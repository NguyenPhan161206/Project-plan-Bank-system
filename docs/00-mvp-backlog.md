# MVP Backlog — Hệ thống Vay Tiền Mặt Online (thuần feature, chưa AI)

**Ngày tạo:** 2026-09-24
**Cập nhật lần cuối:** 2026-09-24
**Tags:** `#MVP`, `#Agile`, `#Backlog`, `#ProductBacklog`, `#Roadmap`
**Tham chiếu:** [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md), [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md), [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md)
**Trạng thái:** Đã chốt — sẵn sàng Sprint 0
**Độ khó:** Cơ bản
**Dự án:** Hệ thống Tài chính Tiêu dùng Modular — giai đoạn MVP **thuần tính năng, không có AI** (website + backend + feature). Backend cố định bằng **Python**.

---

## 📌 Tóm tắt Chính (TL;DR)

- **MVP = không AI.** Chặn tay toàn bộ AI/ML: không trích xuất giấy tờ tự động, không scoring ML. Duyệt hồ sơ bằng **quy tắc có sẵn (rule-based)**.
- Phục vụ **cả hai phía**: khách vay nộp hồ sơ qua web + bàn trực nội bộ cho nhân viên duyệt/tra cứu.
- Có **auth đầy đủ 2 phía** với 3 role mặc định: `customer`, `staff`, `admin` (RBAC động: role là dữ liệu seed, không hardcode — có thể tạo thêm vai trong tương lai). Admin quản trị tài khoản staff + tra cứu nhật ký kiểm toán, nhưng **không được tự duyệt hồ sơ** (ngăn xung đột vai trò).
- **Giải ngân** qua một **cổng e-wallet/ngân hàng giả lập** (pending/success/fail + retry thủ công), chưa tích hợp thật.
- **Backend: Python (FastAPI).** Giữ hình dạng module/public API tương thích với pseudocode docs 02–04 để *nạp backbone (event bus/saga/agent)* ở giai đoạn sau — không implement bus/MCP/agent ngay ở MVP; sẵn sàng cho AI/ML tương lai.
- Bắt đầu theo Agile: **Sprint 0 setup → Sprint 1 lát cắt dọc mỏng nhất →** mới tới các lớp bàn trực, giải ngân, củng cố.
- Chưa có **event bus / saga / Kafka** ở MVP — backend gọi module trong-process; backbone chỉ nạp khi có ≥2 luồng liên kết thật chạy song song.

---

## 1. MVP Scope (IN / OUT)

### ✅ IN (phải có để demo được)
| Hạng mục | Mô tả |
|----------|-------|
| Auth 2 phía | Khách đăng ký/đăng nhập; nhân viên đăng nhập; phân quyền theo role |
| Admin | Quản trị user staff (tạo, khóa/mở khóa, reset mật khẩu); **quản trị role** (tạo/đổi tên/xóa role, gán quyền, gán role cho user); tra cứu nhật ký kiểm toán; **không** được tự duyệt hồ sơ (ngăn xung đột vai trò) |
| 1 sản phẩm | Chỉ **vay tiền mặt online** (số tiền + kỳ hạn cố định) |
| Nộp hồ sơ | Khách mở form, nhập thông tin CCCD **bằng tay** (không trích xuất AI) |
| Quy tắc duyệt | Rule-based: tuổi, hạn mức tối đa, dư nợ tối đa, trùng hồ sơ đang xử lý → auto-duyệt/từ chối/refer |
| Bàn trực nội bộ | Danh sách hồ sơ theo trạng thái, xem chi tiết, duyệt/từ chối kèm lý do |
| Hợp đồng | Sinh hợp đồng vay, khách xem + xác nhận (e-sign giả = tích ô) |
| Giải ngân | Cổng giả lập (pending/success/fail), retry thủ công, thất bại sinh cờ |
| Lịch trả nợ | Sau giải ngân tự sinh lịch trả nợ (gốc + lãi) |
| Ghi nhận trả nợ | Ops ghi nhận khoản thanh toán tay (đóng vòng demo) |
| Audit-basic | Ai duyệt/từ chối hồ sơ, khi nào, lý do; admin tra cứu theo user/hồ sơ/hành động |
| Trạng thái nhất quán | `submitted → decision_made → contract_sent → signed → disbursed → repaying` (+ nhánh `pending` cho hồ sơ refer/duyệt tay, `rejected`) |

