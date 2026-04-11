# AWS S3

![Status](https://img.shields.io/badge/AWS_S3-orange) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Giới thiệu về AWS S3

**[Amazon S3](https://aws.amazon.com/s3/)** (Amazon Simple Storage Service) là dịch vụ lưu trữ dữ liệu đơn giản của Amazon cung cấp. Amazon S3 cung cấp khả nẳng mở rộng, tính khả dụng của dữ liệu, bảo mật cao.

### S3 Buckets và Object trong AWS

Amazon S3 cho phép người dùng có thể lưu trữ Objects (files) trong Buckets (directories).

- **Bucket**: 
  - Là một container để lưu trữ dữ liệu. 
  - Mỗi bucket có tên duy nhất trên toàn cầu và có thể chứa một số lượng không giới hạn các object. 
  - Bucket có scope là region, khi tạo bucket cần chọn region.
  - Quy tắc đặt tên: 
    - Tên bucket phải có độ dài từ 3 đến 63 ký tự.
    - Tên bucket chỉ được chứa các ký tự chữ thường, số, dấu gạch ngang (-) và dấu chấm (.).
    - Tên bucket phải được bắt đầu bằng ký tự chữ thường hoặc số.
    - Không được có format địa chỉ IP.

- **Object**: 
  - Là một file được lưu trữ trong bucket. 
  - Object có Key chính là path tên object trong bucket. Key có thể là kết hợp của: prefix + object_name (vd: `path_1/path_2/file.txt`).
  - Object value là nội dung của body. Object size tối đa là 5TB. Nếu muốn upload nhiều hơn 5GB, cần dùng "multi-path upload" để chia nhỏ upload nhiều phần.

- **S3 versioning**: 
  - Chúng ta có thể tạo các version của file.
  - Tính năng này được enable ở "bucket level".
  - Khi 1 file có chung key, version sẽ tự động tạo ra.
  - Khi đánh version cho 1 file, chúng ta có thể dễ dàng phục hồi các version của 1 file.

  Luu ý: Nếu enable versioning của một bucket thì những file đã tồn tại trước đó sẽ có version ID = null.

## Mã hóa dữ liệu trong S3

### S3 Encryption trong AWS

Để tránh việc lưu trữ dữ liệu dưới dạng thô, Amazon S3 cung cấp phương thức mã hóa dữ liệu. Cách thức hoạt động của mã hóa là dùng **key** và **thuật toán (algorithm)** để biến dữ liệu ban đầu thành dữ liệu được mã hóa. Vậy nên, vấn đề cần quan tâm là lưu trữ key ở đâu. 

Trong S3 có 2 cách chính để mã hóa.

- Server-side encryption: Mã hóa phía server (S3).
- Client-side encryption: Mã hóa phía client (dùng các libs để mã hóa) rồi upload dữ liệu được mã hóa lên S3 Amazon S3 cung cấp 4 phương thức mã hóa object:
  - SSE-S3: Mã hóa S3 objects sử dụng key quản lý bởi AWS.
  - SSE-KMS: Sử dụng AWS Key Management Service (KMS) để quản lý encryption keys.
  - SSE-C: Sử dụng khi bạn muốn quản lý encryption keys riêng của mình.
  - Client Side Encryption.

#### SSE-S3 trong AWS

- Mã hóa sử dụng key quản lý bởi Amazon S3.
- Object được mã hóa phía server side.
- Phương thức mã hóa: AES-256.
- Phải set header: "x-amz-server-side-encryption":"AES256".

![alt text](../image/SSE-S3.png)

---

#### SSE-KMS trong AWS

- Mã hóa sử dụng key quản lý bởi AWS Key Management Service (KMS).
- Object được mã hóa phía server side.
- Phải set header: "x-amz-server-side-encryption":"aws:kms".

![alt text](../image/SSE-S3.png)

---

#### SSE-C trong AWS

- Là server-side encryption sử dụng key cung cấp bởi khách hàng (AWS không quản lý key này).
- **Phải dùng HTTPS**.
- Encryption key phải được cung cấp trong HTTPS headers trong mỗi request.

![alt text](../image/SSE-C.png)

---

#### Client Side Encryption trong AWS

- Mã hóa phía client trước khi upload lên S3.
- Sử dụng client libs chẳng hạn như: Amazon S3 Encryption Client.
- Khi đọc dữ liệu trả về cần decrypt chúng.

![alt text](../image/Client-Side-Encryption.png)

## Bảo mật trong S3

### S3 Security trong AWS

S3 security là quản lý quyền truy cập dữ liệu trong Amazon S3. Chúng ta có 3 phương thức để quản lý truy cập, đó là:

- **IAM policies**: Cấp quyền truy cập cho user nhất định.
- **ACLs**:
  - **Bucket ACL**: Quản lý cấp độ bucket.
  - **Objects ACL**: Quản lý cấp độ Objects.
- **Bucket policies**: Có thể add/deny quyền truy cập một cách linh hoạt được đinh nghĩa trong file JSON.

---

### S3 Bucket Policies trong AWS

![alt text](../image/s3-bucket-policies.png)

- S3 bucket policies được định nghĩa dưới dạng file JSON.
- **Resource**: Tài nguyên thực thi (Bucket hoặc Objects).
- **Effect**: Allow hoặc Deny.
- **Principal**: Account hay user được apply (* là tất cả user).
- **Action**: Quền thực thi (GetObject, Put, Delete...).

Ví dụ như trên hình vẽ đang định nghĩa policy: Tất cả user có thể đọc được tất cả object trong bucket foobucket.

**Sử dụng S3 bucket policies thường cho**:

- Cấp quyền truy cập đến bucket, object.
- Bắt buộc object cần được mã hóa trước khi upload lên S3.
- Cấp quyền truy cập cho account khác (cross account).

---

### Block public access

Mặc định khi tạo một bucket trên S3, Amazon sẽ block tất cả public access. Có nghĩa là khi bạn upload một file ảnh lên bucket đó, bạn sẽ không thể view file đó được luôn. Để có thể xem được file ảnh đó, bạn cần cấp quyền để có thể xem từ mọi nơi.

---

### Bảo mật người dùng trong S3

- **MFA Delete**: Chúng ta có thể anable MFA (multi factor authentication) khi muốn delete object. Việc này đảm bảo không phải ai cũng có thể xóa dữ liệu của bạn.
- **Pre-Signed URLs**: Ví dụ video của bạn là premium (giới hạn cho những user đã trả phí). Khi tạo Pre-Signed URLs cho object đó, urls sẽ có expire.


## Giới thiệu S3 Hosting, CORS

### S3 Hosting trong AWS

S3 hosting cho phép bạn có thể tạo 1 public website từ source code html, css, javascript của bạn. Bạn không cần config web server, dns... tất cả Amazon S3 đã làm cho bạn, việc cần làm là đẩy source code của bạn lên bucket.

- URL website sẽ có format: `http://<bucket-name>.s3-website-<region>.amazonaws.com`
- Cần cho phép bucket của bạn access public.

---

### S3 CORS trong AWS

![alt text](../image/s3-cors.png)

Như trên hình vẽ chúng ta có 2 bucket:

- **HTML bucket**: Chứa các file html được host trên S3.
- **Assets bucket**: Chứa các file ảnh của project.

Khi gửi request đến HTML bucket, website cần request đến tiếp những file ảnh trong Assets bucket. Khi đó cần enable CORS trong Assets bucket để những request từ url: http://sample.s3-website-us-east-2.amazonaws.com có thể đọc được những file ảnh này.

## Giới thiệu S3 MFA Delete, Default Encryption

### S3 MFA Delete trong AWS

**MFA (Multi factor authentication)** bắt buộc người dùng phải sử dụng một đoạn code được gen ra trên thiết bị xác thực 2 lớp(thường là điện thoại) trước khi thực hiện các operations quan trọng trên S3. Điều này nhằm mục đích bảo mật những resource quan trọng có trên S3, Amazon S3 muốn chắc chắn rằng chỉ có owner của Bucket mới có quyền để xóa một version hoặc thay đổi version của nó.

- Để sử dụng MFA-Delete, bạn cần enable Versioning trên S3 bucket.
- Khi đã sử dụng MFA-Delete, bạn cần thêm một bước xác thực nữa để có thể thực hiện:
  - Thay đổi Versioning của Bucket.
  - Xóa một Object version.
- Bạn không cần sử dụng MFA để:
  - Enable versioning.
  - Xem những versions đã xóa.
- Chỉ có bucket owner (root account) mới có quền enable/disable MFA-Delete.
- MFA-Delete hiện tại chỉ có thể enabled bằng CLI.

---

### S3 Default Encryption trong AWS

- Khi tạo một bucket trên S3, chúng ta có thể "force encryption" những object được upload lên. Điều này đảm bảo rằng những object được upload lên S3 đã được mã hóa.
- Bạn có thể sử dụng "Default Encryption", khi enable option này lên, object sẽ được Amazon S3 mã hóa mặc định.

## Giới thiệu S3 Access Logs, Replication và Pre-signed

### S3 Access Logs trong AWS

- S3 Access Logs lưu lại thông tin request đến S3 buckets của bạn.
- Như hình vẽ dưới đây, những request đến "S3 Bucket", cho dù accept hay denied đều được ghi lại vào "Log Bucket".
- Dữ liệu này có thể dùng để phân tích bằng những dịch vụ phân tích như Amazon Athena...

![alt text](../image/s3-access-log.png)

---

### S3 Replication (CRR & SRR) trong AWS

- CRR: Cross Region Replication.
- SRR: Same Region Replication.

S3 Replication là tính năng sao chép các object giữa các vùng lưu trữ. Với S3 Replication, bạn có thể config Amazon S3 tự động sao chép S3 Object trên các Region khác nhau (Cross Region Replication), hoặc giữa các vùng lưu trữ trên cùng một Region (Same Region Replication).

- Phải "Enable versioning" trong bucket source và destination.
- Buckets có thể ở account khác nhau.
- Việc copy là asynchronous.
- Cần cung cấp IAM permission cần thiết tới S3.

Lưu ý: 
- Sau khi enable replica, bạn chỉ có thể copy những Object mới, còn objects cũ trước đó sẽ không được copy.
- Copy không thể có tính "chaining". Có nghĩa nếu Bucket A copy sang Bucket B, Bucket B copy sang Bucket C. Thì khi tạo Object D sẽ không được copy sang Bucket C.

---

### S3 Pre-signed trong AWS

Pre-signed URL là URL mà bạn có thể cung cấp cho người dùng của mình để cấp quyền truy cập tạm thời vào một đối tượng S3 cụ thể. Sử dụng URL, người dùng có thể đọc và ghi đối tượng (hoặc cập nhật đối tượng hiện có). URL chứa các thông số cụ thể do ứng dụng mà bạn cài đặt.

- Mặc định Pre-signed URL có hiệu lực 3600s, bạn có thể thay đổi với CLI (--expires-in argument).
- Khi đã quá thời gian hết hạn, người dùng không thể truy cập được đến Object chỉ định.