# Ý tưởng — Hệ thống Tác vụ Modular cho Công ty, Vận hành bởi AI Agents

**Ngày tạo:** 2026-09-23
**Cập nhật lần cuối:** 2026-09-23
**Tags:** `#ProjectIdea`, `#SystemArchitecture`, `#Modularity`, `#AIAgents`, `#CLIFirst`, `#EventDriven`, `#DomainSpecific`
**Trạng thái:** Bản nháp — đang đánh giá
**Loại:** Ghi nhận ý tưởng + đánh giá phản biện (cấu trúc & chất lượng lập luận)
**Dự án:** Hệ thống Tài chính Tiêu dùng Modular — lĩnh vực neo đậu: **vận hành tài chính tiêu dùng**, lấy chuẩn **Home Credit Việt Nam** (một công ty tài chính được cấp phép, chịu giám sát của NHNN). Clarification Gap #1 đã làm rõ: "công ty có chuyên môn cụ thể" là một *bên cho vay*, không phải ngân hàng nhận tiền gửi.

> **📌 Đánh dấu mục tiêu:** Đây là tài liệu **ý tưởng / hướng tới v1**. Mọi phần liên quan AI/agent/backbone đang được **trì hoãn ở MVP** — MVP thuần feature, không AI, không event bus, không MCP (xem [00-mvp-backlog.md](00-mvp-backlog.md)).

---

## 📌 Nhật ký Quyết định (cập nhật 2026-09-24)

Các quyết định định hình kiến trúc về sau — ghi lại ở đây để các chỉnh sửa tương lai luôn nhất quán:

1. **Lĩnh vực neo đậu = cho vay tiêu dùng (chuẩn HomeCredit).** Không phải payment processor, không phải ngân hàng bán lẻ. Luồng chịu lực là **vòng đời khoản vay** (`origination → underwriting → hợp đồng → giải ngân → trả nợ/servicing → sổ sách`), không phải "thanh toán làm trung tâm".
2. **Lát cắt v0 = vay tiền mặt online, end-to-end**, mô phỏng kênh số hoá nhất của Home Credit (KYC/CCCD → auto-duyệt ~3 phút → hợp đồng → giải ngân ~10 phút → trả nợ đầu → hạch toán). Lý do: saga sạch nhất, kiểm chứng yêu cầu exactly-once của luồng tiền di chuyển, chưa cần context merchant/POS.
3. **Bản đồ bounded context đã sửa (thay thế bản đồ cũ theo hướng `payments`-centric hoặc `approval/AML`-centric):** `onboarding/kyc`, `application`, `credit_decisioning`, `contracting`, `disbursement`, `loan_servicing`, `ledger`, `reporting` (+ `collections`, `merchant` trong v1). `Approval/AML` được tách thành `onboarding/kyc` (định danh) và `credit_decisioning` (rủi ro) — chúng là các mô hình, dữ liệu và độ trễ khác nhau.
4. **Ranh giới benchmarking:** hồ sơ HomeCredit chỉ được dùng làm tham chiếu *nghiệp vụ* dựa trên thông tin công khai (khung sản phẩm/SLA, bề mặt tuân thủ NHNN/CIC/Nghị định 13/2023). Không tham chiếu bất kỳ tài liệu quy trình nội bộ độc quyền nào.

---

## 💡 Ý tưởng Gốc (Nguyên văn)

> *"Tôi muốn làm 1 hệ thống lớn chuyên biệt cho các tác vụ của 1 công ty có chuyên môn cụ thể, các tác vụ ấy có khả năng liên kết với nhau thông qua hệ thống back-end vững chắc, tuy nhiên các thành phần, nhân tố trong đó phải có khả năng độc lập cao, có khả năng thích ứng với thời cuộc, dễ dàng sửa đổi và phát triển dài lâu. Đồng thời có khả năng tích hợp AI và có hệ thống CLI để thao tác tự do bằng AI Agent."*

*(Nguyên văn tiếng Việt được giữ nguyên như một trích dẫn được đánh dấu, theo Language Policy A6.)*

**Diễn giải tiếng Anh:** Một hệ thống lớn chuyên biệt cho các tác vụ của một công ty có chuyên môn lĩnh vực cụ thể; các tác vụ đó có thể liên kết với nhau thông qua một hệ thống back-end vững chắc; tuy nhiên mọi thành phần bên trong phải có độ độc lập cao, thích ứng với thời cuộc, dễ sửa đổi và hỗ trợ phát triển lâu dài. Đồng thời, hệ thống phải tích hợp được AI và phải có hệ thống CLI để AI Agents có thể vận hành tự do.

---

## 📌 Tóm tắt Chính (TL;DR)

- Ý tưởng mô tả một **nền tảng nghiệp vụ theo lĩnh vực, có agent vận hành**: dọc (chuyên biệt), modular (các thành phần độc lập), tiến hoá được (dài hạn) và AI-native (CLI + agents).
- Ba bản năng mạnh nhất ở đây là: **(1) chuyên biệt lĩnh vực là hào cạnh tranh**, **(2) độc lập thành phần là mục tiêu thiết kế**, **(3) agent là tác nhân vận hành hạng nhất** — đây chính là ba điểm khác biệt của các nền tảng hiện đại.
- Căng thẳng cốt lõi cần giải quyết là **độc lập vs. tích hợp**: các thành phần phải vừa độc lập *vừa* liên kết với nhau cần một lớp hợp đồng tường minh (events/APIs/schemas). Quyết định "liên kết" này ở back-end chính là nơi thiết kế sẽ sống hay chết.
- Khoảng trống lớn nhất: chưa có lĩnh vực cụ thể đầu tiên; "agents vận hành tự do" chưa có câu chuyện quản trị; **riêng CLI không phải là giao diện agent** (agent còn cần các giao thức có cấu trúc như JSON schema / MCP / OpenAPI).
- Nhận định: một ý tưởng **định hướng tốt, đúng chiến lược**, với bản năng kiến trúc rõ ràng, chỉ bị suy yếu bởi tính trừu tượng (chưa có lĩnh vực neo đậu) và một giả định không an toàn (agent tự do chưa được quản trị).
- **Đánh giá bổ sung (2026-09-23):** +5 insight (Định luật Conway, tiến hoá dữ liệu, kinh tế AI theo thao tác, cực tính bánh đà dữ liệu, mua quyền chọn), danh sách 12 **khoảng trống cần làm rõ** ý tưởng phải giải quyết, và **8 rủi ro phải nhận biết** trước khi cam kết — xem các phần mở rộng bên dưới.