### ❌ OUT (chặn tay, KHÔNG làm ở MVP)
- AI/ML mọi dạng (trích xuất giấy tờ, scoring ML, agent, MCP)
- Tích hợp ngân hàng/e-wallet thật, SMS OTP thật
- Collections, merchant trả góp, đa sản phẩm
- Báo cáo SBV/CIC, dashboard thống kê nâng cao, IFRS 9
- Event bus / saga / Kafka / schema registry
- Webhook tích hợp bên ngoài

---

## 2. Quyết định Tech Stack (đã chốt)

| Lớp | Lựa chọn | Lý do |
|-----|----------|-------|
| Backend | **Python 3.12 + FastAPI** | Khớp pseudocode doc 02–04; import-linter dùng được; hệ sinh thái AI tương lai là Python |
| DB | **PostgreSQL 1 instance, schema riêng/bounded context** | Sẵn sàng modular monolith giai đoạn sau |
| Auth | **OAuth2 + JWT, bcrypt, role-based** | OTP SMS thật là OUT (cần provider + phí) |
| Frontend | **React + Vite + TypeScript** (SPA cho khách + staff trong 1 app) | UI tương tác đa bước, trạng thái realtime |
| Liên kết backend | **Trong-process** (module-call + DB), state máy đơn giản trong DB | Trì hoãn event bus tới khi cần thiết |
| Test/CI | pytest, Playwright (1 luồng E2E), GitHub Actions | Lint + import-linter + pytest |

> Ghi chú: JavaScript được dùng **chỉ ở frontend** (React TS). Toàn bộ backend là Python.

---

## 3. Backlog ưu tiên (MoSCoW + epic)

Mỗi story có tiêu chí chấp nhận riêng; ưu tiên theo giá trị + rủi ro.

| # | Epic | Story | Mức | Sprint |
|---|------|-------|-----|--------|
| S1 | E1 Auth | Khách đăng ký, đăng nhập, đăng xuất, xem profile | Must | 1 |
| S2 | E1 Auth | Nhân viên đăng nhập theo role (`staff`); phân tách `admin` | Must | 2 |
| S3 | E2 Nộp hồ sơ | Khách mở form vay tiền mặt (số tiền, kỳ hạn), nhập thông tin CCCD tay | Must | 1 |
| S4 | E3 Duyệt | Rule auto-duyệt/từ chối (tuổi, hạn mức, dư nợ, trùng hồ sơ) | Must | 1 |
| S5 | E4 Bàn trực | Staff thấy hàng chờ, xem chi tiết, duyệt/từ chối kèm lý do, audit hiển thị | Must | 2 |
| S6 | E4 Bàn trực | Chế độ `pending` cho hồ sơ cần duyệt thủ công (refer) | Should | 2 |
| S7 | E5 Hợp đồng | Sinh hợp đồng, khách xem + xác nhận (e-sign giả = tích ô) | Must | 3 |
| S8 | E6 Giải ngân | Cổng giả lập (pending/success/fail), retry thủ công, thất bại sinh cờ | Must | 3 |
| S9 | E7 Trả nợ | Sau giải ngân sinh lịch trả nợ; ops ghi nhận 1 khoản thanh toán tay | Should | 4 |
| S10 | E8 Củng cố | Trạng thái nhất quán, chống trùng hồ sơ, test E2E luồng đầy đủ | Should | 4 |
| S11 | E9 Admin | Admin quản trị user: tạo staff, khóa/mở khóa, reset mật khẩu | Must | 2 |
| S12 | E9 Admin | Admin tra cứu nhật ký kiểm toán (filter theo user/hồ sơ/hành động) | Must | 3 |
| S13 | E9 Admin | Admin KHÔNG được tự duyệt hồ sơ (tách vai trò quản trị vs vận hành) | Should | 3 |
| S14 | E9 Admin | Admin quản trị role: tạo role mới (tên + chọn tập quyền), đổi tên, xóa role (chỉ role do admin tạo), gán quyền, gán role cho user; **chỉ admin** thao tác; 3 role seed (`customer`/`staff`/`admin`) không được xóa; **admin không được tự thu hồi quyền cốt lõi của role Admin (chống tự-khóa-mình)**; **policy bất biến: user sở hữu role có `role.manage` thì quyền `application.*` bị triệt tiêu (chống tự-phóng-đại-quyền vòng qua rào duyệt)** | Should | 0 (Tầng 1) / 3 (Tầng 2, hoãn) |

---

## 4. Lịch trình Sprint (1–2 tuần/sprint, demo cuối mỗi sprint)

