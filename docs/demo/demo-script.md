# Kịch bản demo – luồng trọng tâm (mục 1.4)

Luồng: **Long kết bạn với Lâm → mở chat riêng → gửi tin qua WebSocket → Lâm vào lại, xem tóm tắt tin
chưa đọc → đọc lịch sử và cập nhật mốc đọc.** Thứ tự giống bảng ở mục 1.4 của báo cáo; mốc đọc và tóm
tắt thuộc MVP lõi nên nằm trong luồng demo.
Toàn bộ chạy trên Prism mock sinh từ `openapi.yaml`, nên mỗi response là example trong hợp đồng.
Thời lượng dự kiến: khoảng 7 phút.

## 0. Chuẩn bị (làm trước buổi review)

Chạy từ thư mục gốc repo, mỗi lệnh một cửa sổ terminal:

```bash
# 1) Mock server REST từ hợp đồng
npx @stoplight/prism-cli mock webchat/contracts/openapi.yaml -p 4010

# 2) Phục vụ Swagger UI demo (trang có ô chọn header Prefer)
npx http-server -p 8081 -c-1
```

- Mở http://localhost:8081/docs/demo/swagger.html
- Ở ô **Servers** chọn `http://localhost:4010 – Prism mock`.
- Bấm **Authorize** → `bearerAuth` → dán token bất kỳ, ví dụ `demo` (Prism chỉ kiểm tra có token).
- Mở sẵn tab https://studio.asyncapi.com/ và dán nội dung `webchat/contracts/asyncapi.bundled.yaml`
  (bản đã bundle, không còn tham chiếu sang `openapi.yaml`).

Tạo lại bản bundle khi sửa `asyncapi.yaml`:

```bash
npx @asyncapi/cli bundle webchat/contracts/asyncapi.yaml -o webchat/contracts/asyncapi.bundled.yaml
```

**Cách gọi response lỗi.** Prism mặc định trả mã 2xx nhỏ nhất. Gõ vào ô `Prefer` trên thanh đen đầu
trang, ví dụ `code=401`, `code=409, example=incomingRequestExists` hoặc `example=empty`, rồi bấm Execute. Dùng curl thì
thêm `-H "Prefer: code=401"`. Để trống ô này trước mỗi bước thành công.

ID cố định dùng trong example:

| Đối tượng | ID |
|---|---|
| Long | `a1000000-0000-4000-8000-000000000001` |
| Lâm | `a1000000-0000-4000-8000-000000000002` |
| Lời mời Long → Lâm | `f1000000-0000-4000-8000-000000000001` |
| Hội thoại riêng | `c1000000-0000-4000-8000-000000000001` |

## 1. Đăng nhập — `POST /api/v1/auth/login`

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | Execute với example `Long logs in` | 200 `TokenResponse`, `expires_in=3600`, `user.username=long` | `01a-login-200.png` |
| ❌ | `Prefer: code=401`, đổi password | 401 `INVALID_CREDENTIALS` | `01b-login-401.png` |

**Lời thoại:** "Sai username hay sai mật khẩu đều trả cùng một mã và một câu, để không lộ username nào
đã tồn tại. Client chỉ rẽ nhánh theo `error.code`."

## 2. Tìm Lâm và gửi lời mời

`GET /api/v1/users/by-username/{username}` với `Lam`: trả `lam`, vì username lưu chữ thường (`02a`).

`POST /api/v1/friend-requests`:

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | `Prefer: code=201`, example `Long invites Lâm` | 201, `status=pending` | `03a-invite-201.png` |
| ❌ | `Prefer: code=409, example=incomingRequestExists` | 409 `INCOMING_REQUEST_EXISTS`, `related_id` trỏ tới lời mời ngược chiều | `03b-invite-409.png` |
| ❌ | Logout trong Authorize rồi Execute | Prism tự trả 401 `UNAUTHENTICATED` | `03c-invite-no-token-401.png` |

**Lời thoại:** "Gửi lại cùng chiều trả 200 với lời mời cũ, nên client bấm hai lần không tạo bản ghi
trùng. Nếu Lâm đã mời Long trước, API không tạo lời mời mới mà chỉ cho client `related_id` để chấp
nhận luôn."

## 3. Chấp nhận — `PATCH /api/v1/friend-requests/{request_id}`

`request_id = f1000000-0000-4000-8000-000000000001`, body `{ "status": "accepted" }`.

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | Execute | 200, `status=accepted` | `04a-accept-200.png` |
| ❌ | `Prefer: code=403` | 403 `ACTION_FORBIDDEN`: người gửi không được tự chấp nhận | `04b-accept-403.png` |

**Lời thoại:** "Dùng PATCH trên trạng thái thay vì endpoint động từ `/accept`. Người nhận chọn
accepted hoặc declined, người gửi chỉ được cancelled."

## 4. Mở chat riêng — `PUT /api/v1/conversations/direct/{friend_id}`

`friend_id = a1000000-0000-4000-8000-000000000002`.

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | `Prefer: code=201` (Long mở lần đầu) | 201, `id=c1…01`, `peer=lam` | `05a-direct-201.png` |
| ✅ | Để trống Prefer (Lâm mở) | 200, **cùng `id`**, `peer=long` | — |
| ❌ | `Prefer: code=403` | 403 `FRIENDSHIP_REQUIRED` | `05b-direct-403.png` |

**Lời thoại:** "Dùng PUT vì thao tác là idempotent: A→B và B→A về cùng một hội thoại. Hai yêu cầu đến
cùng lúc vẫn chỉ ra một hội thoại nhờ ràng buộc UNIQUE trên cặp UUID đã chuẩn hóa."