---

## 🧠 Giải cấu trúc Ý tưởng (6 Yêu cầu Nguyên tử)

| # | Yêu cầu (lời người dùng) | Diễn giải Kỹ thuật | Trục Kiến trúc |
|---|--------------------------|--------------------|----------------|
| 1 | "Hệ thống lớn chuyên biệt cho một công ty có chuyên môn cụ thể" | Một **nền tảng lĩnh vực dọc** xây quanh quy trình của một nghề (không phải công cụ tổng quát) | Phạm vi & lĩnh vực |
| 2 | "Các tác vụ liên kết thông qua back-end vững chắc" | Một **backbone tích hợp**: messaging đáng tin cậy, hợp đồng dữ liệu, choreography/orchestration quy trình | Tích hợp |
| 3 | "Các thành phần có độ độc lập cao" | **Modularity / khớp nối lỏng**: khả năng triển khai độc lập, bounded contexts, cô lập lỗi | Coupling |
| 4 | "Thích ứng với thời cuộc, dễ sửa đổi, lâu dài" | **Kiến trúc tiến hoá**: kỷ luật refactoring, versioning, tương thích ngược, ADRs | Khả năng tiến hoá |
| 5 | "Khả năng tích hợp AI" | **AI như một khả năng nhúng** bên trong tác vụ (models, agents, copilots) | Trí tuệ |
| 6 | "CLI để AI Agents thao tác tự do" | **Giao diện agent-native**: mọi khả năng được phơi bày như một tool gọi được với I/O có cấu trúc | Giao diện |

Ý tưởng này *hoàn chỉnh* một cách bất thường cho một bản nháp đầu tiên — nó bao phủ **phạm vi, tích hợp, coupling, khả năng tiến hoá, trí tuệ và giao diện**, về cơ bản là sáu trong số các hạng mục trình điều khiển kiến trúc kinh điển. Đó là dấu hiệu của tư duy hệ thống.

---

## 💡 Insight & Làm Rõ (Làm Sắc bén Ý tưởng)

### Insight 1 — Quyết định: đây là *product* hay *nền tảng nội bộ*?
"Một công ty có chuyên môn cụ thể" là mập mờ: là **một công ty cụ thể** (công cụ nội bộ) hay **các công ty cùng một nghề** (vertical SaaS product)?
- **Nền tảng nội bộ:** tích hợp sâu với quy trình hiện có, không multi-tenancy, triển khai = hạ tầng của họ.
- **Sản phẩm dọc:** cần multi-tenancy, workflow engine cấu hình được, onboarding, billing, hỗ trợ — một hệ thống lớn hơn về bản chất.
*Vì sao quan trọng:* quyết định này đơn lẻ đổi kiến trúc (single-tenant vs multi-tenant), lộ trình và mô hình kinh doanh. Giải quyết nó trước.

### Insight 2 — "Back-end vững chắc" *chính là* ý tưởng
Yêu cầu #2 là bức tường chịu lực. "Các tác vụ liên kết" hàm ý một **backbone hướng sự kiện** (ví dụ: message bus bền vững + saga/choreography), không chỉ là một chồng lời gọi REST:
- Task A hoàn tất → phát event → Task B kích hoạt, kèm retries, idempotency keys, dead-letter queues và nhật ký kiểm toán.
- "Vững chắc" nên được định nghĩa bằng các thuộc tính đo được: **độ bền, đảm bảo thứ tự, phân phối at-least-once/at-most-once, observability**.
*Vì sao quan trọng:* nếu lớp này bị làm lơ, "các thành phần độc lập" âm thầm trở thành một cục bùn phân tán.

### Insight 3 — Độc lập ≠ cô lập; độc lập cần *hợp đồng*
Độ độc lập cao chỉ sống sót nếu được quản trị bởi **giao diện ổn định, phần bên trong linh hoạt**:
- Mỗi thành phần sở hữu dữ liệu của nó (database-per-context), chỉ giao tiếp qua API/event có version (schema registry, contract tests).
- Trade-off kinh điển: **modular monolith vs microservices**. Bắt đầu với modular monolith (một deployable, ranh giới module cứng rắn nhờ lint rules/archunit), chỉ tách khi một fitness function chứng minh nhu cầu. Điều này khớp với tư duy kiến trúc tiến hoá đã có trong nhật ký (HAL, refactoring, scaling — xem Related Topics).
*Vì sao quan trọng:* thiếu kỷ luật hợp đồng, "độc lập" thoái hoá thành "mỗi đội tự phát minh giao thức riêng" → hỗn loạn tích hợp.

### Insight 4 — "Thích ứng / dễ sửa / lâu dài" là *kỷ luật*, không phải tính năng
Bạn không thể thiết kế khả năng thích ứng một lần; bạn **kiếm được** nó liên tục:
- CI/CD với bộ test nhanh, ADR (Architecture Decision Records) để *cái-tại-sao* sống sót qua các lần đổi người, semantic versioning + cửa sổ deprecation, feature flags để triển khai an toàn, strangler pattern để thay thế phần cũ.
- Fitness functions: các kiểm tra tự động (dependency rules, ngân sách độ trễ, quét bảo mật) *bảo vệ* kiến trúc khi nó tiến hoá.
*Vì sao quan trọng:* đây là yêu cầu mà hầu hết ý tưởng quên, và nó là thứ quyết định hệ thống có còn tồn tại sau 5 năm hay không.

