# SOCIAL APIs

![Status](https://img.shields.io/badge/SOCIAL_APIS-green) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Mục lục

## Instagram Graph API

### Tóm tắt chung
- Instagram Graph API cho phép các nhà phát triển quản lý tài khoản Instagram doanh nghiệp (business) và người sáng tạo nội dung (creator).
- API này sử dụng Facebook Login OAuth 2.0.
- Các endpoints dựa trên GraphQL để truy xuất dữ liệu về người dùng, bài đăng, bình luận, và nhiều hơn nữa.

### Các tính năng chính
- Xuất bản nội dung (ảnh, video, reels, ...).
- Lấy thông tin chi tiết và phân tích về phương tiện.
- Quản lý bình luận, đề cập.
- Nhắn tin trực tiếp (Instagram Graph API + Messenger Platform).
- Theo dõi hashtag, đề cập.
- Quản lý stories.
- Thẻ mua sẵm, sản phẩm.

| Tính năng | Mô tả |
|---|---|
| **API dựa trên đồ thị (Graph)** | Truy cập tài nguyên dựa trên nút |
| **OAuth 2.0** | Xác thực bằng Facebook Login |
| **Webhook** | Thông báo real-time cho bình luận, đề cập |
| **Giới hạn tốc độ** | 200 lượt gọi/giờ/ứng dụng |
| **Xuất bản nội dung** | Ảnh, video. reels,... |
| **Thông tin chi tiết (Insights)** | Chỉ số tương tác, phạm vi tiếp cận, lượt hiển thị |
| **Kiểm duyệt** | Quản lý bình luận, đề cập, tin nhắn |

Lưu ý: API này chỉ hỗ trợ tài khoản Instagram doanh nghiệp và người sáng tạo nội dung, không hỗ trợ tài khoản cá nhân.

### API sử dụng endpoint: https://graph.facebook.com/v18.0/

### Cách sử dụng

#### Bước 1: Tạo tài khoản nhà phát triển Facebook
1. Truy cập [Facebook for Developers](https://developers.facebook.com/) và đăng ký tài khoản nhà phát triển.
2. Đăng nhập Facebook.
3. Tạo ứng dụng Facebook (loại Business).
4. Thêm sản phẩm Instagram Graph API.

#### Bước 2: Liên kết tài khoản Instagram doanh nghiệp
1. Vào Cài đặt Trang Facebook -> Instagram.
2. Nhấn **Kết nối tài khoản**.
3. Đăng nhập Instagram và xác nhận ủy quyền.
4. Đảm bảo Instagram Business đã liên kết.

#### Bước 3: Lấy Access Token ngắn hạn -> dài hạn

#### Bước 4: Lấy ID tài khoản Instagram doanh nghiệp

#### Bước 5: Thực hiện các cuộc gọi API đã xác thực