---
trigger: always_on
description: Bối cảnh dự án Hệ thống Tài chính Tiêu dùng Modular và nguồn sự thật.
---

# Bối cảnh dự án

Hệ thống **Tài chính Tiêu dùng Modular** — chuẩn **Home Credit Việt Nam** (công ty tài chính có phép, giám sát bởi Ngân hàng Nhà nước). Giai đoạn hiện tại: **kế hoạch → chuẩn bị Sprint 0 của MVP** (chưa có code).

## Nguồn sự thật (đọc trước khi làm)

| File | Vai trò |
|---|---|
| `docs/00-mvp-backlog.md` | **Backlog MVP + scope IN/OUT + lịch trình sprint** — nguồn hàng đầu cho mọi quyết định feature |
| `docs/01-project-idea-and-evaluation.md` | Quyết định kiến trúc đã chốt (Decisions Log) + rủi ro |
| `docs/03-modular-monolith-vs-microservices.md` | Banner đồng module map + feature-based layout |
| `docs/02`, `docs/04` | Thiết kế event/saga và MCP/agent (tương lai, không áp dụng ở MVP) |

## Tech stack (đã chốt)

- Backend: **Python 3.12 + FastAPI**
- DB: **PostgreSQL** (1 instance, schema riêng cho mỗi bounded context)
- Auth: OAuth2 + JWT, bcrypt, role-based (`customer`, `staff`, `admin`)
- Frontend: **React + Vite + TypeScript** (SPA, khách + staff trong 1 app)
- Test/CI: pytest, Playwright (E2E), import-linter, GitHub Actions

## Quy ước giao tiếp

- Phản hồi người dùng bằng **tiếng Việt**.
- Code comments, tên biến/hàm, commit message bằng **tiếng Anh**.
- Không thêm comment vào code trừ khi được yêu cầu.