## 5–6. Vào phòng và gửi tin — WebSocket (AsyncAPI Studio)

Chuyển sang tab AsyncAPI Studio (`06a`, `06b`, `06c`). Chỉ vào từng sự kiện theo thứ tự:

1. `session.authenticate` → `session.ready`: token gửi trong frame đầu tiên, trong 5 giây, **không** để trên URL.
2. `conversation.subscribe` → `conversation.subscribed` kèm `latest_seq`: Chat hỏi Social `can_read`.
3. `message.send` (example `Chào Lâm!`) → `message.ack` **sau khi commit**; các socket khác nhận `message.created`.
4. Lỗi: `error` với `code=FRIENDSHIP_REQUIRED`, `retryable=false` (example có sẵn).

**Lời thoại:** "Swagger UI không mô tả được WebSocket nên phần này nằm trong AsyncAPI. Payload
`Message` và `ErrorCode` dùng lại đúng schema của OpenAPI, nên REST và WebSocket trả cùng một hình
dạng tin nhắn."

## 7. Tóm tắt tin chưa đọc

Bối cảnh: sau tin "Chào Lâm!" ở bước 6, Long gửi tiếp tới seq 12 trong lúc Lâm chưa mở hội thoại. Lâm
vào lại. Authorize bằng token của Lâm (ví dụ `demo.lam`), `conversation_id = c1…01`.

`GET /api/v1/conversations/{conversation_id}/read-state`: giao diện biết có tin chưa đọc nên hiện nút
"Tóm tắt tin chưa đọc".

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | Execute | 200, `last_read_seq=0`, `latest_seq=12`, `unread_text_count=12` | `07a-readstate-unread.png` |

`POST /api/v1/conversations/{conversation_id}/unread-summary`, không có body:

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | Execute | 200 `status=ready`, `source_count=12`, nguồn seq 1–12, `last_read_seq` vẫn là 0 | `07b-summary-200.png` |
| ✅ | `Prefer: example=empty` | 200 `status=empty`, `summary=null`: Lâm đã đọc hết nên không gọi LLM | `07c-summary-empty.png` |
| ❌ | `Prefer: code=503` | 503 `SUMMARY_UNAVAILABLE`: dịch vụ LLM lỗi hoặc quá thời gian | `07d-summary-503.png` |

**Lời thoại:** "Khi nhận yêu cầu, server chốt mốc đọc và `snapshot_seq`, rồi chỉ lấy tin văn bản của
người khác trong khoảng đó, tối đa 100 tin mới nhất. Tóm tắt **không** dời mốc đọc: Lâm vẫn còn 12 tin
chưa đọc và tự quyết định có đọc tiếp hay không. Response trả kèm `source_from_seq` và `source_to_seq`
để giao diện ghi rõ bản tóm tắt dựa trên những tin nào. Bản tóm tắt dùng username vì `PublicUser` chỉ
có `username`."

## 8. Đọc lại lịch sử và cập nhật mốc đọc

`GET /api/v1/conversations/{conversation_id}/messages?limit=3`:

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | Execute | 200, seq 12, 11, 10 của long (seq giảm dần), `has_more=true`; nội dung khớp với bản tóm tắt | `08a-history-200.png` |
| ❌ | `Prefer: code=404` | 404 `CONVERSATION_NOT_FOUND` (người ngoài và id không tồn tại trả giống nhau) | `08b-history-404.png` |

`PUT /api/v1/conversations/{conversation_id}/read-state`, body `{ "through_seq": 12 }`:

| | Thao tác | Kết quả | Ảnh |
|---|---|---|---|
| ✅ | Execute | 200, `last_read_seq=12`, `unread_text_count=0` | `08c-readstate-200.png` |
| ❌ | `Prefer: code=422`, `through_seq: 99` | 422 `READ_AHEAD` | `08d-readstate-422.png` |

**Lời thoại:** "Đọc xong thì mốc đọc mới tăng, đúng thứ tự ở mục 1.4 của báo cáo. Mốc đọc chỉ tăng:
server lấy max(mốc cũ, through_seq), nên hai tab không kéo lùi nhau. Khi reconnect, client gọi
`after_seq=<seq cuối đã thấy>&through_seq=<latest_seq>` để lấy đúng phần thiếu rồi gộp theo `message.id`."

## Câu hỏi có thể gặp

- **Sao không dùng Postman?** Swagger UI đọc trực tiếp hợp đồng, nên demo chứng minh hợp đồng đủ để
  frontend làm việc trước khi có backend.
- **Prism trả về dữ liệu cố định, có kiểm tra logic không?** Không. Prism kiểm tra request theo schema
  (thiếu trường, sai UUID, thiếu token) và trả example. Logic nghiệp vụ được kiểm chứng ở tuần 13–15
  bằng pytest và Schemathesis trên backend thật.
- **Mỗi lần bấm tóm tắt đều gọi LLM à?** Chỉ khi có tin để tóm tắt. Không có tin thì trả `empty` mà
  không gọi LLM; mỗi lần gửi tối đa 100 tin, nhiều hơn thì `omitted_count` cho biết số tin cũ bị bỏ qua.
- **Sao tóm tắt dùng POST mà không dùng GET?** Mỗi lần gọi là một lần LLM tạo bản tóm tắt mới, kết quả
  có thể khác nhau và tốn chi phí, nên không phải thao tác đọc an toàn để cache.
- **Ảnh chụp ở đâu?** `docs/demo/screenshots/`, tên file theo số bước ở trên.
