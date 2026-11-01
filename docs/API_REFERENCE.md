<!-- SPDX-License-Identifier: Apache-2.0 -->
# Tham chiếu API

Có hai lớp API:

- **Cổng APISIX** (`:9080`): API của Tầng 1, dành cho mọi ứng dụng cắm vào lõi.
- **BFF** (`:8000/api`): API riêng của ứng dụng Tầng 2, dành cho giao diện web.

## Lấy token

```bash
set -a; . ./.env; set +a
TOKEN=$(curl -s http://localhost:8080/realms/dxos/protocol/openid-connect/token \
  -d grant_type=password -d client_id=dxos-web -d username=nhanvien1 \
  --data-urlencode "password=$DEMO_USER_PASSWORD" | jq -r .access_token)
```

Mọi lời gọi gửi kèm `Authorization: Bearer $TOKEN`.

## Cổng APISIX (Tầng 1)

| Tuyến | Chuyển tới | Ai gọi được |
|---|---|---|
| `/data/*` | PostgREST, schema `api` | Mọi token hợp lệ; dữ liệu trả về đã lọc theo row-level security |
| `/workflow/*` | Flowable REST API (`/flowable-rest/service/*`) | Chỉ token có `azp = dxos-bff` |
| `/workflow-jobs/*` | Flowable external worker API (`/flowable-rest/external-job-api/*`) | Chỉ token có `azp = dxos-bff` |

Không có token: `401`. Token giả hoặc sửa claim `azp`: `401`. Token người dùng gọi `/workflow`: `403`.

`GET /data/` trả về mô tả OpenAPI của toàn bộ API dữ liệu. Ví dụ:

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:9080/data/budgets
curl -H "Authorization: Bearer $TOKEN" "http://localhost:9080/data/rule_versions?status=eq.active"
```

Bảng và hàm có thể gọi: [DATA_MODEL.md](DATA_MODEL.md).

## BFF (Tầng 2)

Mọi API cần token người dùng. Lỗi trả JSON `{ "message": "<thông báo tiếng Việt>" }` với mã HTTP tương ứng (`400`, `403`, `404`...). Lỗi hệ thống trả `500` với thông báo chung, chi tiết chỉ ghi trong log.

### Người dùng, quy trình

| Phương thức, đường dẫn | Ai | Làm gì |
|---|---|---|
| `GET /api/me` | Mọi người | Tên đăng nhập và vai trò |
| `GET /api/processes` | Mọi người | Danh sách quy trình kèm trường form (sinh từ BPMN) |
| `POST /api/agent/draft` | Mọi người | Gửi `{ processKey, text }` (tối đa 2000 ký tự), nhận giá trị form do AI điền |
| `GET /api/health` | Mọi người | Trạng thái từng dịch vụ: `identity`, `gateway`, `data`, `workflow`, `storage`, `ai` |

### Hồ sơ, việc

| Phương thức, đường dẫn | Ai | Làm gì |
|---|---|---|
| `POST /api/requests` | Mọi người | Mở hồ sơ mới theo một quy trình |
| `GET /api/requests` | Mọi người | Hồ sơ do chính mình tạo |
| `GET /api/tasks` | Mọi người | Việc giao cho mình hoặc cho nhóm vai trò của mình |
| `GET /api/tasks/:id` | Người được giao, nhóm được giao | Chi tiết một việc và form |
| `POST /api/tasks/:id` | Người được giao, nhóm được giao | Hoàn thành việc với giá trị form |

### Tệp

| Phương thức, đường dẫn | Ai | Làm gì |
|---|---|---|
| `POST /api/files` | Mọi người | Tải lên PDF, PNG, JPEG hoặc TXT, tối đa 10 MB; lưu trên SeaweedFS |
| `GET /api/files/:id/:name` | Người có khóa tệp | Tải tệp về |

### Luật, vận hành, nhật ký

| Phương thức, đường dẫn | Ai | Làm gì |
|---|---|---|
| `GET /api/rules` | Mọi người | Lịch sử phiên bản và bản nháp của trần chiết khấu |
| `POST /api/rules` | Trưởng phòng, Giám đốc | Lưu bản nháp (số nguyên 0–50) |
| `POST /api/rules/:id/approve` | Giám đốc | Duyệt bản nháp thành phiên bản mới |
| `GET /api/rules/impact?value=&days=` | Trưởng phòng, Giám đốc | Số đơn trong `days` ngày qua (7–180, mặc định 30) vào làn Đỏ theo luật lúc đó và theo trần `value` |
| `GET /api/board` | Trưởng phòng, Giám đốc | Hồ sơ đang mở, gom theo bước đang chờ |
| `GET /api/audit` | Trưởng phòng, Giám đốc | 50 dòng nhật ký mới nhất và kết quả kiểm chuỗi |
