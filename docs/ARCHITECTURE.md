<!-- SPDX-License-Identifier: Apache-2.0 -->
# Kiến trúc

## Mục lục

- [Tổng quan](#tổng-quan)
- [Tầng 1: dịch vụ lõi headless](#tầng-1-dịch-vụ-lõi-headless)
- [Tầng 2: ứng dụng H-P-D-I](#tầng-2-ứng-dụng-h-p-d-i)
- [Luồng một hồ sơ liên phòng](#luồng-một-hồ-sơ-liên-phòng)
- [Mô hình tin cậy](#mô-hình-tin-cậy)
- [Quyết định thiết kế](#quyết-định-thiết-kế)

## Tổng quan

DX-OS theo khung 3 tầng của đề thi DX-OS Open-Core:

```
Tầng 3  Gói quy trình, ứng dụng ngành (chưa làm; cắm vào như một quy trình BPMN mới)
Tầng 2  app/  Vue 3 + BFF Node + agent AI  ── chỉ gọi Tầng 1 qua cổng
Tầng 1  platform/ + docker-compose.yml
        Cổng APISIX :9080 ─┬─ /data           PostgREST (row-level security)
                           ├─ /workflow       Flowable REST (chỉ token dịch vụ nền tảng)
                           └─ /workflow-jobs  Flowable external worker (agent AI lấy việc)
        Keycloak :8080 · PostgreSQL 17 + pgvector · SeaweedFS (S3) · Flowable 7.1
```

| Thư mục | Tầng | Vai trò |
|---|---|---|
| `docker-compose.yml`, `platform/` | 1 | Dịch vụ lõi và cấu hình của chúng. Không có giao diện người dùng |
| `app/` | 2 | Ứng dụng: giao diện, BFF, agent, quy trình BPMN, bảng dữ liệu của ứng dụng |
| `scripts/` | — | Sinh cấu hình, triển khai quy trình, kiểm thử |

## Tầng 1: dịch vụ lõi headless

| Dịch vụ lõi (yêu cầu đề) | Thành phần | Cấu hình |
|---|---|---|
| Định danh, SSO | Keycloak 26 | `platform/keycloak/realm-dxos.json` |
| Cổng API | Apache APISIX 3.11 | `platform/apisix/apisix.yaml`, `config.yaml` |
| Dữ liệu có cấu trúc | PostgreSQL 17 + pgvector, PostgREST 12 | `platform/postgres/init.sh`, `app/sql/` |
| Dữ liệu phi cấu trúc | SeaweedFS 3.80 (S3) | `docker-compose.yml` |
| Quy trình | Flowable REST 7.1 | `app/processes/*.bpmn20.xml` |

Ra ngoài chỉ có cổng `:9080` và trang đăng nhập Keycloak `:8080`. Các dịch vụ khác chỉ nằm trong mạng Docker.

Keycloak có 4 client:

| Client | Loại | Dùng cho |
|---|---|---|
| `dxos-web` | Công khai | Trình duyệt đăng nhập người dùng |
| `dxos-gateway` | Bí mật | APISIX kiểm token |
| `dxos-bff` | Tài khoản dịch vụ | BFF gọi `/workflow`, `/workflow-jobs` |
| `dxos-agent` | Tài khoản dịch vụ | Agent AI đọc `/data`, bị row-level security giới hạn |

## Tầng 2: ứng dụng H-P-D-I

```
Trình duyệt (Vue 3 + keycloak-js)
   │  token người dùng
   ▼
BFF (Node, :8000) ──token người dùng──▶ /data       (RLS: mỗi người chỉ thấy việc của mình)
   │               ──token dxos-bff──▶ /workflow   (BFF kiểm tra quyền trên từng task)
   │
   └─ Agent AI ──token dxos-agent──▶ /data         (agent cũng bị RLS giới hạn)
                ──token dxos-bff──▶ /workflow-jobs  (agent là một bước BPMN)
                ──▶ Ollama qwen3:4b-instruct (chạy tại chỗ)
```

| Không gian | Thành phần |
|---|---|
| [H] Con người | Đăng nhập một lần, vai trò theo phòng ban; màn Việc của tôi |
| [P] Quy trình | Quy trình BPMN, biểu mẫu sinh từ BPMN, luật L-02 và L-07, luật có phiên bản |
| [D] Dữ liệu | Bản đồ vận hành, nhật ký chuỗi băm, tệp đính kèm trên SeaweedFS |
| [I] Trí tuệ | Agent điền form, agent thẩm định và chuẩn bị làn Đỏ |

**Biểu mẫu sinh từ BPMN.** Các trường `flowable:formProperty` trong tệp BPMN là nguồn duy nhất cho form web, cho bước kiểm dữ liệu ở BFF và cho công cụ của agent. Thêm quy trình mới chỉ cần thêm tệp BPMN và chạy `scripts/deploy-process.sh`.

**Agent là một bước BPMN.** Bước thẩm định là `serviceTask` dạng external worker với topic `ai-review`. Agent lấy việc qua `/workflow-jobs`. Bước này có bộ hẹn giờ 2 phút (`PT2M`): agent không chạy thì việc vẫn sang người duyệt.

## Luồng một hồ sơ liên phòng

Quy trình `ban-hang-lien-phong`:

```
Sales tạo đơn ─▶ [L-02] thiếu mã số thuế / số PO? ──có──▶ trả về Sales
                     │ không
                     ▼
               [L-07] chiết khấu > trần (luật đang chạy)?
                     │ có                         │ không
                     ▼                            │
       AI trực ca chuẩn bị (ai-review, ≤ 2 phút)  │
                     ▼                            │
            Giám đốc duyệt làn Đỏ                 │
                     └──────────────┬─────────────┘
                                    ▼
             Kế toán ─▶ Kho ─▶ Giao vận ─▶ CSKH
```

Lúc tạo hồ sơ, BFF đọc phiên bản luật đang chạy rồi chép vào biến `tranChietKhau` và `luatVersion` của hồ sơ. Nhờ vậy hồ sơ đang chạy giữ luật cũ khi Giám đốc duyệt phiên bản mới.

Mỗi bước quan trọng ghi một dòng vào nhật ký chuỗi băm: tạo hồ sơ, hoàn thành bước, AI chuẩn bị xong, lưu nháp luật, duyệt luật.

## Mô hình tin cậy

| Lớp | Chặn gì |
|---|---|
| Cổng APISIX | Lời gọi không có token, token giả, token người dùng gọi `/workflow` |
| PostgREST | Kiểm lại chữ ký token, đặt role CSDL theo token |
| PostgreSQL | Row-level security: người dùng chỉ thấy hồ sơ của mình; chỉ trưởng phòng và Giám đốc đọc nhật ký; chỉ Giám đốc duyệt luật (`api.approve_rule`) |
| Nhật ký | Không ai được `UPDATE`, `DELETE`; mỗi dòng băm nối dòng trước; `api.verify_audit()` chỉ ra dòng đầu tiên bị phá |
| BFF | Kiểm vai trò trên từng task và từng API trước khi gọi Tầng 1 |
| Agent AI | Token riêng, bị RLS như người; không có quyền hoàn thành bước duyệt |

## Quyết định thiết kế

| Quyết định | Lý do |
|---|---|
| PostgREST thay vì tự viết API dữ liệu | API tự sinh từ schema, quyền nằm trong CSDL nên mọi client (kể cả agent) đều bị chặn như nhau |
| Flowable cho quy trình | Chuẩn BPMN, có external worker cho agent, có hẹn giờ |
| Agent là bước BPMN, không phải chatbot đứng ngoài | Quy trình biết lúc nào AI tham gia, và luôn có đường sang người khi AI không chạy |
| Model chạy tại chỗ qua Ollama | Dữ liệu nội bộ không ra ngoài; model Apache-2.0 |
| Chỉ phần mềm có giấy phép OSI | Không dùng Vault, n8n, Redpanda, MongoDB, immudb, model Llama hay Gemma |

