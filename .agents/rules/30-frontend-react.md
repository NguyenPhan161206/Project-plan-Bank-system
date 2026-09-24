---
trigger: glob
globs: "*.ts, *.tsx"
description: Quy tắc frontend React TS — không hardcode tên feature, dùng API contract.
---

# Quy tắc frontend React + TypeScript

1. **Tên feature:** không rải literal string trong UI/route — dùng nguồn đơn nhất (cấu hình/registry).
2. **Gọi backend** qua API contract tường minh (dto/schema chung với backend), không phát sinh endpoint ad-hoc.
3. **Trạng thái UI / máy trạng thái:** khai báo dạng dữ liệu (config/map), không hardcode chuỗi trạng thái rải rác trong component.
4. **Không hiển thị/hide feature bằng hardcode:** feature xuất hiện/biến mất qua feature flags từ backend, không dán `if` cố định trong code UI.
5. SPA phục vụ **cả khách vay và staff** trong 1 app — điều hướng theo role.