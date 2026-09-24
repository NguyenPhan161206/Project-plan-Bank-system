---
description: "Chạy vòng lập kế hoạch → triển khai → review: orchestrator giao planner phân việc theo backlog, fan-out cho worker, rồi đưa reviewer kiểm cổng."
agent: orchestrator
---

Bạn nhận mission sau và chạy đúng vòng điều phối của mình:

$ARGUMENTS

## Quy trình bắt buộc

1. Đọc nguồn sự thật (`AGENTS.md`, `docs/00-mvp-backlog.md`, `docs/03-modular-monolith-vs-microservices.md`). Nếu mission có nguy cơ ngoài MVP → báo và gợi ý `/mvp-check` trước.
2. Gọi `planner` bẻ mission thành task theo epic/sprint + module + worker phụ trách.
3. Fan-out cho đúng worker (`backend-worker`, `frontend-worker`, `explore` cho tra cứu) — song song khi độc lập.
4. Tổng hợp, kiểm tra tích hợp các mảnh (API contract chung, feature flags, tên feature registry).
5. Gọi `reviewer` rà soát theo module map / không hardcode / ranh giới import. Chỉ kết luận xong khi cổng review sạch.
6. Báo cáo tiếng Việt: đã làm gì, kết quả, rủi ro mở, bước tiếp theo theo backlog.