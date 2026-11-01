<!-- SPDX-License-Identifier: Apache-2.0 -->
# Dựng môi trường phát triển

## Yêu cầu

| Phần mềm | Phiên bản | Ghi chú |
|---|---|---|
| Docker Engine hoặc Docker Desktop | 24+ | Windows: bật ảo hóa trong BIOS, dùng backend WSL 2 |
| Docker Compose | v2.20+ | Có sẵn trong Docker Desktop |
| Git | bất kỳ | Windows: Git for Windows (kèm Git Bash để chạy script) |
| Node.js | 22 | Chỉ cần khi sửa giao diện hoặc chạy unit test ngoài Docker |
| `curl`, `jq` | bất kỳ | Cho các script kiểm tra |
| GPU NVIDIA + NVIDIA Container Toolkit | tùy chọn | Có thì model AI chạy nhanh hơn nhiều |

Máy cần khoảng 10 GB RAM trống. Lần đầu tải khoảng 2,5 GB image và model.

## Dựng từ mã nguồn

```bash
git clone https://github.com/thanhheo7749-ui/DX-OS.git
cd DX-OS
./scripts/init-env.sh      # sinh .env: mật khẩu ngẫu nhiên, tự bật GPU nếu Docker dùng được
docker compose up -d       # dựng Tầng 1 + Tầng 2 (image app dựng từ app/Dockerfile)
./scripts/smoke.sh         # kiểm tra 4 dịch vụ lõi và các quy tắc chặn của cổng
```

Mọi cấu hình nằm trong `.env`, không cần sửa tệp mã nào. Danh sách biến: [`.env.example`](../.env.example).
Flowable cần khoảng 1 phút sau `up` mới sẵn sàng.

Chỉ chạy Tầng 1: `docker compose -f docker-compose.yml up -d`.

## Kiểm thử

```bash
cd app && npm ci && npm test   # unit test, không cần Docker
./scripts/smoke.sh             # Tầng 1
./scripts/e2e.sh               # trọn luồng Đề nghị mua sắm
./scripts/e2e-dxos.sh          # trọn luồng Đơn hàng liên phòng
```

## Sửa giao diện

```bash
cd app
npm install
npm run dev      # http://localhost:5173, gọi API qua BFF đang chạy ở :8000
```

Sửa BFF (`app/server/`) thì dựng lại container: `docker compose up -d --build app`.

## Thêm một quy trình

1. Viết tệp `app/processes/<tên>.bpmn20.xml`. Trường của form khai bằng `flowable:formProperty`.
2. Triển khai không cần khởi động lại: `./scripts/deploy-process.sh app/processes/<tên>.bpmn20.xml`.

Ví dụ: `app/processes/extra/dang-ky-cong-tac.bpmn20.xml`.

## Xử lý sự cố

| Triệu chứng | Cách xử lý |
|---|---|
| `required variable POSTGRES_PASSWORD is missing a value: run scripts/init-env.sh` | Chưa có `.env`: chạy `./scripts/init-env.sh` |
| Windows báo không tìm thấy `docker-compose.yml:app/compose.yml` | `.env` thiếu `COMPOSE_PATH_SEPARATOR=:`; chạy lại `./scripts/init-env.sh` |
| Sửa `platform/keycloak/realm-dxos.json` mà không thấy thay đổi | Keycloak chỉ nhập realm khi volume trống: `docker compose down -v && docker compose up -d` |
| `smoke.sh` báo lỗi `workflow` | Flowable chưa khởi động xong, đợi 1 phút rồi chạy lại |
| AI trả lời rất chậm | Chưa dùng GPU; kiểm `docker info \| grep -i nvidia`, chạy lại `init-env.sh` sau khi cài NVIDIA Container Toolkit |
| Hết dung lượng, muốn xóa sạch | `docker compose down -v` (xóa cả dữ liệu và model đã tải) |
