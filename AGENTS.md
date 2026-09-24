# AGENTS.md — Project-plan-Bank-system

## Bối cảnh dự án

Hệ thống **Tài chính Tiêu dùng Modular** — chuẩn **Home Credit Việt Nam** (công ty tài chính có phép, giám sát bởi Ngân hàng Nhà nước). Hiện tại dự án ở giai đoạn **kế hoạch → chuẩn bị Sprint 0 của MVP** (chưa có code).

## Nguồn sự thật (đọc trước khi làm)

| File | Vai trò |
|---|---|
| `docs/00-mvp-backlog.md` | **Backlog MVP + scope IN/OUT + lịch trình sprint** — nguồn hàng đầu cho mọi quyết định feature |
| `docs/01-project-idea-and-evaluation.md` | Quyết định kiến trúc đã chốt (Decisions Log) + rủi ro |
| `docs/03-modular-monolith-vs-microservices.md` | Banner đồng module map + feature-based layout |
| `docs/02`, `docs/04` | Thiết kế event/saga và MCP/agent (tương lai, không áp dụng ở MVP) |

## Làn ranh MVP (bất biến)

Mọi đề xuất thay đổi phải nằm trong `docs/00-mvp-backlog.md`. Không được làm khi chưa có yêu cầu rõ ràng:

- **No AI/ML** (không trích xuất giấy tờ tự động, không scoring ML, không agent, không MCP)
- **No event bus / saga / Kafka / schema registry** (backend trong-process, state máy trong DB)
- **No tích hợp thật** (cổng e-wallet/ngân hàng, SMS OTP đều giả lập)
- **No thêm feature ngoài backlog**

## Tech stack (đã chốt)

- Backend: **Python 3.12 + FastAPI**
- DB: **PostgreSQL** (1 instance, schema riêng cho mỗi bounded context)
- Auth: OAuth2 + JWT, bcrypt, role-based (`customer`, `staff`, `admin`)
- Frontend: **React + Vite + TypeScript** (SPA, khách + staff trong 1 app)
- Test/CI: pytest, Playwright (E2E), import-linter, GitHub Actions

## Quy tắc mã nguồn (bắt buộc — KHÔNG hardcode)

1. **Feature-based structure:** layout theo bounded context (`app/application/`, `app/credit_decisioning/`, `app/loan_servicing/`, `app/shared_kernel/`...). Mỗi module sở hữu dữ liệu riêng; chỉ giao tiếp qua `public_api` khai báo tường minh. Không tổ chức theo layer (`controllers/`, `models/`, `services/`).
2. **Feature flags:** feature bật/tắt bằng **cấu hình/DB**, không `if/else` hardcode để làm feature xuất hiện/biến mất.
3. **Quy tắc nghiệp vụ + cấu hình sản phẩm** (tuổi, hạn mức, dư nợ tối đa, số kỳ hạn, lãi suất...) là **dữ liệu cấu hình/DB**, không phải hằng số rải trong code.
4. **Trạng thái/máy trạng thái:** danh sách state + mapping sự kiện → hành động khai báo **dạng dữ liệu**, không hardcode.
5. **Tên feature:** từ nguồn đơn nhất (cấu hình/registry), không rải literal string trong UI/route.

## Quy ước giao tiếp

- Phản hồi người dùng bằng **tiếng Việt**.
- Code comments, tên biến/hàm, commit message bằng **tiếng Anh**.
- Không thêm comment vào code trừ khi được yêu cầu.
- Không đặt model trong opencode.json (dùng model môi trường).

## Hệ thống agent (mô hình manager + workers)

Dự án dùng cấu hình OpenCode tự dựng (`.opencode/agent/`, chủ yếu `permission.task`), **không** dùng plugin ngoài:

| Agent | Kiểu | Vai trò |
|---|---|---|
| `orchestrator` | primary (mặc định) | **Manager** — nhận mission, kiểm tra MVP scope, giao task, tổng hợp, vặn cổng review |
| `planner` | subagent | Worker — đọc backlog, bẻ mission thành task theo epic/sprint/module (chỉ đọc) |
| `backend-worker` | subagent | Worker — code Python/FastAPI (chỉ sửa file backend) |
| `frontend-worker` | subagent | Worker — code React/TS (chỉ sửa file frontend) |
| `reviewer` | subagent | Worker — cổng rà soát module map / no-hardcode / ranh giới import (chỉ đọc) |

- Quyền `task` của mọi worker đều `deny` (không gọi được subagent khác); chỉ `orchestrator` được fan-out.
- Lệnh kích hoạt vòng lập kế hoạch → triển khai → review: `/orchestrate <mission>`.
- Worker chỉ được sửa file thuộc lớp của mình (globs trong frontmatter); muốn thay đổi chia sẻ/rộng phải thông qua orchestrator.

## Kiểm tra

Chưa có code → chưa có lệnh lint/type/test. Khi Sprint 0 dựng skeleton, bổ sung lệnh `lint`, `typecheck`, `test` vào mục này.