### Insight 5 — "Tích hợp AI" thực ra có nghĩa *ba* thứ khác nhau
Phân định trước khi xây:
1. **AI bên trong tác vụ** — mô hình nhúng trong workflow (ví dụ: trích xuất tài liệu trong một bước phê duyệt).
2. **AI như orchestrator** — một agent lập kế hoạch/thực thi tác vụ đa bước (agent = client của workflow engine).
3. **AI như operator qua CLI** — agent điều khiển hệ thống như một người dùng quyền lực (yêu cầu đã nêu).
Mỗi loại có nhu cầu độ tin cậy khác nhau. Lưu ý AI là **phi tất định** — hệ thống nghiệp vụ cần guardrails: phê duyệt human-in-the-loop, log kiểm toán, fallback tất định, và phạm vi phân quyền.

### Insight 6 — "CLI để agent vận hành tự do" đúng 50% — nửa còn thiếu là *giao thức*
CLI là một giao diện agent tuyệt vời (văn bản vào/ra, tổ hợp được, script được, cờ JSON), nhưng vận hành agent hiện đại cần nhiều hơn:
- **Đầu ra có cấu trúc:** mọi lệnh hỗ trợ `--json` để agent phân tích kết quả chứ không phải văn xuôi.
- **Lệnh tự mô tả:** `help --json` / schema đọc máy được (nghĩ Cobra/Click/Typer + JSON schema) — agent khám phá khả năng lúc runtime.
- **Một lớp giao thức:** MCP (Model Context Protocol) hoặc tool schema OpenAPI cho phép agent gọi khả năng *theo lập trình*, không cần vòng lặp shell. CLI là vỏ cho người/agent; giao thức là hợp đồng.
- **Quản trị:** "tự do" phải bị giới hạn — token quyền-tối-thiểu, sandboxing, luồng phê duyệt cho thao tác phá hoại, nhật ký kiểm toán đầy đủ. Một agent không được quản trị với CLI của hệ thống công ty là gánh nặng, không phải tính năng.
*Insight bổ sung:* **core không đầu + nhiều client mỏng** — một engine, phơi bày qua CLI (agent/scripts), Web UI (người dùng nghiệp vụ), API (tích hợp). UX chỉ có CLI sẽ giới hạn sự chấp nhận ở người dùng kỹ thuật.

### Insight 7 — Định luật Conway là người thiết kế ĐỒNG HÀNH (cấu trúc phản chiếu đội ngũ)
Ranh giới thành phần của hệ thống sẽ phản chiếu cấu trúc của người xây nó. Xây solo (hoặc một đội nhỏ), "các thành phần độc lập" **không có áp lực người tiêu dùng bên ngoài** để giữ tách rời — chúng âm thầm thoái hoá thành các thư mục với import tuỳ tiện. Độc lập chỉ *thật sự* khi có các đội hoặc release train thực sự độc lập (ba module tự triển khai và tiến hoá). Nếu xây một mình, hãy trung thực về đơn vị độc lập thực tế của bạn: **ranh giới module + luật được ràng buộc** (import-linting, contract tests) — không phải service vật lý. Và dùng điều này một cách có chủ đích: đưa sự độc lập thực sự dần dần khi người tiêu dùng thực sự xuất hiện (strangler pattern), thay vì đóng gói sẵn sự tự trị chưa ai cần.

### Insight 8 — "Back-end vững chắc" chủ yếu là bài toán *tiến hoá dữ liệu*
Liên kết tác vụ nghĩa là dữ liệu/event vượt qua ranh giới thành phần — nên **mọi thay đổi schema đều là thay đổi phá vỡ tiềm năng** cho consumer hạ nguồn. Một hệ thống liên kết trường tồn do đó cần, từ ngày đầu: schema registry, semantic versioning của hợp đồng, và chính sách deprecation ("version cũ được hỗ trợ trong N tháng"). "Vững chắc" không phải thuộc tính bạn cài đặt; nó là *kỷ luật versioning* bạn áp dụng trong nhiều năm. Bỏ lỡ điều này, back-end đóng băng — không ai dám đụng vào schema, và yêu cầu "thích ứng, dễ sửa" (4) âm thầm chết.

### Insight 9 — Tích hợp AI biến chi phí một lần thành chi phí *theo thao tác*
Một hệ thống nghiệp vụ do agent vận hành trả **chi phí biến đổi mỗi hành động** (tokens, lời gọi model, độ trễ). Hệ thống nghiệp vụ có lưu lượng cao. Bạn phải mô hình hoá **kinh tế đơn vị mỗi tác vụ**: chi phí/tác vụ, ngân sách độ trễ, ngân sách tỷ lệ lỗi, chi phí fallback. Một hệ thống modular đẹp, agent-operable tốn $0.50/tác vụ trên một thao tác có biên $0.05 là chết kinh tế. Ngoài ra, nhà cung cấp model thay đổi giá/mô hình/khả dụng một cách khó lường — hãy trừu tượng hoá lớp AI tại *giao diện* (tool schema trung lập nhà cung cấp + adapter mỏng), không phải bằng cách bọc từng tính năng hiếm.

### Insight 10 — Bánh đà AI-vs-chuyên-gia có thể chạy ngược
Lợi thế của ý tưởng = tri thức lĩnh vực + AI. Nhưng nếu AI dần thay thế *việc làm* của chuyên gia, dòng dữ liệu thực/đã-thẩm-định mới cạn kiệt, và AI trôi khỏi thực tế (concept drift). Quy trình phải giữ các chuyên gia lĩnh vực làm **người xác thực** (human-in-the-loop) — vì tín hiệu xác thực đó *chính là* dữ liệu đào tạo/eval của ngày mai. Thiết kế cổng kiểm con người ngay trong quy trình, không phải nghĩ thêm sau. Cũng nhớ: tri thức lĩnh vực nằm trong đầu chuyên gia; hệ thống bị con tin của người mã hoá nó đầu tiên — hãy lên kế hoạch khám phá lĩnh vực liên tục (phỏng vấn, quan sát công việc, khai thác log → thiết kế tác vụ mới).

