# AWS SQS

![Status](https://img.shields.io/badge/AWS_SQS-orange) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Giới thiệu Amazon SQS - Standard Queues

### Amazon SQS là gì

Amazon Simple Queue Service (SQS) là một dịch vụ hàng đợi (queue) lưu trữ message nhanh chóng, đáng tin cậy, có khả năng mở rộng và quản lý một cách đầy đủ. Với SQS, bạn có thể gửi, nhận và lưu trữ message giữa các thành phần trong một phần mềm.

Attributes:

- Không giới hạn số lượng messages trong queue.
- Mặc định message có thể tồn tại trong 4 ngày, tối đa là 14 ngày.
- Độ trễ thấp khi gửi và nhận message.
- Mỗi message dung lượng tối đa là 256KB.

### Producing message trong SQS

- Sử dụng SDK để tạo mới message và gửi message đó đến SQS (SendMessage API).
- Message sẽ nằm trong SQS cho đến khi consumer nhận, xử lý và xóa nó đi khỏi queue.
- Message có thể nằm trong SQS 4 ngày và tối đa là 14 ngày.

### Consuming message trong SQS

- Consumers (EC2 instance, server, Lambda function...).
- Receive message từ SQS (có thể nhận tối đa 10 message từ SQS).
- Xử lý message nhận được.
- Xóa message sử dụng DeleteMessage API.

### SQS Multiple EC2 consumer

- Consumers nhận message parallel.
- Mỗi message chỉ được process một lần duy nhất.
- Consumers sẽ xóa message sau khi message đó được process.

### Bảo mật trong SQS

- **Encryption**:
  - In-flight encryption bằng cách sử dụng HTTPS.
  - Sử dụng KMS keys.
  - Client-side encryption.
- **Access Controls**: sử dụng IAM policies
- **SQS Access Policies** (tương tự như S3 bucket policies)

## Access Policy trong SQS Queue

### Cross Account Access là gì

![alt text](../image/cross-account-access.png)

Khi Account 2 muốn lấy message từ SQS của Account 1 thì cần config Policy như sau:

```json
{
  "Version": "2026-04-28",
  "Statement": [
    {
      "Sid": "Queue1_AllActions",
      "Effect": "Allow",
      "Principal": {
        "AWS": ["222222222222"]
      },
      "Action": "sqs:ReceiveMessage",
      "Resource": "arn:aws:sqs:us-east-1:111111111111"
    }
  ]
}
```

### Publish S3 Event Notifications

![alt text](../image/publish-s3-event-notification.png)

```json
{
  "Version": "2026-04-28",
  "Statement": [
    {
      "Sid": "example-statement-ID",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": ["SQS:SendMessage"],
      "Resource": "arn:aws:sqs:<region>:<account-id></account-id>:queue1",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:*:*:bucket1"
        },
        "StringEquals": {
          "aws:SourceAccount": "<bucket-owner-account-id>"
        }
      }
    }
  ]
}
```
