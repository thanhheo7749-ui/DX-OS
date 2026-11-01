<!-- SPDX-License-Identifier: Apache-2.0 -->
<h1 align="center">DX-OS Open-Core</h1>

<p align="center">
  Hệ điều hành chuyển đổi số nối các phòng ban thành một dòng chảy công việc.<br/>
  <b>Luật chạy việc thường. AI trực ca chuẩn bị sẵn và báo trước. Con người quyết định.</b>
</p>

<p align="center">
  <a href="https://github.com/thanhheo7749-ui/DX-OS/issues/new?template=bug_report.yml">🐛 Báo lỗi</a>
  ·
  <a href="https://github.com/thanhheo7749-ui/DX-OS/issues/new?template=feature_request.yml">✨ Đề xuất tính năng</a>
  ·
  <a href="https://github.com/thanhheo7749-ui/DX-OS/tree/docs/docs">📚 Tài liệu</a>
  ·
  <a href="CONTRIBUTING.md">🤝 Đóng góp</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="Giấy phép Apache-2.0"/></a>
  <a href="https://github.com/thanhheo7749-ui/DX-OS/issues"><img src="https://img.shields.io/github/issues/thanhheo7749-ui/DX-OS?label=Issues" alt="Issues"/></a>
  <img src="https://img.shields.io/badge/OLP_PMNM-2026-orange" alt="OLP PMNM 2026"/>
</p>

---

Bài dự thi **OLP Phần mềm Nguồn mở 2026**, đề *Xây dựng Hệ điều hành Chuyển đổi số bằng Công nghệ Lõi nguồn mở (DX-OS Open-Core)*.

## Trạng thái

🚧 **Đang dựng.** Nhánh `main` hiện chứa khung dự án: giấy phép, quy trình đóng góp, mẫu issue và PR. Mã nguồn được đưa lên `develop` theo từng phần qua Pull Request, và lên `main` khi phát hành bản đầu tiên `v0.1.0`. Tài liệu nằm ở nhánh [`docs`](https://github.com/thanhheo7749-ui/DX-OS/tree/docs/docs).

## Vấn đề

Ở doanh nghiệp vừa và nhỏ, mỗi phòng dùng một phần mềm riêng. Việc đi qua nhiều phòng (một đơn hàng, một đề nghị mua sắm) được nối bằng tin nhắn, email và trí nhớ: việc nằm chờ không ai biết ở bàn ai, đơn thiếu thông tin vẫn lọt qua, ngoại lệ được duyệt miệng, và không còn dấu vết ai đã quyết gì.

## Hướng giải quyết

DX-OS đứng trên các phần mềm doanh nghiệp đang dùng, không thay thế chúng:

- **Hồ sơ liên phòng** chạy theo quy trình BPMN, xong bước nào tự sang phòng kế tiếp.
- **Luật canh ở cửa** chặn dữ liệu sai (ví dụ mã số thuế sai chữ số kiểm tra) và đưa ngoại lệ sang người có thẩm quyền.
- **Luật có phiên bản**, xem trước tác động trên dữ liệu đã có trước khi duyệt.
- **AI trực ca chạy tại chỗ** chuẩn bị phân tích, không tự duyệt. AI không chạy thì việc vẫn đến người.
- **Nhật ký niêm phong** bằng chuỗi băm: sửa lén một dòng cũ là bị phát hiện.

## Kiến trúc 3 tầng

| Tầng | Thành phần | Giấy phép |
|---|---|---|
| **1. Dịch vụ lõi** (headless, không có giao diện) | Keycloak (định danh, SSO) · Apache APISIX (cổng API) · PostgreSQL + PostgREST (dữ liệu có cấu trúc) · SeaweedFS (dữ liệu phi cấu trúc) · Flowable (quy trình BPMN) | Apache-2.0, PostgreSQL, MIT |
| **2. Ứng dụng H-P-D-I** | Giao diện Vue 3, BFF Node, agent AI (Ollama, model chạy tại chỗ) | MIT, Apache-2.0 |
| **3. Gói quy trình** | Quy trình BPMN cắm thêm không cần sửa mã | — |

## Cấu trúc thư mục

```
├── docker-compose.yml   Tầng 1: dịch vụ lõi headless
├── platform/            Tầng 1: cấu hình APISIX, Keycloak, PostgreSQL
├── app/                 Tầng 2: giao diện, BFF, agent AI, quy trình BPMN, bảng dữ liệu
├── scripts/             Sinh cấu hình, triển khai quy trình, kiểm thử
├── docs/                Tài liệu (nhánh docs)
└── .github/             Mẫu issue, mẫu PR, workflow
```

## Nhánh

| Nhánh | Dùng cho |
|---|---|
| `main` | Bản đã phát hành, mỗi bản một tag `vX.Y.Z` |
| `develop` | Nhánh tích hợp, mọi Pull Request việc hằng ngày vào đây |
| `docs` | Tài liệu |
| `release/X.Y.Z` | Chuẩn bị phát hành, giữ lại sau khi phát hành |

Chi tiết: [CONTRIBUTING.md](CONTRIBUTING.md).

## Đóng góp và báo lỗi

Báo lỗi, đề xuất tính năng: [Issues](https://github.com/thanhheo7749-ui/DX-OS/issues). Mọi thay đổi đi qua Pull Request và cần một thành viên khác duyệt. Lỗ hổng bảo mật: xem [SECURITY.md](SECURITY.md).

## Giấy phép

Apache License 2.0, xem [LICENSE](LICENSE). Các thành phần bên thứ ba chạy từ image Docker chính thức của từng dự án, không đóng gói kèm mã nguồn.

Lõi Tầng 1 và ứng dụng mẫu ban đầu dựa trên mã nguồn mẫu `olp-2026-os` của giảng viên hướng dẫn (Apache-2.0), giữ nguyên tác giả các commit gốc. Xem [NOTICE](NOTICE).
