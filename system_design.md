# SYSTEM DESIGN NOTE

![Status](https://img.shields.io/badge/Status-Learning-blue) ![Topic](https://img.shields.io/badge/Topic-System%20Design-orange)

System Design cho người mới bắt đầu.
Những kiến thức cơ bản giúp bạn phỏng vấn hiệu quả.

> **Lưu ý:** Đây là tài liệu ghi chép, tổng hợp và tóm tắt lại kiến thức cá nhân của tôi trong quá trình đọc Ebook về System Design.
>
> **© Bản quyền gốc thuộc về:** Tác giả **Nguyễn Thế Huy** (https://huynt.dev). Mọi trích dẫn và nội dung cốt lõi đều nhằm mục đích học tập và lưu trữ kiến thức.

---

## Mục lục

### [Lời mở đầu](#lời-mở-đầu-1)

### [Giới thiệu](#giới-thiệu-1)

- [System Design Interview là gì?](#system-design-interview-là-gì)
- [Tại sao vòng phỏng vấn này quan trọng?](#tại-sao-vòng-phỏng-vấn-này-quan-trọng)
- [Những khó khăn mà người mới thường gặp](#những-khó-khăn-mà-người-mới-thường-gặp)
- [Các kỹ năng mà nhà tuyển dùng mong muốn](#các-kỹ-năng-mà-nhà-tuyển-dùng-mong-muốn)

### [Các khái niệm nền tảng trong Thiết kế hệ thống](#các-khái-niệm-nền-tảng-trong-thiết-kế-hệ-thống-1)

- [Latency (Độ trễ)](#latency-độ-trễ)
- [Throughput (Thông lượng)](#throughput-thông-lượng)
- [Scalability (Khả năng mở rộng)](#scalability-khả-năng-mở-rộng)
- [Availability (Tính khả dụng)](#availability-tính-khả-dụng)
- [Reliability (Độ tin cậy)](#reliability-độ-tin-cậy)
- [High-Level Architecture (Kiến trúc cấp cao)](#high-level-architecture-kiến-trúc-cấp-cao)
- [Các thành phần phổ biến trong hệ thống](#các-thành-phần-phổ-biến-trong-hệ-thống)
    - [Web Server (Máy chủ Web)](#web-server-máy-chủ-web)
    - [Application Server (Máy chủ ứng dụng)](#application-server-máy-chủ-ứng-dụng)
    - [Database (Cơ sở dữ liệu)](#database-cơ-sở-dữ-liệu)
    - [Load Balancer (Bộ cân bằng tải)](#load-balancer-bộ-cân-bằng-tải)
    - [Cache (Bộ nhớ đệm)](#cache-bộ-nhớ-đệm)
    - [CDN (Mạng phân phối nội dung)](#cdn-mạng-phân-phối-nội-dung)
    - [Message Queue (Hàng đợi thông điệp)](#message-queue-hàng-đợi-thông-điệp)

### [Tính nhất quán và khả dụng trong System Design](#tính-nhất-quán-và-khả-dụng-trong-system-design-1)

- [Giới thiệu](#giới-thiệu-2)
- [Tính nhất quán và tính khả dụng](#tính-nhất-quán-và-tính-khả-dụng)
- [Các mô hình nhất quán (Consistency Models)](#các-mô-hình-nhất-quán-consistency-models)
    - [Nhất quán mạnh (Strong Consistency)](#nhất-quán-mạnh-strong-consistency)
    - [Nhất quán cuối cùng (Eventual Consistency)](#nhất-quán-cuối-cùng-eventual-consistency)
    - [Nhất quán yếu (Weak Consistency)](#nhất-quán-yếu-weak-consistency)
- [Cơ chế đảm bảo tính nhất quán và khả dụng](#cơ-chế-đảm-bảo-tính-nhất-quán-và-khả-dụng)
    - [Nhân bản dữ liệu (Replication)](#nhân-bản-dữ-liệu-replication)
    - [Chuyển đổi dự phòng (Failover)](#chuyển-đổi-dự-phòng-failover)
    - [Quorum (Đa số phiếu)](#quorum-đa-số-phiếu)
- [Định lý CAP và các đánh đổi kinh điển](#định-lý-cap-và-các-đánh-đổi-kinh-điển)
- [Cách trình bày về nhất quán và khả dụng khi phỏng vấn](#cách-trình-bày-về-nhất-quán-và-khả-dụng-khi-phỏng-vấn)

### [Lựa chọn Cơ sở dữ liệu, Sharding và Tối ưu hóa lưu trữ](#lựa-chọn-cơ-sở-dữ-liệu-sharding-và-tối-ưu-hóa-lưu-trữ-1)

- [SQL vs NoSQL: Lựa chọn cơ sở dữ liệu phù hợp](#sql-vs-nosql-lựa-chọn-cơ-sở-dữ-liệu-phù-hợp)
- [Phân mảnh dữ liệu (Sharding/Partitioning)](#phân-mảnh-dữ-liệu-shardingpartitioning)
- [Chỉ mục (Index): Tối ưu hóa truy vấn](#chỉ-mục-index-tối-ưu-hóa-truy-vấn)
- [Cân nhắc khi chọn kiểu lưu trữ cho hệ thống lớn](#cân-nhắc-khi-chọn-kiểu-lưu-trữ-cho-hệ-thống-lớn)
- [Dữ liệu phân tán và tính nhất quán (Consistency)](#dữ-liệu-phân-tán-và-tính-nhất-quán-consistency)
- [Gợi ý trình bày phần thiết kế Database khi phỏng vấn](#gợi-ý-trình-bày-phần-thiết-kế-database-khi-phỏng-vấn)

### [Mở rộng hệ thống và tối ưu hiệu suất](#mở-rộng-hệ-thống-và-tối-ưu-hiệu-suất-1)

- [Cân bằng tải (Load Balancer)](#cân-bằng-tải-load-balancer)
    - [Load Balancer là gì và vai trò của nó?](#load-balancer-là-gì-và-vai-trò-của-nó)
    - [Cách hoạt động và các thuật toán phân phối](#cách-hoạt-động-và-các-thuật-toán-phân-phối)
    - [Load Balancer tầng 4 vs tầng 7 (Layer 4 vs Layer 7)](#load-balancer-tầng-4-vs-tầng-7-layer-4-vs-layer-7)
    - [Khi nào cần sử dụng Load Balancer?](#khi-nào-cần-sử-dụng-load-balancer)
- [Bộ nhớ đệm (Caching)](#bộ-nhớ-đệm-caching)
    - [Cache là gì và có lợi ích gì?](#cache-là-gì-và-có-lợi-ích-gì)
    - [Các vị trí có thể đặt cache](#các-vị-trí-có-thể-đặt-cache)
    - [Các chiến lược cập nhật cache (Cache update strategies)](#các-chiến-lược-cập-nhật-cache-cache-update-strategies)
    - [Gợi ý khi phỏng vấn về Caching](#gợi-ý-khi-phỏng-vấn-về-caching)
- [Hàng đợi thông điệp (Message Queue)](#hàng-đợi-thông-điệp-message-queue)
    - [Khái niệm hàng đợi thông điệp & xử lý bất đồng bộ](#khái-niệm-hàng-đợi-thông-điệp--xử-lý-bất-đồng-bộ)
    - [Khi nào nên dùng hàng đợi](#khi-nào-nên-dùng-hàng-đợi)
    - [Một số hệ thống Message Queue phổ biến](#một-số-hệ-thống-message-queue-phổ-biến)
    - [Mô hình Producer - Queue - Consumer](#mô-hình-producer---queue---consumer)
    - [Cơ chế backpressure (phản áp lực) trong hàng đợi](#cơ-chế-backpressure-phản-áp-lực-trong-hàng-đợi)
    - [Ví dụ minh họa sử dụng Message Queue](#ví-dụ-minh-họa-sử-dụng-message-queue)
    - [Gợi ý khi phỏng vấn về Message Queue](#gợi-ý-khi-phỏng-vấn-về-message-queue)

### [Idempotency và cơ chế khóa trong hệ thống phân tán](#idempotency-và-cơ-chế-khóa-trong-hệ-thống-phân-tán-1)

- [Pessimistic Locking (Khóa bi quan)](#pessimistic-locking-khóa-bi-quan)
- [Optimistic Locking (Khóa lạc quan)](#optimistic-locking-khóa-lạc-quan)
- [Khi nào dùng & lựa chọn](#khi-nào-dùng--lựa-chọn)

### [Các phương thức giao tiếp và thiết kế API](#các-phương-thức-giao-tiếp-và-thiết-kế-api-1)

- [HTTP: Giao thức request/response phổ biến nhất](#http-giao-thức-requestresponse-phổ-biến-nhất)
- [TCP: Giao thức kết nối tin cậy](#tcp-giao-thức-kết-nối-tin-cậy)
- [UDP: Giao thức "gửi là quên"](#udp-giao-thức-gửi-là-quên)
- [RPC: Gọi thủ tục từ xa như gọi hàm nội bộ](#rpc-gọi-thủ-tục-từ-xa-như-gọi-hàm-nội-bộ)
- [REST: Phong cách thiết kế API rõ ràng, thống nhất](#rest-phong-cách-thiết-kế-api-rõ-ràng-thống-nhất)

### [Đảm bảo khả năng chịu lỗi và phục hồi hệ thống](#đảm-bảo-khả-năng-chịu-lỗi-và-phục-hồi-hệ-thống-1)

- [Khả năng chịu lỗi là gì và tại sao quan trọng?](#khả-năng-chịu-lỗi-là-gì-và-tại-sao-quan-trọng)
- [Các sự cố phổ biến trong hệ thống phân tán](#các-sự-cố-phổ-biến-trong-hệ-thống-phân-tán)
- [Chiến lược tăng độ bền vững (resilience) cho hệ thống](#chiến-lược-tăng-độ-bền-vững-resilience-cho-hệ-thống)
- [Phân biệt High Availability và Fault Tolerance](#phân-biệt-high-availability-và-fault-tolerance)
- [Đánh giá độ tin cậy: Uptime, SLA, MTTR, MTBF](#đánh-giá-độ-tin-cậy-uptime-sla-mttr-mtbf)
- [Gợi ý trình bày chiến lược phục hồi sự cố trong phỏng vấn](#gợi-ý-trình-bày-chiến-lược-phục-hồi-sự-cố-trong-phỏng-vấn)

### [Bảo mật và phân quyền](#bảo-mật-và-phân-quyền-1)

- [Vai trò của bảo mật trong hệ thống phân tán hiện đại](#vai-trò-của-bảo-mật-trong-hệ-thống-phân-tán-hiện-đại)
    - [Các khái niệm cơ bản](#các-khái-niệm-cơ-bản)
        - [Xác thực (Authentication)](#xác-thực-authentication)
        - [Phân quyền (Authorization)](#phân-quyền-authorization)
        - [Bảo mật API](#bảo-mật-api)
        - [Bảo vệ dữ liệu](#bảo-vệ-dữ-liệu)
        - [HTTPS, TLS và bảo mật cookie](#https-tls-và-bảo-mật-cookie)
    - [Ví dụ minh họa](#ví-dụ-minh-họa)
    - [Lời khuyên khi phỏng vấn về bảo mật trong System Design](#lời-khuyên-khi-phỏng-vấn-về-bảo-mật-trong-system-design)
- [Monitoring và Observability](#monitoring-và-observability)
    - [Monitoring là gì?](#monitoring-là-gì)
    - [Observability là gì?](#observability-là-gì)
    - [Trình bày về Monitoring và Observability trong buổi phỏng vấn](#trình-bày-về-monitoring-và-observability-trong-buổi-phỏng-vấn)

### [Lời kết](#lời-kết-1)

---

## <a id="lời-mở-đầu-1"></a>Lời mở đầu

> "Chào bạn, và cám ơn bạn đã đọc ebook này của mình.
>
> Mình là Huy, hiện tại mình là một Technical Leader với trên 10 năm kinh nghiệm.
> Ebook này của mình nhằm cung cấp cho bạn những khái niệm cốt lõi, cũng như kinh nghiệm tham gia phỏng vấn System Design (thiết kế hệ thống).
>
> Dù bạn là người mới bắt đầu hay đã có kinh nghiệm, mình tin những kiến thức trong ebook này sẽ giúp bạn rất nhiều trong việc review lại kiến thức về thiết kế, cũng như chuẩn bị cho các cuộc phỏng vấn về System Design.
>
> Chúc bạn có những giây phút thú vị, bổ ích khi đọc ebook."

## <a id="giới-thiệu-1"></a> Giới thiệu

### System Design Interview là gì?

### Tại sao vòng phỏng vấn này quan trọng?

### Những khó khăn mà người mới thường gặp

### Các kỹ năng mà nhà tuyển dùng mong muốn

## <a id="các-khái-niệm-nền-tảng-trong-thiết-kế-hệ-thống-1"></a>Các khái niệm nền tảng trong Thiết kế hệ thống

### Latency (Độ trễ)

### Throughput (Thông lượng)

### Scalability (Khả năng mở rộng)

### Availability (Tính khả dụng)

### Reliability (Độ tin cậy)

### High-Level Architecture (Kiến trúc cấp cao)

### Các thành phần phổ biến trong hệ thống

1. <a id="web-server-máy-chủ-web"></a>**Web Server (Máy chủ Web)**

2. <a id="application-server-máy-chủ-ứng-dụng"></a>**Application Server (Máy chủ ứng dụng)**

3. <a id="database-cơ-sở-dữ-liệu"></a>**Database (Cơ sở dữ liệu)**

4. <a id="load-balancer-bộ-cân-bằng-tải"></a>**Load Balancer (Bộ cân bằng tải)**

5. <a id="cache-bộ-nhớ-đệm"></a>**Cache (Bộ nhớ đệm)**

6. <a id="cdn-mạng-phân-phối-nội-dung"></a>**CDN (Mạng phân phối nội dung)**

7. <a id="message-queue-hàng-đợi-thông-điệp"></a>**Message Queue (Hàng đợi thông điệp)**

## <a id="tính-nhất-quán-và-khả-dụng-trong-system-design-1"></a>Tính nhất quán và khả dụng trong System Design

### <a id="giới-thiệu-2"></a>Giới thiệu

### Tính nhất quán và tính khả dụng

### Các mô hình nhất quán (Consistency Models)

1. <a id="nhất-quán-mạnh-strong-consistency"></a>**Nhất quán mạnh (Strong Consistency)**

2. <a id="nhất-quán-cuối-cùng-eventual-consistency"></a>**Nhất quán cuối cùng (Eventual Consistency)**

3. <a id="nhất-quán-yếu-weak-consistency"></a>**Nhất quán yếu (Weak Consistency)**

### Cơ chế đảm bảo tính nhất quán và khả dụng

1. <a id="nhân-bản-dữ-liệu-replication"></a>**Nhân bản dữ liệu (Replication)**

2. <a id="chuyển-đổi-dự-phòng-failover"></a>**Chuyển đổi dự phòng (Failover)**

3. <a id="quorum-đa-số-phiếu"></a>**Quorum (Đa số phiếu)**

### Định lý CAP và các đánh đổi kinh điển

### Cách trình bày về nhất quán và khả dụng khi phỏng vấn

## <a id="lựa-chọn-cơ-sở-dữ-liệu-sharding-và-tối-ưu-hóa-lưu-trữ-1"></a>Lựa chọn Cơ sở dữ liệu, Sharding và Tối ưu hóa lưu trữ

### SQL vs NoSQL: Lựa chọn cơ sở dữ liệu phù hợp

### Phân mảnh dữ liệu (Sharding/Partitioning)

### Chỉ mục (Index): Tối ưu hóa truy vấn

### Cân nhắc khi chọn kiểu lưu trữ cho hệ thống lớn

### Dữ liệu phân tán và tính nhất quán (Consistency)

### Gợi ý trình bày phần thiết kế Database khi phỏng vấn

## <a id="mở-rộng-hệ-thống-và-tối-ưu-hiệu-suất-1"></a>Mở rộng hệ thống và tối ưu hiệu suất

### Cân bằng tải (Load Balancer)

1. <a id="load-balancer-là-gì-và-vai-trò-của-nó"></a>**Load Balancer là gì và vai trò của nó?**

2. <a id="cách-hoạt-động-và-các-thuật-toán-phân-phối"></a>**Cách hoạt động và các thuật toán phân phối**

3. <a id="load-balancer-tầng-4-vs-tầng-7-layer-4-vs-layer-7"></a>**Load Balancer tầng 4 vs tầng 7 (Layer 4 vs Layer 7)**

4. <a id="khi-nào-cần-sử-dụng-load-balancer"></a>**Khi nào cần sử dụng Load Balancer?**

### Bộ nhớ đệm (Caching)

1. <a id="cache-là-gì-và-có-lợi-ích-gì"></a>**Cache là gì và có lợi ích gì?**

2. <a id="các-vị-trí-có-thể-đặt-cache"></a>**Các vị trí có thể đặt cache**

3. <a id="các-chiến-lược-cập-nhật-cache-cache-update-strategies"></a>**Các chiến lược cập nhật cache (Cache update strategies)**

4. <a id="gợi-ý-khi-phỏng-vấn-về-caching"></a>**Gợi ý khi phỏng vấn về Caching**

### Hàng đợi thông điệp (Message Queue)

1. <a id="khái-niệm-hàng-đợi-thông-điệp--xử-lý-bất-đồng-bộ"></a>**Khái niệm hàng đợi thông điệp & xử lý bất đồng bộ**

2. <a id="khi-nào-nên-dùng-hàng-đợi"></a>**Khi nào nên dùng hàng đợi**

3. <a id="một-số-hệ-thống-message-queue-phổ-biến"></a>**Một số hệ thống Message Queue phổ biến**

4. <a id="mô-hình-producer---queue---consumer"></a>**Mô hình Producer - Queue - Consumer**

5. <a id="cơ-chế-backpressure-phản-áp-lực-trong-hàng-đợi"></a>**Cơ chế backpressure (phản áp lực) trong hàng đợi**

6. <a id="ví-dụ-minh-họa-sử-dụng-message-queue"></a>**Ví dụ minh họa sử dụng Message Queue**

7. <a id="gợi-ý-khi-phỏng-vấn-về-message-queue"></a>**Gợi ý khi phỏng vấn về Message Queue**

## <a id="idempotency-và-cơ-chế-khóa-trong-hệ-thống-phân-tán-1"></a>Idempotency và cơ chế khóa trong hệ thống phân tán

### Pessimistic Locking (Khóa bi quan)

### Optimistic Locking (Khóa lạc quan)

### Khi nào dùng & lựa chọn

## <a id="các-phương-thức-giao-tiếp-và-thiết-kế-api-1"></a>Các phương thức giao tiếp và thiết kế API

### HTTP: Giao thức request/response phổ biến nhất

### TCP: Giao thức kết nối tin cậy

### UDP: Giao thức "gửi là quên"

### RPC: Gọi thủ tục từ xa như gọi hàm nội bộ

### REST: Phong cách thiết kế API rõ ràng, thống nhất

## <a id="đảm-bảo-khả-năng-chịu-lỗi-và-phục-hồi-hệ-thống-1"></a>Đảm bảo khả năng chịu lỗi và phục hồi hệ thống

### Khả năng chịu lỗi là gì và tại sao quan trọng?

### Các sự cố phổ biến trong hệ thống phân tán

### Chiến lược tăng độ bền vững (resilience) cho hệ thống

### Phân biệt High Availability và Fault Tolerance

### Đánh giá độ tin cậy: Uptime, SLA, MTTR, MTBF

### Gợi ý trình bày chiến lược phục hồi sự cố trong phỏng vấn

## <a id="bảo-mật-và-phân-quyền-1"></a>Bảo mật và phân quyền

### Vai trò của bảo mật trong hệ thống phân tán hiện đại

1. <a id="các-khái-niệm-cơ-bản"></a>**Các khái niệm cơ bản**
    - <a id="xác-thực-authentication"></a>**Xác thực (Authentication)**
    - <a id="phân-quyền-authorization"></a>**Phân quyền (Authorization)**
    - <a id="bảo-mật-api"></a>**Bảo mật API**
    - <a id="bảo-vệ-dữ-liệu"></a>**Bảo vệ dữ liệu**
    - <a id="https-tls-và-bảo-mật-cookie"></a>**HTTPS, TLS và bảo mật cookie**

2. <a id="ví-dụ-minh-họa"></a>**Ví dụ minh họa**

3. <a id="lời-khuyên-khi-phỏng-vấn-về-bảo-mật-trong-system-design"></a>**Lời khuyên khi phỏng vấn về bảo mật trong System Design**

### Monitoring và Observability

1. <a id="monitoring-là-gì"></a>**Monitoring là gì?**

2. <a id="observability-là-gì"></a>**Observability là gì?**

3. <a id="trình-bày-về-monitoring-và-observability-trong-buổi-phỏng-vấn"></a>**Trình bày về Monitoring và Observability trong buổi phỏng vấn**

## <a id="lời-kết-1"></a>Lời kết