# Thiết kế API — Module Auth

> Tài liệu này ghi lại các endpoint, cách gọi, dữ liệu trả về và các quy tắc nghiệp vụ cần tuân thủ phía backend.

---

## POST /auth/login

Xác thực người dùng bằng email và mật khẩu.

**Request body**
```json
{
  "email": "gv.nguyenvana@school.edu.vn",
  "password": "Matkhau123"
}
```

**Response 200 OK**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "user": {
    "user_id": "usr_7k3m9x",
    "full_name": "Nguyễn Văn A",
    "email": "gv.nguyenvana@school.edu.vn",
    "role_id": 2,
    "must_change_password": false
  }
}
```
Đồng thời server set cookie `refresh_token` (HttpOnly, Path=/auth).

**Khi đăng nhập thất bại** (email không tồn tại hoặc sai mật khẩu):
```json
{ "detail": "Email hoặc mật khẩu không đúng" }
```
Status: 401. Thông báo chung, không nói rõ cái nào sai.

**Lỗi định dạng** (email sai chuẩn):
```json
{ "detail": [{ "loc": ["body", "email"], "msg": "value is not a valid email address" }] }
```
Status: 422 — FastAPI tự sinh lỗi validation.

**Nghiệp vụ cần nhớ**:
- Token access sống 30 phút, refresh sống 7 ngày.
- Client dựa vào `must_change_password` để quyết định chuyển trang hay ở lại.
- Refresh token nằm trong cookie, không nằm trong body.

---

## POST /auth/change-password

Đổi mật khẩu của chính tài khoản đang đăng nhập.

**Request body**
```json
{
  "current_password": "Matkhaucu",
  "new_password": "MatkhauMoi@2026",
  "confirm_password": "MatkhauMoi@2026"
}
```

**Response 200 OK** — giống hệt response của /auth/login, kèm access_token mới và user info cập nhật.

**Lỗi mật khẩu cũ sai**:
```json
{ "detail": "Mật khẩu hiện tại không đúng" }
```
Status: 401.

**Lỗi mật khẩu mới trùng cũ**:
```json
{ "detail": "Mật khẩu mới phải khác mật khẩu hiện tại" }
```
Status: 400.

**Lỗi xác nhận không khớp**:
```json
{
  "detail": [{
    "loc": ["body", "confirm_password"],
    "msg": "Mật khẩu xác nhận không khớp với mật khẩu mới"
  }]
}
```
Status: 422.

**Nghiệp vụ cần nhớ**:
- Sau khi đổi thành công, `token_version` tăng 1 → mọi phiên cũ trên mọi thiết bị bị vô hiệu ngay.
- Cờ `must_change_password` reset về false.
- Endpoint này chỉ tác động lên chính tài khoản của người gọi, không thể đổi mật khẩu người khác.

**Yêu cầu**: Bearer token hợp lệ trong Authorization header.

---

## POST /auth/logout

Kết thúc phiên đăng nhập trên thiết bị hiện tại.

**Request body**: rỗng.

**Response**: `204 No Content` — không có body. Cookie `refresh_token` bị xoá ở phía server.

**Điểm quan trọng**: logout **không** tăng `token_version`, nên các thiết bị khác vẫn đăng nhập bình thường.

**Yêu cầu**: Không bắt buộc có access token (vì mục đích là xoá cookie, nên gọi được kể cả khi token đã hết hạn).

---

## POST /auth/refresh

Lấy access token mới mà không cần đăng nhập lại.

**Request body**: rỗng (đọc từ cookie).

**Response 200 OK**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer"
}
```

**Lỗi hết hạn hoặc chữ ký sai**:
```json
{ "detail": "Phiên đăng nhập đã hết hạn" }
```
Status: 401.

**Lỗi token bị thu hồi** (token_version không khớp):
```json
{ "detail": "Phiên đăng nhập không còn hợp lệ" }
```
Status: 401. Đồng thời xoá cookie refresh_token.

**Nghiệp vụ cần nhớ**:
- Refresh token **không bị thay đổi** sau mỗi lần gọi (không rotate).
- Client nên gọi refresh trước khi access token hết hạn, không phải đợi nhận 401 rồi mới refresh.

**Yêu cầu**: Cookie `refresh_token` hợp lệ. Không cần Authorization header.

---

## POST /teachers

Tạo tài khoản giáo viên. Chỉ admin được phép.

**Request body**
```json
{
  "full_name": "Trần Thị B",
  "email": "gv.tranthib@school.edu.vn"
}
```

**Response 201 Created**
```json
{
  "user_id": "usr_9p2n4x",
  "full_name": "Trần Thị B",
  "email": "gv.tranthib@school.edu.vn",
  "role_id": 2,
  "temporary_password": "Kx8mNp3qR"
}
```

**Lỗi email đã tồn tại**:
```json
{ "detail": "Email đã tồn tại" }
```
Status: 400.

**Nghiệp vụ cần nhớ**:
- `role_id` luôn cố định = 2, bỏ qua bất kỳ giá trị nào client truyền lên.
- `must_change_password` luôn = true.
- Mật khẩu tạm chỉ trả về **đúng một lần** — sau khi tạo, không ai xem lại được.
- Mật khẩu tạm gửi cho giáo viên qua kênh riêng (email nội bộ, tin nhắn, tra tay).

**Yêu cầu**: Bearer token với role = ADMIN.

---

## POST /students

Tạo tài khoản sinh viên đơn lẻ.

