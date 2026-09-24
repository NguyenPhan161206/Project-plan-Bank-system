# Modular Monolith vs Microservices — Một Khung Quyết định

**Ngày tạo:** 2026-09-23
**Cập nhật lần cuối:** 2026-09-23
**Tags:** `#SystemArchitecture`, `#ModularMonolith`, `#Microservices`, `#DomainDrivenDesign`, `#FitnessFunctions`, `#Independence`
**Tham chiếu:** [Martin Fowler — MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html), [Sam Newman — Monolith to Microservices](https://samnewman.io/books/monolith-to-microservices/), [DDD — Eric Evans](https://www.domainlanguage.com/), [Pure vs applied modularization — Vlad Khononov](https://vladikk.com/2018/02/11/modular-monolith/)
**Trạng thái:** Đã xuất bản
**Độ khó:** Trung cấp
**Dự án:** Hệ thống Tài chính Tiêu dùng Modular — bản đồ module cho các bounded context cho vay (onboarding/KYC, application, credit decisioning, contracting, disbursement, loan servicing, ledger, reporting).

---

## 📌 Tóm tắt Chính (TL;DR)

- Ý tưởng gốc muốn "các thành phần độc lập cao **và** các tác vụ liên kết qua back-end vững chắc" — ghi chú này giải quyết căng thẳng đó: độc lập là thuộc tính của **ranh giới + hợp đồng**, không phải của *bạn chạy bao nhiêu server*.
- **Modular monolith** = một đơn vị triển khai với *ranh giới module cứng rắn* (được ràng buộc bằng tests/linters) và dữ liệu cô lập mỗi module. Nó mang lại ~80% lợi ích của sự độc lập với ~20% chi phí vận hành — default đúng cho hệ thống công ty của đội nhỏ / người-xây-một-mình.
- **Microservices** mua khả năng scale độc lập, triển khai độc lập, cô lập lỗi — nhưng mỗi service tốn lương một operator cho giám sát, mạng, versioning và testing. Tách sớm là sai lầm đắt nhất trong không gian này.
- **Distributed monolith** là anti-pattern phải tránh bằng mọi giá: nhiều service *khớp nối chặt* về dữ liệu và lời gọi — tệ nhất của cả hai thế giới.
- Tách khi **fitness functions / bằng chứng** bảo bạn tách (tự trị của đội, scale một hotspot, cô lập quy định), không phải khi kiến trúc *cảm thấy* sạch hơn.
- Định luật Conway là lực quyết định: **cấu trúc của hệ thống sẽ phản chiếu cấu trúc của đội ngũ** — với một người xây/đội, hãy giữ số module nhỏ và tôn vinh các ranh giới.

---

## 🧠 Ghi chú Chi tiết

### 1. Khái niệm Lõi

**Monolith** — một tiến trình, một deployable: toàn bộ code trong một đơn vị codebase. Vận hành đơn giản, debug đơn giản, nhưng kỷ luật ranh giới là *thủ công* — qua nhiều năm, "big ball of mud" hình thành.

**Modular monolith** — vẫn một deployable, nhưng codebase được tổ chức thành **các module với ranh giới được ràng buộc**:
- Mỗi module sở hữu **dữ liệu** của nó (bảng/schema riêng; không có truy cập bảng xuyên module).
- Các module giao tiếp qua **giao diện tường minh** (export-only `internal` của Python, hạn chế `from module import`, REST/function APIs).
- **Quy tắc import được kiểm tra bằng máy** (ví dụ: `import-linter` cho Python) để không ai vô tình khớp nối các module.
- Có thể tách thành microservices **sau này** — modular monolith là một hình dạng thân thiện với microservices, không phải ngõ cụt.

**Microservices** — nhiều deployable, mỗi cái độc lập deploy/scale/cô-lập-lỗi, giao tiếp qua mạng (thường là events/REST).

**Bounded context (DDD)** — ranh giới khái niệm của một mô hình lĩnh vực: *cùng một từ* (ví dụ: "Invoice") có thể mang nghĩa khác nhau trong các context khác nhau, và mỗi context sở hữu mô hình của nó. Bounded contexts là chỉ dẫn *độ hạt* cho cả modular monolith lẫn microservices.

**Distributed monolith** — các service trông như microservices nhưng dùng chung database, gọi nhau đồng bộ trong chuỗi sâu, và phải triển khai cùng nhau. Hệ thống khó vận hành nhất, không lợi ích nào.

**Fitness functions** — các kiểm tra tự động *bảo vệ* ràng buộc kiến trúc qua thời gian (dependency rules, độ trễ tối đa, không truy vấn DB xuyên module). Chúng chuyển "chúng tôi coi trọng tính modular" thành các cổng CI kiểm chứng được.

### 2. Nó Hoạt động Thế nào (Cơ chế)

**Công thức tích hợp-độc lập (giống nhau cho cả hai kiến trúc):**

1. **Tìm các bounded contexts** — các đường nét thật của lĩnh vực bạn: `Billing`, `Inventory`, `Ledger`, `Approval`, `Reporting`…
2. **Cấp cho mỗi context dữ liệu của nó** — không có bảng dùng chung; dữ liệu xuyên context chảy như *events* (xem [[01-event-driven-architecture-task-orchestration]]) hoặc qua API tường minh.
3. **Ràng buộc ranh giới bằng máy** — dependency rules trong CI; một module chỉ được import các package của chính nó + giao diện đã công bố.
4. **Version hoá các hợp đồng** — mọi giao diện công khai/event schema tuân theo SemVer; thay đổi phá vỡ = major version + cửa sổ di cư.
5. **Đo bằng fitness functions** — ví dụ: "không module nào import trực tiếp `ledger.db`", "độ trễ event < 500 ms p95", "module X có ≥ 1 release train test".

**Đường dẫn quyết định:**

```
Bạn có một người tiêu dùng thứ hai cụ thể / hotspot scale rõ ràng / đội độc lập?
        │
   ┌────┴─────────────────────────┐
   ▼                             ▼
  KHÔNG                        CÓ
   │                             │
   ▼                             ▼
Bắt đầu MODULAR MONOLITH   Cân nhắc tách CHỈ bounded
(ranh giới + hợp đồng     context đó thành một service
đã có sẵn)                 (strangler: mỗi lần một module)
        │
        └──▶ Distributed monolith nếu tách mà không có ranh giới/hợp đồng ❌
```

### 3. Triển khai (Pseudocode / Cấu trúc)

**Bố cục modular monolith trong Python:**

```
app/
├── application/           # khởi tạo khoản vay / tiếp nhận hồ sơ
│   ├── domain/            # entities, policies (không import framework)
│   ├── application/       # use-cases (logic orchestration)
│   ├── infrastructure/    # DB adapters, external clients
│   └── public_api.py      # THỨ DUY NHẤT mà module khác được import
├── credit_decisioning/    # (hình dạng tương tự)
├── loan_servicing/        # (hình dạng tương tự)
└── shared_kernel/         # thực sự xuyên suốt: ids, money, time
```

**Ranh giới được ràng buộc (ví dụ import-linter):**

```python
# .importlinter
[Main]
root_packages = ["app"]
include_external_packages = false
ignore_imports = ["app.shared_kernel.* -> app.*"]

[Contracts.guard-module-data]
type = forbid
source_modules = ["app.application.infrastructure"]
forbidden_modules = ["app.ledger.infrastructure"]
reason = "Modules must never touch each other's data layer."
```

**Một fitness function (kiểm tra CI):**

```python
# ci/gate_modularity.py — làm build thất bại nếu ranh giới bị vi phạm
from importlinter import lint_filesystem
failures = lint_filesystem(".importlinter")
if failures:
    raise SystemExit(f"Modularity gate FAILED: {failures}")
```

**Khi bạn sau này tách một context ra (strangler):**

```python
# bước 1: chỉ định tuyến lưu lượng Origination tới service mới sau một feature flag
if feature_flag("origination_service"):
    return await remote("origination")  # microservice mới
return origination_local()               # module cũ — giữ cho tới khi lưu lượng cạn
```

### 4. Ứng dụng Thực tế — cho Company Task System

**Đề xuất của tôi: modular monolith trước.** Lý do:

1. **Một người xây / đội nhỏ** → chi phí vận hành của microservices (giám sát, triển khai, versioning, test mạng, debug phân tán) không thể trả bằng một người.
2. Toàn bộ *giá trị* của ý tưởng nằm ở workflow lĩnh vực và tích hợp AI — hãy dành nỗ lực ở đó, không phải vào vận hành.
3. Một modular monolith **đã mang lại** các phẩm chất chính của ý tưởng: độ độc lập thành phần cao (bounded contexts + ranh giới ràng buộc + dữ liệu cô lập), dễ sửa đổi (một codebase để refactor, hợp đồng có version), phát triển lâu dài (fitness functions bảo vệ khả năng tiến hoá).
4. Khi scale/đội trở thành thật → **strangler-extract** bounded context bận nhất (nhiều khả năng là `credit_decisioning`, hotspot scoring theo hồ sơ) thành một service, mỗi lần một cái, không viết lại hệ thống.

**Bản đồ module cụ thể cho v1 (vòng đời cho vay):**

| Bounded context | Sở hữu | Phát sự kiện |
|-----------------|--------|--------------|
| `identity` / `access_control` | tài khoản người dùng (customer/staff/admin), mật khẩu (bcrypt), session/JWT, phân quyền theo role, chính sách quyền | `user.registered`, `user.role.changed`, `user.deactivated` |
| `onboarding` / `kyc` | định danh khách hàng, xác minh scan CCCD, sàng lọc PEP/sanctions, đồng thuận Nghị định 13 | `customer.verified`, `kyc.failed` |
| `application` (origination) | chọn sản phẩm, số tiền/kỳ hạn, kênh (app/e-wallet/POS), điều khoản giá | `application.submitted` |
| `credit_decisioning` | scoring, policy/rules engine, dữ liệu bureau (CIC), auto-duyệt vs referral | `credit.decision.approved`, `credit.decision.rejected`, `credit.decision.referred` |
| `contracting` | sinh hợp đồng vay, e-sign, hồ sơ thoả thuận | `contract.signed`, `contract.expired` |
| `disbursement` | trả tiền cho khách hàng/merchant qua adapter ngân hàng/e-wallet (exactly-once) | `disbursement.completed` |
| `loan_servicing` | lịch trả nợ, dồn tích lãi, trả trước, phát hiện quá hạn | `repayment.received`, `loan.in.arrears`, `schedule.activated` |
| `ledger` | hạch toán kế toán khoản vay double-entry, tách gốc/lãi, **dự phòng IFRS 9**, dồn tích | `ledger.posted` |
| `reporting` | báo cáo SBV/CIC, dashboards (đọc sự kiện) | — |
| *(v1)* `collections` | nhắc nợ, thu hồi, tái cấu trúc | `collection.action` |
| *(v1)* `merchant` | đăng ký đối tác POS, hoa hồng/quyết toán cho trả góp | `merchant.registered`, `settlement.due` |

Mọi mũi tên giữa các context là một **event trên bus** — độc lập bên trong, liên kết qua hợp đồng. Đó là ý tưởng gốc, được hiện thực hoá cho một bên cho vay. Lưu ý việc tách context "Approval/AML" cũ: xác minh định danh giờ sống trong `onboarding/kyc`; quyết định *rủi ro* sống trong `credit_decisioning`. Các mô hình, dữ liệu, độ trễ khác nhau — ép chúng lại với nhau là cái bẫy god-context kinh điển.

`identity`/`access_control` là một context hạng nhất (không chỉ là "tiện ích xuyên suốt"): các context khác gọi nó qua public API để kiểm tra quyền (trong-process lúc MVP, qua network khi tách service), không sở hữu dữ liệu người dùng. Điều này hoá giải rủi ro "Bảo mật & identity vắng mặt" ở [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md).

### 5. Bảng So sánh

| Chiều | Classic Monolith | Modular Monolith | Microservices |
|--------|------------------|------------------|---------------|
| Triển khai độc lập | ❌ | Một đơn vị (module chia sẻ deploy) | ✅ theo service |
| Scale độc lập | ❌ | ❌ (scale cả app) | ✅ theo hotspot |
| Cô lập lỗi | ❌ (crash = tất cả) | ❌ (tiến trình chung) | ✅ một phần |
| Ràng buộc ranh giới | Thủ công (trôi dạt) | **Kiểm tra bằng máy** ✅ | Mạng = ràng buộc (nhưng đắt) |
| Độ phức tạp vận hành | Thấp | **Thấp–Trung bình** | Cao |
| Chi phí refactoring | Thấp (một repo) | Thấp–Trung bình | Cao (thay đổi xuyên service) |
| Độc lập dữ liệu | ❌ DB dùng chung | ✅ schema mỗi module | ✅ DB mỗi service |
| Đau đầu debug phân tán | Không | Không | Cao |
| Tự trị đội (mỗi module) | Không | Trung bình | Cao |
| Tốt nhất cho | Prototypes | **Đội nhỏ, hệ thống nặng lĩnh vực** | Tổ chức lớn, nhu cầu scale thật |

---

## 📝 Hành trình Nghiên cứu

- **Tại sao:** Căng thẳng trung tâm của ý tưởng gốc — "các thành phần độc lập cao" trong khi "các tác vụ liên kết" — buộc tôi cuối cùng hiểu rằng độc lập và tích hợp *không đối lập*; chúng là hai mặt của cùng một hợp đồng. Ghi chú này là quá trình tôi rèn điều đó ra.
- **Vật lộn:** Tôi đã dành thời gian dài tin microservices = "kiến trúc tốt" như một tuyệt đối. Mọi talk hội nghị làm monolith nghe như điều đáng hổ thẹn. Cú lật tinh thần cần tách *chất lượng kiến trúc* (ranh giới, hợp đồng, sở hữu dữ liệu) khỏi *topology triển khai* (một tiến trình vs nhiều tiến trình).
- **Khoảnh khắc à-ha:** (1) **Distributed monolith là phương án tệ nhất xa thẳm** — microservices không có bounded contexts chỉ là một monolith chậm với mạng. (2) **Định luật Conway**: "dịch vụ độc lập" của một người xây solo chỉ là các thư mục cộng thêm đau đớn — đơn vị độc lập trung thực cho tôi là *module + ranh giới ràng buộc*. (3) Fitness functions biến giá trị kiến trúc thành cổng CI — kiến trúc trở thành *test được*, cùng tinh thần với các cổng đánh giá ML tôi đã biết từ MLOps.
- **Liên kết sự nghiệp:** Người phỏng vấn và sản phẩm thực đều dò xét quyết định này liên tục; *bảo vệ được* một chiến lược modular-monolith-trước với fitness functions chính xác là phán đoán kỹ thuật cấp cao mà một AI Engineer trong hệ thống nghiệp vụ cần.

## 🔗 Chủ đề Liên quan

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — ý tưởng dự án (Insight 3: độc lập cần hợp đồng; Insight 7: Định luật Conway).
- [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md) — lớp hợp đồng/sự kiện cho phép các context độc lập *liên kết*.
- [04-mcp-and-function-calling-agent-interfaces.md](04-mcp-and-function-calling-agent-interfaces.md) — cách agents phơi bày/tiêu thụ cùng các module API.
- *(bên ngoài, Nhật ký Nghiên cứu)* `books_summaries/agile_project_management/notes/07-agile-architecture-hal-refactoring-scaling.md` — kiến trúc tiến hoá, refactoring, và khung scaling điều khiển *khi nào* tách.
- *(bên ngoài, Nhật ký Nghiên cứu)* `iot_aiot/03-hardware-abstraction-layer-hal.md` — cùng kỷ luật ranh giới ở cấp phần cứng (HAL = lớp hợp đồng của phần cứng).
- [ĐỀ XUẤT] **Ghi chú cầu nối: "Bounded contexts trong cho vay tiêu dùng (KYC vs application vs credit decisioning vs disbursement vs loan servicing)"** — cách *khám phá* ranh giới module thật của bên cho vay trước khi viết module. *Tại sao quan trọng:* sai lầm ranh giới là loại đắt nhất; khám phá DDD là thuốc giải.

## 🤔 Câu hỏi Mở

- [ ] Bounded context nào *chứng minh được* là sai trong bản đồ module v1 tôi đề xuất — quy trình cho vay nào sẽ lộ ra điều đó?
- [ ] Ngăn xếp fitness-function rẻ nhất cho một dự án Python solo là gì (import-linter + structural tests + contract tests)?
- [ ] Khi nào một *công ty tài chính đơn* thực sự cần microservices (cô lập quy định? burst multi-tenant? hot-path mô hình scoring?) — xem lại với bằng chứng.
- [ ] Các cổng đánh giá AI (từ MLOps) đóng vai trò fitness functions modularity trong lớp xuyên suốt `ai_ops` thế nào?

---
*Độc lập không phải là bạn chạy bao nhiêu tiến trình; nó là các module của bạn có ít lý do phải thay đổi cùng nhau đến mức nào.*