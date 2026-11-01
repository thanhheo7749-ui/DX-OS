<!-- SPDX-License-Identifier: Apache-2.0 -->
# Chính sách bảo mật

## Phiên bản được hỗ trợ

Chỉ bản phát hành mới nhất trên trang [Releases](https://github.com/thanhheo7749-ui/dx-os/releases) được sửa lỗi bảo mật.

## Báo lỗ hổng

**Không mở issue công khai cho lỗ hổng.** Dùng [Security → Report a vulnerability](https://github.com/thanhheo7749-ui/dx-os/security/advisories/new) của kho mã nguồn.

Xin ghi: phiên bản hoặc commit, cách tái hiện, ảnh hưởng. Đội phản hồi trong 3 ngày làm việc và báo lại khi đã sửa.

## Mô hình bảo vệ hiện có

- Mọi lời gọi từ ngoài vào Tầng 1 đi qua cổng APISIX và phải có access token Keycloak.
- `/workflow` và `/workflow-jobs` chỉ nhận token của dịch vụ nền tảng `dxos-bff`.
- PostgREST tự kiểm tra lại token và áp row-level security theo người dùng.
- Agent AI có token riêng và bị row-level security giới hạn như người dùng.

## Hạn chế đã biết

- SeaweedFS chưa bật xác thực S3; chỉ truy cập được trong mạng Docker và từ máy chủ. Không đưa cổng 8333 ra Internet.
- Ai đăng nhập mà biết khóa ngẫu nhiên của tệp đính kèm thì tải được tệp đó.
- Keycloak chạy `start-dev` (HTTP) cho môi trường demo; triển khai thật cần `start` với HTTPS.
