# Đặc tả Use-case — Module Auth

> Bản ghi chú làm việc nội bộ. Mỗi use-case mô tả hành vi mong muốn của hệ thống từ góc nhìn người dùng.

---

## 1. UC-AUTH-01: Đăng nhập

**Ai dùng**: Admin, Giáo viên, Sinh viên

**Mục đích**: Xác thực danh tính để nhận quyền truy cập hệ thống.

**Luồng chính**:
Người dùng mở trang đăng nhập, nhập email và mật khẩu rồi nhấn nút Đăng nhập. Hệ thống tìm tài khoản theo email. Nếu không tìm thấy hoặc mật khẩu sai, hiển thị thông báo "Email hoặc mật khẩu không đúng" — không cho biết cụ thể cái nào sai để tránh brute-force. Nếu đúng, hệ thống cấp access token (30 phút) và refresh token (7 ngày), trả về thông tin người dùng kèm role.

**Lưu ý đặc biệt**:
- Cờ `must_change_password` quyết định người dùng có bị chuyển sang trang đổi mật khẩu bắt buộc hay vào dashboard thẳng.
- Refresh token được lưu trong HttpOnly cookie, không nằm trong response body.

**Từ chối đăng nhập khi**: tài khoản bị khoá (status = inactive), token_version trong DB không khớp token đang dùng.

---

## 2. UC-AUTH-02: Đổi mật khẩu

**Ai dùng**: Mọi người dùng đã đăng nhập

**Mục đích**: Thay đổi mật khẩu vì lý do bảo mật hoặc do bị buộc đổi sau lần đầu đăng nhập.

**Luồng chính**:
Người dùng nhập mật khẩu hiện tại, mật khẩu mới, rồi nhập lại mật khẩu mới để xác nhận. Hệ thống kiểm tra: mật khẩu mới phải khác mật khẩu cũ (không cho phép đổi giống y), mật khẩu mới phải đủ mạnh (tuỳ quy định của đề tài), mật khẩu mới phải trùng với ô xác nhận. Nếu mật khẩu cũ sai, hệ thống báo "Mật khẩu hiện tại không đúng". Khi đổi thành công, hệ thống tăng `token_version` lên 1 — điều này khiến mọi token cũ trên mọi thiết bị của tài khoản đó trở nên vô hiệu ngay lập tức, buộc đăng nhập lại ở những nơi khác.

**Sau khi đổi**: cờ `must_change_password` được đặt về false, người dùng không còn bị buộc đổi mật khẩu nữa.

---

## 3. UC-AUTH-03: Đăng xuất

**Ai dùng**: Mọi người dùng đã đăng nhập

**Mục đích**: Kết thúc phiên làm việc trên thiết bị hiện tại, không ảnh hưởng thiết bị khác.

**Luồng chính**:
Người dùng nhấn nút Đăng xuất. Hệ thống xoá cookie refresh_token (đặt Expires=0). Access token phía client tự huỷ (client tự xoá khỏi bộ nhớ). Điểm quan trọng: logout **không thay đổi** `token_version`, nên nếu ai đó lén copy token cũ trước khi logout thì token đó vẫn còn hiệu lực cho đến khi hết hạn hoặc bị invalid khi đổi mật khẩu. Đây là trade-off — nếu muốn logout invalid token ngay thì tăng token_version, nhưng sẽ ảnh hưởng các thiết bị khác đang đăng nhập.

---

## 4. UC-AUTH-04: Làm mới phiên (Refresh token)

**Ai dùng**: Hệ thống (gọi tự động mỗi khi access token sắp hết hạn)

**Mục đích**: Cấp access token mới mà không yêu cầu người dùng đăng nhập lại.

**Luồng chính**:
Client phát hiện access token sắp hết hạn (hoặc nhận HTTP 401 từ API) → tự động gửi request đến endpoint refresh kèm cookie refresh_token. Hệ thống kiểm tra chữ ký JWT và thời hạn của refresh_token. Nếu hết hạn hoặc chữ ký sai → trả 401, client chuyển sang trang đăng nhập. Nếu hợp lệ → kiểm tra `token_version` trong token có khớp với DB không. Nếu không khớp (tài khoản vừa đổi mật khẩu ở nơi khác) → trả 401, xoá cookie, chuyển đăng nhập. Nếu khớp → cấp access token mới, giữ nguyên refresh token (không rotate).

**Lưu ý**: refresh_token không bị thay đổi sau mỗi lần refresh — đây là thiết kế đơn giản hoá. Nếu muốn an toàn hơn (refresh token rotation), mỗi lần refresh sẽ sinh refresh_token mới và revoke token cũ.

---

## 5. UC-AUTH-05: Cấp phát tài khoản Giáo viên

**Ai dùng**: Admin

**Mục đích**: Tạo tài khoản cho giáo viên mới, gửi mật khẩu tạm cho người dùng qua kênh riêng.