### Insight 11 — "Thích ứng thời cuộc" là mua quyền chọn, không phải tiên đoán
Bạn không thể tiên đoán tương lai; kiến trúc nên mua **các quyền chọn (khả đảo ngược)** thay vì: feature flags, strangler patterns, ADR, và một hợp đồng lõi cố tình mỏng. Mọi cam kết không thể đảo ngược (schema, topology, tech stack) là một vụ cược — hãy đếm số cược và giới hạn chúng. Đây chính là tư duy "kiến trúc tiến hoá + fitness functions" đã có trong các ghi chú agile/kiến trúc của nhật ký.

---

## ❓ Khoảng trống Cần Làm Rõ — Các Điểm Chưa Rõ (Điểm chưa được làm rõ)

Phát biểu ý tưởng để lại những điều này chưa giải quyết. Chúng là *vật cản thiết kế*, không phải câu hỏi về phong cách — mỗi cái đều thay đổi kiến trúc:

1. **Định danh neo đậu:** MỘT công ty cụ thể vs cả một nghề? (Quyết định multi-tenancy, tuỳ biến theo khách hàng, licensing, công sức.)
2. **Độ hạt của tác vụ:** Hành động nguyên tử (một lệnh đơn) vs quy trình kéo dài (một case kéo dài nhiều ngày)? Orchestration khác biệt rất lớn (lời gọi đơn giản vs sagas có bù trừ).
3. **Ngữ nghĩa "liên kết":** Luồng dữ liệu (đầu ra A nuôi B), luồng điều khiển (A kích hoạt B), hay cả hai? Và *chuyện gì xảy ra khi B thất bại sau khi A đã commit*? (hành vi bù trừ/rollback)
4. **Phạm vi AI:** Chỉ generative, hay còn ML cổ điển (routing, dự báo, phân loại)? Tác vụ nào chấp nhận lỗi AI (đề xuất) so với cái không chấp nhận (hạch toán tài chính, đầu ra pháp lý)?
5. **Mô hình operator CLI:** Chỉ con người, chỉ agent, hay cả hai với phân quyền khác nhau? Có role agent chỉ-đọc và role thực thi không? Mọi hành động agent có được log và có thể phê duyệt không?
6. **Triển khai & nơi lưu trú dữ liệu:** Cloud / on-prem / hybrid? Cloud công cộng có thể không được phép với một số dữ liệu công ty (tài chính, y tế, dữ liệu kinh doanh bí mật; Nghị định 13/2023 của Việt Nam về bảo vệ dữ liệu cá nhân).
7. **Hạng nhạy cảm của dữ liệu:** PII, tài chính, y tế, bí mật thương mại? → quyết định phạm vi tuân thủ (quy tắc bảo vệ dữ liệu, quy định ngành, có thể EU AI Act nếu từng phơi bày cho người dùng EU).
8. **Công cụ hiện có:** Người dùng mục tiêu đang chạy gì hôm nay (Excel, ERP, CRM, email)? Hệ thống thay thế, bổ sung, hay import/export với chúng? — Adapter tích hợp thường là 50–80% thời gian dự án ẩn.
9. **Tiêu chí thành công v1:** Kết quả đo được nào chứng minh ý tưởng? (ví dụ: thời gian tác vụ −60%, tỷ lệ lỗi −40%, chi phí/tác vụ dưới X) Không có con số, "hệ thống lớn" mãi không kiểm chứng được.
10. **Quyền sở hữu "chuyên môn":** Tri thức lĩnh vực là của bạn, của đối tác, hay phải học? Hào cạnh tranh ~90% là hiểu lĩnh vực, ~10% là code.
11. **Mô hình kinh doanh (nếu là product):** per-seat, per-task, license, theo-kết-quả?
12. **Vai trò con người sau tự động hoá:** Bước nào *phải* do con người, và điều đó được ràng buộc thế nào? (trách nhiệm pháp lý, niềm tin, quy định)

---

## ✅ Điểm mạnh (Điểm tốt)

1. **Chuyên biệt lĩnh vực là hào cạnh tranh thật.** Nền tảng ngang tổng quát thì chật chội và hàng hoá hoá; một hệ thống tinh chỉnh theo quy trình của một nghề có chi phí chuyển đổi sâu, ROI rõ ràng và khó bị thay thế. Dọc đánh bại ngang với một người xây nhỏ.
2. **Độc lập thành phần là default đúng.** Nó mua cô lập lỗi, triển khai độc lập, làm việc song song của đội, và tự do công nghệ mỗi thành phần — trực tiếp chống lại "big ball of mud" giết chết các hệ thống nội bộ trường tồn.
3. **Tích hợp trước ("back-end vững chắc") là bản năng chín chắn.** Nhiều ý tưởng solo nhảy vào UI trước; cái này đặt đúng mô liên kết (luồng dữ liệu, liên kết tác vụ) làm trung tâm — phần quyết định hệ thống có vượt ngoài demo hay không.
4. **CLI agent-native đi trước xu hướng.** Thiết kế mọi khả năng vận hành được bằng máy (I/O có cấu trúc, tool schemas) khớp với sự dịch chuyển agentic-workflow 2025–2026 (MCP, function calling). Hệ thống được thiết kế *cho agents* sẽ tự động hoá được chặt chẽ hơn hẳn hệ thống retrofit.
5. **Khả năng tiến hoá dài hạn được ưu tiên.** Chủ động coi trọng "thích ứng thời cuộc, dễ sửa, lâu dài" nghĩa là thiết kế bắt đầu từ khả năng bảo trì thay vì coi nó là điều nghĩ thêm — khác biệt giữa tài sản 5 năm và bản viết lại 6 tháng.
6. **Rất khả thi với bộ kỹ năng hiện tại.** Kinh nghiệm Backend + AI/ML + CLI/TUI (xem dự án Custom Multi-TUI) + ghi chú DevOps (IaC, MLOps) bao phủ gần như mọi lớp ý tưởng này cần. Đây là một dự án tham vọng nhưng *đạt được*, không phải vapourware.

---

## ⚠️ Điểm yếu & Rủi ro (Điểm xấu)

