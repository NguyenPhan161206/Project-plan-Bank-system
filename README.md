# Kế hoạch Dự án — Hệ thống Vận hành Tài chính Tiêu dùng (Modular)

> Một **nền tảng vận hành tài chính tiêu dùng** theo lĩnh vực cụ thể — modular, liên kết bằng sự kiện, có khả năng tiến hoá, và được vận hành bởi **AI Agents** thông qua giao diện CLI/MCP có kiểm soát.

Kho lưu trữ này là *ngôi nhà dự án* để biến ý tưởng **Modular Company Task System** (từ [Nhật ký Nghiên cứu](https://github.com/NguyenPhan161206/reaserching-diary/blob/mine/random_ideas/06-modular-company-task-system-ai-agents.md)) thành một hệ thống cụ thể. Lĩnh vực neo đậu là **vận hành tài chính tiêu dùng**, lấy chuẩn là **Home Credit Việt Nam** — một công ty tài chính được cấp phép và chịu sự giám sát của Ngân hàng Nhà nước Việt Nam — nơi mà "công ty có chuyên môn cụ thể" trong ý tưởng gốc là một *bên cho vay*, không phải ngân hàng nhận tiền gửi.

---

## 🎯 Tầm nhìn

Xây dựng một hệ thống lớn chuyên biệt cho **các tác vụ vận hành cốt lõi của một công ty tài chính** (khởi tạo khoản vay, quyết định tín dụng, giải ngân, quản lý vòng đời khoản vay, sổ sách, báo cáo). Các tác vụ liên kết với nhau qua một **back-end hướng sự kiện vững chắc**, trong khi mọi thành phần vẫn **độc lập cao** — thích ứng với biến đổi của quy định, sản phẩm và thị trường, dễ sửa đổi và xây dựng để tồn tại lâu dài. Hệ thống **tích hợp AI** và phơi bày mọi khả năng thông qua **CLI + giao thức MCP** để AI Agents có thể vận hành các quy trình — *trong phạm vi quyền hạn được kiểm soát và ghi nhật ký kiểm toán*.

## 📊 Mục lục Tài liệu

| Doc | Chủ đề | Trả lời |
|-----|-------|---------|
| [01-project-idea-and-evaluation.md](docs/01-project-idea-and-evaluation.md) | Ghi nhận ý tưởng + đánh giá phản biện | Tại sao nên xây? Điểm mạnh, điểm yếu, rủi ro phải nhận |
| [02-event-driven-architecture-and-task-orchestration.md](docs/02-event-driven-architecture-and-task-orchestration.md) | Nền tảng hướng sự kiện + điều phối saga | Các tác vụ *liên kết* đáng tin cậy thế nào? |
| [03-modular-monolith-vs-microservices.md](docs/03-modular-monolith-vs-microservices.md) | Khung quyết định modularity + fitness functions | Các thành phần *độc lập* bằng cách nào? |
| [04-mcp-and-function-calling-agent-interfaces.md](docs/04-mcp-and-function-calling-agent-interfaces.md) | MCP + function calling + agent có kiểm soát | AI Agents *vận hành* hệ thống thế nào? |

## 🏗️ Tổng quan Kiến trúc (mục tiêu v1)

```
                             ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
 Client mỏng            →   │  CLI (người/ │  │  Web UI      │  │ MCP server   │
                             │  agent)      │  │ (bàn trực    │  │ (agents)     │
                             │              │  │  nghiệp vụ)  │  │              │
                             └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                                    └─────────────────┼─────────────────┘
                                                      ▼
                             ┌──────────────────────────────────────────┐
                             │ Lớp giao thức/hợp đồng (JSON Schema,     │
                             │ auth, versioning, nhật ký kiểm toán bất  │
                             │ biến)                                    │
                             └────────────────┬─────────────────────────┘
                                               ▼
   Các bounded context độc lập (modular monolith, dữ liệu riêng mỗi context)
   [Identity/Access Control] ── phân quyền theo role ──▶ mọi context
   [KYC/Onboarding] ──customer.verified──▶ [Application] ──application.submitted──▶
   [Credit Decisioning] ──approved/rejected/referred──▶ [Contracting] ──contract.signed──▶
   [Disbursement] ──disbursement.completed──▶ [Loan Servicing] ──repayment.received──▶
   [Ledger (hạch toán khoản vay, dự phòng IFRS 9)] ◀── sự kiện ──▶ [Reporting]
                                               ▼
                              ┌──────────────────────────────────────────┐
                              │ Xuyên suốt: observability, dịch vụ AI,   │
                              │ schema registry                          │
                              └──────────────────────────────────────────┘
```

- **Backbone:** hướng sự kiện với một **orchestrated saga** cho mỗi quy trình nghiệp vụ (ví dụ: `application.submitted → credit.decision.approved → contract.signed → disbursement.completed → repayment.received → ledger.posted`), phân phối ít-nhất-một-lần (at-least-once) + consumer idempotent + DLQ.
- **Modularity:** bắt đầu bằng **modular monolith** với ranh giới được ràng buộc (import-linter, contract tests); chỉ tách context thành service khi fitness functions/bằng chứng yêu cầu (strangler pattern).
- **Agents:** mọi khả năng = một MCP **tool** (ví dụ: `get_loan_status`, `approve_loan_application`), CLI `--json` là mặt dành cho người/script; role ánh xạ sang bộ tool; các tool có hậu quả lớn yêu cầu phê duyệt của con người; mọi lời gọi đều được ghi kiểm toán.

## 🗺️ Lộ trình

- **v0 — Vay tiền mặt online, end-to-end:** các bounded context `identity`/`access_control`, `onboarding/kyc`, `application`, `credit_decisioning`, `contracting`, `disbursement`, `ledger`; orchestrated saga (KYC → quyết định tự động → hợp đồng → giải ngân → trả nợ đầu tiên → hạch toán); CLI + `--json`; nhật ký kiểm toán; admin quản trị user + tra cứu audit. Mục tiêu: chứng minh ý tưởng + backbone trên một luồng tiền di chuyển với cổng quyết định tự động.
- **v1 — Vòng đời khoản vay + mở rộng LOB:** thêm các context `loan_servicing`, `collections`, `reporting` tiêu thụ sự kiện; kỷ luật schema registry + versioning; adapter e-wallet/merchant (POS trả góp); các cổng xác thực human-in-the-loop cho thành phần AI.
- **v2 — Quản trị Agent + vận hành AI:** MCP server phơi bày toàn bộ bộ khả năng; phân quyền tool theo role; eval harness & shadow mode trước khi thực thi tự trị; phòng thủ prompt-injection theo chiều sâu (kiểm toán đạt chuẩn quy định).

## ⚠️ Rủi ro hàng đầu cần nhận thức (xem doc 01 để có bảng đầy đủ)

1. **Tuân thủ & trách nhiệm pháp lý** — AI tác động lên các quyết định cho vay hệ trọng (phê duyệt, giải ngân); xem doc 04 về thiết kế quản trị.
2. **Prompt injection** — agent tiếp xúc nội dung bên thứ ba (hồ sơ khách hàng, tài liệu đối tác/merchant, payload e-wallet).
3. **Khuếch đại GIGO** — các tác vụ liên kết tự động truyền dữ liệu sai; cần provenance + cổng xác thực.
4. **Bus-factor = 1** — hệ thống xây solo, dài hạn: cần ADR + phạm vi nhỏ trung thực.
5. **Kinh tế đơn vị** — chi phí AI mỗi thao tác phải được mô hình hoá so với biên lợi nhuận mỗi khoản vay (cho vay nhỏ, SLA duyệt tự động).

## 📚 Nguồn gốc Sự thật

Các ghi chú thiết kế được duy trì trong **[Nhật ký Nghiên cứu](https://github.com/NguyenPhan161206/reaserching-diary)** — repo này phản chiếu các tài liệu liên quan đến dự án. Mọi thay đổi ở ghi chú gốc đều được phản ánh tại đây.

## 🔗 Liên kết Liên quan

- Nhật ký Nghiên cứu: <https://github.com/NguyenPhan161206/reaserching-diary>
- Chủ repo: Phan Hữu Bình Nguyên (hướng AI Engineer, Kinh tế/Kinh doanh)

---
*Chất lượng của một ý tưởng được đo bằng những mâu thuẫn mà nó giải quyết được, không phải bằng danh sách tính năng của nó.*