# AWS EVENT BRIDGE

![Status](https://img.shields.io/badge/AWS_EVENT_BRIDGE-orange) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Mục lục

- [AWS CloudWatch Events là gì](#aws-cloudwatch-events-là-gì)
- [AWS EventBridge là gì](#aws-eventbridge-là-gì)
- [EventBridge Schema Registry là gì](#eventbridge-schema-registry-là-gì)
- [Amazon EventBridge Scheduler là gì](#amazon-eventbridge-scheduler-là-gì)

## AWS CloudWatch Events là gì

CloudWatch Events cung cấp các event để mô tả những thay đổi resource trên AWS. Bằng cách sử dụng những rules, bạn có thể định nghĩa những target mà khi một event được gọi tới.

Ví dụ: Bạn có thể cài đặt một Event: nhận biết state của EC2 instance chuyển sang "pending". Target của event đó sẽ là một SNS Topic gửi mail đến người dùng.

- **Events**: là sự kiện xác định thay đổi resource trên môi trường AWS.
- **Rules**: định nghĩa những event và target khi event đó xảy ra.
- **Target**: nơi xử lý khi event xảy ra. Target có thể là: Lambda, SNS, Batch jobs, SQS...

## AWS EventBridge là gì

- **EventBridge** là giải pháp thay thế cho CloudWatch Events. Là một dịch vụ serverless của AWS, giúp bạn dễ dàng kết nối các ứng dụng thông qua các sự kiện (event-driven). EventBridge hỗ trợ lập lịch công việc và chuyển tiếp sự kiện từ nhiều nguồn khác nhau (AWS services, SaaS apps, custom applications) đến các mục tiêu như Lambda, SQS, Step Functions..
- **Default event bus**: được tạo ra bởi AWS (giống với CloudWatch Events).
- **Partner event bus**: tiếp nhập event từ một dịch vụ SaaS hoặc một Application (DataDog, Zendesk...).
- **Custom Event Buses**: custom để sử dụng cho application của riêng bạn.

## EventBridge Schema Registry là gì

- EventBridge có thể phân tích các events.
- EventBridge Schema Registry lưu trữ các cấu trúc của event hoặc schema.
- Khi bạn đã có Schema cho một event, EventBridge sẽ tạo ra những source code sample. Bạn có thể download code ở nhiểu ngôn ngữ (java, js...) để tiếp tục phát triển thêm.

## Amazon EventBridge Scheduler là gì

Amazon EventBridge Scheduler là một tính năng được tích hợp sẵn trong EventBridge, cho phép lập lịch và thực thi các công việc định kỳ hoặc vào các thời điểm cụ thể trong tương lai. Nó giúp tự động hóa các tác vụ như gửi thông báo, kích hoạt Lambda function, chạy Step Functions, và gửi sự kiện đến các dịch vụ khác trong AWS theo lịch trình được xác định trước.

Các tính năng chính:

- Lập lịch công việc định kỳ.
- Hỗ trợ các loại khác nhau: Rate-based, Cron-based, One-time.
- Tích hợp với các dịch vụ AWS như Lambda, Step Functions, SQS. ...
- Quản lý lịch trình dễ dàng qua AWS Management Console, SDK, CLI, API.

EventBridge Scheduler hỗ trợ **3** loại schedule:

- **Rate-based schedule** (Định kỳ theo chu kỳ).
- **Cron-based schedule** (Định kỳ theo lịch cụ thể).
- **One-time schedule** (Chạy một lần vào thời điểm cụ thể).