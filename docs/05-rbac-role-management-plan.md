# RBAC 2 tầng — Kế hoạch triển khai Quản trị Role (MVP)

**Ngày tạo:** 2026-09-24
**Cập nhật lần cuối:** 2026-09-24
**Tags:** `#MVP`, `#RBAC`, `#Identity`, `#AccessControl`, `#Backlog`, `#SprintPlan`
**Tham chiếu:** [00-mvp-backlog.md](00-mvp-backlog.md) (story S14), [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) (module map — bounded context `identity`/`access_control`)
**Trạng thái:** Đã chốt kế hoạch — sẵn sàng phân công khi vào sprint
**Độ khó:** Trung cấp

---

## 📌 Tóm tắt Chính (TL;DR)

- **Mục tiêu:** Không hardcode role trong code. Role + Permission là **dữ liệu bảng + seed data**. Admin quản trị role động qua màn hình/API.
- **2 tầng:**
  - **Tầng 1 (data model):** `role`, `permission`, `role_permission`, `user_role` + seed 3 role (`customer`/`staff`/`admin`) + 1 hàm kiểm tra quyền chung `has_permission(user, permission)`.
  - **Tầng 2 (luồng admin):** Admin tạo role mới (tên + chọn tập quyền), đổi tên, xóa role (chỉ role do admin tạo), gán quyền cho role, gán role cho user.
- **Ràng buộc đã chốt (bắt buộc):**
  1. Chỉ `admin` được quản trị role (tạo/đổi tên/xóa) và gán role cho user.
  2. 1 user = 1 role ở MVP (schema nhiều-nhiều vẫn để sẵn cho tương lai).
  3. **3 role seed** (`customer`/`staff`/`admin`) đánh dấu `is_system_role = true` — **không được xóa**, chỉ đổi tên/thêm-bớt quyền phụ.
  4. **Admin không được tự thu hồi quyền cốt lõi** của role Admin (chống tự-khóa-mình) — **ràng buộc ở tầng server**, không chỉ ẩn ở UI.
  5. Mọi thay đổi role/quyền/gán role phải **ghi audit**.
- **Customer:** MVP chỉ có **1 loại khách cá nhân** — không có doanh nghiệp/hộ kinh doanh ở MVP (OUT). "Loại khách hàng" là thuộc tính nghiệp vụ, **không trộn** vào hệ thống role.
- **Vị trí triển khai:** data model RBAC động dựng ngay ở **Sprint 0** (khung auth); luồng admin quản trị role ở **Sprint 3**; test E2E ở Sprint 4.

---

## 1. Lý do (Tại sao cần 2 tầng)

- Quy tắc AGENTS.md: quy tắc nghiệp vụ + cấu hình sản phẩm là **dữ liệu cấu hình/DB**, không phải hằng số rải trong code → role cũng phải là dữ liệu.
- Backlog yêu cầu phân quyền theo role nhưng có rủi ro "hardcode 3 role" ngay khi viết Sprint 0. Làm data model RBAC động từ đầu rẻ hơn làm lại sau này.
- Admin cần khả năng tạo role mới theo nghiệp vụ (ví dụ: role chuyên viên chỉ xem không duyệt) mà không cần sửa code.

## 2. Phạm vi (IN / OUT)

### ✅ IN (thuộc kế hoạch)
- Data model RBAC động + seed 3 role (`customer`/`staff`/`admin`) + catalog permission.
- `has_permission(user, permission)` + dependency check bảo vệ mọi endpoint.
- API quản trị role (admin-only): tạo/đổi tên/xóa role, gán quyền, gán role cho user.
- Ràng buộc server-side: không xóa role seed; admin không tự-thu-hồi quyền cốt lõi; audit mọi thay đổi.
- UI admin: màn hình quản trị role + gán role trên trang quản trị user.
- E2E: admin tạo role → gán cho staff → staff dùng đúng quyền; vi phạm bị chặn.