**Request body**
```json
{
  "full_name": "Lê Văn C",
  "email": "sv.levanc@school.edu.vn"
}
```

**Response 201 Created** — cấu trúc y hệt `/teachers`, khác mỗi `role_id = 3` và `temporary_password` khác.

**Lỗi tương tự** `/teachers`.

**Yêu cầu**: Bearer token với role = ADMIN hoặc TEACHER.

---

## POST /students/import

Tạo nhiều tài khoản sinh viên từ file Excel.

**Request**: `multipart/form-data`, field `file` chứa file .xlsx.

File Excel có 2 cột bắt buộc: `full_name`, `email`.

**Response 201 Created**
```json
{
  "created": [
    {
      "user_id": "usr_1a2b3c",
      "full_name": "Phạm Duy E",
      "email": "sv.phamduye@school.edu.vn",
      "temporary_password": "Ab7cDx2k"
    },
    {
      "user_id": "usr_4d5e6f",
      "full_name": "Hoàng Thị F",
      "email": "sv.hoangthif@school.edu.vn",
      "temporary_password": "Ym9nPq8t"
    }
  ],
  "errors": [
    {
      "row": 3,
      "email": "sv.email_sai_dinh_dang",
      "reason": "Định dạng email không hợp lệ"
    },
    {
      "row": 7,
      "email": "sv.trunglap@school.edu.vn",
      "reason": "Email đã tồn tại trong hệ thống"
    }
  ]
}
```

**Lỗi file sai định dạng**:
```json
{ "detail": "File phải có định dạng .xlsx và chứa cột 'full_name' và 'email'" }
```
Status: 422.

**Lỗi file rỗng hoặc vượt giới hạn**:
```json
{ "detail": "File rỗng hoặc vượt quá 500 dòng cho phép" }
```
Status: 400.

**Nghiệp vụ cần nhớ**:
- Xử lý best-effort: dòng hợp lệ vẫn tạo, dòng lỗi vẫn báo. Ngay cả khi `created` rỗng → vẫn trả 201 kèm danh sách lỗi đầy đủ.
- Mỗi dòng hợp lệ sinh mật khẩu tạm **riêng**.
- Giới hạn 500 dòng mỗi lần upload.

**Yêu cầu**: Bearer token với role = ADMIN hoặc TEACHER.

---

## POST /users/{user_id}/reset-password

Đặt lại mật khẩu cho một tài khoản bất kỳ.

**Path param**: `user_id` — ví dụ `usr_9p2n4x`

**Request body**: rỗng.

**Response 200 OK**
```json
{
  "user_id": "usr_9p2n4x",
  "temporary_password": "Wz3rTm7nP"
}
```

**Lỗi tài khoản không tồn tại**:
```json
{ "detail": "Không tìm thấy tài khoản" }
```
Status: 404.

**Lỗi không có quyền** (giáo viên cố reset tài khoản giáo viên khác):
```json
{ "detail": "Bạn không có quyền đặt lại mật khẩu cho tài khoản này" }
```
Status: 403.

**Nghiệp vụ cần nhớ**:
- Sau reset, `must_change_password = true` → người dùng bị buộc đổi mật khẩu ở lần đăng nhập tiếp theo.
- `token_version` tăng 1 → toàn bộ phiên cũ của tài khoản mục tiêu bị vô hiệu trên mọi thiết bị.
- Admin: reset được mọi tài khoản. Giáo viên: chỉ reset được tài khoản sinh viên (`role_id = 3`).

**Yêu cầu**: Bearer token với role = ADMIN hoặc (TEACHER + tài khoản mục tiêu có role_id = 3).

---

## Bảng tổng hợp quyền gọi API

| Endpoint | ADMIN | TEACHER | STUDENT | Không đăng nhập |
|---|---|---|---|---|
| POST /auth/login | ✅ | ✅ | ✅ | ✅ |
| POST /auth/change-password | ✅ | ✅ | ✅ | ❌ |
| POST /auth/logout | ✅ | ✅ | ✅ | ✅* |
| POST /auth/refresh | ✅ | ✅ | ✅ | ✅* |
| POST /teachers | ✅ | ❌ | ❌ | ❌ |
| POST /students | ✅ | ✅ | ❌ | ❌ |
| POST /students/import | ✅ | ✅ | ❌ | ❌ |
| POST /users/{id}/reset-password | ✅ | ✅ (chỉ sv) | ❌ | ❌ |

\* Có thể gọi được nếu cookie còn, không cần token.

---

## Các quy tắc validation chung

**Email**: phải đúng định dạng RFC 5322 (thư viện Pydantic tự xử).

**Mật khẩu mới**: tối thiểu 8 ký tự, có ít nhất 1 chữ hoa, 1 chữ thường, 1 số.

**full_name**: không rỗng, tối đa 100 ký tự, không chứa ký tự đặc biệt ngoài dấu cách và dấu.

**user_id format**: chuỗi bắt đầu bằng `usr_` theo sau là 7 ký tự alphanumeric ngẫu nhiên (tạo bằng `secrets`).

---

## Cấu trúc lỗi chuẩn

Mọi lỗi nghiệp vụ trả về JSON:
```json
{ "detail": "Mô tả lỗi bằng tiếng Việt, rõ ràng" }
```

Lỗi validation (422) do FastAPI tự sinh, có cấu trúc phức tạp hơn — giữ nguyên mặc định của Pydantic.

Không trả lỗi 500 (Internal Server Error) ra ngoài — ghi log phía server, trả về body rỗng hoặc `{"detail": "Lỗi hệ thống"}`.