1. **Bùng nổ phạm vi / cái bẫy "hệ thống lớn".** "Lớn + chuyên biệt + AI + CLI + lâu dài" mà chưa có lĩnh vực cụ thể đầu tiên là cái bẫy v1 kinh điển: mọi thứ đều khả thi, không gì ra đời. Không có một lát cắt khởi đầu có giới hạn, dự án phình ra hoặc đình trệ.
   *Giảm thiểu:* chọn MỘT lĩnh vực công ty thật + 3 tác vụ cụ thể cho v0; phát triển qua strangler pattern.
2. **Mâu thuẫn độc lập-vs-tích hợp chưa được giải quyết.** Các thành phần "độc lập cao" không tự nhiên "liên kết qua back-end vững chắc" — liên kết *chính là* coupling. Nếu lớp hợp đồng (events, schemas, versioning) không được thiết kế, bạn có một trong hai chế độ lỗi: hỗn loạn phân tán (độc lập nhưng không liên kết được) hoặc monolith ẩn (liên kết được nhưng không độc lập).
   *Giảm thiểu:* thiết kế lớp hợp đồng trước: event catalog, chính sách schema versioning, contract tests.
3. **Không có người dùng, người mua, hay mô hình kinh doanh được định nghĩa.** Ý tưởng mô tả *kiến trúc*, không phải *giá trị*: ai dùng hàng ngày, ai trả tiền, nỗi đau nào được loại bỏ? Quy trình thực của công ty bừa bộn; thiếu lĩnh vực neo đậu và người dùng thật, mọi quyết định thiết kế vẫn trừu tượng và không kiểm chứng được.
   *Giảm thiểu:* viết một trang phát biểu vấn đề cho mỗi tác vụ: người dùng, nỗi đau, cách đối phó hiện tại, thắng-lợi đo lường được.
4. **"Agents vận hành tự do" là giả định không an toàn cho hệ thống nghiệp vụ.** Vận hành agent tự do trên tác vụ công ty = ghi dữ liệu nghiệp vụ tự trị, lệnh phá hoại, bề mặt prompt-injection. Không có phân quyền, sandboxing, cổng phê duyệt và log kiểm toán, một lượt agent tồi có thể làm hỏng vận hành hoặc rò rỉ dữ liệu.
   *Giảm thiểu:* role agent quyền-tối-thiểu, dry-run mặc định, phê duyệt con người cho thao tác không thể đảo ngược, nhật ký kiểm toán bất biến.
5. **UX chỉ-CLI giới hạn sự chấp nhận.** Nhân sự nghiệp vụ (kế toán, operator, quản lý) sẽ không sống trong terminal. Chỉ-CLI nghĩa là hệ thống phục vụ developer và agent, loại trừ những con người xác thực giá trị lĩnh vực — thu hẹp thị trường xuống các công ty rành kỹ thuật.
   *Giảm thiểu:* core không đầu; CLI là bề mặt power-user/agent, Web UI là client mỏng trên cùng API.
6. **"Lâu dài" nâng sàn bảo trì.** Một hệ thống đa thành phần modular, tiến hoá được, quan sát được tốn kém hơn để vận hành so với app đơn giản: giám sát liên module, quản lý trôi dạt version, nâng cấp dependency, bảo trì contract test. Không có CI/CD và kỷ luật testing từ ngày đầu, mục tiêu thích ứng âm thầm chết dưới nợ kỹ thuật.
   *Giảm thiểu:* tự động hoá fitness functions sớm (kiểm tra dependency rules, smoke tests, IaC cho môi trường tái lập).
7. **Rủi ro độ tin cậy AI trong đường tới hạn nghiệp vụ.** Ảo giác, phi tất định, và trôi dạt mô hình bên trong tác vụ liên kết có thể thác đổ (task B tin đầu ra AI của task A). AI cần pipeline riêng: evaluation, guardrails, quy tắc fallback (xem ghi chú MLOps lifecycle).

---

## 🚨 Rủi ro phải Nhận biết Trước khi Cam kết (Rủi ro phải nhận)

Đây là những rủi ro bạn *thừa hưởng do bản chất của chính ý tưởng* — hãy nhận biết ngay, hoặc chúng sẽ nổi lên sau này như bất ngờ:

1. **Bus-factor = 1 (hội chứng người-xây-đơn độc).** Một hệ thống "lớn, lâu dài" do phần lớn một người xây tập trung toàn bộ tri thức và năng lượng vào một cái đầu. Nếu ưu tiên của bạn đổi (tốt nghiệp, công việc đầu, sở thích đổi), hệ thống không ai tiếp quản. *Chỉ chấp nhận nếu* bạn coi "sống sót khi không có tôi 6 tháng" là một thuộc tính thật: ADR, bản đồ module, phạm vi nhỏ trung thực.
2. **Prompt injection ở quy mô nghiệp vụ.** Agent xử lý nội dung bên thứ ba (hồ sơ khách hàng — bản scan CCCD, ảnh chứng minh thu nhập, tài liệu đối tác) mở một bề mặt prompt-injection *gián tiếp*: chỉ thị độc hại giấu bên trong tài liệu nghiệp vụ có thể khiến agent trích xuất dữ liệu, phê duyệt khoản vay, hoặc kích hoạt giải ngân. Đây là rủi ro an ninh AI #1 cho hệ thống nghiệp vụ agentic — không phải giả thuyết. Giảm thiểu: tách nội dung/chỉ thị, allowlist cấp tool, quyền tối thiểu, phê duyệt con người cho hành động hệ trọng.
3. **Tuân thủ & trách nhiệm pháp lý.** Đầu ra AI điều khiển quyết định nghiệp vụ thật tạo trách nhiệm pháp lý (hành động tài chính sai, đầu ra pháp lý tồi, rò rỉ dữ liệu). Là một *product*, bạn thừa hưởng phơi nhiễm quy định (bảo vệ dữ liệu, quy tắc ngành, có thể EU AI Act). *Chỉ chấp nhận khi* có phạm vi tuân thủ định nghĩa rõ, tuyên bố từ chối trách nhiệm, và nhật ký kiểm toán bất biến.
4. **Khuếch đại garbage-in-garbage-out.** Hệ thống liên kết tự động truyền lỗi: dữ liệu tồi ở task A làm ô nhiễm B và C — đó đúng nghĩa là "liên kết". Chất lượng dữ liệu là vấn đề văn hoá công ty bạn không thể sửa trọn trong code, mà hệ thống của bạn lại bị đổ lỗi cho dữ liệu nó không tạo ra. Giảm thiểu: theo dõi provenance + cổng xác thực ở mọi ranh giới.
5. **Lock-in nhà cung cấp ở lớp AI.** Nhà cung cấp đổi giá/mô hình/API ngay dưới bạn. Phần "thích ứng" của hệ thống phải gồm một giao diện mỏng trung lập nhà cung cấp, hoặc bạn chấp nhận làm lại định kỳ.
6. **Mục nát công nghệ.** Thứ bạn xây hôm nay sẽ già đi; "dễ sửa lâu dài" là lời hứa với future-bạn người sẽ trả giá bằng refactoring. Một hệ thống trường tồn là một *đăng ký nỗ lực*, không phải bản xây một lần — bảo trì cộng dồn mỗi năm.
7. **Chi phí cơ hội cho một sinh viên.** Đây là cam kết nhiều năm. Mỗi tháng cho nó là một tháng không dành cho các khoá ML sâu hơn, thực tập, hoặc dự án portfolio cạnh tranh. *Chấp nhận nó có ý thức*, hoặc thu nhỏ v1 xuống một module + một khách hàng.
8. **Tổng quát hoá sớm (premature generic-ization).** Thiết kế "độc lập/thích ứng" *trước* một tập tác vụ cụ thể thường sinh ra các trừu tượng sai — và sửa trừu tượng sai là loại làm-lại đắt nhất. *Quy tắc ba:* xây cụ thể trước; chỉ trừu tượng hoá khi nhu cầu thứ ba thật xuất hiện.

