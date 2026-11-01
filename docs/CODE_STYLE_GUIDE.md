<!-- SPDX-License-Identifier: Apache-2.0 -->
# Quy ước mã

## Chung

- Đầu mỗi tệp mã: một dòng `SPDX-License-Identifier: Apache-2.0` theo cú pháp comment của loại tệp.
- Comment viết tiếng Anh, giải thích **vì sao**, không lặp lại mã làm gì. Tài liệu và chữ trên giao diện viết tiếng Việt.
- Thụt lề 2 dấu cách, xuống dòng LF, UTF-8 (xem `.editorconfig`).
- Không để bí mật trong mã. Mọi giá trị đổi theo môi trường đọc từ biến môi trường, sinh bởi `scripts/init-env.sh`.

## JavaScript (BFF, `app/server/`)

- ES modules (`import`/`export`), `async`/`await`.
- Lỗi trả cho người dùng ném `HttpError(<mã>, '<thông báo tiếng Việt>')`.
- Kiểm vai trò ở đầu handler, trước khi gọi Tầng 1.
- Gọi Tầng 1 luôn qua cổng (`/data`, `/workflow`), không gọi thẳng PostgREST hay Flowable.
- Unit test đặt cạnh mã: `<tên>.test.js`, chạy bằng `node --test`.

## Vue (`app/web/src/`)

- Single File Component, `<script setup>`.
- Không giữ khóa hay token dịch vụ ở trình duyệt; mọi thao tác đi qua BFF.
- Ẩn nút theo vai trò chỉ để dễ dùng. Quyền thật luôn được kiểm lại ở BFF và CSDL.

## SQL (`app/sql/`)

- Chạy lại được nhiều lần: `CREATE ... IF NOT EXISTS`, `CREATE OR REPLACE`, `DROP POLICY IF EXISTS` trước `CREATE POLICY`.
- Bảng mới trong schema `api` luôn bật row-level security và có chính sách rõ ràng.
- Đánh số tệp theo thứ tự chạy: `001_...`, `002_...`.
- Cuối tệp: `NOTIFY pgrst, 'reload schema';` khi đổi cấu trúc.

## BPMN (`app/processes/`)

- `id` của process viết thường, gạch nối: `ban-hang-lien-phong`.
- Trường form khai bằng `flowable:formProperty`; đây là nguồn duy nhất cho form web, kiểm dữ liệu và công cụ của agent.
- Bước AI là external worker với topic `ai-review`, luôn có hẹn giờ đẩy việc sang người.

## Shell (`scripts/`)

- Bắt đầu bằng `#!/usr/bin/env bash` và `set -euo pipefail`.
- Chạy được trên Linux, macOS và Git Bash của Windows.
