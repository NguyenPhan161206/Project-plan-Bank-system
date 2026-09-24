# MCP & Function Calling — Thiết kế Giao diện cho AI Agents

**Ngày tạo:** 2026-09-23
**Cập nhật lần cuối:** 2026-09-23
**Tags:** `#AIAgents`, `#MCP`, `#FunctionCalling`, `#ToolUse`, `#AgentInterfaces`, `#LLM`, `#StructuredOutput`
**Tham chiếu:** [Model Context Protocol (Anthropic)](https://modelcontextprotocol.io/), [OpenAI Function Calling docs](https://platform.openai.com/docs/guides/function-calling), [JSON Schema](https://json-schema.org/), [Ollama Tools](https://docs.ollama.com/guides/function-calling)
**Trạng thái:** Đã xuất bản
**Độ khó:** Trung cấp
**Dự án:** Hệ thống Tài chính Tiêu dùng Modular — giao diện agent có kiểm soát (MCP + CLI `--json`) qua đó AI agents vận hành các workflow cho vay (KYC, quyết định tín dụng, giải ngân, loan servicing).

> **📌 Đánh dấu mục tiêu:** Đây là thiết kế **hướng tới v1** (MCP + AI agents). MVP **trì hoãn toàn bộ** AI/ML, agents và MCP (xem [00-mvp-backlog.md](00-mvp-backlog.md)). Giữ lại làm mục tiêu giai đoạn sau, **không** triển khai trong MVP.

---

## 📌 Tóm tắt Chính (TL;DR)

- Một **giao diện agent** không phải là CLI — nó là một *hợp đồng đọc máy được* (JSON Schema tools, đầu ra có cấu trúc) cho phép một LLM khám phá khả năng lúc runtime và gọi chúng an toàn. CLI chỉ là một *mặt* của giao diện đó (mặt người/script).
- **Function calling (tool use):** LLM xuất một lời gọi có cấu trúc (`tool: name`, `arguments: {...}`) thay vì văn xuôi; app thực thi nó và trả kết quả lại. Vòng lặp là: *prompt → quyết định tool → thực thi → quan sát → quyết định tiếp theo*.
- **MCP (Model Context Protocol)** tiêu chuẩn hoá cách một agent *host* kết nối tới *tool servers*: một giao thức, mọi tool, mọi agent (Claude, opencode, custom). Ba nguyên thuỷ của nó — **tools** (hành động), **resources** (dữ liệu chỉ đọc), **prompts** (bản mẫu) — ánh xạ 1:1 lên một hệ thống tác vụ công ty.
- **Yêu cầu "CLI để AI Agent thao tác" nên được xây thành: giao thức (MCP) + CLI với `--json` + quyền hạn có kiểm soát.** "Agents vận hành tự do" phải được thay bằng "agents vận hành trong các phong bì quyền-tối-thiểu, được kiểm toán".
- An toàn là một phần của kiến trúc, không phải nghĩ thêm: **tool allowlists, role chỉ đọc, cổng phê duyệt cho tool có hậu quả, log kiểm toán bất biến và phòng thủ prompt-injection** là một phần của chính thiết kế giao diện.

---

## 🧠 Ghi chú Chi tiết

### 1. Khái niệm Lõi

**Function calling (tool use)** — cơ chế mà một LLM *gọi hệ thống của bạn*:
1. Bạn khai báo các tool khả dụng dưới dạng JSON Schemas (name + description + parameter schema).
2. Mô hình thấy các mô tả tool trong context; nếu thích hợp, nó trả lời bằng một payload `tool_calls` *có cấu trúc* — **không phải văn bản** — chứa các đối số.
3. App của bạn xác thực các đối số, thực thi hàm thật, gửi kết quả lại cho mô hình.
4. Mô hình tiếp tục (gọi thêm tool hoặc tạo câu trả lời cuối).

Insight chính: **tool schema là API contract với một LLM.** Chất lượng `description` và `parameters` quyết định mô hình có gọi tool đúng hay không. Schema tồi ⇒ mô hình bịa đối số hoặc từ chối dùng tool.

**Structured outputs** — buộc câu trả lời *cuối* của mô hình cũng là JSON (được xác thực theo schema), để hệ thống hạ nguồn tiêu thụ kết quả mà không cần parse văn xuôi. Thiết yếu khi đầu ra LLM nuôi một tác vụ khác trong pipeline của bạn.

**Model Context Protocol (MCP)** — một chuẩn mở (Anthropic, 11/2024) để kết nối agents với tools/dữ liệu. Kiến trúc:

```
┌──────────────┐    MCP    ┌──────────────────┐
│   MCP Host   │ ◀───────▶ │   MCP Server(s)  │
│  (agent app: │  JSON-RPC │  (tools/nguồn    │
│   Claude,    │  stdio /  │   dữ liệu của    │
│   opencode…) │  HTTP/SSE │   bạn)           │
└──────────────┘           └──────────────────┘
   │ (nhiều clients)
   ▼
 MCP Client(s)  ── kết nối tới n servers, mỗi server phơi bày:
        • tools      → hành động (ví dụ: approve_loan_application)
        • resources  → dữ liệu chỉ đọc (ví dụ: ledger:current)
        • prompts    → bản mẫu dùng lại (ví dụ: "audit một chuỗi tác vụ")
```

**Ba nguyên thuỷ MCP vs hệ thống của bạn:**

| Nguyên thuỷ MCP | Nghĩa | Ví dụ System Tài chính Tiêu dùng |
|------------------|-------|-----------------------------------|
| **Tool** | hành động gọi được (biến đổi hay không) | `approve_loan_application(id)`, `disburse_loan(id)`, `get_loan_status(id)`, `generate_report(period)` |
| **Resource** | dữ liệu chỉ đọc với một URI | `loan://{id}/status`, `kyc://{id}/documents`, `ledger://loans/current`, `agent://session/{id}/log` |
| **Prompt** | bản mẫu chỉ dẫn dùng lại | `audit_chain`, `draft_response_to_customer` |

**CLI + `--json`** — mặt người/script của hệ thống bạn: `taskcli loan decision --id APP-42 --verdict approve --json`. CLI là một *client mỏng trên cùng hợp đồng* mà MCP phục vụ. Cùng một engine, ba người tiêu dùng (người, script, agent).

### 2. Nó Hoạt động Thế nào (Cơ chế)

**Vòng lặp tool-call (đơn giản hoá):**

```
User: "Duyệt application 42 sau kiểm tra KYC, rồi kích hoạt giải ngân, và cho tôi biết trạng thái."
  │
  ▼
[Agent] ──tools: [approve_loan_application, disburse_loan, get_loan_status]──▶ [LLM]
  │                                                                        │
  └──────────────  tool_calls: ◀───────────────────────────────────────────┘
      [{name: approve_loan_application, args: {application_id: "42"}}]     (có cấu trúc!)
      ├─▶ [Hệ thống của bạn: xác thực → thực thi → kết quả "ok, approved"]
      │
      ▼  (kết quả được trả lại, vòng lặp lặp lại)
      └── tool_calls: [{name: disburse_loan, args: {...}}]  … v.v.
```

**Luồng yêu cầu MCP (transport stdio):**

1. **Initialize:** bắt tay client ↔ server, đàm phán version, đàm phán khả năng.
2. **List tools:** client hỏi `tools/list` → server trả JSON Schema của mọi tool.
3. **Call:** client gửi `tools/call` kèm tên tool + đối số → server thực thi → trả kết quả có cấu trúc hoặc lỗi.
4. **Notifications/resources:** server có thể đẩy `resources/list_changed`; client lấy resources qua `resources/read`.

**Vì sao MCP đánh bại keo dán theo-nhà-cung-cấp:** thay vì viết N adapter (OpenAI SDK, Anthropic SDK, mô hình cục bộ, khung agent tự chế), bạn viết **một MCP server** cho mỗi bộ khả năng và mọi host MCP-capable đều điều khiển được nó. Đây là lớp tiêu chuẩn hoá mà nền kinh tế-agent đang thiếu.

### 3. Triển khai (Pseudocode)

**Một định nghĩa tool dưới dạng JSON Schema (đây CHÍNH LÀ hợp đồng agent):**

```json
{
  "name": "approve_loan_application",
  "description": "Approve a loan application for disbursement. Requires the application to be in 'decision_pending' state and the caller to hold role 'credit_decider'. Consequential action - requires explicit consent when run by an autonomous agent.",
  "input_schema": {
    "type": "object",
    "properties": {
      "application_id": {"type": "string", "pattern": "^APP-[0-9]+$"},
      "reason": {"type": "string", "minLength": 10}
    },
    "required": ["application_id", "reason"],
    "additionalProperties": false
  }
}
```

**Một MCP server tối thiểu (pseudocode kiểu FastMCP):**

```python
from mcp.server.fastmcp import FastMCP   # khái niệm; SDK tương đương

mcp = FastMCP("consumer-finance")

@mcp.tool()
def approve_loan_application(application_id: str, reason: str) -> str:
    """Approve a loan application for disbursement (role-checked, audit-logged)."""
    if not current_agent_has_role("credit_decider"):
        return {"error": "FORBIDDEN", "audit_id": audit.begin(application_id, "denied")}
    result = saga_coordinator.advance("credit.decision.approved", application_id, by=current_agent())
    audit.log("approve_loan_application", application_id, by=current_agent(), result=result)
    return result

@mcp.resource("loan://{id}/status")
def loan_status(id: str) -> dict:
    """Read-only snapshot of a loan's lifecycle (never mutates)."""
    return loan_servicing.snapshot(id)

# Transport: stdio (local) hoặc streamable HTTP — cùng server, tuỳ cách.
```

**Lớp bọc vòng lặp agent có kiểm soát (an toàn như kiến trúc):**

```python
async def run_with_guardrails(goal: str, allowed_tools: set[str], require_human: set[str]):
    for step in agent_loop(goal, tools=allowed_tools):        # 1. tool allowlist
        if step.tool in require_human:                        # 2. cổng phê duyệt
            await human_approve(step)                          #    (tools có hậu quả)
        if not audit.within_budget(step):                     # 3. ngân sách chi phí/độ trễ
            abort("budget exceeded")
        outcome = await execute(step)                          # 4. thực thi
        await audit.record(step, outcome)                      # 5. vết bất biến
```

**Mặt CLI trên cùng hợp đồng:**

```bash
$ taskcli loan decision --id APP-42 --verdict approve --reason "Bureau check passed, DTI within limit" --json
{"ok": true, "state": "decision_made", "next": ["contract.generation.requested"], "audit_id": "a-9f3"}
```

Cùng engine. Agent chạm nó qua MCP; người chạm nó qua CLI; script chạm nó qua `--json`.

### 4. Ứng dụng Thực tế — cho Company Task System

**Biến "CLI để AI Agent thao tác tự do" thành một thiết kế an toàn:**

1. **Mọi khả năng có một tool schema** (MCP) — khả năng trở nên *khám phá được*, điều làm cho "vận hành tự do" trở nên *có năng lực*, không chỉ hợp lệ.
2. **Roles ánh xạ sang bộ tool:** `read_only_agent` → chỉ resources; `operator_agent` → tools không có hậu quả; `approver_agent` → + tools phê duyệt yêu cầu đồng thuận người. Quyền tối thiểu, như các role DB bạn đã biết.
3. **Các tool có hậu quả gọi cổng con người:** phê duyệt tín dụng, giải ngân, xoá → không bao giờ được agent tự trị thực thi nếu không có sự đồng ý của người (cấu hình theo chính sách).
4. **Mọi thứ đều được kiểm toán:** mỗi lời gọi tool được lưu với định danh agent, đối số, kết quả, timestamp → hành trình kiểm toán *chính là* bằng chứng bàn trực nghiệp vụ và cơ quan quản lý cần.
5. **Phòng thủ prompt-injection:** tách nội dung không tin cậy (tài liệu, email đang *được xử lý*) khỏi chỉ thị: không bao giờ để văn bản tài liệu trở thành chỉ thị hệ thống; nạp tài liệu như *resources/dữ liệu*, giữ chỉ thị trong system prompt; allowlist tools theo tác vụ; ưu tiên trích xuất có cấu trúc hơn Q&A tự do trên nội dung không tin cậy.

**Ngăn xếp giao diện (kết cấu đã sửa từ ý tưởng gốc):**

```
Core engine (bounded contexts, events, saga)
        │  lớp hợp đồng: JSON Schema tools + MCP server + REST + CLI --json
        ├─── CLI        (power-user người, scripts)
        ├─── Web UI     (người dùng nghiệp vụ — client mỏng, không nằm trong ghi chú này)
        └─── MCP server (agents: Claude, opencode, custom…)
```

### 5. Bảng So sánh

| Chiều | Function Calling (raw SDK) | MCP | Plain REST/OpenAPI | CLI không có --json |
|--------|----------------------------|-----|--------------------|----------------------|
| Khả năng khám phá agent | ✅ (schemas trong prompt) | ✅ (`tools/list`) | ⚠️ (cần agent framework) | ❌ |
| Tái dùng đa agent | ❌ keo dán theo-vendor | ✅ một server, mọi host | ⚠️ OpenAPI tooling tồn tại | ❌ |
| Kiểm soát đầu ra có cấu trúc | ✅ | ✅ | ✅ | ❌ (văn xuôi) |
| Truy cập resource/dữ liệu cho agent | ❌ (tuỳ chỉnh) | ✅ resources bản địa | ✅ nhưng bespoke | ❌ |
| Khả dụng với người | ❌ | ❌ (hướng dev) | ⚠️ qua Swagger UI | ✅ |
| Độ phức tạp | Thấp | Trung bình | Trung bình | Thấp |
| **Khớp với ý tưởng của bạn** | Starter 🔧 | **Hợp đồng lõi** ✅ | Đồng hành (REST client reg) | Mặt của CLI ✅ |

Người thắng cho hệ thống công ty: **MCP làm hợp đồng + CLI với `--json` làm mặt** — vì bạn cần cả khả năng khám phá agent LẪN khả năng dùng của người LẪN khả năng kiểm toán từ cùng một engine.

---

## 📝 Hành trình Nghiên cứu

- **Tại sao:** Câu nói táo bạo nhất của ý tưởng gốc là "hệ thống CLI để thao tác tự do bằng AI Agent" — tôi cần biết liệu đó có phải một mẫu kỹ thuật thật hay chỉ là buzzword. Đáp án: nó thật, nhưng *hợp đồng* (không phải CLI) mới là lõi kỹ thuật.
- **Vật lộn:** Ban đầu tôi nghĩ agents "dùng" hệ thống như con người (gõ trong terminal). LLM *không gõ* — nó phát ra các lời gọi JSON có cấu trúc. Nhận ra CLI chỉ là một *biểu diễn* của hợp đồng cho con người đã thay đổi toàn bộ thiết kế của tôi.
- **Khoảnh khắc à-ha:** (1) **Tool schema = prompt engineering**: field `description` CHÍNH LÀ tài liệu của mô hình — sự chăm chút bạn dành cho `README.md` phải vào trong `description`. (2) Ba nguyên thuỷ của MCP ánh xạ 1:1 tới một hệ thống nghiệp vụ (tools/resources/prompts = hành động/dữ liệu/bản mẫu). (3) **Prompt injection là rủi ro #1 của hệ thống agent**, và nó là vấn đề *thiết kế giao diện*: nội dung không bao giờ được trở thành chỉ thị.
- **Liên kết sự nghiệp:** Mọi công ty nghiêm túc xây trên AI sẽ cần hệ thống agent-operable với quản trị. Là một trong các kỹ sư *thiết kế được lớp giao diện* (MCP + tools + audit) là một kỹ năng trực tiếp, tạo khác biệt cho vai AI Engineer trong Kinh tế/Kinh doanh — không chỉ "gọi một API".

## 🔗 Chủ đề Liên quan

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — ý tưởng dự án (Insight 6: CLI là một mặt; giao thức là hợp đồng; quản trị).
- [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md) — engine đằng sau các tools (saga, events, audit) mà agents điều khiển.
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — các bounded contexts định nghĩa tool nào tồn tại và ai được gọi chúng.
- *(bên ngoài, Nhật ký Nghiên cứu)* `ai_ml/05-mlops-lifecycle-and-deployment-architecture.md` — cổng eval và giám sát cho phía *mô hình* của vòng lặp (trôi dạt, lỗi).
- *(bên ngoài, Nhật ký Nghiên cứu)* `random_ideas/04-custom-tui-project.md` — dự án TUI/CLI cho AGY & opencode — phía tương tác-con-người của cùng câu chuyện giao diện này.
- [ĐỀ XUẤT] **"An toàn agentic & phòng thủ prompt-injection theo chiều sâu"** — một lớp bảo mật trọn vẹn (tách nội dung/chỉ thị, allowlist tools, sandboxing, red-teaming) mà các ghi chú chính thống bỏ qua. *Tại sao quan trọng:* đây là khác biệt giữa một demo và một hệ thống một bên cho vay tin tưởng giao tiền — kiểm toán đạt chuẩn quy định mới đáng tin.
- [ĐỀ XUẤT] **"Đánh giá workflow agent (eval harness cho LLM dùng tool)"** — cách đo liệu agent của bạn có thực sự hoàn tất chuỗi tác vụ đúng không (pass@k trên trace tác vụ thật). *Tại sao quan trọng:* tư duy eval của MLOps áp cho agent trước khi chạm dữ liệu cho vay thật.

## 🤔 Câu hỏi Mở

- [ ] MCP stdio vs streamable HTTP transport cho triển khai doanh nghiệp đa-agent — hàm ý bảo mật?
- [ ] Tool schema nên mô tả *ràng buộc nghiệp vụ* (roles, states) thế nào để mô hình hiếm khi thử gọi bị cấm?
- [ ] Định dạng audit nào thoả mãn cả bàn trực nghiệp vụ lẫn cơ quan quản lý tiềm năng (log bất biến, hash-chained)?
- [ ] Cách đánh giá chất lượng hoàn-thành-tác-vụ của agent TRƯỚC khi nó chạm dữ liệu thật (eval harness, shadow mode)?

---
*CLI là mặt người của một hợp đồng; schema là mặt của agent. Thiết kế hợp đồng một lần, đeo nó cả hai chiều.*