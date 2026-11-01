<!-- SPDX-License-Identifier: Apache-2.0 -->
# Mô hình dữ liệu

Dữ liệu của ứng dụng nằm trong schema `api` của PostgreSQL. PostgREST tự sinh API cho mọi bảng và hàm trong schema này (`/data/*` qua cổng).
Tạo bởi `app/sql/*.sql`, chạy lại mỗi lần `docker compose up` và an toàn khi chạy nhiều lần.

Quyền dựa vào token Keycloak đã được PostgREST kiểm chữ ký. Các hàm trợ giúp:

| Hàm | Trả về |
|---|---|
| `api.jwt_user()` | `preferred_username` của người gọi |
| `api.jwt_roles()` | Mảng vai trò realm của người gọi |
| `api.jwt_actor()` | Tên người gọi, hoặc client id (`azp`) với tài khoản dịch vụ như agent AI |

## `api.requests` — hồ sơ đã gửi

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | bigint | Khóa chính, tự tăng |
| `owner` | text | Mặc định là người gọi |
| `process_instance_id` | text | Duy nhất; nối với process instance của Flowable |
| `process_key` | text | Ví dụ `ban-hang-lien-phong`, `de-nghi-mua-sam` |
| `title` | text | |
| `created_at` | timestamptz | |

Row-level security: mỗi người chỉ đọc và ghi hồ sơ có `owner` là chính mình.

## `api.budgets` — ngân sách theo dự án

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `project` | text | Khóa chính, mã dự án |
| `amount` | bigint | Tổng ngân sách được duyệt (VND) |
| `spent` | bigint | Đã chi (VND); còn lại = `amount - spent` |

Mọi người đọc được. Dữ liệu mẫu: `X` (200 triệu, đã chi 170 triệu), `Y` (500 triệu, đã chi 100 triệu).

## `api.rule_versions` — luật có phiên bản

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | bigint | Khóa chính |
| `rule_key` | text | Ví dụ `tran_chiet_khau` |
| `label` | text | Tên hiển thị |
| `value` | bigint | 0–100 |
| `version` | int | Có giá trị khi đã duyệt |
| `status` | text | `draft`, `active`, `retired`, `discarded` |
| `note` | text | |
| `created_by`, `approved_by` | text | |
| `created_at`, `approved_at` | timestamptz | |

- Mỗi `rule_key` có tối đa một dòng `active` (chỉ mục duy nhất `rule_one_active`).
- Mọi người đọc được. Chỉ trưởng phòng và Giám đốc thêm được bản nháp. Không ai `UPDATE`, `DELETE` trực tiếp.
- `api.approve_rule(draft_id)`: chỉ Giám đốc gọi được. Ngừng phiên bản đang chạy, đưa bản nháp thành phiên bản kế tiếp.
- Luật ban đầu: `tran_chiet_khau` = 15, phiên bản 1.

## `api.cases` — sổ hồ sơ

Mỗi hồ sơ liên phòng một dòng: làn lúc mở, phiên bản luật, kết quả. Flowable chạy quy trình; bảng này giữ dữ liệu để thống kê trên bản đồ vận hành và mô phỏng tác động khi sửa luật.

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | bigint | Khóa chính, tự tăng |
| `case_ref` | text | Duy nhất; process instance id của Flowable (`MAU-xxxx` với dữ liệu minh họa) |
| `process_key` | text | Ví dụ `ban-hang-lien-phong` |
| `title` | text | |
| `customer` | text | Tên khách hàng |
| `mst` | text | Kiểm bằng `api.mst_valid(mst)` |
| `amount` | bigint | Giá trị trước chiết khấu (VND) = số lượng × đơn giá (`>= 0`) |
| `discount` | int | Chiết khấu 0–100 (%) |
| `rule_version`, `rule_value` | int | Phiên bản và giá trị luật lúc mở hồ sơ |
| `lane` | text | Làn lúc mở: `vang` = thiếu MST hoặc PO, `do` = vượt trần chiết khấu, `xanh` = tự đi |
| `opened_by` | text | Mặc định `api.jwt_user()` |
| `opened_at`, `closed_at` | timestamptz | Lúc mở và lúc đóng |
| `outcome` | text | `hoan-tat` hoặc `tu-choi` |
| `source` | text | `live` = hồ sơ thật trên hệ thống; `seed` = dữ liệu minh họa |

- Chính sách RLS: người mở và quản lý (`truong-phong`, `giam-doc`) đọc; chỉ người mở thêm dòng `live` đang mở; không ai `UPDATE`, `DELETE` trực tiếp; đóng qua `api.close_case`.
- `api.mst_valid(mst)`: kiểm mã số thuế Việt Nam 10 chữ số (chữ số thứ 10 là chữ số kiểm tra theo trọng số `31, 29, 23, 19, 17, 13, 7, 5, 3` mod 11), tùy chọn chi nhánh `-NNN`.
- `api.close_case(ref, result)`: chỉ vai trò kết thúc hồ sơ (`cskh`, `giam-doc`) gọi được; đặt `closed_at = now()` và `outcome` (`hoan-tat` hoặc `tu-choi`).
- `api.rule_impact(new_value, window_days DEFAULT 30)`: trả về `(cases, red_now, red_new)` của quy trình bán hàng trong `window_days` ngày qua nếu áp trần mới `new_value`.
- `api.case_stats(window_days DEFAULT 30)`: trả về `(opened, closed, green_pct, avg_hours)` của quy trình bán hàng trong `window_days` ngày qua.

## `api.audit_log` — nhật ký chuỗi băm

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | bigint | Khóa chính |
| `at` | timestamptz | |
| `actor` | text | Mặc định `api.jwt_actor()` |
| `action` | text | `case.start`, `task.complete`, `ai.review`, `rule.draft`, `rule.approve` |
| `case_ref` | text | Hồ sơ liên quan |
| `detail` | jsonb | |
| `prev_hash`, `hash` | text | Do trigger điền |

- Trigger `seal` (trước `INSERT`): khóa bảng theo giao dịch, lấy `hash` của dòng trước (`genesis` cho dòng đầu), tính `hash` = SHA-256 của `prev_hash | id | at (UTC) | actor | action | case_ref | detail`.
- Mỗi người chỉ ghi được dòng có `actor` là chính mình. Chỉ trưởng phòng và Giám đốc đọc được. Không ai `UPDATE`, `DELETE`.
- `api.verify_audit()`: trả về `id` của dòng đầu tiên bị phá chuỗi, hoặc `NULL` khi chuỗi nguyên vẹn.

## Dữ liệu ngoài schema `api`

| Nơi | Giữ gì |
|---|---|
| CSDL `keycloak` | Người dùng, vai trò, client (Keycloak quản lý) |
| CSDL `flowable` | Định nghĩa quy trình, process instance, task, biến (Flowable quản lý) |
| SeaweedFS | Tệp đính kèm |