### Bảng tổng hợp sổ rủi ro

| # | Rủi ro | Mức độ nghiêm trọng nội tại | Giảm thiểu được ngay? | Chấp nhận nguyên trạng? |
|---|------|---------------------------|------------------------|-------------------------|
| 1 | Bus-factor = 1 | Cao | Một phần (docs, phạm vi nhỏ) | Có điều kiện |
| 2 | Prompt injection | Nghiêm trọng | Có (lúc thiết kế) | Không — phải thiết kế để đối phó |
| 3 | Tuân thủ & trách nhiệm pháp lý | Cao (nếu product) | Một phần (rà soát pháp lý) | Có điều kiện |
| 4 | Khuếch đại GIGO | Trung bình | Một phần (cổng xác thực) | Có điều kiện |
| 5 | Lock-in nhà cung cấp AI | Trung bình | Có (giao diện mỏng) | Có — với adapter mỏng |
| 6 | Mục nát công nghệ | Trung bình | Một phần (ADR, quyền chọn) | Có — ngân sách refactoring |
| 7 | Chi phí cơ hội | Cá nhân | Có (thu nhỏ v1) | Bạn tự quyết |
| 8 | Trừu tượng sớm | Cao | Có (quy tắc ba) | Không — phải kỷ luật |

---

## 🏗️ Đánh giá Kết cấu (Đánh giá về KẾT CẤU)

**Kết cấu làm đúng điều gì:**
- Bản năng phân lớp rõ ràng: **hệ thống → tác vụ → thành phần → giao diện (CLI) → trí tuệ (AI)**. Ý tưởng tách *hệ thống làm gì* (tác vụ) khỏi *nó được xây thế nào* (các thành phần độc lập) khỏi *nó được vận hành ra sao* (CLI/agents) — một phân rã ba trục khoẻ mạnh.
- Các yêu cầu phi chức năng được nêu tường minh (độc lập, thích ứng, khả sửa, trường tồn). Hầu hết ý tưởng bản nháp đầu chỉ nêu mục tiêu chức năng; nêu tên các *phẩm chất* là cách kiến trúc thật sự bắt đầu.

**Các vấn đề kết cấu:**
1. **Thiếu thứ tự ưu tiên.** Độc lập, thích ứng, khả sửa, và trường tồn *xung đột* khi chịu áp lực (ví dụ: ship-nhanh vs ranh giới sạch). Kiến trúc = làm tường minh các đánh đổi; không có danh sách trình điều khiển được xếp hạng, mọi quyết định tương lai mặc định thành "cái gì nhanh nhất".
2. **Lớp dữ liệu vắng mặt trong kết cấu.** Trạng thái sống ở đâu — shared database, database-per-component, event store? Độc lập thành phần chủ yếu là câu hỏi *sở hữu dữ liệu*; bỏ nó ra là lỗ hổng kết cấu lớn nhất.
3. **Bảo mật & identity vắng mặt.** Không nhắc persona, role, hay ranh giới quyền — trong khi "agents vận hành tự do" làm cho authorization trở thành lớp thiếu chịu lực nhất.
4. **Observability vắng mặt.** Hệ thống phân tán, liên kết, trường tồn không có tracing/logging/metrics thì không debug hay tiến hoá được; "back-end vững chắc" không observability là không kiểm chứng được.
5. **CLI nằm sai mức trừu tượng.** Yêu cầu trộn một *giao diện người dùng* (CLI) với một *hợp đồng tích hợp* (agent gọi khả năng thế nào). Về kết cấu, hình dạng sạch là: `Core engine → Lớp API/protocol (MCP/OpenAPI) → {CLI, Web UI, Agent adapters}`. CLI nên là *client của giao thức*, không phải chính giao thức.

**Phác thảo kết cấu đã sửa:**
```
                    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   Client mỏng →    │  CLI (người/ │  │   Web UI     │  │ Agent adapter│
                    │  agent)      │  │ (nghiệp vụ)  │  │ (MCP/tools)  │
                    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                           └─────────────────┼─────────────────┘
                                             ▼
                          ┌────────────────────────────────────┐
                          │  Lớp API + Giao thức (hợp đồng,    │
                          │  auth, versioning, audit)          │
                          └────────────────┬───────────────────┘
                                             ▼
   Các thành phần độc lập (bounded contexts, DB-per-context)
   [Task A] ──event──▶ [Task B] ──event──▶ [Task C]   ← backbone tích hợp
                                             ▼
                          ┌────────────────────────────────────┐
                          │  Xuyên suốt: observability,        │
                          │  dịch vụ AI, identity/phân quyền   │
                          └────────────────────────────────────┘
```