- **Sprint 0 — Setup (trước code):** backlog này thành nguồn chuẩn; skeleton repo (monorepo); CI (lint + import-linter + pytest); khung auth (3 role mặc định `customer`, `staff`, `admin`); **nền RBAC Tầng 1: data model role/permission là dữ liệu seed + `has_permission` (không hardcode role trong code)**; xác nhận tech stack.
- **Sprint 1 — Lát cắt dọc mỏng nhất:** `đăng nhập khách → tạo hồ sơ → rule tự duyệt → hiện trạng thái`. **Chưa event bus, chưa saga, chưa gate cổng.** Backend gọi hàm trong-process. Mục tiêu: có thứ demo được.
- **Sprint 2 — Bàn trực + Minh bạch:** đăng nhập staff, hàng chờ, duyệt/từ chối, audit-basic, chế độ `pending`; **admin quản trị user** (tạo staff, khóa/mở khóa, reset mật khẩu).
- **Sprint 3 — Giải ngân + hợp đồng:** e-sign giả, cổng giả lập 3 trạng thái + retry, sinh lịch trả nợ; **tra cứu nhật ký kiểm toán cho admin**; chặn admin tự duyệt hồ sơ; **Tầng 2 quản trị role** (tạo/đổi tên/xóa role, gán quyền, gán role cho user, ràng buộc chống tự-khóa-mình + chống tự-phóng-đại-quyền) — **hoãn nếu chưa có role thứ 4 có tên thật** (ví dụ tách `ops` để SoD), giữ nguyên Tầng 1 RBAC động từ Sprint 0.
- **Sprint 4 — Củng cố & đóng vòng:** ghi nhận trả nợ thủ công, bằng chứng "vững chắc" (trạng thái nhất quán, test); cân nhắc nạp event backbone cho các luồng thật sự chạy song song.

### Định nghĩa "Done" cho mỗi sprint
- Một kịch bản demo chạy được end-to-end trước khán giả (khách hoặc staff).
- Code trên nhánh chính, CI xanh (lint + type + test).
- Audit-basic của hành động chính được ghi nhận.

---

## 5. Nơi bắt đầu trước tiên (Câu trả lời Agile)

> **Bắt đầu bằng Sprint 0**, không phải code: chốt tech stack (đã xong — Python backend), ghi MVP scope + backlog thành file (chính là doc này), dựng skeleton repo với CI. Rồi **Sprint 1 = lát cắt dọc**: một luồng khách điền → hồ sơ có trạng thái → hiển thị, phủ backend tối giản, **không backbone trước**.

Nguyên tắc giữ trong suốt MVP:
1. **Lát cắt dọc mỏng nhất**, không build theo lớp ngang (đừng "làm DB trước, rồi API, rồi UI").
2. **Working software là thước đo tiến độ** — mỗi sprint có thứ demo được.
3. **De-risk thứ đắt nhất trước** — ở phiên bản thuần feature, đó là sự nhất quán trạng thái khi nhiều hồ sơ chạy, không phải AI.
4. **Không xây cầu trước khi có sông** — event bus/saga chỉ tới khi ≥2 luồng liên kết thật chạy song song.

---

## 📝 Hành trình

- **Tại sao:** Ý tưởng gốc (doc 01) nhấn mạnh đơn vị kinh tế AI chưa đủ chín; học hỏi theo chu kỳ nhỏ. MVP thuần feature là bước chứng minh sản phẩm/quy trình trước, bơm AI sau.
- **Khoảnh khắc à-ha:** "Website" và "back-end vững chắc" không mâu thuẫn ở giai đoạn này — MVP dùng backend trong-process; backbone là thứ *nạp* khi có bằng chứng cần, không phải thứ xây ngay ngày một.

## 🔗 Chủ đề Liên quan

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — lý do MVP thuần feature (rủi ro kinh tế đơn vị AI, chu kỳ học hỏi nhỏ).
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — 8 bounded context lõi vòng đời khoản vay là đích v1 (+ `identity`/`access_control` là context xuyên suốt, không tính vào 8; `collections`/`merchant` sau v1); MVP triển khai một lát của chúng, định hình module ngay từ đầu.

## 🤔 Câu hỏi Mở

- [ ] Mức xác thực CCCD ở MVP: nhập tay số + tự in thành ảnh xem trước, hay cần nhập đầy đủ (họ tên, ngày sinh, quê quán…)?
- [ ] Website có cần hiển thị bằng tiếng Việt với chữ số/lãi minh hoạ trước khi duyệt (theo Nghị định liên quan) hay chỉ hợp đồng?
- [ ] Số kỳ hạn / khoản vay hỗ trợ ở MVP: 1 combo cố định hay vài tuỳ chọn cho demo?

---
*MVP là hệ thống nhỏ nhất cho phép bạn học điều người dùng thực sự cần — không phải hệ thống nhỏ nhất mà bạn có thể phát hành.*