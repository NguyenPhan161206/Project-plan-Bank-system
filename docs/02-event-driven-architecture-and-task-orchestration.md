# Kiến trúc Hướng Sự kiện & Điều phối Tác vụ

**Ngày tạo:** 2026-09-23
**Cập nhật lần cuối:** 2026-09-23
**Tags:** `#EventDrivenArchitecture`, `#MessageBroker`, `#Saga`, `#TaskOrchestration`, `#SystemDesign`, `#Backend`
**Tham chiếu:** [Saga pattern — Microservices.io](https://microservices.io/patterns/data/saga.html), [Kafka design docs](https://kafka.apache.org/documentation/), [Event-driven architecture — AWS](https://aws.amazon.com/event-driven-architecture/)
**Trạng thái:** Đã xuất bản
**Độ khó:** Trung cấp
**Dự án:** Hệ thống Tài chính Tiêu dùng Modular — thiết kế "back-end vững chắc" cho các chuỗi tác vụ cho vay (origination → KYC → quyết định tín dụng → hợp đồng → giải ngân → servicing/trả nợ → sổ sách → báo cáo).

---

## 📌 Tóm tắt Chính (TL;DR)

- **Events là sự thật của quá khứ; commands là yêu cầu cho tương lai.** "Task B phải phản ứng với điều Task A đã làm" ⇒ phát một event; "Task A làm X giúp tôi" ⇒ gửi một command. Trộn lẫn chúng là nguồn #1 của hệ thống bị khớp nối.
- Một **message broker** (Kafka/RabbitMQ/Redis Streams) là "back-end vững chắc" của Hệ thống Tác vụ Modular cho Công ty: nó cho độ bền, retries và khả năng kiểm toán — nhưng nó chỉ *giao hàng*, không bao giờ đảm bảo consumer của bạn đúng.
- Ba đảm bảo làm hệ thống liên kết thực sự vững chắc: **phân phối at-least-once + consumer idempotent + dead-letter queues**. Thứ tự là một phần thưởng tốn throughput — chỉ mua nó nơi lĩnh vực cần.
- Các tác vụ trải dài nhiều thành phần phải được bọc trong một **saga** (workflow phân tán có bù trừ). Bắt đầu bằng **orchestration** (một bộ điều phối trung tâm) để rõ nghiệp vụ; chỉ dùng choreography khi đội ngũ chín chắn và workflow ổn định.
- **Mọi thay đổi schema là một thay đổi hợp đồng.** Schema registry + SemVer + cửa sổ deprecation là thứ giữ hệ thống hướng sự kiện *tiến hoá được* — chính là yêu cầu "dễ sửa đổi lâu dài" của ý tưởng gốc.

---

## 🧠 Ghi chú Chi tiết

### 1. Khái niệm Lõi

**Event vs Command — sự phân biệt nền tảng.**

| | Event | Command |
|--|-------|---------|
| Ngữ nghĩa | *Một điều gì đó đã xảy ra* (thì quá khứ, sự thật) | *Hãy làm điều này* (yêu cầu, ý định tương lai) |
| Nhà sản xuất kỳ vọng | Không có trả lời (fire-and-forget) | Một kết quả / lời xác nhận |
| Ngôn ngữ | `InvoiceApproved`, `PaymentCaptured` | `ApproveInvoice`, `CapturePayment` |
| Thất bại | Đã xảy ra — không thể thất bại | Có thể bị từ chối/retry |
| Vai trò broker | Nhà phân phối (thông báo subscriber) | Bộ định tuyến (giao cho *một* worker) |

**Vì sao điều này quan trọng cho ý tưởng của bạn:** "Các tác vụ liên kết với nhau" gần như luôn là mối quan hệ *event* — Task A hoàn tất, vậy nên Task B bắt đầu. Nếu bạn mô hình hoá sự liên kết đó thành chuỗi command đồng bộ (`A → gọi B → gọi C`), bạn xây một monolith phân tán: một tác vụ chậm/thất bại chặn toàn chuỗi. Events cho bạn **tách rời thời gian** (temporal decoupling — B không đợi A) và **tách rời không gian** (spatial decoupling — B không cần biết địa chỉ của A).

**Message broker** — người trung gian bền vững:
- **Kafka** — log append-only, topic phân vùng, replay được, throughput khổng lồ; tuyệt cho event streams + audit.
- **RabbitMQ** — queue kinh điển + AMQP exchanges, routing, xác nhận từng message; tuyệt cho commands/work queues.
- **Redis Streams** — nhẹ, gần-in-memory, tốt cho hệ thống nhỏ/vừa với chi phí vận hành thấp.

**Pub/Sub** — publisher không biết subscriber (mỗi subscriber nhận một bản sao riêng của event) vs **queue** — một consumer nhận mỗi message.

**Event Sourcing** — lưu *chuỗi sự kiện* làm nguồn sự thật, suy ra trạng thái hiện tại bằng cách replay (projection). Khả năng kiểm toán mạnh; độ phức tạp cao hơn. Tuỳ chọn cho v1 — chỉ thêm khi bạn cần event log làm chân lý (lĩnh vực tài chính/nặng kiểm toán).

**CQRS (Command Query Responsibility Segregation)** — tách mô hình ghi khỏi mô hình đọc; thường đi cặp event sourcing; hữu ích cho dashboard nặng đọc. Lại là tuỳ chọn.

### 2. Nó Hoạt động Thế nào (Cơ chế)

Một pipeline đáng tin cậy tối thiểu:

```
[Producer] --publish--> [Broker: topic "repayment.received"] --> [Consumer Group: ledger-service]
                                                              │
                                                              └─ kiểm tra idempotency → xử lý → commit offset
```

**Ngăn xếp đảm bảo:** broker cho **at-least-once** (giao hàng được retry cho tới khi xác nhận). Nghĩa là consumer có thể thấy cùng một event hai lần. Vì vậy: **consumer PHẢI idempotent** — áp event hai lần cùng hiệu ứng như áp một lần (dùng một key event/nghiệp vụ duy nhất lưu tại consumer; bỏ qua nếu đã thấy).

**Thứ tự:** Kafka đảm bảo thứ tự *trong một partition* — khai báo theo thực thể (ví dụ: `application_id`), nên mọi event của một khoản vay đổ vào một partition, đúng thứ tự. Thứ tự toàn cục xuyên thực thể về cơ bản là bất khả; lĩnh vực gần như không bao giờ cần.

**Dead Letter Queue (DLQ):** sau N lần thất bại (poison messages), park event vào một DLQ để sửa thủ công/ngoài giờ thay vì chặn pipeline mãi mãi.

**Exponential backoff với jitter** (nhịp retry):

$$t_n = \text{base} \cdot 2^n + \text{rand}(0, \text{jitter})$$

Ngăn "thundering herd" nơi mọi consumer cùng retry đồng loạt sau một trục trặc broker.

**Schema registry & tiến hoá:**
- Mọi event mang `schema_version`.
- **Thay đổi cộng dồn** (thêm field tuỳ chọn mới) = tương thích ngược → an toàn trên cùng version.
- **Thay đổi phá vỡ** (field bị gỡ/đổi kiểu) = major version mới + **cửa sổ dual-write** (sản xuất cả hai version trong N tuần) trong khi consumer di cư.

Đây là cơ sở *cơ học* của "thích ứng với thời cuộc / dễ dàng sửa đổi" — thứ giữ một hệ thống 5 năm tuổi an toàn để tiến hoá.

### 3. Điều phối Tác vụ — Choreography vs Orchestration vs Saga

Ba cách liên kết tác vụ trong một workflow:

**A. Choreography (hướng sự kiện, không có bộ điều phối trung tâm):**
```
LoanServicing → phát "repayment.received" → LedgerSvc phản ứng → phát "ledger.posted" → ReportSvc phản ứng
```
- ✅ Tách rời tối đa, không có điểm lỗi đơn, mỗi service sở hữu logic của mình.
- ❌ Toàn bộ quy trình nghiệp vụ là *ngầm định* — không ai trả lời được "chuyện gì xảy ra nếu repayment.received không bao giờ tới?" mà không đọc mọi consumer. Debug và giám sát nghiệp vụ khó.
- ⚠️ Nó là một **giao dịch phân tán không có rollback**: nếu Ledger thất bại sau khi Payment đã thành công, ai sửa sự không nhất quán?

**B. Orchestration (workflow engine/bộ điều phối trung tâm):**
```
Orchestrator: 1. gọi KYCSvc (command) → 2. nếu ok, gọi CreditDecisionSvc → 3. nếu thất bại, gọi Compensation
```
- ✅ Quy trình nghiệp vụ tường minh (dễ đọc, giám sát và version); phục hồi là một state machine hạng nhất.
- ❌ Orchestrator trở thành điểm khớp nối (một "god service" nếu lạm dụng); mỗi bước là một round-trip.
- Điều này khớp bản năng của ý tưởng bạn: *"các tác vụ có khả năng liên kết với nhau"* với **tính nhìn thấy** — một bàn trực nghiệp vụ cần *thấy* chuỗi tác vụ đang kẹt ở đâu.

**C. Saga = cách đúng để làm các quy trình nghiệp vụ dài, đa thành phần.**

Saga là một chuỗi các giao dịch cục bộ, mỗi cái có một **bù trừ** (undo):

```
ApplyLoan ──▶ KYCVerify ──▶ CreditDecision ──▶ Disburse ──▶ ActivateSchedule
                │                 │              │
              (fail)           (fail)          (fail)
                ▼                 ▼              ▼
         CloseApplication   RejectCase    CancelDisbursement
```

Saga có hai hoá thân:
- **Orchestrating saga** (bộ điều phối trung tâm quyết định mỗi bước + bù trừ) — khởi đầu được khuyến nghị.
- **Choreographed saga** (mỗi service phát events kích hoạt bước tiếp; bù trừ cũng qua events) — cho các đội chín chắn, tách rời.

**Cái nào cho Company Task System?** Orchestrated saga. Lý do: workflow nghiệp vụ cần *observability, versioning và pause/resume* (một chuỗi tác vụ có thể chờ nhiều ngày để phê duyệt con người). Một state machine trung tâm cho cả ba miễn phí; bạn vẫn có thể *phát events từ mỗi bước* để audit và analytics (events + orchestrator là bổ sung, không cạnh tranh).

### 4. Triển khai (Pseudocode)

**Một orchestrating saga tối thiểu cho chuỗi vay tiền mặt online (v0):**

```python
# saga_coordinator.py — orchestrating saga
# Workflow: ApplicationSubmitted → KYCVerified → CreditDecision(auto/human) → ContractSigned → Disbursed → RepaymentReceived

class CashLoanSagaCoordinator:
    STATES = ("submitted", "kyc_verified", "decision_made", "signed", "disbursed", "repaying", "compensated")

    def __init__(self, bus, store):
        self.bus = bus        # event bus / broker
        self.store = store    # saga-state store (DB with application_id as key)

    async def start(self, application_id: str):
        saga = self.store.create(application_id)
        await self.bus.publish("application.submitted", {"application_id": application_id,
                                                         "schema_version": 1})

    async def on_event(self, event: dict):
        key = event["application_id"]
        saga = self.store.get(key)
        if not self._is_expected(event, saga.state):
            return  # event cũ/nghịch thứ tự → bỏ qua (idempotency)

        if event["type"] == "kyc.verified" and event["ok"]:
            saga.state = "kyc_verified"
            await self.bus.publish("credit.decision.requested", {"application_id": key})
        elif event["type"] == "credit.decision.approved":
            saga.state = "decision_made"
            await self.bus.publish("contract.generation.requested", {"application_id": key})
        elif event["type"] == "contract.signed":
            saga.state = "signed"
            await self.bus.publish("disbursement.requested", {"application_id": key})
        elif event["type"] == "disbursement.completed":
            saga.state = "disbursed"
            await self.bus.publish("repayment.schedule.activated", {"application_id": key})
        elif event["type"] == "repayment.received":
            saga.state = "repaying"
            self.store.complete(key)   # v0: khoản trả nợ đầu tiên đóng đường chính
        elif event["type"] in ("kyc.failed", "credit.decision.rejected", "contract.expired"):
            await self._compensate(saga)  # ví dụ: đóng application + thông báo kênh

    async def _compensate(self, saga):
        saga.state = "compensated"
        await self.bus.publish("application.compensated", {"application_id": saga.id})
        # bù trừ cho từng bước đã commit nằm NGAY ĐÂY,
        # như một chuỗi ngược các hành động bù trừ (nhả hold tín dụng, huỷ giải ngân).
```

**Phía consumer idempotent:**

```python
# disbursement_consumer.py — tiêu thụ "disbursement.requested"
# Di chuyển tiền là bước rủi ro nhất: tái-áp nó phải là bất khả.
async def handle_disburse(event, db, gateway):
    key = event["disbursement_id"]
    if await db.dedupe_exists(key):        # 1) rào idempotency
        return
    try:
        await gateway.payout(event)        # 2) gọi e-wallet/chuyển khoản ngân hàng đúng một lần
        await db.post_disbursement(event)
    except RetryableError:
        await bus.retry_later(event, backoff=exponential_with_jitter(base=1_000, jitter=500))
    except PoisonError:
        await bus.dead_letter(event)       # 3) park vào DLQ, cảnh báo con người — không bao giờ auto-retry mãi
    await db.mark_dedupe(key)
```

### 5. Ứng dụng Thực tế — cho Company Task System

**Định nghĩa "back-end vững chắc" theo vận hành.** Với ý tưởng của bạn, "vững chắc" nên có nghĩa là những *thuộc tính kiểm chứng được* này:

| Thuộc tính | Triển khai | Cách bạn kiểm chứng |
|------------|------------|---------------------|
| Độ bền | Broker lưu trữ events (Kafka retention/compaction) | Test: khởi động lại broker, events vẫn còn |
| Không mất mát ngầm | At-least-once + consumer offsets | Test: giết consumer giữa batch → không mất event |
| *Hiệu ứng* exactly-once | Idempotency keys + dedupe store | Test: replay cùng event → cùng trạng thái |
| Phục hồi workflow | Orchestrated saga state machine | Test: crash coordinator → tiếp tục từ trạng thái cuối |
| Không chặn vô hạn | DLQ + cảnh báo | Test: poison message đáp vào DLQ, cảnh báo kích hoạt |
| Hợp đồng tiến hoá được | Schema registry + SemVer + dual-write | Test: event v2 được consumer v1 tiêu thụ trong cửa sổ |

**Ví dụ task graph cho neo đậu v0 (vay tiền mặt online)** — chuỗi theo kiểu HomeCredit:

```
[KYC/Onboarding] ──kyc.verified──▶ [Application] ──credit.decision.approved (cổng auto hoặc người)──▶
[Contracting] ──contract.signed──▶ [Disbursement] ──disbursement.completed──▶
[Loan Servicing] ──repayment.received──▶ [Ledger] ──ledger.posted──▶ [Reporting]
```

Mỗi mắt xích là một **event**; mọi thành phần vẫn độc lập (tự triển khai/tiến hoá); **orchestrated saga** làm cho sức khoẻ của chuỗi hiện rõ trên bàn trực nghiệp vụ; **event log** kiêm nhiệm nhật ký kiểm toán mà một công ty do agent vận hành cần.

### 6. Bảng So sánh

| Chiều | Choreography | Orchestration (saga) | Chuỗi RPC đồng bộ |
|--------|--------------|----------------------|-------------------|
| Coupling | Thấp nhất | Trung bình (một coordinator) | Cao nhất |
| Tính nhìn thấy quy trình | Kém (ngầm định) | Xuất sắc (tường minh) | Tốt (nhưng một request) |
| Xử lý thất bại | Bù trừ ad-hoc | Bù trừ có cấu trúc | Chỉ timeout/retries |
| Phục hồi / pause-resume | Thủ công | Tự nhiên (state machine) | Thủ công |
| Trần mở rộng | Xuất sắc | Tốt (coordinator phải scale) | Kém (độ sâu request) |
| Tốt nhất cho | Luồng ổn định, đội chín chắn | Nghiệp vụ then chốt, đa bước, cổng người | Chuỗi 2-hop tầm thường |
| **Khớp với ý tưởng của bạn** | ⚠️ Sau khi chín chắn | ✅ **Khởi đầu khuyến nghị** | ❌ Bẫy monolith phân tán |

---

## 📝 Hành trình Nghiên cứu

- **Tại sao:** Ý tưởng gốc nói các tác vụ phải "liên kết với nhau thông qua hệ thống back-end vững chắc" — ghi chú này tồn tại để trả lời *"vững chắc" cơ học nghĩa là gì* và làm sao các thành phần độc lập vẫn tạo thành các quy trình nghiệp vụ mạch lạc.
- **Vật lộn:** Tôi cứ trộn events và commands, và giả định "tách rời hơn = tốt hơn". Thực tế: choreography tách rời *code* nhưng *che giấu quy trình nghiệp vụ* — với một hệ thống công ty, quy trình ẩn là không thể chấp nhận.
- **Khoảnh khắc à-ha:** (1) Idempotency không phải thứ tốt-có-thì-sao — "at-least-once" *bắt buộc* nó; (2) orchestrated saga không vi phạm events — bạn có thể (và nên) có cả hai: orchestrator cho luồng điều khiển, events cho audit và analytics; (3) kỷ luật tiến hoá schema chính là cách "dễ sửa đổi lâu dài" trở thành sự thật trong một hệ thống phân tán.
- **Liên kết sự nghiệp:** Hệ thống hướng sự kiện là backbone của mọi nền tảng fintech/logistics/nghiệp vụ nghiêm túc — hệ thống Kinh tế/Kinh doanh tôi muốn kỹ sư hoá. Thông thạo ở đây là khác biệt giữa "lớp tích hợp" và "monolith phân tán".

## 🔗 Chủ đề Liên quan

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — ý tưởng dự án ghi chú này triển khai (Insight 2: "back-end vững chắc" *chính là* ý tưởng).
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — cách giữ các thành phần độc lập *trong khi* chúng trao đổi events (kỷ luật hợp đồng + ranh giới).
- [04-mcp-and-function-calling-agent-interfaces.md](04-mcp-and-function-calling-agent-interfaces.md) — cách agents tiêu thụ các khả năng tác vụ này (lớp giao thức phía trên cùng backend).
- *(bên ngoài, Nhật ký Nghiên cứu)* `ai_ml/05-mlops-lifecycle-and-deployment-architecture.md` — khả năng vận hành dài hạn của các phần AI bên trong chuỗi tác vụ.
- *(bên ngoài, Nhật ký Nghiên cứu)* `git_github/11-infrastructure-as-code-and-devops-automation.md` — brokers và sagas phải được triển khai tái lập được (IaC).
- [ĐỀ XUẤT] **Ghi chú cầu nối: "Từ tri thức quy trình/vòng đời khoản vay đến thiết kế saga"** — workflow của một bên cho vay (quyết định tín dụng, KYC, giải ngân) đã tồn tại như các quy trình; khai thác log thật (hoặc hồ sơ sản phẩm/SLA công khai của một chuẩn như HomeCredit) để suy ra các bước saga sẽ neo hệ thống vào thực tế. *Tại sao quan trọng:* khám phá các quy trình — đừng bịa ra chúng.

## 🤔 Câu hỏi Mở

- [ ] Kafka vs RabbitMQ vs Redis Streams cho *backbone* tích hợp công ty đầu tiên — chi phí vận hành ở quy mô đơn công ty là gì?
- [ ] Event sourcing cho ledger khoản vay: giá trị kiểm toán có đáng độ phức tạp trong v1 không?
- [ ] Cổng quyết định tín dụng mô hình hoá một *tạm dừng nhiều ngày* (referral sang duyệt thủ công) bên trong saga thế nào — chính sách timeout, nhắc nhở, escalation?
- [ ] Quy trình cảnh báo + sửa chữa DLQ thế nào khi operator là một agent?
- [ ] Đối soát giải ngân & trả nợ với adapter e-wallet/ngân hàng — một dòng tiền đi có được xác nhận và đối soát thế nào (*hiệu ứng* exactly-once, không chỉ giao hàng)?

---
*Events kết nối những gì phải giữ kết nối, mà không sở hữu những gì phải giữ độc lập.*