# Chương 2: Pessimistic Locking - Sức Mạnh "Bạo Chúa"

![Status](https://img.shields.io/badge/Pessimistic_Locking-green) ![Topic](https://img.shields.io/badge/FOR_UPDATE_&_Deadlock-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
  - [1. Bản chất "Bi Quan": Đẩy trách nhiệm xuống tận Database Engine](#1-bản-chất-bi-quan-đẩy-trách-nhiệm-xuống-tận-database-engine)
  - [2. Nhắc lại: Đối lập hoàn toàn với Optimistic Locking](#2-nhắc-lại-đối-lập-hoàn-toàn-với-optimistic-locking)
- [II. `PESSIMISTIC_WRITE` - Khóa Ghi Độc Quyền](#ii-pessimistic_write---khóa-ghi-độc-quyền)
  - [1. Hibernate dịch `find()` thành `FOR UPDATE` như thế nào?](#1-hibernate-dịch-find-thành-for-update-như-thế-nào)
  - [2. Dưới đáy MySQL InnoDB: Index-Record Lock](#2-dưới-đáy-mysql-innodb-index-record-lock)
- [III. `PESSIMISTIC_READ` - Khóa Đọc Chia Sẻ](#iii-pessimistic_read---khóa-đọc-chia-sẻ)
  - [1. `FOR SHARE` / `LOCK IN SHARE MODE`](#1-for-share--lock-in-share-mode)
  - [2. Use-case thực tế: Tính toán báo cáo an toàn](#2-use-case-thực-tế-tính-toán-báo-cáo-an-toàn)
- [IV. Tử Huyệt Của Khóa Bi Quan](#iv-tử-huyệt-của-khóa-bi-quan)
  - [1. Cạn Kiệt Connection Pool](#1-cạn-kiệt-connection-pool)
  - [2. Deadlock Kinh Điển Ở Tầng Database](#2-deadlock-kinh-điển-ở-tầng-database)
- [V. Giải Pháp: Timeouts - Đừng Bao Giờ Khóa Mù Quáng](#v-giải-pháp-timeouts---đừng-bao-giờ-khóa-mù-quáng)
- [VI. Vũ Khí Nâng Cao: `NOWAIT` \& `SKIP LOCKED`](#vi-vũ-khí-nâng-cao-nowait--skip-locked)
  - [1. `FOR UPDATE NOWAIT`: Thà báo lỗi còn hơn chờ đợi](#1-for-update-nowait-thà-báo-lỗi-còn-hơn-chờ-đợi)
  - [2. `FOR UPDATE SKIP LOCKED`: Vũ khí bí mật cho hệ thống Queue](#2-for-update-skip-locked-vũ-khí-bí-mật-cho-hệ-thống-queue)
- [VII. Bảng So Sánh: 4 Biến Thể Của Khóa Bi Quan](#vii-bảng-so-sánh-4-biến-thể-của-khóa-bi-quan)
- [VIII. Trade-off: Optimistic vs Pessimistic - Chọn Sao Cho Đúng?](#viii-trade-off-optimistic-vs-pessimistic---chọn-sao-cho-đúng)
- [IX. Cheat Sheet Phỏng vấn (Interview Q\&A)](#ix-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

### 1. Bản chất "Bi Quan": Đẩy trách nhiệm xuống tận Database Engine

Nếu [Optimistic Locking](1_OPTIMISTIC_LOCKING.md) đặt cược rằng "xung đột hiếm khi xảy ra", thì Pessimistic Locking đi theo triết lý hoàn toàn ngược lại: **"xung đột chắc chắn sẽ xảy ra, nên phải phòng thủ ngay từ đầu"**.

Khác với Optimistic Locking (toàn bộ logic nằm ở tầng Application, DB "vô tư" không biết gì), Pessimistic Locking **đẩy thẳng trách nhiệm khóa xuống tận Database Engine**. Khi 1 Transaction chạm vào dòng dữ liệu nào, nó yêu cầu DB **đóng băng (lock)** ngay dòng đó lại — cấm tiệt bất kỳ ai khác đụng vào cho tới khi Transaction hiện tại gọi `commit()` hoặc `rollback()`.

```
User A: SELECT * FROM products WHERE id = 1 FOR UPDATE;
        ┌─────────────────────────────────┐
        │  DB khóa CỨNG dòng id=1 lại     │
        └─────────────────────────────────┘

User B: SELECT * FROM products WHERE id = 1 FOR UPDATE;
        ⛔ BỊ TREO (BLOCKED) — phải xếp hàng chờ
        ... chờ ... chờ ... chờ ...

User A: COMMIT; (hoặc ROLLBACK)
        └──► DB nhả khóa
User B: được cấp Lock, tiếp tục chạy
```

### 2. Nhắc lại: Đối lập hoàn toàn với Optimistic Locking

| | Optimistic Locking | Pessimistic Locking |
| :--- | :--- | :--- |
| Ai chịu trách nhiệm khóa? | Tầng Application (Hibernate + cột `version`) | Tầng Database Engine (Row Lock thật sự) |
| Khi nào phát hiện xung đột? | **Sau khi** cố ghi (kiểm tra `update count`) | **Ngay từ lúc đọc** (chặn luôn từ đầu) |
| Transaction khác bị ảnh hưởng ra sao? | Không bị chặn, chỉ có thể `rollback` sau này | Bị **Block, treo cứng** cho tới khi Lock được nhả |

Cơ chế này được JPA hỗ trợ thông qua 2 `LockModeType` chính: **`PESSIMISTIC_WRITE`** và **`PESSIMISTIC_READ`** — Mục II và III sẽ mổ xẻ từng loại.

---

## II. `PESSIMISTIC_WRITE` - Khóa Ghi Độc Quyền

### 1. Hibernate dịch `find()` thành `FOR UPDATE` như thế nào?

Khi bạn gọi `EntityManager.find()` kèm `LockModeType.PESSIMISTIC_WRITE`, Hibernate lập tức gắn thêm mệnh đề `FOR UPDATE` vào đuôi câu SQL:

```java
Product product = entityManager.find(
    Product.class,
    1L,
    LockModeType.PESSIMISTIC_WRITE
);
```

SQL sinh ra:

```sql
SELECT * FROM products WHERE id = 1 FOR UPDATE;
```

Với Spring Data JPA, cú pháp gọn hơn nhiều nhờ annotation `@Lock`:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

### 2. Dưới đáy MySQL InnoDB: Index-Record Lock

Ở tầng vật lý (lấy ví dụ MySQL InnoDB), câu `FOR UPDATE` tìm đến đúng dòng `id=1` trên B+Tree Index và đặt một **Index-Record Lock** (khóa bản ghi) lên đó — về bản chất tương tự cơ chế **Record Lock** đã nhắc tới ở [Chương Isolation Level & MVCC, Mục VII](../consistency%20%26%20transaction%20management/2_ISOLATION_LEVEL_AND_MVCC.md), nhưng lần này Lock được xin **tường minh, chủ động** bởi code Application, chứ không phải hệ quả ngầm của Next-Key Lock khi chạy DML.

```
Timeline thực tế
─────────────────
T1  User A: SELECT * FROM products WHERE id=1 FOR UPDATE;
    → DB cấp Exclusive Lock (X-Lock) trên record id=1

T2  User B: SELECT * FROM products WHERE id=1 FOR UPDATE;
    → DB phát hiện record đang bị khóa
    → Thread của User B bị BLOCK ngay tại tầng Database
    → Connection HikariCP của User B "đứng hình" tại đây, chưa trả lại Pool

T3  User B: (cũng có thể là 1 câu UPDATE/DELETE bình thường trên id=1)
    → CŨNG bị chặn tương tự, vì PESSIMISTIC_WRITE là khóa ĐỘC QUYỀN
      (không ai được đọc-để-sửa HAY ghi trên dòng đó cho tới khi User A xong)

T4  User A: connection.commit();  (hoặc rollback())
    → DB nhả X-Lock

T5  User B: được cấp Lock ngay lập tức, câu lệnh của B tiếp tục chạy
```

> **Ghi nhớ**: `PESSIMISTIC_WRITE` là loại khóa **mạnh nhất** — không một ai khác (dù chỉ muốn đọc để chuẩn bị sửa) được phép chen vào dòng đang bị khóa. Nó phù hợp cho các thao tác **chắc chắn sẽ ghi** ngay sau khi đọc, ví dụ: đọc số dư ví để trừ tiền, đọc số lượng vé để giảm tồn kho.

---

## III. `PESSIMISTIC_READ` - Khóa Đọc Chia Sẻ

### 1. `FOR SHARE` / `LOCK IN SHARE MODE`

Nếu `PESSIMISTIC_WRITE` là "cấm tiệt mọi người", thì `PESSIMISTIC_READ` "khoan dung" hơn — nó chỉ cấm **ghi**, chứ không cấm **đọc**:

```java
Product product = entityManager.find(
    Product.class,
    1L,
    LockModeType.PESSIMISTIC_READ
);
```

Hibernate dịch thành:

```sql
-- PostgreSQL / MySQL 8+ (chuẩn ANSI SQL)
SELECT * FROM products WHERE id = 1 FOR SHARE;

-- Cú pháp cũ của MySQL (trước 8.0)
SELECT * FROM products WHERE id = 1 LOCK IN SHARE MODE;
```

### 2. Use-case thực tế: Tính toán báo cáo an toàn

`PESSIMISTIC_READ` cho phép nhiều Transaction **cùng lúc** đọc (SELECT) 1 dòng dữ liệu — User B, C, D thoải mái đọc song song mà không ai bị chặn. Nhưng nếu **bất kỳ ai** trong số họ (hoặc 1 người khác) định `UPDATE`/`DELETE` dòng đó, người đó sẽ bị treo lại cho tới khi tất cả các "khóa đọc chia sẻ" đang giữ được giải phóng.

```
User A: SELECT ... FOR SHARE;   → Lock chia sẻ (Shared Lock)
User B: SELECT ... FOR SHARE;   → Cũng được cấp Shared Lock, KHÔNG bị chặn
                                    (nhiều Shared Lock có thể tồn tại đồng thời)

User C: UPDATE products SET stock = 90 WHERE id = 1;
        ⛔ BỊ CHẶN — vì đang có Shared Lock của A và B chưa giải phóng
```

**Ví dụ đời thực**: Bạn cần đọc `balance` của 1 tài khoản để tính toán 1 báo cáo tài chính phức tạp (nhiều bước tính toán, mất vài giây), và muốn chắc chắn **không ai được phép sửa số dư đó** trong lúc bạn đang tính — nhưng đồng thời vẫn cho phép nhân viên khác cùng đọc để đối chiếu song song. `PESSIMISTIC_READ` chính là công cụ dành cho tình huống này — mạnh hơn 1 câu `SELECT` thường (đảm bảo dữ liệu "đứng yên"), nhưng vẫn nhẹ nhàng hơn `PESSIMISTIC_WRITE` (không chặn những người chỉ muốn đọc).

---

## IV. Tử Huyệt Của Khóa Bi Quan

### 1. Cạn Kiệt Connection Pool

Đây chính là cái giá đắt nhất của Pessimistic Locking. Hãy nhớ lại kiến trúc HikariCP ở [Chương HikariCP](../connectivity%20internals/3_HIKARI_CP.md): mỗi Connection lấy ra từ Pool là 1 tài nguyên **hữu hạn**.

Khi User A bị `BLOCK` chờ Lock từ User B, **Connection của User A vẫn đang bị giữ nguyên trong lúc chờ** — nó không hề được trả về Pool. Nếu hàng loạt Request cùng dồn vào tranh chấp 1 vài dòng "hot" (ví dụ cùng mua 1 sản phẩm đang sale sốc), Connection Pool có thể cạn kiệt cực nhanh:

```
maximum-pool-size = 20

Request 1-20: đều xin SELECT ... FOR UPDATE trên CÙNG 1 dòng "hot"
    → Chỉ có 1 Request thực sự cầm được Lock và chạy tiếp
    → 19 Request còn lại BỊ TREO, giữ nguyên 19 Connection trong trạng thái chờ
    → Pool 20/20 Connection đều bị chiếm dụng (dù chỉ có 1 cái đang "làm việc" thật sự)
    → Request thứ 21 tới sau, HOÀN TOÀN không xin được Connection nào
    → SQLTransientConnectionException: Connection is not available
```

### 2. Deadlock Kinh Điển Ở Tầng Database

Nguy hiểm hơn nữa là **Deadlock thật sự ở tầng Database** (khác với "Application-level Deadlock" do `REQUIRES_NEW` đã phân tích ở [Chương Propagation, Mục V](../consistency%20%26%20transaction%20management/3_PROPAGATION_AND_CONNECTION_MANAGEMENT.md) — lần này là deadlock kinh điển do 2 Transaction khóa chéo tài nguyên của nhau):

```
Transaction A                          Transaction B
──────────────                         ──────────────
SELECT * FROM accounts
WHERE id = 1 FOR UPDATE;               SELECT * FROM accounts
→ giữ Lock trên id=1                   WHERE id = 2 FOR UPDATE;
                                        → giữ Lock trên id=2

SELECT * FROM accounts
WHERE id = 2 FOR UPDATE;               SELECT * FROM accounts
⛔ chờ Lock của B trên id=2             WHERE id = 1 FOR UPDATE;
                                        ⛔ chờ Lock của A trên id=1

        A đang chờ B  ────────────────────►  B đang chờ A
                    (VÒNG LẶP CHỜ NHAU VÔ TẬN)
```

Đây là 2 Transaction cùng chuyển tiền theo thứ tự **ngược nhau** (A: chuyển từ tài khoản 1 sang 2; B: chuyển từ tài khoản 2 sang 1) — kinh điển đến mức hầu hết Database Engine (MySQL InnoDB, PostgreSQL) đều có sẵn cơ chế **Deadlock Detector** chạy ngầm: phát hiện chu trình chờ nhau này, và **chủ động** chọn 1 Transaction làm "vật hi sinh" (thường là Transaction rẻ hơn để rollback), ném lỗi `Deadlock found when trying to get lock` để giải thoát Transaction còn lại.

> **Bài học thực chiến**: Để giảm thiểu Deadlock kiểu này, nguyên tắc kinh điển là **luôn khóa tài nguyên theo cùng 1 thứ tự cố định** trong toàn bộ code nghiệp vụ (ví dụ luôn khóa theo `id` tăng dần) — nếu cả A và B đều khóa `id=1` trước rồi mới tới `id=2`, chu trình chờ chéo nhau ở trên sẽ không bao giờ xảy ra.

---

## V. Giải Pháp: Timeouts - Đừng Bao Giờ Khóa Mù Quáng

Từ 2 "tử huyệt" trên, nguyên tắc bất di bất dịch khi dùng Pessimistic Locking: **không bao giờ để 1 Transaction chờ Lock vô thời hạn**. Luôn thiết lập thời gian chờ tối đa (Timeout) thông qua Query Hint chuẩn của JPA — `javax.persistence.lock.timeout` (đơn vị mili-giây):

```java
Map<String, Object> hints = new HashMap<>();
hints.put("javax.persistence.lock.timeout", 3000); // chờ tối đa 3 giây

Product product = entityManager.find(
    Product.class,
    1L,
    LockModeType.PESSIMISTIC_WRITE,
    hints
);
```

Với Spring Data JPA, dùng kèm `@QueryHints`:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({@QueryHint(name = "javax.persistence.lock.timeout", value = "3000")})
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

Nếu không lấy được Lock trong khoảng thời gian quy định, Hibernate sẽ ném `PessimisticLockException` — Connection được **giải phóng ngay lập tức** thay vì treo lơ lửng chờ mãi, giữ cho Connection Pool luôn "thở" được dù hệ thống đang có tranh chấp cao.

---

## VI. Vũ Khí Nâng Cao: `NOWAIT` & `SKIP LOCKED`

Với PostgreSQL hoặc các bản MySQL mới (8.0+), Hibernate có thể tận dụng thêm 2 biến thể mạnh mẽ của `FOR UPDATE`, cho phép kiểm soát hành vi khi gặp Lock một cách tinh vi hơn hẳn so với chỉ đơn thuần chờ-rồi-timeout.

### 1. `FOR UPDATE NOWAIT`: Thà báo lỗi còn hơn chờ đợi

```sql
SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;
```

Nếu dòng `id=1` đang bị Transaction khác khóa, `NOWAIT` khiến DB **không chờ đợi một chút nào cả** — nó ném lỗi (`PessimisticLockException` ở tầng Hibernate) **ngay lập tức**, giải phóng Connection tức thì. Về bản chất, đây là phiên bản "cực đoan" của Timeout ở Mục V (tương đương `timeout = 0`) — hữu ích khi nghiệp vụ của bạn thà báo lỗi ngay cho User "thử lại sau" còn hơn để họ đứng chờ mù mờ không biết bao lâu.

### 2. `FOR UPDATE SKIP LOCKED`: Vũ khí bí mật cho hệ thống Queue

```sql
SELECT * FROM jobs
WHERE status = 'PENDING'
ORDER BY id
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

Đây là "chiêu" cực kỳ hữu dụng khi xây dựng hệ thống **Hàng đợi (Message Queue) bằng chính 1 bảng SQL thông thường** — một pattern rất phổ biến khi chưa cần tới hạ tầng Kafka/RabbitMQ đầy đủ. Thay vì chờ đợi (như `FOR UPDATE` thường) hoặc báo lỗi ngay (như `NOWAIT`), `SKIP LOCKED` chọn cách **lịch sự nhất**: nếu gặp dòng đang bị khóa, nó **bỏ qua luôn dòng đó**, tự động lấy tiếp dòng kế tiếp đang "rảnh rỗi" để xử lý.

```
Bảng jobs (status = PENDING): id 1, 2, 3, 4, 5, 6

Worker 1: SELECT ... WHERE status='PENDING' LIMIT 3 FOR UPDATE SKIP LOCKED;
          → Khóa và lấy được: id 1, 2, 3

Worker 2 (chạy GẦN NHƯ ĐỒNG THỜI):
          SELECT ... WHERE status='PENDING' LIMIT 3 FOR UPDATE SKIP LOCKED;
          → Thấy id 1, 2, 3 đang bị khóa → TỰ ĐỘNG BỎ QUA
          → Khóa và lấy được: id 4, 5, 6

Kết quả: Worker 1 và Worker 2 xử lý 2 tập Job HOÀN TOÀN KHÁC NHAU,
         chạy song song thật sự, không ai giẫm chân lên ai,
         và KHÔNG AI PHẢI CHỜ ĐỢI AI CẢ.
```

Ví dụ với Spring Data JPA (native query, vì cú pháp `SKIP LOCKED` chưa có sẵn trong JPQL chuẩn):

```java
public interface JobRepository extends JpaRepository<Job, Long> {

    @Query(value = """
        SELECT * FROM jobs
        WHERE status = 'PENDING'
        ORDER BY id
        LIMIT :batchSize
        FOR UPDATE SKIP LOCKED
        """, nativeQuery = true)
    List<Job> pollPendingJobs(@Param("batchSize") int batchSize);
}
```

> **So sánh nhanh 3 hành vi khi gặp Lock**: `FOR UPDATE` thường → **chờ** (có thể kèm Timeout); `NOWAIT` → **báo lỗi ngay**, không chờ 1 giây nào; `SKIP LOCKED` → **bỏ qua, lấy dòng khác**, không chờ và cũng không báo lỗi. Đây chính là bí quyết giúp nhiều Worker instance chạy song song xử lý cùng 1 bảng Job Queue mà không cần thêm bất kỳ cơ chế phân tán (Distributed Lock) phức tạp nào khác.

---

## VII. Bảng So Sánh: 4 Biến Thể Của Khóa Bi Quan

| Biến thể | SQL sinh ra | Hành vi khi gặp Lock | Phù hợp khi nào |
| :--- | :--- | :--- | :--- |
| **`PESSIMISTIC_WRITE`** | `FOR UPDATE` | Chờ (hoặc theo Timeout cấu hình) | Chắc chắn sẽ ghi ngay sau khi đọc (trừ tiền, giảm tồn kho) |
| **`PESSIMISTIC_READ`** | `FOR SHARE` / `LOCK IN SHARE MODE` | Cho phép đọc song song, chỉ chặn ghi | Cần dữ liệu "đứng yên" để tính toán, nhưng vẫn cho người khác đọc |
| **`FOR UPDATE NOWAIT`** | `FOR UPDATE NOWAIT` | Báo lỗi **ngay lập tức**, không chờ | Nghiệp vụ cần phản hồi nhanh, chấp nhận báo lỗi "thử lại sau" |
| **`FOR UPDATE SKIP LOCKED`** | `FOR UPDATE SKIP LOCKED` | **Bỏ qua** dòng đang khóa, lấy dòng khác | Worker Pool xử lý Job Queue song song, không ai phải chờ ai |

---

## VIII. Trade-off: Optimistic vs Pessimistic - Chọn Sao Cho Đúng?

Nhắc lại bảng so sánh tổng thể đã có ở [Chương 1 - Optimistic Locking, Mục VIII](1_OPTIMISTIC_LOCKING.md#viii-bảng-so-sánh-optimistic-locking-vs-pessimistic-locking) — nguyên tắc cốt lõi vẫn là:

* **Contention thấp** (ít khi 2 người cùng đụng vào 1 dòng) → ưu tiên **Optimistic Locking**, tránh phí tổn giữ Lock/Connection không cần thiết.
* **Contention cực cao trên đúng 1 dòng** (ví dụ: kho chỉ còn 1 vé cuối cùng, hàng nghìn người bấm mua cùng lúc) → **Pessimistic Locking** với Timeout hợp lý thường ổn định hơn, vì để Optimistic Locking "đấu" tự do sẽ sinh ra hàng loạt `rollback`/`retry` gây lãng phí CPU tương đương (hoặc tệ hơn) so với việc xếp hàng chờ Lock đàng hoàng.
* **Xử lý hàng đợi/công việc phân tán bằng bảng SQL** (Job Queue, Task Scheduler nhiều Worker) → `FOR UPDATE SKIP LOCKED` gần như là lựa chọn mặc định, vì nó cho phép song song hóa tối đa mà không cần Lock phân tán riêng (Redis Lock, ZooKeeper...).
* Luôn nhớ đi kèm **Timeout** khi dùng Pessimistic Locking — không có Timeout, 1 Transaction "đứng hình" của 1 User có thể kéo sập cả Connection Pool của toàn hệ thống.

---

## IX. Cheat Sheet Phỏng vấn (Interview Q&A)

### Q1: Bản chất của Pessimistic Locking là gì? Nó khác Optimistic Locking ở điểm cốt lõi nào?
> **Trả lời**: Pessimistic Locking giả định xung đột dữ liệu chắc chắn sẽ xảy ra, nên nó đẩy trách nhiệm khóa xuống tận Database Engine — Transaction chạm vào dòng nào sẽ khóa cứng dòng đó lại (`SELECT ... FOR UPDATE`), cấm người khác đụng vào cho tới khi commit/rollback. Khác biệt cốt lõi với Optimistic Locking: Pessimistic phát hiện xung đột **trước** (chặn ngay từ lúc đọc, Transaction khác bị Block thật sự dưới DB), còn Optimistic chỉ phát hiện xung đột **sau** khi đã cố gắng ghi (dựa vào `update count`).

### Q2: Phân biệt `PESSIMISTIC_WRITE` và `PESSIMISTIC_READ`.
> **Trả lời**: `PESSIMISTIC_WRITE` dịch thành `FOR UPDATE` — khóa độc quyền, không cho bất kỳ ai khác đọc-để-sửa hay ghi lên dòng đó. `PESSIMISTIC_READ` dịch thành `FOR SHARE`/`LOCK IN SHARE MODE` — cho phép nhiều Transaction cùng đọc song song (Shared Lock), nhưng chặn mọi thao tác ghi cho tới khi tất cả Shared Lock được giải phóng. `PESSIMISTIC_WRITE` dùng khi chắc chắn sẽ ghi ngay sau đó; `PESSIMISTIC_READ` dùng khi chỉ cần đảm bảo dữ liệu "đứng yên" để tính toán mà vẫn cho phép người khác đọc cùng lúc.

### Q3: Vì sao Pessimistic Locking dễ gây cạn kiệt Connection Pool?
> **Trả lời**: Vì khi 1 Transaction bị Block chờ Lock từ Transaction khác, Connection JDBC mà nó đang giữ **không được trả về Pool** — nó đứng treo trong trạng thái chờ. Nếu nhiều Request cùng tranh chấp 1 vài dòng "hot" (ví dụ cùng mua 1 sản phẩm sale sốc), phần lớn Connection trong Pool sẽ bị chiếm dụng bởi các Thread đang chờ đợi thay vì thực sự xử lý, khiến Pool nhanh chóng cạn kiệt và ném `SQLTransientConnectionException` cho các Request tới sau.

### Q4: Deadlock ở tầng Database xảy ra như thế nào? Cách phòng tránh kinh điển là gì?
> **Trả lời**: Xảy ra khi 2 Transaction khóa chéo tài nguyên của nhau — ví dụ Transaction A khóa row 1 rồi cố khóa row 2 (đang bị B giữ), trong khi Transaction B khóa row 2 rồi cố khóa row 1 (đang bị A giữ), tạo thành vòng chờ vô tận. Hầu hết DB Engine có Deadlock Detector tự động phát hiện và chủ động rollback 1 trong 2 Transaction để giải thoát. Cách phòng tránh kinh điển: luôn khóa tài nguyên theo **cùng 1 thứ tự cố định** (ví dụ luôn khóa theo `id` tăng dần) trong toàn bộ code nghiệp vụ, để 2 Transaction không bao giờ chờ chéo nhau.

### Q5: Vì sao không bao giờ nên dùng Pessimistic Locking mà không cấu hình Timeout?
> **Trả lời**: Vì nếu không có Timeout, 1 Transaction bị Block sẽ chờ **vô thời hạn** cho tới khi Lock được nhả — nếu Transaction đang giữ Lock đó gặp sự cố (treo, chạy chậm bất thường, hoặc rơi vào Deadlock không được phát hiện), toàn bộ chuỗi Transaction chờ theo sau sẽ bị "đóng băng" cùng với Connection của chúng, có thể kéo sập cả Connection Pool. Cấu hình `javax.persistence.lock.timeout` đảm bảo Transaction chờ quá lâu sẽ bị hủy và giải phóng Connection ngay, thay vì treo lơ lửng vô hạn.

### Q6: Phân biệt hành vi của `FOR UPDATE` thường, `FOR UPDATE NOWAIT`, và `FOR UPDATE SKIP LOCKED` khi gặp dòng đang bị khóa.
> **Trả lời**: `FOR UPDATE` thường sẽ **chờ** cho tới khi Lock được nhả (hoặc tới khi hết Timeout nếu có cấu hình). `NOWAIT` **không chờ một chút nào** — ném lỗi ngay lập tức nếu phát hiện dòng đang bị khóa. `SKIP LOCKED` thì **không chờ và cũng không báo lỗi** — nó lặng lẽ bỏ qua dòng đang bị khóa, tự động chuyển sang lấy dòng tiếp theo còn "rảnh rỗi".

### Q7: Tại sao `FOR UPDATE SKIP LOCKED` lại phù hợp để xây dựng hệ thống Message Queue bằng bảng SQL thông thường?
> **Trả lời**: Vì nó cho phép nhiều Worker instance cùng chạy `SELECT ... FOR UPDATE SKIP LOCKED` để lấy Job đang `PENDING` mà không ai giẫm chân lên ai: mỗi Worker chỉ khóa và nhận đúng những dòng đang "rảnh" (chưa bị Worker khác khóa), Worker khác tự động bỏ qua các dòng đã bị khóa để lấy tiếp dòng kế tiếp. Kết quả là nhiều Worker chạy song song thật sự, không ai phải chờ đợi ai, không cần tới cơ chế Distributed Lock phức tạp bên ngoài (Redis, ZooKeeper) chỉ để điều phối việc lấy Job.

### Q8: Khi nào nên ưu tiên Pessimistic Locking thay vì Optimistic Locking?
> **Trả lời**: Khi mức độ tranh chấp (Contention) trên cùng 1 dòng dữ liệu cực cao và hệ thống không chấp nhận việc liên tục `rollback`/`retry` (vốn là hệ quả tất yếu của Optimistic Locking khi Contention cao) — ví dụ hàng nghìn người cùng tranh mua 1 vé cuối cùng, hoặc trừ tiền dồn dập trên cùng 1 tài khoản ví điện tử. Trong các tình huống này, xếp hàng chờ Lock có kiểm soát (kèm Timeout hợp lý) ổn định và dễ dự đoán hơn so với vòng lặp retry vô tận của Optimistic Locking.

---

> **Tài liệu tham khảo:**
> - [Pessimistic Locking in JPA - Baeldung](https://www.baeldung.com/jpa-pessimistic-locking)
> - [Enabling Transaction Locks in Spring Data JPA - Baeldung](https://www.baeldung.com/java-jpa-transaction-locks)
> - [Pessimistic locking in JPA and Hibernate - Arnold Galovics](https://arnoldgalovics.com/jpa-pessimistic-locking/)
> - [How to implement a database job queue using SKIP LOCKED - Vlad Mihalcea](https://vladmihalcea.com/database-job-queue-skip-locked/)
> - [Pessimistic vs Optimistic Locking: Khi nào nên "khóa chặt", khi nào nên "thả lỏng"? - Viblo](https://viblo.asia/p/pessimistic-vs-optimistic-locking-khi-nao-nen-khoa-chat-khi-nao-nen-tha-long-ym4008OW491)
