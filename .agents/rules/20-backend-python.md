---
trigger: glob
globs: "*.py"
description: Quy tắc backend Python — feature-based structure, không hardcode quy tắc nghiệp vụ, cấu hình, state machine, feature flags.
---

# Quy tắc backend Python (KHÔNG hardcode)

1. **Feature-based structure:** layout theo bounded context (`app/application/`, `app/credit_decisioning/`, `app/loan_servicing/`, `app/shared_kernel/`...). Mỗi module sở hữu dữ liệu riêng; chỉ giao tiếp qua `public_api` khai báo tường minh. Không tổ chức theo layer (`controllers/`, `models/`, `services/`).
2. **Feature flags:** feature bật/tắt bằng **cấu hình/DB**, không `if/else` hardcode để làm feature xuất hiện/biến mất.
3. **Quy tắc nghiệp vụ + cấu hình sản phẩm** (tuổi, hạn mức, dư nợ tối đa, số kỳ hạn, lãi suất...) là **dữ liệu cấu hình/DB**, không phải hằng số rải trong code.
4. **Trạng thái/máy trạng thái:** danh sách state + mapping sự kiện → hành động khai báo **dạng dữ liệu**, không hardcode.
5. **Tên feature:** từ nguồn đơn nhất (cấu hình/registry), không rải literal string trong UI/route.