### ❌ OUT (không làm ở MVP)
- Không tạo role riêng cho "loại khách hàng" (cá nhân/doanh nghiệp) — customer chỉ 1 loại: **cá nhân**. Doanh nghiệp/hộ kinh doanh là OUT của sản phẩm.
- Không phân quyền theo phòng ban/department (ngoài backlog).
- Không permission mức widget/màn hình con (quá mịn — chỉ mức action).
- Không role hệ thống động do staff tự tạo (chỉ admin).

---

## 3. Thiết kế Dữ liệu (Tầng 1)

Mô hình RBAC trong bounded context `identity`/`access_control`:

```
user ── 1..1(user_role) ── role ── role_permission ── permission
```

| Bảng | Vai trò |
|---|---|
| `permission` | Catalog quyền (mức action): `application.approve`, `user.manage`, `audit.view`, `role.manage`... Đây là nguồn đơn nhất (seed). |
| `role` | Tên role + cờ `is_system_role` (3 role seed = true). |
| `role_permission` | Gán quyền cho role (nhiều-nhiều). |
| `user_role` | Gán role cho user — **MVP: 1 user = 1 role**; để sẵn many-to-many trong schema. |

Seed data: 3 role mặc định + catalog permission. Migration **idempotent** (chạy nhiều lần không trùng/double).

Hàm kiểm tra chung:

```python
def has_permission(user: User, permission: str) -> bool: ...
```

- Chỉ đọc từ DB (không so role name hardcode).
- Được dùng trong FastAPI dependency cho mọi endpoint bảo vệ (backend-worker triển khai, orchestrator sẽ đưa vào tầng 3/4).

> **Lưu ý thiết kế:** category of guard ở tầng 1 chỉ là nền *chip*; danh sách permission phải **khớp chính xác** tên giữa catalog seed và tên kiểm trong `has_permission` — dùng hằng số duy nhất, không rải literal thủ công (chi tiết ở Rủi ro).

---

## 4. Luồng Admin (Tầng 2)

| Hành động | Ai được phép | Ghi chú / ràng buộc |
|---|---|---|
| Tạo role mới (tên + chọn tập quyền) | Chỉ `admin` | Tự đặt tên role; chọn quyền từ catalog |
| Đổi tên role | Chỉ `admin` | Cho phép cả role seed |
| Xóa role | Chỉ `admin` | **Chặn xóa role `is_system_role=true`** (ở API, không chỉ ẩn nút UI) |
| Gán/thu hồi quyền cho role | Chỉ `admin` | Không cho thu hồi quyền cốt lõi của role Admin |
| Gán role cho user | Chỉ `admin` | MVP 1 user = 1 role |
| Audit | Tự động | Mọi hành động ở trên ghi nhật ký kiểm toán |

### Ràng buộc chống tự-khóa-mình (bắt buộc, ở server)
- Role Admin (`is_system_role`) có tập quyền cốt lõi tối thiểu không được thu hồi (ít nhất `user.manage` + `role.manage` + `audit.view`).
- Kiểm tra **trong service layer/API** trước khi cập nhật `role_permission`, không chỉ ẩn nút ở UI.

---

## 5. Phân rã Task (theo Sprint)

| ID | Task | Sprint | Loại | Phụ thuộc |
|---|---|---|---|---|
| **T1** | Bảng + seed RBAC động: `role`, `permission`, `role_permission`, `user_role` (1 user = 1 role ở MVP); seed 3 role + catalog permission + cờ `is_system_role` | 0 | migration + seed (backend) | — |
| **T2** | `has_permission(user, permission)` + FastAPI dependency check bảo vệ endpoint | 0–2 | backend | T1 |
| **T3** | Test: 3 role mặc định hoạt động đúng (đăng nhập → chạy đúng endpoint, chặn sai quyền) | 0 | test | T2 |
| **T4** | API quản trị role (admin-only): create/rename/delete, list/add/remove permission, assign role → user | 3 | backend | T1, T2 |
| **T5** | Ràng buộc server-side bắt buộc: chặn xóa role seed; chặn admin tự thu hồi quyền cốt lõi của role Admin; ghi audit mọi thay đổi | 3 | backend | T4 |
| **T6** | UI admin: màn hình Quản trị role (danh sách, tạo role: tên + checkbox tập quyền, đổi tên, xóa — **ẩn nút xóa** với role seed) | 3 | frontend | T4 |
| **T7** | UI admin: gán role cho user trên trang quản trị user | 3 | frontend | T4 |
| **T8** | E2E Playwright: admin tạo role mới → gán cho staff mới → staff đăng nhập dùng đúng quyền; vi phạm bị chặn ở API | 4 | test | T5–T7 |

