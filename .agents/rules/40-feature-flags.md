---
trigger: model_decision
description: Áp dụng khi thêm, sửa, xoá hoặc bật/tắt một feature (công việc liên quan đến phạm vi feature hoặc hành vi cửa sổ sản phẩm).
---

# Feature flags (no hardcode)

- Mọi feature bật/tắt bằng **cấu hình/DB**, không dùng `if/else` hardcode để làm feature xuất hiện/biến mất.
- Tên feature có nguồn đơn nhất (registry/cấu hình), không rải literal string.
- Khi thêm feature mới: kiểm tra trước với `/mvp-check` xem có nằm trong phạm vi MVP không (`docs/00-mvp-backlog.md`).
- Flag mặc định nên tắt (opt-in) cho tính năng chưa hoàn thiện; dữ liệu cấu hình nằm ở DB/config, không nằm trong code.