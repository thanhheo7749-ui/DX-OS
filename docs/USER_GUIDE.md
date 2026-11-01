<!-- SPDX-License-Identifier: Apache-2.0 -->
# Hướng dẫn sử dụng

Mở http://localhost:8000 và đăng nhập. Mật khẩu mọi tài khoản thử: giá trị `DEMO_USER_PASSWORD` trong tệp `.env`.

## Tài khoản thử

| User | Vai trò | Thấy gì thêm |
|---|---|---|
| `sales1` | Sales | Tạo đơn hàng |
| `ketoan` | Kế toán | Xác nhận thanh toán, xác nhận đề nghị mua sắm |
| `kho1` | Kho | Xuất hàng |
| `giaovan1` | Giao vận | Giao hàng |
| `cskh1` | CSKH | Chăm sóc sau bán |
| `truongphong` | Trưởng phòng | Bản đồ vận hành, Nhật ký, đề xuất sửa luật, duyệt đề nghị mua sắm |
| `giamdoc` | Giám đốc | Duyệt làn Đỏ, duyệt luật, Bản đồ vận hành, Nhật ký |
| `nhanvien1`, `nhanvien2` | Nhân viên | Gửi đề nghị mua sắm |

## Các màn hình

| Màn | Ai thấy | Dùng để |
|---|---|---|
| **Việc của tôi** | Mọi người | Việc đang chờ chính mình, theo làn Xanh, Vàng, Đỏ |
| **Tạo mới** | Mọi người | Chọn quy trình, điền form (hoặc mô tả bằng lời để AI điền giúp), gửi |
| **Hồ sơ tôi tạo** | Mọi người | Theo dõi hồ sơ mình đã gửi, nhận xét của AI |
| **Bản đồ vận hành** | Trưởng phòng, Giám đốc | Hồ sơ đang mở gom theo bước đang chờ |
| **Luật** | Mọi người xem; trưởng phòng, Giám đốc đề xuất; Giám đốc duyệt | Lịch sử phiên bản trần chiết khấu, bản nháp |
| **Nhật ký** | Trưởng phòng, Giám đốc | 50 thao tác mới nhất, kết quả kiểm chuỗi băm |

## Kịch bản 1: Đơn hàng liên phòng

1. `sales1` → Tạo mới → **Đơn hàng liên phòng**. Bỏ trống mã số thuế rồi gửi: luật **L-02** chặn và trả đơn về.
2. Gửi lại đơn đủ thông tin, chiết khấu **trong** trần: đơn đi thẳng sang Kế toán, không ai phải duyệt.
3. Gửi một đơn chiết khấu **vượt** trần: đơn vào **làn Đỏ**. AI trực ca chuẩn bị phân tích, rồi việc hiện trong Việc của tôi của `giamdoc`.
4. `giamdoc` đọc phân tích của AI, quyết định duyệt hoặc từ chối.
5. Đăng nhập lần lượt `ketoan`, `kho1`, `giaovan1`, `cskh1` để đưa đơn đi hết vòng.
6. `truongphong` hoặc `giamdoc` mở **Bản đồ vận hành** để xem hồ sơ đang ở bước nào, mở **Nhật ký** để xem từng thao tác.

## Kịch bản 2: Sửa luật có phiên bản

1. (Chạy `./scripts/seed-demo.sh` trước để có dữ liệu lịch sử.) `truongphong` → **Luật** → đề xuất trần chiết khấu mới (số nguyên từ 0 đến 50). Ngay dưới ô nhập hiện tác động trên 30 ngày qua: bao nhiêu đơn sẽ vào làn Đỏ với trần mới. Bản nháp được lưu, chưa có hiệu lực.
2. `giamdoc` → **Luật** → duyệt bản nháp. Bản nháp thành phiên bản mới, phiên bản cũ ngừng.
3. Đơn tạo **trước** khi duyệt vẫn dùng trần cũ; đơn tạo **sau** dùng trần mới.

## Kịch bản 3: Đề nghị mua sắm có AI

1. `nhanvien1` → Tạo mới → **Đề nghị mua sắm** → mô tả bằng lời, ví dụ "mua 2 màn hình cho dự án X, khoảng 12 triệu". AI điền form. Kiểm lại rồi gửi.
2. AI thẩm định: tra ngân sách dự án (dự án `X` còn 30 triệu, `Y` còn 400 triệu) và ghi nhận xét.
3. `truongphong` duyệt, rồi `ketoan` xác nhận.

## Chạy khi không có Internet

Mọi thành phần (đăng nhập, cổng, CSDL, quy trình, kho tệp, model AI, phông chữ) chạy trên máy. Sau lần cài đầu tiên (cần mạng để tải image và model), có thể rút mạng mà hệ thống vẫn chạy đủ.

## Câu hỏi thường gặp

**AI có tự duyệt không?** Không. AI chỉ điền giúp và chuẩn bị nhận xét. Người có thẩm quyền mới duyệt.

**AI không chạy thì sao?** Sau 2 phút, việc tự chuyển sang người duyệt.

**Tôi có xem được hồ sơ của người khác không?** Không. Quyền chặn ngay trong cơ sở dữ liệu, không chỉ ẩn trên giao diện.
