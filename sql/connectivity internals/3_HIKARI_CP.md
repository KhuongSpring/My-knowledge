# Chương 3: HikariCP - Connection Pool Quốc Dân

![Status](https://img.shields.io/badge/HikariCP-green) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Mục lục

- [I. Tổng quan](#i-tổng-quan)
  - [1. HikariCP là gì?](#1-hikaricp-là-gì)
  - [2. Tại sao HikariCP là mặc định của Spring Boot (từ 2.0+)?](#2-tại-sao-hikaricp-là-mặc-định-của-spring-boot-từ-20)
  - [3. So sánh nhanh với các Pool khác](#3-so-sánh-nhanh-với-các-pool-khác)
- [II. Bí mật hiệu năng (Top 4 "Killer Features")](#ii-bí-mật-hiệu-năng-top-4-killer-features)
  - [1. Tối ưu Bytecode (JIT Inlining)](#1-tối-ưu-bytecode-jit-inlining)
  - [2. FastList (Thay thế ArrayList)](#2-fastlist-thay-thế-arraylist)
  - [3. ConcurrentBag (Lock-free Architecture)](#3-concurrentbag-lock-free-architecture)
  - [4. Direct Delegate Proxy (Mã hóa bằng Javassist)](#4-direct-delegate-proxy-mã-hóa-bằng-javassist)
- [III. Luồng mượn & trả Connection (Core Workflow)](#iii-luồng-mượn--trả-connection-core-workflow)
  - [1. Quy trình lấy Connection (`getConnection()`)](#1-quy-trình-lấy-connection-getconnection)
  - [2. Quy trình trả Connection (`close()`)](#2-quy-trình-trả-connection-close)
  - [3. HouseKeeper & Leak Detection](#3-housekeeper--leak-detection)
- [IV. Cấu hình Production & Tham số cốt lõi](#iv-cấu-hình-production--tham-số-cốt-lõi)
  - [1. Giải thích tham số & Công thức tính Pool Size](#1-giải-thích-tham-số--công-thức-tính-pool-size)
  - [2. Mẫu cấu hình chuẩn (`application.yml`)](#2-mẫu-cấu-hình-chuẩn-applicationyml)
- [V. Giám sát Metrics & Bắt bệnh Lỗi thường gặp](#v-giám-sát-metrics--bắt-bệnh-lỗi-thường-gặp)
  - [1. Key Metrics cần giám sát](#1-key-metrics-cần-giám-sát)
  - [2. Troubleshooting (Xử lý sự cố phỏng vấn)](#2-troubleshooting-xử-lý-sự-cố-phỏng-vấn)
- [VI. Cheat-sheet Phỏng vấn (Q&A Flashcards)](#vi-cheat-sheet-phỏng-vấn-qa-flashcards)

---

## I. Tổng quan

### 1. HikariCP là gì?
[HikariCP](https://github.com/brettwooldridge/hikaricp) là một JDBC Connection Pool cực nhẹ (~130KB), siêu nhanh và chuẩn mực hàng đầu cho Java/Spring Boot.

**Tính năng nổi bật:**
- Tự động quản lý vòng đời Connection, kiểm tra tính sống sót trước khi giao cho client.
- Tự động rollback các transaction chưa hoàn thành và xóa cảnh báo SQL warning khi trả Connection về pool.
- Zero-dependency: Không phụ thuộc vào thư viện bên thứ 3.

### 2. Tại sao HikariCP là mặc định của Spring Boot (từ 2.0+)?
Từ Spring Boot 2.0, Spring chính thức chọn HikariCP thay cho Tomcat JDBC nhờ **4 ưu điểm vượt trội**:
1. **Hiệu năng vượt trội**: Nhanh gấp **2 - 13 lần** so với Tomcat JDBC, DBCP2, C3P0. Chi phí lấy Connection tiệm cận mức micro-seconds.
2. **Siêu nhẹ (~130KB)**: Zero-dependency, giúp ứng dụng khởi động cực nhanh và tránh rủi ro xung đột thư viện.
3. **Cơ chế Lock-free**: Giảm tối đa hiện tượng nghẽn luồng (Thread Contention) khi tải cao nhờ cấu trúc `ConcurrentBag`.
4. **Triết lý Fail-Fast & Tự phục hồi**: Ném lỗi dứt khoát khi đứt mạng/DB sập thay vì treo luồng, tự phục hồi ngay khi DB sống lại.

### 3. So sánh nhanh với các Pool khác

| Tiêu chí | HikariCP | Tomcat JDBC | Commons DBCP2 | C3P0 |
| :--- | :--- | :--- | :--- | :--- |
| **Mặc định Spring Boot** | **2.0+** | 1.x | Không | Không |
| **Tốc độ** | **Nhanh nhất** | Khá | Trung bình | Chậm |
| **Đồng bộ hóa (Lock)** | **Lock-free (`ConcurrentBag`)** | Synchronized / Locks | Synchronized / Locks | Heavy Lock |
| **Dung lượng JAR** | **~130 KB** | ~150 KB | ~200 KB | > 600 KB |

---

## II. Bí mật hiệu năng (Top 4 "Killer Features")

### 1. Tối ưu Bytecode (JIT Inlining)
- JIT Compiler chỉ tự động **inline** (gộp mã hàm) nếu kích thước Bytecode của method nhỏ hơn 35 bytes.
- Tác giả HikariCP đã tối ưu từng dòng code Java để Bytecode cực nhỏ, giúp JIT inline phương thức `getConnection()` trực tiếp thành mã máy CPU.

### 2. FastList (Thay thế ArrayList)
- Khi thực thi SQL, Connection tạo ra các `Statement`. JDBC yêu cầu đóng `Statement` theo thứ tự LIFO (mở sau đóng trước).
- `ArrayList` chuẩn xóa phần tử từ đầu mảng ($0 \to n-1$), gây tốn $O(n)$ và di chuyển mảng.
- **`FastList` của HikariCP**:
  - Bỏ kiểm tra chỉ số `rangeCheck()`.
  - **Duyệt ngược từ cuối mảng ($n-1 \to 0$)**, tìm thấy `Statement` cần đóng ngay ở phần tử đầu tiên $\to$ Đạt độ phức tạp **$O(1)$**.

### 3. ConcurrentBag (Lock-free Architecture)
Cấu trúc dữ liệu "xương sống" giúp mượn/trả Connection mà **không cần Lock (`synchronized`)**:
- Sử dụng **`ThreadLocal`**: Luồng vừa dùng xong Connection sẽ cất vào túi riêng `ThreadLocal`. Lần sau cần, lấy ra dùng ngay **$O(1)$ không tranh chấp**.
- Nếu túi riêng trống $\to$ Quét túi chung (`CopyOnWriteArrayList`) dùng kỹ thuật **CAS (Compare-And-Swap)**.
- Nếu túi chung hết $\to$ Đứng chờ ở `SynchronousQueue` để nhận Connection trực tiếp từ luồng khác vừa `close()` (Hand-off mechanism).

### 4. Direct Delegate Proxy (Mã hóa bằng Javassist)
- JDK Dynamic Proxy dùng Reflection nên rất chậm và tốn CPU.
- HikariCP dùng **Javassist** sinh ra lớp Proxy riêng (`HikariProxyConnection`) tại runtime.
- Proxy gọi thẳng đến phương thức gốc (`delegate.prepareStatement()`) mà **không dùng Reflection**, đưa chi phí gọi Proxy về gần như **bằng 0**.

---

## III. Luồng mượn & trả Connection (Core Workflow)

### 1. Quy trình lấy Connection (`getConnection()`)
Tốc độ lấy Connection đi qua **3 tầng tìm kiếm**:
1. **Tầng 1 (Túi riêng - ThreadLocal)**: Lấy lại Connection luồng vừa dùng trước đó (Lock-free, cực nhanh).
2. **Tầng 2 (Túi chung - Shared List)**: Quét danh sách dùng chung và lấy bằng thuật toán CAS.
3. **Tầng 3 (Hàng chờ - HandOff Queue)**: Nếu pool bận hết, luồng chờ tại `SynchronousQueue`. Khi luồng khác trả Connection, nó nhận trực tiếp tại đây mà không cần cất vào pool.

### 2. Quy trình trả Connection (`close()`)
1. Reset các trạng thái của Connection (Auto-commit, Rollback nếu còn transaction lở dở, clear SQL Warnings).
2. Kiểm tra có luồng nào đang chờ ở HandOff Queue không $\to$ Nối trực tiếp cho luồng đó.
3. Nếu không ai chờ $\to$ Cất vào `ThreadLocal` của luồng hiện tại để tái sử dụng.

### 3. HouseKeeper & Leak Detection
- **HouseKeeper**: Tiến trình chạy ngầm định kỳ (mỗi 30s) để dọn dẹp các Connection hết hạn (`maxLifetime`) hoặc rảnh quá lâu (`idleTimeout`).
- **Leak Detection**: Nếu bật `leakDetectionThreshold`, nếu một luồng giữ Connection lâu hơn mốc này mà chưa trả, HikariCP sẽ log ngay **Stack Trace** cảnh báo rò rỉ kết nối (Connection Leak).

---

## IV. Cấu hình Production & Tham số cốt lõi

### 1. Giải thích tham số & Công thức tính Pool Size

- **`maximum-pool-size`**: Số connection tối đa.
  - *Công thức khuyến nghị của PostgreSQL/HikariCP*:
    $$\text{Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindle Count}$$
  - *Thực tế*: Với server 4 Core, pool size chỉ cần khoảng **10 - 20** connection là đủ đáp ứng hàng nghìn request/s. Pool quá to sẽ gây nghẽn CPU do Context Switching ở DB Server.
- **`minimum-idle`**: Số connection rảnh tối thiểu.
  - *Khuyên dùng*: **Đặt bằng `maximum-pool-size`** (Fixed Pool) để tránh chi phí tạo/hủy connection liên tục.
- **`connection-timeout`**: Thời gian tối đa chờ xin Connection (Mặc định 30s, nên giảm xuống **3s - 5s** để Fail-Fast).
- **`max-lifetime`**: Thời gian sống tối đa của 1 Connection (Mặc định 30 phút). **Bắt buộc ngắn hơn timeout của DB/Firewall 30-60 giây**.

### 2. Mẫu cấu hình chuẩn (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: secret
    hikari:
      maximum-pool-size: 15
      minimum-idle: 15
      connection-timeout: 5000       # 5 giây (Fail-fast)
      max-lifetime: 1800000          # 30 phút
      idle-timeout: 600000           # 10 phút
      leak-detection-threshold: 2000 # Cảnh báo nếu giữ connection > 2 giây
      pool-name: HikariPool-Primary
```

---

## V. Giám sát Metrics & Bắt bệnh Lỗi thường gặp

### 1. Key Metrics cần giám sát
Giám sát qua **Micrometer + Prometheus + Grafana**:
- `hikaricp.connections.active`: Số connection đang thực thi query.
- `hikaricp.connections.idle`: Số connection rảnh rỗi.
- `hikaricp.connections.pending`: **(Rất quan trọng)** Số luồng đang phải đứng chờ lấy connection. Nếu > 0 liên tục $\to$ Pool đang bị quá tải.
- `hikaricp.connections.timeout.total`: Tổng số lần bị timeout do hết connection.

### 2. Troubleshooting (Xử lý sự cố phỏng vấn)

#### Lỗi 1: `Connection is not available, request timed out after 5000ms`
- **Nguyên nhân**: Hết Connection trong pool. Do query quá chậm, bể pool quá nhỏ, hoặc **Connection Leak** (quên đóng connection/transaction chạy quá lâu).
- **Cách xử lý**:
  1. Kiểm tra metric `pending` và log `leak-detection-threshold`.
  2. Tối ưu lại Slow Query (thêm Index) để giải phóng Connection nhanh hơn.
  3. Kiểm tra mã nguồn xem có chỗ nào mở Connection thủ công mà chưa close không.

#### Lỗi 2: `Connection has been closed / Connection reset by peer`
- **Nguyên nhân**: `max-lifetime` của HikariCP đặt dài hơn timeout dọn dẹp kết nối rảnh của DB Server hoặc Firewall.
- **Cách xử lý**: Giảm `max-lifetime` xuống thấp hơn cấu hình `wait_timeout` của Database (ví dụ DB timeout 10 phút thì đặt `max-lifetime` 9 phút).

---

## VI. Cheat-sheet Phỏng vấn (Q&A Flashcards)

**Q1: Vì sao HikariCP lại nhanh hơn các Connection Pool khác?**
> *Trả lời ngắn*: Nhờ 4 yếu tố: (1) Cấu trúc `ConcurrentBag` Lock-free sử dụng `ThreadLocal`, (2) `FastList` xóa Statement theo LIFO với $O(1)$, (3) Mã hóa Proxy trực tiếp bằng Javassist loại bỏ Reflection, (4) Bytecode cực gọn giúp JIT Inlining.

**Q2: Tại sao đặt `maximumPoolSize` quá lớn lại làm giảm hiệu năng hệ thống?**
> *Trả lời ngắn*: Vì Database Server bị giới hạn bởi số Cores của CPU. Quá nhiều connection đồng thời sẽ khiến DB Server tốn tài nguyên cho CPU Context Switching và tranh chấp Disk I/O thay vì xử lý query thực tế.

**Q3: Sự khác biệt giữa `connectionTimeout` và `maxLifetime` là gì?**
> *Trả lời ngắn*: `connectionTimeout` là thời gian tối đa **ứng dụng đứng chờ** để mượn 1 Connection. `maxLifetime` là **thời gian sống tối đa** của 1 Connection trong pool trước khi bị dọn dẹp và làm mới.

**Q4: Cách phát hiện Connection Leak trong Spring Boot?**
> *Trả lời ngắn*: Cấu hình `spring.datasource.hikari.leak-detection-threshold=2000` (2 giây). Nếu luồng nào giữ connection quá 2s mà chưa trả, HikariCP sẽ in ra Stack Trace chỉ rõ dòng code đang gây rò rỉ.
