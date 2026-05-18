# ARCHITECTURE PATTERN

## Mục lục

## Phần 1: Các cách phân loại kiến trúc phần mềm

### 1. Cách phân loại phổ biến

Các mẫu kiến trúc phần mềm thường được chia thành hai loại chính:
- Kiến trúc nguyên khối (*Monolithic Architecture*)
- Kiến trúc phân tán (*Distributed Architecture*)

#### 1.1 Kiến trúc nguyên khối (Monolithic Architecture)

![alt text](../image/monolithic_architecture.png)

**Kiến trúc nguyên khối** (*Monolithic architecture*) được triển khai trên một đơn vị duy nhất (single deployment unit). Ví dụ, trên một server, bạn có thể cài một database và một service. Ở đây, chúng ta có hai đơn vị triển khai.

**Ưu điểm:**
- Thiết kế đơn giản và triển khai, vận hành dễ dàng.
- Dễ dàng test và debug.
- Hiệu suất cao trong các ứng dụng ít phức tạp.

**Nhược điểm:**
- Khó khăn khi scale ứng dụng.
- Độ tin cậy thấp.
- Thiếu linh hoạt với thay đổi.

**Các mẫu kiến trúc phầm mềm nằm trong nhóm kiến trúc nguyên khối:**
- Kiến trúc phân lớp (*Layered architecture*)
- Kiến trúc client-server
- Kiến trúc đường ống (*Pipeline architecture*)
- Kiến trúc vi nhân (*Microkernel architecture*)

#### 1.2 Kiến trúc phân tán (Distributed Architecture)

![alt text](../image/distributed_architecture.png)

**Kiến trúc phân tán** (*Distributed architecture*) được triển khai trên nhiều đơn vị (multiple deployment units) làm việc cùng nhau để thực hiện một số loại chức năng business gắn kết.

**Ưu điểm:**
- **Khả năng chịu lỗi**: Nếu một dịch vụ bị lỗi, các dịch vụ khác có thể tiếp tục thực hiện các yêu cầu dịch vụ như thể không có lỗi nào xảy ra. 
- **Khả năng thích ứng**: Dễ dàng xác định vị trí và áp dụng thay đổi hơn, phạm vi thử nghiệm được giảm xuống chỉ còn dịch vụ bị ảnh hưởng và rủi ro triển khai giảm đáng kể vì thường chỉ triển khai dịch vụ bị ảnh hưởng.

**Nhược điểm:**
- Chịu ảnh hưởng lớn bởi [8 sai lầm trong tính toán phân tán](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing).
- Quản lý các giao dịch phân tán, tính nhất quán cuối cùng, quản lý quy trình làm việc, xử lý lỗi, đồng bộ hóa dữ liệu.

**Các mẫu kiến trúc phầm mềm nằm trong nhóm kiến trúc phân tán:**
- Kiến trúc hướng sự kiện (*Event-driven architecture*)
- Kiến trúc microservice (*Microservices architecture*)
- Kiến trúc hướng không gian (*Space-based architecture*)
- Kiến trúc hướng dịch vụ (*Service-oriented architecture*)

### 2. Cách phân loại khác

Ngoài cách phân loại trên, các mẫu kiến trúc phần mềm cũng có thể được phân loại theo cách chia cấu trúc tổng thể của hệ thống. Theo cách chia này, chúng ta có hai loại chính là:

- Phân vùng kỹ thuật (*Technically partitioned*)
- Phân vùng theo domain (*Domain partitioned*)

#### 2.1 Phân vùng kỹ thuật (Technically partitioned)

Trong phân vùng kỹ thuật, các component của hệ thống được tổ chức theo cách sử dụng kỹ thuật (technical usage). Ở ví dụ dưới sử dụng kiến trúc phân tầng.

![alt text](../image/technically_partitioned_layered_architecture.png)

**Ưu điểm:**
- Phù hợp với các team được tổ chức phân nhóm theo vai trò. Ví dụ: nhóm BE, nhóm FE, nhóm designer, ...
- Khi mã nguồn của bạn được tổ chức độc lập với nhau. Sự thay đổi của một layer sẽ không ảnh hưởng tới layer khác.
- Testing dễ dàng khi chỉ cần viết test cho từng layer và tạo mock test cho các layer còn lại.

**Nhược điểm:**
- Nếu chúng ta sửa business thì khả năng cao sẽ phải sửa lại tất cả các tầng do không có sự phân tách business giữa các tầng.

#### 2.2 Phân vùng theo domain (Domain partitioned)

Trong phân vùng theo domain (*Domain partitioned*), các component của hệ thống được tổ chức theo domain.

![alt text](../image/domain_partitioned.png)

**Ưu điểm:**
- Nếu business có thay đổi thì chỉ ảnh hưởng tới những service chứa domain đó.
- Phù hợp với các nhóm đa chức năng có chuyên môn (cross-functional team). Mỗi nhóm sẽ phụ trách một domain riêng biệt.

**Nhược điểm:**
- Viết unit test không dễ dàng do logic của các tầng phục vụ cho cùng một domain nên viết end-to-end test sẽ phù hợp hơn và dễ dàng hơn là unit test.

### 3. Tổng kết

- Hiện nay, có 2 cách phân loại kiến trúc phần mềm: cách phân loại phổ biến và cách phân loại dựa trên cách chia cấu trúc hệ thống.
- Các phân loại phổ biến chia kiến trúc phần mềm ra làm hai loại chính là *kiến trúc nguyên khối* và *kiến trúc phân tán*.
- Ngoài ra, có cách phân loại khác dựa trên cách chia cấu trúc của hệ thống bao gồm *phân vùng kỹ thuật* và *phân vùng theo domain*.
- Mỗi loại kiến trúc phần mềm đều có ưu điểm và nhược điểm riêng, do vậy, tuỳ yêu cầu của bài toán, chúng ta cần cân nhắc lựa chọn cho phù hợp.