---

## 🧠 Đánh giá Lập luận (Đánh giá về KHẢ NĂNG TƯ DUY)

**Nơi tư duy mạnh:**
1. **Khung tầm hệ thống.** Ý tưởng suy luận về một *hệ thống toàn thể* (thành phần, liên kết, giao diện, tiến hoá) thay vì một tính năng đơn lẻ — bản năng kiến trúc thật, không phải liệt kê tính năng.
2. **Nhận thức phi chức năng.** Coi trọng độc lập, thích ứng, trường tồn cho thấy nhận thức về các *phẩm chất* phần mềm — dấu hiệu của tư duy kỹ thuật chín chắn (hầu hết người mới chỉ tối ưu "nó chạy").
3. **Hướng tương lai.** Neo vào agents + tích hợp AI cho thấy ý tưởng được thiết kế cho nơi ngành công nghiệp đang đi (hệ thống agent-operable), không phải nơi nó đã qua.
4. **Nhất quán với kinh nghiệm thực.** Thương vụ độc lập đa-repo đã chọn trong dự án Custom TUI (chấp nhận trùng lặp code để có độc lập tuyệt đối) là cùng triết lý ở quy mô lớn hơn — lập luận tổng quát hoá đúng.

**Nơi tư duy sụp đổ:**
1. **Trừu tượng không có neo đậu.** Ý tưởng lơ lửng ở mức "hệ thống lớn" mà không nêu tên lĩnh vực, tác vụ, hay người dùng cụ thể. Tư duy không bao giờ chạm ví dụ cụ thể thì không thể kiểm chứng — và yêu cầu không kiểm chứng được ("thích ứng thời cuộc") âm thầm trở nên không đạt được.
2. **Một mâu thuẫn không được soi xét.** "Các thành phần độc lập cao" + "các tác vụ liên kết qua back-end vững chắc" + "một hệ thống lớn" kéo về ba hướng đối nhau. Ý tưởng *khẳng định* chúng cùng tồn tại nhưng không bao giờ hỏi *bằng cách nào* — cái "bằng cách nào" còn thiếu (lớp hợp đồng) chính là nơi 80% công việc thực nằm.
3. **"Tự do" bị nhầm với "đủ năng lực".** "Thao tác tự do" nhầm truy cập không ràng buộc với năng lực agent. Trong thực tế, agents *năng lực hơn* bên trong ranh giới quyền được chỉ định rõ (chúng biết mình được làm gì) hơn là một bãi mở (chúng phải đoán, và đoán = rủi ro). Đây là điểm mù quản trị, không chỉ bảo mật.
4. **Nhầm lẫn công cụ:** CLI ≈ giao diện agent. Lập luận nhảy từ "agents cần vận hành nó" đến "vậy nên CLI", bỏ qua câu hỏi *agent thực sự tiêu thụ cái gì* (schemas, đầu ra có cấu trúc, giao thức). CLI là một *phần* tốt của câu trả lời, không phải câu trả lời.
5. **"Lớn" được xem như đức tính, không phải chi phí.** Không gì trong ý tưởng cân nhắc chi phí duy mang của sự lớn (gánh nặng vận hành, chi phí phối hợp, refactor chậm hơn). Tư duy trường tồn nên hỏi trước *"hệ thống nhỏ nhất vẫn mang lại giá trị là gì?"*

**Kết luận đánh giá lập luận:** định hướng xuất sắc (đúng mối quan tâm, đúng tương lai), nhưng hiện tại mang tính **khẳng định hơn là phân tích** — nó phát biểu các thuộc tính mong muốn mà không giải quyết các căng thẳng của chúng. Chuyển từng "Tôi muốn X" thành "X đạt được bởi Y, kiểm chứng bởi Z" sẽ nâng nó từ tầm nhìn lên kiến trúc.

---

## 🎯 Các Quyết định Cần Chốt Trước Khi Viết Code

- [x] Chọn MỘT lĩnh vực neo đậu + tác vụ cụ thể cho v0 → **cho vay tiêu dùng (chuẩn HomeCredit), v0 = vay tiền mặt online** *(đã giải quyết 2026-09-24)*.
- [ ] Product vs nền tảng nội bộ? (multi-tenant hay không) — mặc định: hệ thống học tập single-tenant.
- [x] Vẽ task graph: tác vụ nào liên kết, dữ liệu nào vượt ranh giới → **chuỗi vòng đời khoản vay** trong docs 02–03.
- [ ] Chọn backbone tích hợp: event bus / queue / REST orchestration — và định nghĩa "vững chắc" (đảm bảo phân phối, thứ tự, retries).
- [ ] Chọn mô hình sở hữu dữ liệu: database-per-component vs shared store.
- [ ] Ưu tiên AI mode: embedded AI vs orchestrator vs operator (Insight 5).
- [ ] Giao thức agent: CLI với `--json` + MCP server? Định nghĩa role phân quyền & cổng phê duyệt.
- [ ] Xếp hạng trình điều khiển chất lượng (độc lập vs tốc độ vs chi phí) — thứ tự đánh đổi.
- [ ] Bắt đầu bằng modular monolith? Đặt fitness functions kích hoạt việc tách sau này.
- [ ] Chính sách versioning hợp đồng: schema registry, semantic versioning, cửa sổ deprecation.
- [ ] Phân loại dữ liệu & phạm vi cơ quan quản lý: PII / tài chính / y tế? Quy tắc tuân thủ nào áp dụng (ví dụ: Nghị định 13/2023, quy tắc ngành)?
- [ ] Mô hình an toàn agent: role quyền hạn, agent chỉ-đọc vs thực thi, cổng phê duyệt, nhật ký kiểm toán.
- [ ] Mức trừu tượng hoá nhà cung cấp AI (giữ mỏng) + kinh tế đơn vị: ngân sách mỗi tác vụ (tokens, độ trễ, tỷ lệ lỗi, chi phí fallback).
- [ ] Thiết kế human-in-the-loop: bước nào giữ người xác thực, và sự xác thực nuôi dữ liệu đào tạo/eval tương lai ra sao.