**Luồng chính**:
Admin mở trang quản lý tài khoản, chọn "Cấp phát Giáo viên", nhập họ tên và email. Hệ thống kiểm tra email chưa tồn tại (nếu trùng → báo lỗi). Khi hợp lệ, hệ thống sinh mật khẩu tạm ngẫu nhiên (8-12 ký tự, không dùng mật khẩu yếu), băm và lưu vào DB. Tài khoản được gán `role_id = 2` và `must_change_password = true`. Mật khẩu tạm **chỉ hiển thị đúng 1 lần** trong response — sau đó không ai (kể cả admin) có thể xem lại được. Admin gửi mật khẩu này cho giáo viên qua email nội bộ, tin nhắn, hoặc tra tay.

**Giới hạn quyền**: chỉ Admin mới được tạo tài khoản giáo viên. Giáo viên không có quyền tạo tài khoản giáo viên khác.

---

## 6. UC-AUTH-06: Cấp phát tài khoản Sinh viên

**Ai dùng**: Admin hoặc Giáo viên

**Mục đích**: Tương tự UC-05 nhưng cho sinh viên, và mở rộng quyền cho giáo viên.

**Luồng chính**:
Giống hệt UC-05 nhưng `role_id = 3`. Điểm khác: Giáo viên được phép tạo tài khoản sinh viên (phục vụ trường hợp giáo viên chủ nhiệm tự cấp tài khoản cho lớp mình). Admin thì được tạo cả giáo viên lẫn sinh viên.

---

## 7. UC-AUTH-07: Nhập danh sách Sinh viên từ Excel

**Ai dùng**: Admin hoặc Giáo viên

**Mục đích**: Tạo nhiều tài khoản sinh viên cùng lúc thay vì nhập tay từng người.

**Luồng chính**:
Người dùng tải file Excel (.xlsx) gồm 2 cột: họ tên và email. Hệ thống đọc từng dòng, kiểm tra định dạng từng email và xem có trùng trong DB không. Dòng nào lỗi → bỏ qua, ghi vào danh sách lỗi. Dòng nào hợp lệ → tạo tài khoản sinh viên với mật khẩu tạm riêng cho mỗi người.

**Xử lý lỗi một phần**: Không fail toàn bộ request khi có vài dòng lỗi. Nếu cả file 100 dòng đều lỗi → vẫn trả 201 kèm danh sách 100 lỗi, không phải lỗi HTTP. Lỗi HTTP 422 chỉ xảy ra khi file sai định dạng hoàn toàn (không phải .xlsx, thiếu cột bắt buộc).

**Giới hạn**: tối đa 500 dòng mỗi lần upload.

---

## 8. UC-AUTH-08: Đặt lại mật khẩu

**Ai dùng**: Admin (mọi tài khoản) hoặc Giáo viên (chỉ tài khoản sinh viên)

**Mục đích**: Cấp mật khẩu mới khi người dùng quên hoặc cần reset bắt buộc.

**Luồng chính**:
Người dùng chọn tài khoản cần reset trong danh sách, nhấn nút Đặt lại mật khẩu, xác nhận hành động. Nếu là Giáo viên đang thực hiện và tài khoản mục tiêu không phải sinh viên → hệ thống từ chối với thông báo "Bạn không có quyền đặt lại mật khẩu cho tài khoản này". Nếu hợp lệ → hệ thống sinh mật khẩu tạm mới, cập nhật vào DB, tăng `token_version` (vô hiệu mọi phiên đăng nhập cũ của tài khoản đó), đặt `must_change_password = true`.

**Lưu ý**: Giáo viên không thể reset mật khẩu của giáo viên khác hay của admin — đây là cơ chế phân quyền quan trọng.

---

## 9. Yêu cầu kỹ thuật chung cho toàn module

**Mật khẩu tạm**:
- Độ dài 8-12 ký tự, gồm chữ hoa + chữ thường + số
- Sinh ngẫu nhiên bằng thư viện chuẩn ( secrets hoặc random)
- **Không bao giờ** trả về mật khẩu gốc trong bất kỳ API response nào
- Chỉ lưu hash (bcrypt/argon2), không lưu plaintext

**JWT token**:
- Thuật toán HS256
- Payload chứa: user_id, email, role_id, tv (token_version)
- Access token: 30 phút
- Refresh token: 7 ngày, lưu trong HttpOnly cookie với Secure flag (production)

**Token version**:
- Số nguyên, bắt đầu từ 0
- Tăng 1 mỗi khi: đổi mật khẩu, reset mật khẩu bắt buộc từ admin
- Mỗi lần verify token, so sánh `tv` trong JWT với `tv` trong DB
- Khớp → token hợp lệ; không khớp → reject với 401

**Rate limiting** (nếu có thời gian):
- Đăng nhập: 5 lần/phút/IP
- Refresh token: 10 lần/giờ/tài khoản

**Thứ tự ưu tiên triển khai**: UC-AUTH-01 (đăng nhập) → UC-AUTH-04 (refresh) → UC-AUTH-03 (đăng xuất) → UC-AUTH-02 (đổi mật khẩu) → UC-AUTH-05/06/07/08 (quản lý tài khoản).
