<!-- SPDX-License-Identifier: Apache-2.0 -->
# Đóng góp cho DX-OS Open-Core

Cảm ơn bạn quan tâm đến DX-OS. Mọi thay đổi đi theo một đường: **Issue → nhánh → Pull Request → duyệt → merge**.

## Mục lục

- [Quy tắc ứng xử](#quy-tắc-ứng-xử)
- [Dựng môi trường](#dựng-môi-trường)
- [Mô hình nhánh](#mô-hình-nhánh)
- [Quy trình làm việc](#quy-trình-làm-việc)
- [Đặt tên nhánh và commit](#đặt-tên-nhánh-và-commit)
- [Phát hành](#phát-hành)
- [Pull Request](#pull-request)
- [Quy tắc mã](#quy-tắc-mã)
- [Báo lỗi bảo mật](#báo-lỗi-bảo-mật)

## Quy tắc ứng xử

Mọi người tham gia dự án tuân theo [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

## Dựng môi trường

Mã nguồn ứng dụng đang được đưa lên theo từng phần. Khi đã có trên `develop`, các bước dưới đây dùng được ngay.

Cần Docker 24+, Docker Compose v2.20+, Git; Node 22 nếu sửa giao diện; `curl`, `jq` cho script kiểm tra. Chi tiết: [docs/DEVELOPMENT_SETUP.md](https://github.com/thanhheo7749-ui/DX-OS/blob/docs/docs/DEVELOPMENT_SETUP.md) (nhánh `docs`).

```bash
./scripts/init-env.sh
docker compose up -d
./scripts/smoke.sh
cd app && npm ci && npm test
```

## Mô hình nhánh

| Nhánh | Vai trò | Ai ghi vào |
|---|---|---|
| `main` | Chỉ chứa bản đã phát hành. Mỗi bản có tag `vX.Y.Z` | Chỉ PR từ `release/*` hoặc `hotfix/*` |
| `develop` | Nhánh tích hợp, luôn chạy được | PR từ các nhánh việc |
| `docs` | Nhánh dài hạn cho tài liệu (`docs/`), merge dần vào `develop`. Việc tài liệu lẻ làm trên `doc/<việc>` tạo từ `docs` | Người viết tài liệu |
| `feature/*`, `fix/*`, `test/*`, `ci/*`, `chore/*` | Một việc, tạo từ `develop` | Người làm việc đó |
| `release/X.Y.Z` | Chuẩn bị phát hành: chốt CHANGELOG, sửa lỗi cuối | Trưởng nhóm |
| `hotfix/*` | Sửa gấp lỗi trên bản đã phát hành, tạo từ `main` | Người sửa |

```
docs ──────PR──▶ develop
feature/* ──PR──▶ develop ──release/X.Y.Z──PR──▶ main ──tag vX.Y.Z──▶ Release
                     ▲                                │
                     └──────── hotfix/* ◀─────────────┘
```

## Quy trình làm việc

1. Mở hoặc nhận một issue. Ghi rõ **tiêu chí xong**. Gắn nhãn và milestone.
2. Tạo nhánh từ `develop` mới nhất:
   ```bash
   git switch develop && git pull
   git switch -c feature/<việc>
   ```
3. Commit nhỏ, mỗi commit một việc.
4. Thêm một dòng vào `CHANGELOG.md`, mục `[Unreleased]`.
5. Mở PR **vào `develop`**. Một thành viên khác duyệt rồi mới merge.

## Đặt tên nhánh và commit

| Loại | Nhánh | Commit |
|---|---|---|
| Tính năng | `feature/<việc>` | `feat: thêm luật chặn đơn thiếu mã số thuế` |
| Sửa lỗi | `fix/<lỗi>` | `fix: bản đồ vận hành đếm trùng hồ sơ` |
| Tài liệu | `doc/<tài liệu>` (từ nhánh `docs`) | `docs: hướng dẫn cài ngoài thư mục nguồn` |
| Kiểm thử | `test/<phần>` | `test: e2e cho làn Đỏ` |
| CI, cấu hình | `ci/<việc>`, `chore/<việc>` | `ci: chạy reuse lint`, `chore: ghim phiên bản image` |

Commit theo [Conventional Commits](https://www.conventionalcommits.org/vi/v1.0.0/): `<loại>(<phạm vi>): <mô tả>`, phạm vi tùy chọn (ví dụ `feat(bff): ...`, `fix(keycloak): ...`).

Email trong `git config user.email` phải trùng một email đã gắn với tài khoản GitHub của bạn, để commit được tính đúng người.

## Pull Request

- Nhắm vào `develop` (trừ PR phát hành và hotfix nhắm vào `main`).
- Mô tả theo mẫu, ghi `Closes #<số issue>`.
- CI phải xanh.
- Ít nhất **1 người khác** duyệt. Không tự duyệt PR của mình.
- Merge bằng **Create a merge commit**. Nhánh việc tự xóa sau khi merge; `main` và `develop` được bảo vệ nên không bị xóa.

## Phát hành

1. Trưởng nhóm tạo `release/X.Y.Z` từ `develop`, đổi mục `[Unreleased]` của `CHANGELOG.md` thành `[X.Y.Z] - YYYY-MM-DD`.
2. Mở PR `release/X.Y.Z` → `main`, duyệt, merge.
3. Gắn tag trên `main`: `git tag -a vX.Y.Z -m "vX.Y.Z" && git push origin vX.Y.Z`. Workflow phát hành tạo Release có `.tar.gz` và `.sha256`.
4. Merge `main` ngược về `develop` để hai nhánh khớp nhau.

## Quy tắc mã

- Tầng 1 (`platform/`, `docker-compose.yml`) **không có giao diện người dùng**. Màn hình chỉ nằm ở Tầng 2 (`app/`).
- Quyền chặn ở BFF **và** ở PostgreSQL (row-level security). Không chỉ ẩn nút trên giao diện.
- Mọi lời gọi vào Tầng 1 đi qua cổng APISIX bằng token Keycloak.
- AI chỉ gợi ý và chuẩn bị, không tự duyệt, không tự ghi thay người.
- Mọi tệp mã mới có header `SPDX-License-Identifier: Apache-2.0` ở đầu (`#` cho shell/YAML, `//` cho JS, `<!-- -->` cho Vue/HTML/Markdown, `--` cho SQL).
- Chỉ thêm thành phần có giấy phép được OSI phê duyệt. Dùng image chính thức, ghim phiên bản. Không chép mã của dự án khác vào repo, không sửa mã của thư viện.
- Không commit `.env`, `node_modules/`, `app/dist/`, khóa hay mật khẩu.
- Comment trong mã viết tiếng Anh; tài liệu viết tiếng Việt. Phong cách: [docs/CODE_STYLE_GUIDE.md](docs/CODE_STYLE_GUIDE.md).

## Báo lỗi bảo mật

Không mở issue công khai. Xem [SECURITY.md](SECURITY.md).