---

## 📝 Hành trình Nghiên cứu

- **Tại sao:** Tôi cứ quay vòng về cùng một hình dạng dự án — một hệ thống quy mô công ty nơi các tác vụ liên kết nhưng mỗi phần vẫn độc lập và AI có thể vận hành nó. Ghi lại nó đúng cách (thay vì để nó mãi là ý tưởng "một ngày nào đó" mơ hồ) buộc tôi đối mặt với việc liệu tôi có thực sự hiểu cách xây các hệ thống như vậy, hay chỉ thích cách chúng nghe.
- **Vật lộn:** Phần khó nhất là mâu thuẫn bề ngoài giữa *độc lập* và *liên kết*. Trong một thời gian dài tôi giả định "độc lập hơn = tốt hơn", mà không nhận ra độc lập không có hợp đồng chỉ là phân mảnh — và liên kết không kỷ luật chỉ là monolith mặc trang phục.
- **Khoảnh khắc à-ha:** (1) Độc lập được mua bằng **giao diện ổn định + sở hữu dữ liệu**, không phải tách biệt vật lý. (2) CLI là *vỏ*, nhưng agent thực sự cần một **giao thức** (schemas/MCP) — nhận ra điều này đổi khung "CLI cho agents" thành "nền tảng agent-native, CLI là một mặt của nó". (3) "Khả năng thích ứng" là một *kỷ luật* (tests, CI, ADR), không phải thuộc tính bạn hoàn tất.
- **Liên kết sự nghiệp:** Đây chính xác là hình dạng sản phẩm AI Engineer tôi muốn xây trong Kinh tế/Kinh doanh: một nền tảng nghiệp vụ dọc mà quy trình của nó được AI nâng cấp và mọi khả năng đều agent-callable. Thiết kế hệ thống có quản trị, quan sát được, agent-operable là điểm giao chính xác của các ghi chú backend, AI và DevOps của tôi — và là điểm khác biệt tôi muốn trên CV/portfolio.

---

## 🔗 Chủ đề Liên quan

**Trong repo này:**
- [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md) — triển khai yêu cầu #2 ("các tác vụ liên kết thông qua một back-end vững chắc").
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — giải quyết yêu cầu #3 (độc lập thành phần vs tích hợp).
- [04-mcp-and-function-calling-agent-interfaces.md](04-mcp-and-function-calling-agent-interfaces.md) — triển khai yêu cầu #6 (CLI → giao thức agent có kiểm soát).

**Nguồn gốc sự thật (Nhật ký Nghiên cứu, bên ngoài):**
- `random_ideas/04-custom-tui-project.md` — bản năng giao diện agent một cấp thấp hơn (TUI/CLI cho AGY & Opencode).
- `random_ideas/05-projects-review-blindsight-snakeann-joblink.md` — cấu trúc modular theo tính năng của Joblink + bảo mật phân lớp như tiền lệ cụ thể.
- `iot_aiot/03-hardware-abstraction-layer-hal.md` — độc lập thông qua trừu tượng hoá ở cấp phần cứng.
- `ai_ml/05-mlops-lifecycle-and-deployment-architecture.md` — khả năng vận hành dài hạn của các phần AI trong chuỗi tác vụ.
- `git_github/11-infrastructure-as-code-and-devops-automation.md` — môi trường tái lập cho một hệ thống trường tồn.
- `books_summaries/agile_project_management/notes/07-agile-architecture-hal-refactoring-scaling.md` — kiến trúc tiến hoá & khung scaling.

**[ĐỀ XUẤT] bước tiếp theo:** backbone hướng sự kiện (→ doc 02) và khám phá bounded context cho cho vay tiêu dùng — KYC/onboarding, application, credit decisioning, contracting, disbursement, loan servicing, ledger, reporting (→ doc 03).

---

## 🤔 Câu hỏi Mở

- [x] **Lĩnh vực neo đậu (Gap #1):** công ty tài chính tiêu dùng, chuẩn Home Credit Việt Nam → *đã giải quyết 2026-09-24*. Lát cắt v0 = vay tiền mặt online.
- [ ] Những luồng con *cụ thể* nào của vay tiền mặt online tạo thành các bước saga của v0 (kênh giải ngân? quy tắc định giá/kỳ hạn? chính sách referral sang duyệt thủ công)?
- [ ] Product (vertical SaaS) hay nền tảng nội bộ? Điều đó đổi mô hình dữ liệu thế nào? *(hệ thống học tập single-tenant — xem lại nếu từng product hoá)*
- [ ] Đảm bảo phân phối nào làm back-end "vững chắc" đủ khi tiền di chuyển (giải ngân/trả nợ exactly-once, đối soát với adapter ngân hàng/e-wallet)?
- [ ] Quyền agent ánh xạ sang ranh giới thành phần thế nào (role = tập tool được phép)?
- [ ] Fallback khi thành phần AI sai (ví dụ: trích xuất tài liệu KYC) là gì — và log kiểm toán ghi lại nó ra sao?
- [ ] Trình điều khiển chất lượng nào thắng khi xung đột: độc lập, tốc độ giao hàng, hay chi phí?
- [ ] Kinh tế đơn vị mỗi tác vụ (tokens, độ trễ, chi phí/tác vụ) trên một khoản vay 1–40 triệu VND với SLA duyệt 3 phút là gì?
- [ ] Khi AI sai trong một hành động hệ trọng (phê duyệt/giải ngân), ai chịu trách nhiệm — và hành trình kiểm toán chứng minh chính xác điều gì?

---
*Chất lượng của một ý tưởng được đo bằng những mâu thuẫn mà nó giải quyết được, không phải bằng danh sách tính năng của nó.*