**Thứ tự ưu tiên:** T1→T2→T3 ngay ở **Sprint 0** (nền data model RBAC động — rẻ hơn làm lại ở Sprint 3). T4–T7 ở **Sprint 3**, T8 ở Sprint 4 (củng cố).

---

## 6. Đề xuất (Quyết định mở / khuyến nghị)

1. **Permission mức hạt:** đề xuất mức **action** (`application.approve`, `user.manage`, `audit.view`, `role.manage`...) — đủ cho MVP; không đi sâu theo widget (quá mịn, phình scope).
2. **1 user = 1 role** ở MVP: đơn giản, đủ cho 3 loại user; many-to-many chỉ giữ sẵn trong schema cho tương lai.
3. **Dựng data model RBAC ở Sprint 0**: khung auth không hardcode vai trò ngay từ đầu; tiết kiệm refactor so với làm ở Sprint 3.
4. **Role seed chỉ-chỉnh-phụ:** 3 role seed được đổi tên/thêm-bớt quyền phụ nhưng không xóa — giữ tính toàn vẹn ("mọi user phải có role").
5. **Chỉ admin quản trị role:** giữ quyền tối thiểu; không mở cho staff ở MVP.

## 7. Rủi ro & Cách xử lý

| # | Rủi ro | Cách xử lý |
|---|---|---|
| R1 | **Chống tự-khóa-mình chỉ ở UI** → admin gọi API thẳng bỏ quyền `user.manage` của role Admin | Kiểm **ở service layer/API** (T5), không phụ thuộc UI |
| R2 | **Xóa role seed** dù UI ẩn nút → API `delete_role` vẫn chấp nhận | API từ chối mọi role `is_system_role=true` (400/403) |
| R3 | **Xung đột tên permission** giữa catalog seed và tên kiểm trong `has_permission` | Dùng hằng số duy nhất (một nguồn sự thật), không rải literal thủ công; test so khớp |
| R4 | **Seed vs migration order:** bảng role/permission phải seed trước khi app khởi động đọc | Migration idempotent; seed chạy trong cùng luồng khởi tạo DB |
| R5 | **Role bị xóa khi vẫn có user gán** → user mất quyền / sample rỗng | API chặn xóa role đang được gán (hoặc yêu cầu gỡ user trước) |
| R6 | **Thiếu audit** → không truy vết ai đổi role | Mọi mutation role/permission gán đều ghi audit (T5) |

---

## 8. Ghi chú Bổ sung — Phân loại Khách hàng

- MVP chỉ có **1 loại customer: khách cá nhân** (nhập thông tin cá nhân + CCCD tay — backlog S3).
- **Không** có tài khoản doanh nghiệp/hộ kinh doanh ở MVP — OUT và không nằm trong hệ thống role.
- "Loại khách hàng" (nếu sau này cần: đi làm hưởng lương, tự kinh doanh, hưu trí...) là **dữ liệu cấu hình nghiệp vụ** cho scoring/rule duyệt, **không phải role** — không trộn trục quyền truy cập với trục phân khúc khách hàng.

---

## 🔗 Chủ đề Liên quan

- [00-mvp-backlog.md](00-mvp-backlog.md) — story S14 (Admin quản trị role) + scope IN/OUT.
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — bounded context `identity`/`access_control` (tài khoản user, phân quyền theo role, chính sách quyền).

## 🤔 Câu hỏi Mở

- [ ] Verified permission names: danh sách action permission seed nào là tối thiểu cho MVP (application.*, user.*, audit.*, role.*)?
- [ ] Role seed có được phép "đổi tên" qua UI hay chỉ được chỉnh quyền phụ (4.2)?
- [ ] Khi xóa role (R5): chặn cứng hay cho phép xóa với cảnh báo + gỡ user?