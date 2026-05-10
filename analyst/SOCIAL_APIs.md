# SOCIAL APIs

![Status](https://img.shields.io/badge/SOCIAL_APIS-green) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Mục lục

* [Instagram Graph API](#instagram-graph-api)
    - [Tóm tắt chung](#tóm-tắt-chung-instagram)
    - [Các tính năng chính](#các-tính-năng-chính-instagram)
    - [Cách sử dụng](#cách-sử-dụng-instagram)
* [TikTok API for Business](#tiktok-api-for-business)
    - [Tóm tắt chung](#tóm-tắt-chung-tiktok)
    - [Các tính năng chính](#các-tính-năng-chính-tiktok)
    - [Cách sử dụng](#cách-sử-dụng-tiktok)
* [Astream](#astream)
    - [Tóm tắt chung](#tóm-tắt-chung-astream)
    - [Các tính năng chính](#các-tính-năng-chính-astream)
    - [Bản chất](#bản-chất-astream)

## Instagram Graph API

<div id="tóm-tắt-chung-instagram"></div>

### Tóm tắt chung
- Instagram Graph API cho phép các nhà phát triển quản lý tài khoản Instagram doanh nghiệp (business) và người sáng tạo nội dung (creator).
- API này sử dụng Facebook Login OAuth 2.0.
- Các endpoints dựa trên GraphQL để truy xuất dữ liệu về người dùng, bài đăng, bình luận, và nhiều hơn nữa.

<div id="các-tính-năng-chính-instagram"></div>

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

<div id="cách-sử-dụng-instagram"></div>

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

## TikTok API for business

<div id="tóm-tắt-chung-tiktok"></div>

### Tóm tắt chung
TikTok API for Business là bộ API do TikTok cung cấp, giúp nhà phát triển tích hợp với hệ sinh thái TikTok để quản lý quảng cáo, tài khoản TikTok và làm việc với creator/influencer.

<div id="các-tính-năng-chính-tiktok"></div>

### Các tính năng chính
- **TikTok Login Kit**: Cho phép đăng nhập các ứng dụng bên thứ 3 bằng tài khoản TikTok.
- **TikTok Share to TikTok SDK**: Cho phép chia sẻ nội dung từ các ứng dụng khác trực tiếp lên TikTok.
- **TikTok API for Developers** bao gồm 3 nhóm API chính:
- **Marketing API**
  Dùng để quản lý và tối ưu chiến dịch quảng cáo TikTok: tạo quảng cáo, theo dõi hiệu suất, tự động hóa vận hành và mở rộng quy mô marketing.
- **Organic API**
  Hỗ trợ phân tích và tối ưu nội dung organic (không trả phí), đánh giá hiệu quả video, tìm cơ hội hợp tác creator và quyết định nội dung nào nên chạy quảng cáo.
- **Business Messaging API**
  Cho phép xây dựng hệ thống nhắn tin với người dùng trên TikTok theo thời gian thực để tăng tương tác khách hàng và hỗ trợ chiến dịch marketing/quảng cáo.

<div id="cách-sử-dụng-tiktok"></div>

### Cách sử dụng

#### Bước 1: Truy cập Cổng Thông Tin Dành Cho Nhà Phát Triển Của TikTok [TikTok for Developers](https://developers.tiktok.com/)

#### Bước 2: Đăng ký ứng dụng
Bạn cần tạo một ứng dụng trên cổng thông tin nhà phát triển và mô tả rõ mục đích sử dụng API

#### Bước 3: Yêu cầu quyền truy cập
Chọn các quyền (scopes) phù hợp với nhu cầu của ứng dụng (ví dụ: quyền đọc dữ liệu người dùng, quyền đăng video, v.v.).

#### Bước 4: Chờ xét duyệt
TikTok sẽ xem xét yêu cầu của bạn. Quá trình này có thể mất thời gian và đòi hỏi ứng dụng của bạn phải tuân thủ các chính sách của TikTok.

#### Bước 5: Tích hợp và Phát triển
Sau khi được cấp quyền, bạn có thể bắt đầu tích hợp API vào hệ thống của mình bằng cách sử dụng access tokens và API endpoints được cung cấp.

## Astream

<div id="tóm-tắt-chung-astream"></div>

### Tóm tắt chung
Astream là một nền tảng/phần mềm phục vụ Influencer Marketing và Social Media Analytics.
Nói đơn giản, Astream giúp doanh nghiệp:

- Tìm kiếm influencer/KOL phù hợp.
- Phân tích dữ liệu tài khoản social.
- Quản lý chiến dịch booking KOL.
- Đo hiệu quả marketing trên social media.

<div id="các-tính-năng-chính-astream"></div>

### Các tính năng chính
#### Tìm kiếm & phân tích influencer
Có thể lọc influencer theo:

- Số follower
- Engagement
- Giới tính/độ tuổi follower
- Lĩnh vực quan tâm
- Brand affinity
- Location
- Hashtag performance

=> Giúp chọn KOL phù hợp với chiến dịch.

#### Social Analytics
Phân tích:

- Lượt like/comment/share
- Engagement rate
- Tăng trưởng follower
- Audience insight
- Hiệu quả bài đăng

#### Campaign Management
Quản lý:

- Danh sách influencer
- Gửi DM/email hàng loạt
- Trạng thái campaign
- Bài đăng PR
- Report hiệu quả

#### Reporting
Tự động tạo:

- Dashboard
- Biểu đồ
- Campaign report

<div id="bản-chất-astream"></div>

### Bản chất
Astream thực chất là một hệ thống SaaS xây dựng trên các social APIs như:

- Instagram Graph API
- TikTok API for Business
- YouTube API
- X API

Tức là:

- Instagram/TikTok API = API gốc do platform cung cấp.
- Astream = ứng dụng/business platform dùng các API đó để làm influencer marketing & analytics.