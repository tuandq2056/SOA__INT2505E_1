# WebChat – Hệ thống nhắn tin trên trình duyệt

WebChat cho phép chat riêng, chat nhóm, gửi tệp và bản ghi âm, tìm kiếm lịch sử, đọc tin mới thành
tiếng và tóm tắt tin chưa đọc. Hệ thống gồm ba dịch vụ **Identity**, **Social** và **Chat** (FastAPI),
một web frontend (React/TypeScript), và Nginx làm cổng vào. Frontend tích hợp qua REST và WebSocket.

## Trạng thái hiện tại (tuần 12 – Design Review)

| Hạng mục | Trạng thái |
|---|---|
| Contract REST `webchat/contracts/openapi.yaml` (OpenAPI 3.1.1, 25 operation) | Xong, đã lint bằng Spectral |
| Contract WebSocket `webchat/contracts/asyncapi.yaml` (AsyncAPI 3.0.0, 10 sự kiện) | Xong |
| Tài liệu thiết kế, sơ đồ (resource model, ERD, kiến trúc, hexagonal) | Xong, trong `docs/` |
| Backend Identity / Social / Chat | Chưa bắt đầu (tuần 13–15) |
| Web frontend | Chưa bắt đầu (tuần 13–15) |

## Cấu trúc thư mục

```
.
├── README.md
├── docs/
│   ├── report/            # Báo cáo tuần, project brief, báo cáo tiến độ
│   ├── diagrams/          # WebChat_Diagrams.drawio và ảnh PNG xuất ra
│   ├── demo/              # Kịch bản demo, Swagger UI cho Prism, ảnh chụp
│   └── slides/            # Slide thuyết trình
└── webchat/
    └── contracts/
        ├── openapi.yaml            # REST: Public API /api/v1 và Internal API /internal/v1
        ├── asyncapi.yaml           # WebSocket /ws/v1/chat
        └── asyncapi.bundled.yaml   # Bản bundle để mở trên AsyncAPI Studio
```

Các thư mục `webchat/identity`, `webchat/social`, `webchat/chat`, `webchat/web` và
`docker-compose.yml` sẽ được thêm từ tuần 13.

## Xem Contract API

**Swagger UI (REST)**: chạy trong thư mục `webchat/contracts`.

```bash
docker run --rm -p 8081:8080 \
  -e SWAGGER_JSON=/spec/openapi.yaml \
  -v "$PWD:/spec" swaggerapi/swagger-ui
```

Mở http://localhost:8081. Trên PowerShell, thay `\` bằng dấu `` ` `` và `$PWD` bằng `${PWD}`.
Cách khác: mở https://editor.swagger.io/ rồi dán nội dung `openapi.yaml`.

**AsyncAPI (WebSocket)**: mở https://studio.asyncapi.com/ rồi dán nội dung `asyncapi.yaml`. File này
tham chiếu schema `Message` và `ErrorCode` trong `openapi.yaml`, nên khi xem bằng CLI cần để hai file
cùng thư mục.

**Kiểm tra Contract**

```bash
npx @stoplight/spectral-cli lint webchat/contracts/openapi.yaml   # cần file .spectral.yaml: extends: ["spectral:oas"]
npx @asyncapi/cli validate webchat/contracts/asyncapi.yaml
```

**Mock server (Prism) và demo**: chạy trong thư mục gốc repo.

```bash
npx @stoplight/prism-cli mock webchat/contracts/openapi.yaml -p 4010
npx http-server -p 8081 -c-1          # rồi mở http://localhost:8081/docs/demo/swagger.html
npx @asyncapi/cli bundle webchat/contracts/asyncapi.yaml -o webchat/contracts/asyncapi.bundled.yaml
```

Trong Swagger UI chọn server `http://localhost:4010` để dùng "Try it out". Prism trả example thành công
mặc định; muốn xem response lỗi thì gửi header `Prefer: code=401` (trang demo có sẵn ô nhập). Bản
`asyncapi.bundled.yaml` không còn tham chiếu ngoài nên dán thẳng được vào AsyncAPI Studio. Kịch bản
demo (gồm cả bước tóm tắt tin chưa đọc) và ảnh chụp nằm trong `docs/demo/`.

## Quy ước API

- Public API: `/api/v1`; Internal API (chỉ giữa các dịch vụ): `/internal/v1`; WebSocket: `/ws/v1/chat`.
- JSON UTF-8, `snake_case`, ID là UUID, thời gian theo RFC 3339 UTC (hậu tố `Z`).
- Mọi lỗi HTTP trả `ErrorResponse { request_id, error: { code, message, fields?, related_id? } }`;
  client xử lý theo `error.code`.
- Xác thực: JWT người dùng (`aud=webchat`, 3600 giây); JWT dịch vụ cho `/internal/v1`.
- Phân trang: cursor (`limit`, `cursor`, `next_cursor`) cho Social; theo `seq`
  (`before_seq`, `after_seq`, `through_seq`, `has_more`) cho tin nhắn.

## Phân công

| Thành viên | MSSV | Phụ trách chính |
|---|---|---|
| Đinh Quang Tuân | 24021654 | Repo, Contract OpenAPI/AsyncAPI, CI/CD, Docker Compose, Nginx, Identity, kiểm thử |
| Phạm Minh Nghĩa | 24021584 | Social Service: kết bạn, hội thoại, nhóm, API kiểm tra quyền nội bộ |
| Lê Thành Long | 24021552 | Chat Service: REST (lịch sử, tìm kiếm, tệp, mốc đọc, tóm tắt) và WebSocket |
| Lại Tùng Lâm | 24021544 | Web frontend (React/TypeScript), TTS, E2E bằng Playwright |

Backlog được quản lý bằng GitHub Issues, mỗi issue gắn label theo dịch vụ
(`identity`, `social`, `chat`, `web`, `infra`, `contract`) và milestone theo tuần (`W13`, `W14`, `W15`).

## Kế hoạch

| Tuần | Mục tiêu |
|---|---|
| 13 | Docker Compose, Nginx, CI; Identity hoàn chỉnh; khung Social và Chat; frontend đăng nhập và kết bạn trên mock |
| 14 | Social hoàn chỉnh; Chat REST (lịch sử, mốc đọc, tóm tắt tin chưa đọc) và WebSocket gửi/nhận text; frontend nối backend thật – **demo được MVP lõi** |
| 15 | Mở rộng: chat nhóm, tìm kiếm, tệp/audio, TTS; E2E hai trình duyệt; đo tải |

## Quy trình làm việc

- Nhánh `main` được bảo vệ; mỗi việc làm trên nhánh `feat/<dịch-vụ>-<mô-tả>` và gộp qua Pull Request
  có ít nhất một người review.
- Commit theo Conventional Commits, ví dụ `feat(social): accept friend request`.
- Mọi thay đổi Contract API sửa `webchat/contracts/*.yaml` trước, rồi mới sửa code.
