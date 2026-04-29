# AWS LAMBDA

![Status](https://img.shields.io/badge/AWS_LAMBDA-orange) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Mục lục

- [Giới thiệu Lambda](#giới-thiệu-lambda)
  - [Lambda là gì](#lambda-là-gì)
  - [Lợi ích của Lambda](#lợi-ích-của-lambda)
  - [Lambda hoạt động như thế nào](#lambda-hoạt-động-như-thế-nào)
  - [Kiến trúc AWS Lambda](#kiến-trúc-aws-lambda)
  - [Giới hạn & Tinh chỉnh hiệu suất AWS Lambda](#giới-hạn--tinh-chỉnh-hiệu-suất-aws-lambda)
  - [Ứng dụng thực tế của AWS Lambda](#ứng-dụng-thực-tế-của-aws-lambda)
  - [Lambda@Edge là gì](#lambdaedge-là-gì)

## Giới thiệu Lambda

### Lambda là gì

Hiện nay, các giải pháp serverless ngày càng trở nên phổ biến. Đây là công nghệ mà lập trình viên chỉ cần thực hiện quá trình viết code mà không cần lo lắng đến việc thiết lập server hay chạy các đoạn code đã viết. Đây được coi là lời giải cho bài toán chi phí cũng như giải pháp cho quá trình vận hành, khi lập trình viên chỉ cần tập trung vào việc hoàn thành các đoạn code.

AWS Lambda là một dịch vụ tính toán nơi mà bạn có thể upload code của mình lên, và AWS Lambda sẽ giúp bạn chạy đoạn code đó bằng việc sử dụng các tài nguyên sẵn có của AWS. Sau khi bạn upload code, và bạn tạo ra một Lambda function, AWS sẽ cung cấp và quản lý các server mà bạn sử dụng để chạy code.

- Là virtual functions (không cần quản lý server).
- Giới hạn thời gian execute (short execute).
- Run on-demand.
- Scaling tự động.

---

### Lợi ích của Lambda

- Chi phí rẻ:
  - Trả tiền theo request và compute time.
  - Mỗi tháng sẽ có 1,000,000 request free và 400,000 GBs compute time (có nghĩa: nếu bạn sử dụng RAM 1GB thì bạn sẽ có free 400,000s execute).
- Dễ dàng tích hợp với những dịch vụ khác của AWS.
- Hỗ trợ nhiều ngôn ngữ lập trình: Java, Javascript, Python...
- Dễ dàng scale resource trên mỗi function.
- Tăng RAM sẽ cải thiện CPU và Network.

---

### Lambda hoạt động như thế nào

Lambda có 2 phần chính: **Lambda Function** và **Event Source**

- **Event Source**: là nơi phát sinh ra sự kiện để gọi đến Lambda function.
- **Lambda Function**: function thực thi code logic của bạn. Bạn chỉ cần upload source code lên, phần còn lại AWS sẽ quản lý.

---

### Kiến trúc AWS Lambda

AWS Lambda có thể hoạt động như một phần trong kiến trúc microservices, kết hợp với các dịch vụ AWS khác:

- API Gateway + Lambda: Xây dựng API serverless, loại bỏ nhu cầu về backend truyền thống.
- S3 + Lambda: Xử lý ảnh, video, hoặc dữ liệu khi được tải lên S3.
- DynamoDB + Lambda: Tự động xử lý sự kiện khi dữ liệu trong DynamoDB thay đổi.
- Kinesis + Lambda: Phân tích dữ liệu thời gian thực từ các nguồn khác nhau.
- Step Functions + Lambda: Xây dựng luồng công việc (workflow) phức tạp.

---

### Giới hạn & Tinh chỉnh hiệu suất AWS Lambda

- Memory allocation: 128MB - 10 GB.
- Maximum timeout: 900s (15min).
- Environment variables: 4KB.
- Source deployment (file .zip): tối đa 50MB.
- Có thể sử dụng folder /tmp để chứa những file tạm thời.
- Tối ưu thời gian khởi động (Cold Start): Sử dụng Provisioned Concurrency để giảm độ trễ khi khởi chạy Lambda.
- Giảm kích thước package: Sử dụng tree-shaking và tối giản dependency để giảm thời gian tải mã nguồn.
- Chia nhỏ chức năng: Thiết kế Lambda theo hướng single responsibility để dễ bảo trì.

---

### Ứng dụng thực tế của AWS Lambda

- Xử lý dữ liệu theo thời gian thực: AWS Lambda có thể xử lý log, dữ liệu từ IoT hoặc phân tích dữ liệu từ Kinesis Stream.
- Tích hợp với API Gateway: Xây dựng API serverless mà không cần quản lý backend.
- Tự động hóa tác vụ: Kích hoạt Lambda từ cron jobs hoặc sự kiện trên S3 để xử lý ảnh, video.
- Hỗ trợ CI/CD: Dùng Lambda để tự động triển khai hoặc kiểm thử mã nguồn.
- Xây dựng chatbot: Lambda có thể kết hợp với Lex hoặc Telegram API để tạo chatbot thông minh

---

### Lambda@Edge là gì

Lambda@Edge cho phép bạn chạy các hàm của Lamda để customize nội dung mà CloudFront deliver tới người dùng, cụ thể hơn 1 chút nó cho phép chạy các function ở AWS location gần với người dùng nhất từ đó nâng cao tốc độ vận chuyển content.

Bạn có thể chạy function của Lambda để thay đổi CloudFront request và CloudFront response ở các thời điểm sau:

- Sau khi CloudFront nhận request từ viewer (viewer request).
- Trước khi CloudFront chuyển tiếp request tới origin (origin request).
- Sau khi CloudFront nhận response từ origin (origin response).
- Trước khi CloudFront chuyển tiếp response tới viewer (viewer response).