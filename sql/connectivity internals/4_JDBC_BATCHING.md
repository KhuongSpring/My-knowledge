# Chương 4: JDBC Batching - Tối Ưu Hóa I/O & Network Round-Trips

![Status](https://img.shields.io/badge/JDBC_Batching-green) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## Mục lục

- [I. Tổng quan về JDBC Batching](#i-tổng-quan-về-jdbc-batching)
  - [1. Bài toán Network Round-trip Overhead](#1-bài-toán-network-round-trip-overhead)
  - [2. JDBC Batching là gì?](#2-jdbc-batching-là-gì)
- [II. Cơ chế vận hành bên dưới (Internals)](#ii-cơ-chế-vận-hành-bên-dưới-internals)
  - [1. So sánh Single-Execution vs Batch-Execution](#1-so-sánh-single-execution-vs-batch-execution)
  - [2. Luồng xử lý Packet trên Network & Socket Buffer](#2-luồng-xử-lý-packet-trên-network--socket-buffer)
    - [2.1. Client-Side (JDBC Driver)](#21-client-side-jdbc-driver)
    - [2.2. Server-Side (Database Engine)](#22-server-side-database-engine)
    - [2.3. Rewrite Batched Statements](#23-rewrite-batched-statements)
- [III. Hướng dẫn triển khai (JDBC, Spring JDBC & Hibernate)](#iii-hướng-dẫn-triển-khai-jdbc-spring-jdbc--hibernate)
  - [1. Thuần JDBC (`addBatch()` & `executeBatch()`)](#1-thuần-jdbc-addbatch--executebatch)
  - [2. Spring JdbcTemplate (`batchUpdate()`)](#2-spring-jdbctemplate-batchupdate)
  - [3. Spring Data JPA & Hibernate Batching](#3-spring-data-jpa--hibernate-batching)
- [IV. Các bẫy thường gặp (Pitfalls & Gotchas)](#iv-các-bẫy-thường-gặp-pitfalls--gotchas)
  - [1. Bẫy `GenerationType.IDENTITY` (Khai tử Batching)](#1-bẫy-generationtypeidentity-khai-tử-batching)
  - [2. Quên bật tham số Driver (`reWriteBatchedInserts`)](#2-quên-bật-tham-số-driver-rewritebatchedinserts)
  - [3. Tràn bộ nhớ (OutOfMemoryError) do Batch Size quá lớn](#3-tràn-bộ-nhớ-outofmemoryerror-do-batch-size-quá-lớn)
- [V. Cấu hình Production mẫu (application.yml)](#v-cấu-hình-production-mẫu-applicationyml)
- [VI. Cheat-sheet Phỏng vấn (Q&A Flashcards)](#vi-cheat-sheet-phỏng-vấn-qa-flashcards)

---

## I. Tổng quan về JDBC Batching

### 1. Bài toán Network Round-trip Overhead
**Round-trip Time (RTT):** là toàn bộ quá trình: Ứng dụng gửi request qua mạng $\rightarrow$ DB nhận request $\rightarrow$ DB xử lý $\rightarrow$ DB trả kết quả về qua mạng $\rightarrow$ Ứng dụng nhận kết quả.

Khi ứng dụng Java cần chèn (`INSERT`) hoặc cập nhật (`UPDATE`) $N$ bản ghi (ví dụ: 10.000 bản ghi) vào Database theo cách thông thường (từng câu lệnh một):

$$\text{Tổng thời gian} = N \times (\text{Thời gian truyền mạng RTT} + \text{Thời gian DB Parse/Compile} + \text{Thời gian Ghi Disk})$$

- Nếu độ trễ mạng giữa App và DB chỉ là **1ms**, việc thực thi 10.000 câu lệnh riêng lẻ sẽ tốn **tối thiểu 10 giây** chỉ riêng cho việc truyền nhận gói tin trên mạng (Network Latency).
- Database Engine phải tốn tài nguyên liên tục cho việc Parse, Optimize và Compile 10.000 câu SQL riêng biệt.

### 2. JDBC Batching là gì?
**JDBC Batching** là kỹ thuật gom nhóm nhiều câu lệnh SQL (hoặc cùng một câu lệnh `PreparedStatement` với mảng các tập tham số khác nhau) thành một gói (Batch) duy nhất và gửi sang Database Server trong **1 Network Round-trip duy nhất**.

- Giảm từ $N$ lượt gọi mạng xuống còn $N / \text{Batch Size}$ lượt.
- Database Server chỉ cần Parse/Compile câu lệnh SQL **đúng 1 lần** duy nhất, sau đó áp dụng hàng loạt mảng dữ liệu.

---

## II. Cơ chế vận hành bên dưới (Internals)

### 1. So sánh Single-Execution vs Batch-Execution

| Tiêu chí | Single Execution (Chạy từng câu) | Batch Execution (Gom nhóm) |
| :--- | :--- | :--- |
| **Số lần gọi mạng (Network RTT)** | $N$ lần | $1$ hoặc $N / \text{Batch Size}$ lần |
| **Chi phí DB Parsing & Compile** | Lặp lại $N$ lần | **Chỉ 1 lần** (PreparedStatement) |
| **Mức độ tiêu tốn Socket I/O** | Rất cao (liên tục mở/đóng TCP Packet) | Cực thấp (truyền 1 Buffer lớn) |
| **Tốc độ xử lý (Throughput)** | Chậm (100 - 500 ops/sec) | **Cực nhanh (10.000 - 50.000+ ops/sec)** |

### 2. Luồng xử lý Packet trên Network & Socket Buffer
#### 2.1. Client-Side (JDBC Driver):
   - Khi gọi `addBatch()`, JDBC Driver lưu mảng tham số vào bộ nhớ đệm nội bộ (Client-side Buffer).
   - Khi gọi `executeBatch()`, Driver đóng gói toàn bộ mảng tham số này vào các gói tin TCP (Socket Stream) và gửi 1 lần qua đường truyền mạng sang DB Server.
#### 2.2. Server-Side (Database Engine):
   - Database Engine tiếp nhận Stream dữ liệu, thực hiện compile SQL template đúng 1 lần.
   - Tiến hành ghi hàng loạt bản ghi vào WAL (Write-Ahead Log) / Redo Log trong một giao dịch duy nhất, giúp tối ưu hóa đáng kể chi phí Disk I/O.
#### 2.3. Rewrite Batched Statements:
Nếu bạn chỉ cấu hình Batching ở phía code Java (tức là dùng `addBatch()` và `executeBatch()`), các Driver của Database (như MySQL, PostgreSQL) mặc định vẫn sẽ gửi các câu lệnh SQL riêng lẻ đóng gói trong một gói tin TCP, và Database Server vẫn phải parse, execute từng câu lệnh một.

Để thực sự tối ưu, bạn phải ép Driver "viết lại" các câu lệnh đó thành cú pháp Multi-value Insert.

- Đối với MySQL: Thêm cờ `rewriteBatchedStatements=true` vào chuỗi JDBC URL.
  - Trước khi rewrite: INSERT INTO user (name) VALUES ('A'); INSERT INTO user (name) VALUES ('B');
  - Sau khi rewrite: INSERT INTO user (name) VALUES ('A'), ('B');
  - Hiệu quả: DB chỉ cần parse cú pháp SQL đúng 1 lần và thực thi 1 lần. Hiệu năng có thể tăng gấp 10-20 lần.

- Đối với PostgreSQL: Thêm cờ reWriteBatchedInserts=true. Cơ chế của Postgres là đóng gói nhiều tham số vào một bản tin Bind duy nhất ở tầng giao thức.

---

## III. Hướng dẫn triển khai (JDBC, Spring JDBC & Hibernate)

### 1. Thuần JDBC (`addBatch()` & `executeBatch()`)

```java
String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
try (Connection conn = dataSource.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql)) {
    
    conn.setAutoCommit(false); // Bắt đầu Transaction
    int batchSize = 100;
    int count = 0;

    for (User user : userList) {
        pstmt.setString(1, user.getName());
        pstmt.setString(2, user.getEmail());
        pstmt.addBatch(); // Thêm vào bộ nhớ đệm của Driver

        if (++count % batchSize == 0) {
            pstmt.executeBatch(); // Gửi batch sang DB
            pstmt.clearBatch();   // Xóa bộ nhớ đệm Driver
        }
    }
    pstmt.executeBatch(); // Gửi phần còn dư
    conn.commit();        // Commit Transaction
}
```

### 2. Spring JdbcTemplate (`batchUpdate()`)

Spring `JdbcTemplate` cung cấp phương thức `batchUpdate()` giúp xử lý ngắn gọn và tự động quản lý kết nối:

```java
String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
jdbcTemplate.batchUpdate(sql, userList, 100, (pstmt, user) -> {
    pstmt.setString(1, user.getName());
    pstmt.setString(2, user.getEmail());
});
```

### 3. Spring Data JPA & Hibernate Batching

Để Hibernate tự động gom batch khi gọi `saveAll()`, bắt buộc phải cấu hình các thuộc tính sau trong Spring Boot:

```properties
# Bật batching trong Hibernate
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true
```

- **`hibernate.jdbc.batch_size`**: Số lượng câu lệnh tối đa trong 1 Batch (khuyên dùng 30 - 100).
- **`order_inserts` / `order_updates`**: Sắp xếp các câu lệnh Insert/Update theo thứ tự Entity để Hibernate có thể gộp chúng vào cùng 1 Batch một cách tối ưu nhất.

---

## IV. Các bẫy thường gặp (Pitfalls & Gotchas)

### 1. Bẫy `GenerationType.IDENTITY` (Khai tử Batching)
> [!CAUTION]
> Đây là câu hỏi phỏng vấn "kinh điển" về Spring Data JPA & Hibernate!

- **Vấn đề**: Khi sử dụng `@GeneratedValue(strategy = GenerationType.IDENTITY)` (tự tăng Auto-increment trong MySQL), Hibernate bắt buộc phải `INSERT` câu lệnh ngay lập tức xuống DB để lấy ID vừa được sinh ra gán ngược lại cho đối tượng Entity trong Java.
- **Hậu quả**: **Hibernate tự động ngắt (Disable) toàn bộ cơ chế Batching cho các lệnh Insert!** Mọi câu lệnh `saveAll()` sẽ bị lùi về chạy từng câu một.
- **Giải pháp**:
  1. Chuyển sang chiến lược `GenerationType.SEQUENCE` (PostgreSQL, Oracle) kết hợp với `allocationSize`.
  2. Tự sinh ID ở ứng dụng bằng **UUID**, **UUID v7**, **TSID**, hoặc **Snowflake ID**.
  3. Sử dụng `JdbcTemplate.batchUpdate()` nếu bắt buộc phải giữ `IDENTITY` trong MySQL.

### 2. Quên bật tham số Driver (`reWriteBatchedInserts`)
- Mặc định, dù có gọi `executeBatch()`, JDBC Driver của MySQL vẫn gửi $N$ câu lệnh riêng biệt trong 1 gói tin (`INSERT...; INSERT...;`).
- **Giải pháp**: Phải thêm tham số cấu hình vào JDBC URL connection string:
  - **MySQL**: `jdbc:mysql://localhost:3306/db?reWriteBatchedInserts=true`
  - **PostgreSQL**: `jdbc:postgresql://localhost:5432/db?reWriteBatchedInserts=true`
- Khi bật `reWriteBatchedInserts=true`, Driver sẽ tự động ghi lại (rewrite) nhiều câu Insert thành 1 câu duy nhất dạng multi-values:
  `INSERT INTO users (name, email) VALUES ('A', 'a@a.com'), ('B', 'b@b.com'), ...;` (Tăng tốc thêm từ 2x - 5x lần!).

### 3. Tràn bộ nhớ (OutOfMemoryError) do Batch Size quá lớn
- **Vấn đề**: Lưu quá nhiều Entity trong Persistence Context (L1 Cache của Hibernate) mà không xả bộ nhớ.
- **Giải pháp**: Khi lưu file lớn (ví dụ 100.000 bản ghi), phải định kỳ gọi `entityManager.flush()` và `entityManager.clear()` để giải phóng L1 Cache:

```java
for (int i = 0; i < entities.size(); i++) {
    entityManager.persist(entities.get(i));
    if (i % 50 == 0) {
        entityManager.flush(); // Đẩy SQL xuống JDBC
        entityManager.clear(); // Xóa sạch L1 Cache giải phóng RAM
    }
}
```

---

## V. Cấu hình Production mẫu (application.yml)

Cấu hình tối ưu hoàn chỉnh cho ứng dụng Spring Boot sử dụng Spring Data JPA:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb?reWriteBatchedInserts=true
    username: admin
    password: secret
    hikari:
      maximum-pool-size: 15
      minimum-idle: 15

  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          batch_versioned_data: true
        order_inserts: true
        order_updates: true
        generate_statistics: false # Đặt true ở Dev để debug kiểm tra số batch
```

---

## VI. Cheat-sheet Phỏng vấn (Q&A Flashcards)

**Q1: JDBC Batching giúp tối ưu hệ thống ở những điểm nào?**
> *Trả lời ngắn*: Giảm thiểu thời gian chờ trễ mạng (Network Round-trips), tối ưu hóa Socket I/O, giảm chi phí Parsing/Compile SQL ở Database Server và giảm chi phí Ghi đĩa (Disk I/O) nhờ gom nhóm vào 1 Transaction.

**Q2: Tại sao `@GeneratedValue(strategy = GenerationType.IDENTITY)` lại làm hỏng (disable) Hibernate Batch Insert?**
> *Trả lời ngắn*: Vì `IDENTITY` yêu cầu Database tự sinh ID khi insert. Hibernate cần giá trị ID này ngay lập tức để quản lý Entity trong Persistence Context, do đó nó buộc phải gửi lệnh `INSERT` ngay thay vì chờ gom batch.

**Q3: Làm thế nào để giải quyết vấn đề Batch Insert bị disable do `IDENTITY` trong Spring Boot?**
> *Trả lời ngắn*: Có 3 cách: (1) Đổi sang `GenerationType.SEQUENCE`, (2) Tự sinh ID dạng UUID/TSID/Snowflake trên ứng dụng, (3) Dùng `JdbcTemplate.batchUpdate()` cho tác vụ Bulk Insert.

**Q4: Vai trò của tham số `reWriteBatchedInserts=true` trong JDBC URL là gì?**
> *Trả lời ngắn*: Cho phép JDBC Driver gộp nhiều câu lệnh `INSERT` đơn lẻ trong Batch thành 1 câu `INSERT` có nhiều dòng `VALUES (..), (..)` duy nhất, giúp giảm dung lượng gói tin và tăng tốc độ xử lý của DB lên gấp nhiều lần.
