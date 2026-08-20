# Chương 1: Optimistic Locking - Nghệ Thuật "Không Khóa"

![Status](https://img.shields.io/badge/Optimistic_Locking-green) ![Topic](https://img.shields.io/badge/Version_&_Concurrency-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
  - [1. Bài toán Lost Update: Khi 2 người cùng sửa 1 dòng dữ liệu](#1-bài-toán-lost-update-khi-2-người-cùng-sửa-1-dòng-dữ-liệu)
  - [2. Hai triết lý đối lập: Bi quan vs Lạc quan](#2-hai-triết-lý-đối-lập-bi-quan-vs-lạc-quan)
- [II. Bản chất "Lạc Quan": Không hề có khóa vật lý nào cả](#ii-bản-chất-lạc-quan-không-hề-có-khóa-vật-lý-nào-cả)
- [III. Cơ chế `@Version` dưới lớp vỏ Hibernate](#iii-cơ-chế-version-dưới-lớp-vỏ-hibernate)
  - [1. Khai báo `@Version` trên Entity](#1-khai-báo-version-trên-entity)
  - [2. Câu `UPDATE` "có điều kiện" mà Hibernate âm thầm sinh ra](#2-câu-update-có-điều-kiện-mà-hibernate-âm-thầm-sinh-ra)
- [IV. Nút Thắt Quan Trọng: Hibernate phát hiện xung đột bằng cách nào?](#iv-nút-thắt-quan-trọng-hibernate-phát-hiện-xung-đột-bằng-cách-nào)
  - [1. Vũ khí bí mật: `update count` của `executeUpdate()`](#1-vũ-khí-bí-mật-update-count-của-executeupdate)
  - [2. Kịch bản thành công vs Kịch bản xung đột](#2-kịch-bản-thành-công-vs-kịch-bản-xung-đột)
- [V. Bài toán "Long UI Conversation" - Sức mạnh thực sự của Optimistic Locking](#v-bài-toán-long-ui-conversation---sức-mạnh-thực-sự-của-optimistic-locking)
- [VI. Các kiểu `@Version` \& `LockModeType`](#vi-các-kiểu-version--lockmodetype)
  - [1. Kiểu dữ liệu cho cột Version](#1-kiểu-dữ-liệu-cho-cột-version)
  - [2. `LockModeType.OPTIMISTIC` vs `OPTIMISTIC_FORCE_INCREMENT`](#2-lockmodetypeoptimistic-vs-optimistic_force_increment)
- [VII. Cạm bẫy thực chiến (Pitfalls)](#vii-cạm-bẫy-thực-chiến-pitfalls)
  - [1. Bulk Update bằng JPQL "lách qua mặt" Version](#1-bulk-update-bằng-jpql-lách-qua-mặt-version)
  - [2. Retry Pattern: Xử lý `OptimisticLockException` đúng cách](#2-retry-pattern-xử-lý-optimisticlockexception-đúng-cách)
  - [3. Đừng bao giờ đưa `version` vào `equals()`/`hashCode()`](#3-đừng-bao-giờ-đưa-version-vào-equalshashcode)
- [VIII. Bảng So Sánh: Optimistic Locking vs Pessimistic Locking](#viii-bảng-so-sánh-optimistic-locking-vs-pessimistic-locking)
- [IX. Trade-off: Khi nào chọn cái nào?](#ix-trade-off-khi-nào-chọn-cái-nào)
- [X. Cheat Sheet Phỏng vấn (Interview Q\&A)](#x-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

### 1. Bài toán Lost Update: Khi 2 người cùng sửa 1 dòng dữ liệu

Hãy tưởng tượng một tình huống cực kỳ đời thường: bạn và đồng nghiệp cùng mở chung 1 trang admin để sửa thông tin sản phẩm `id=1` (đang có `stock = 100`).

```
Thời điểm     User A (Trình duyệt)                User B (Trình duyệt)
──────────    ──────────────────────────          ──────────────────────────
T1            Mở trang sửa, thấy stock=100
T2                                                 Mở trang sửa, thấy stock=100
T3            Sửa thành stock=90, bấm Save
              → DB: stock=90
T4                                                 Sửa thành stock=80, bấm Save
                                                    → DB: stock=80
```

Kết quả cuối cùng dưới DB là `stock = 80`. Thay đổi của User A (`90`) **biến mất không dấu vết**, dù A đã bấm Save thành công và không hề nhận được bất kỳ lỗi nào! Đây chính là hiện tượng kinh điển gọi là **Lost Update** — một trong những "kẻ thù" nguy hiểm nhất của các hệ thống có nhiều người dùng thao tác đồng thời trên cùng 1 dòng dữ liệu.

> **Lưu ý quan trọng**: Ở [Isolation Level & MVCC](../consistency%20%26%20transaction%20management/2_ISOLATION_LEVEL_AND_MVCC.md), chúng ta đã học rằng ngay cả `Repeatable Read` hay `Serializable` ở tầng DB cũng **không tự động cứu được** bài toán này, vì đây là 2 Transaction **hoàn toàn tách biệt** (A commit xong rồi B mới bắt đầu sửa) — DB không hề biết B đang "ghi đè" lên một giá trị mà A vừa đọc trước đó ở một Transaction khác. Bài toán Lost Update nằm ở tầng **Application logic**, không phải tầng Isolation Level thuần túy.

### 2. Hai triết lý đối lập: Bi quan vs Lạc quan

Để giải bài toán Lost Update, ngành phần mềm sinh ra 2 trường phái tư duy hoàn toàn trái ngược nhau:

* 🔒 **Pessimistic Locking (Khóa bi quan)**: "Tôi *chắc chắn* sẽ có người vào tranh giành dòng dữ liệu này, nên tôi khóa nó lại **ngay từ lúc đọc**, không cho ai đụng vào cho tới khi tôi xong việc." → Dùng `SELECT ... FOR UPDATE`.
* 🍀 **Optimistic Locking (Khóa lạc quan)**: "Xung đột thực ra **rất hiếm khi xảy ra**, tội gì phải khóa cho tốn tài nguyên. Cứ để mọi người tự do đọc/sửa, chỉ cần **kiểm tra lại đúng khoảnh khắc lưu xuống DB** xem có ai vừa sửa trước mình không."

Chương này đào sâu vào triết lý thứ hai — **Optimistic Locking** — cơ chế mặc định và phổ biến nhất trong các ứng dụng JPA/Hibernate hiện đại.

---

## II. Bản chất "Lạc Quan": Không hề có khóa vật lý nào cả

Điều khiến Optimistic Locking khác biệt hoàn toàn so với Pessimistic Locking nằm ngay ở cái tên: nó **"lạc quan"** vì hệ thống giả định xung đột dữ liệu là **hiếm gặp**. Do đó:

* **Không** có bất kỳ câu lệnh `LOCK`, `FOR UPDATE` nào được gửi xuống Database.
* **Không** có Thread/Transaction nào bị `BLOCK` chờ đợi Thread khác.
* Database hoàn toàn "vô tư" — nó không biết và không quan tâm khái niệm "Optimistic Locking" là gì cả!

Toàn bộ "phép thuật" nằm ở **tầng Application** — cụ thể là sự phối hợp giữa **Hibernate** (sinh SQL thông minh) và **JDBC Driver** (trả về kết quả thực thi). Nói cách khác: Optimistic Locking không phải một tính năng của Database, mà là một **kỹ thuật lập trình** dùng chính cơ chế `WHERE` sẵn có của SQL để tự dựng nên một "hàng rào" kiểm tra tại tầng ứng dụng.

```
Pessimistic Locking                         Optimistic Locking
──────────────────────                      ──────────────────────
SELECT ... FOR UPDATE                       SELECT ... (đọc bình thường)
        │                                           │
        ▼                                           ▼
DB khóa row lại (Row Lock)                  Không có Lock nào cả
        │                                           │
Transaction khác PHẢI CHỜ                   Transaction khác vẫn đọc/sửa vô tư
        │                                           │
        ▼                                           ▼
UPDATE bình thường                          UPDATE kèm điều kiện WHERE version = ?
                                                     │
                                             Kiểm tra SAU khi đã UPDATE xong
```

---

## III. Cơ chế `@Version` dưới lớp vỏ Hibernate

### 1. Khai báo `@Version` trên Entity

Trong JPA, bạn chỉ cần thêm đúng 1 field, đánh dấu bằng annotation `@Version`:

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    private String name;
    private Integer stock;

    @Version
    private Integer version;   // Hibernate tự động quản lý field này!
}
```

Từ giây phút này, Hibernate **tự động** làm 3 việc mà bạn không cần viết thêm 1 dòng code nào:
1. Mỗi khi `INSERT`, gán `version = 0` (hoặc `1` tùy cấu hình).
2. Mỗi khi `UPDATE`, tự động tăng `version` thêm `1` và gắn thêm điều kiện kiểm tra vào mệnh đề `WHERE`.
3. Nếu điều kiện đó không khớp → ném `OptimisticLockException`.

### 2. Câu `UPDATE` "có điều kiện" mà Hibernate âm thầm sinh ra

Giả sử User A tải lên Entity `id=1` với `version=1` và sửa `name` thành `"New Name"`. Nếu là 1 câu `UPDATE` bình thường (không có Optimistic Locking), Hibernate sẽ sinh ra:

```sql
UPDATE products SET name = 'New Name' WHERE id = 1;
```

Nhưng vì có `@Version`, câu lệnh thực tế Hibernate gửi xuống DB sẽ là:

```sql
UPDATE products
SET name = 'New Name', version = 2
WHERE id = 1 AND version = 1;
```

Chú ý 2 điểm mấu chốt:
* **Vế `SET`**: `version` được tăng lên `2` — Hibernate tự tính `version + 1`, không phải bạn tự tay set.
* **Vế `WHERE`**: có thêm điều kiện `AND version = 1` — đây chính là "chốt chặn" để kiểm tra xem dữ liệu có còn "tươi mới" đúng như lúc bạn đọc lên hay không.

---

## IV. Nút Thắt Quan Trọng: Hibernate phát hiện xung đột bằng cách nào?

Đây là câu hỏi hay bị hỏi nhất khi phỏng vấn về chủ đề này: **Java code làm sao "biết" được có người khác đã sửa dữ liệu trước mình, trong khi DB không hề ném ra bất kỳ Exception nào cả?**

### 1. Vũ khí bí mật: `update count` của `executeUpdate()`

Bí mật nằm ở đối tượng `PreparedStatement` trong tầng JDBC thuần. Hàm `executeUpdate()` của nó **luôn** trả về 1 số nguyên (`int`), gọi là **update count** — đại diện cho **số dòng thực sự bị tác động** dưới Database sau khi câu lệnh chạy xong.

```java
// Mô phỏng những gì Hibernate làm bên dưới lớp vỏ
PreparedStatement stmt = connection.prepareStatement(
    "UPDATE products SET name = ?, version = ? WHERE id = ? AND version = ?"
);
stmt.setString(1, "New Name");
stmt.setInt(2, 2);      // version mới
stmt.setLong(3, 1L);    // id
stmt.setInt(4, 1);      // version cũ (điều kiện WHERE)

int updateCount = stmt.executeUpdate();

if (updateCount == 0) {
    // Không có dòng nào khớp điều kiện → dữ liệu đã bị người khác sửa!
    throw new OptimisticLockException("Row was updated or deleted by another transaction");
}
```

Đây chính là toàn bộ "phép thuật" — không có gì thần bí cả, chỉ là tận dụng đúng 1 con số mà JDBC vốn đã trả về sẵn nhưng ít người để ý tới.

### 2. Kịch bản thành công vs Kịch bản xung đột

```
Kịch bản 1: KHÔNG có xung đột (User B chưa đụng vào)
────────────────────────────────────────────────────
User A: UPDATE products SET ..., version=2 WHERE id=1 AND version=1
DB: tìm thấy đúng 1 dòng thỏa điều kiện (id=1 AND version=1) → UPDATE thành công
DB trả về: updateCount = 1
Hibernate: "OK, mọi thứ ổn!" → Transaction tiếp tục, commit bình thường.


Kịch bản 2: CÓ xung đột (User B đã lưu trước, version dưới DB đã là 2)
────────────────────────────────────────────────────
User A: UPDATE products SET ..., version=2 WHERE id=1 AND version=1
DB: quét toàn bảng, KHÔNG tìm thấy dòng nào có (id=1 AND version=1)
    vì version thực tế dưới DB giờ đã là 2 (do User B vừa cập nhật)
DB trả về: updateCount = 0     ← DB KHÔNG báo lỗi gì cả! Chỉ âm thầm trả về 0.
Hibernate: nhận thấy updateCount == 0
    → LẬP TỨC ném ra OptimisticLockException
    (Hibernate: StaleObjectStateException / StaleStateException)
    → Transaction hiện tại bị ROLLBACK ngay lập tức.
```

> **Điểm cực kỳ tinh tế**: Database **không hề có khái niệm lỗi** trong tình huống này. Với DB, câu `UPDATE ... WHERE id=1 AND version=1` đơn giản là "không tìm thấy dòng nào khớp" — một kết quả hoàn toàn hợp lệ về mặt cú pháp SQL. Toàn bộ việc "diễn giải con số 0 đó thành 1 xung đột dữ liệu nghiêm trọng" là **quyết định thuần túy ở tầng Application (Hibernate)**. Đây là lý do vì sao Optimistic Locking được gọi là kỹ thuật ở tầng ứng dụng, không phải tính năng của Database.

---

## V. Bài toán "Long UI Conversation" - Sức mạnh thực sự của Optimistic Locking

Đây chính là lý do Optimistic Locking tồn tại và được ưa chuộng hơn hẳn Pessimistic Locking trong phần lớn ứng dụng Web hiện đại.

Hãy tưởng tượng: User A mở màn hình "Sửa hồ sơ cá nhân" trên trình duyệt. Entity được load lên (giả sử `version=1`), Transaction ở Backend đã **kết thúc từ lâu** (Entity đã chuyển sang trạng thái **Detached** — xem lại [Persistence Context & L1 Cache](../hibernate%20%26%20jpa%20internals/1_PERSISTENCE_CONTEXT_AND_L1_CACHE.md)). Sau đó, User A... đứng dậy đi pha cà phê **30 phút**.

```
Timeline
────────
T1 (User A)   Mở form sửa hồ sơ → GET /profile/1 → trả về DTO (version=1)
              Backend Transaction đã ĐÓNG ngay sau khi trả response.

T2 (User B)   Vào sửa cùng hồ sơ đó, bấm Save thành công
              → DB: version giờ là 2

... 30 phút trôi qua, User A vẫn đang... uống cà phê ...

T3 (User A)   Quay lại, bấm nút "Save" trên form đang mở
              → POST /profile/1  (payload JSON vẫn mang version=1, dữ liệu cũ từ T1)
```

Điều thú vị là: **không hề có 1 Transaction nào "sống" xuyên suốt 30 phút đó cả** — hoàn toàn khác với Pessimistic Locking (nếu dùng `SELECT FOR UPDATE`, Connection sẽ phải giữ Lock treo lơ lửng suốt 30 phút, gây cạn kiệt Connection Pool ngay lập tức, xem lại [Connection Pool](../connectivity%20internals/2_CONNECTION_POOL.md)).

Khi request `Save` ở T3 đi tới Backend:
1. Backend mở 1 Transaction **hoàn toàn mới**.
2. Lấy dữ liệu DTO (mang `version=1`) từ Request, "đắp" vào lại Entity (thường bằng `merge()` hoặc gán trực tiếp field).
3. Hibernate sinh câu `UPDATE ... WHERE id = 1 AND version = 1`.
4. DB không tìm thấy dòng nào khớp (vì thực tế dưới DB `version` đã là `2` từ T2) → `updateCount = 0`.
5. Hibernate ném `OptimisticLockException` → giao dịch bị từ chối, dữ liệu của User B được bảo toàn nguyên vẹn.

> **Đây chính là điểm mạnh cốt lõi**: Optimistic Locking bảo vệ được cả những **"cuộc trò chuyện dài hơi với UI"** (user nghĩ ngợi, đi pha cà phê, để tab mở cả tiếng đồng hồ) — thứ mà Pessimistic Locking **không thể** làm được, vì không Database nào chịu nổi việc giữ Row Lock hàng chục phút chỉ vì người dùng... lười bấm Save.

---

## VI. Các kiểu `@Version` & `LockModeType`

### 1. Kiểu dữ liệu cho cột Version

JPA cho phép 3 kiểu dữ liệu phổ biến cho field `@Version`:

| Kiểu dữ liệu | Cách hoạt động | Ưu / nhược điểm |
| :--- | :--- | :--- |
| **`int` / `Integer`** | Tăng dần `1, 2, 3, ...` mỗi lần UPDATE | Phổ biến nhất, tốn ít byte, dễ đọc log |
| **`long` / `Long`** | Giống Integer nhưng không lo tràn số với bảng cực lớn, cập nhật cực nhiều | Dùng cho bảng có tần suất UPDATE khổng lồ |
| **`Timestamp`** | Version = thời điểm sửa gần nhất, tăng theo giờ hệ thống thay vì đếm số | Tiện lợi vì "tự nhiên" đọc hiểu (biết ngay lần sửa cuối lúc nào), nhưng rủi ro nếu 2 Server lệch đồng hồ (Clock Skew) |

Trong thực tế, **`Integer`/`Long`** vẫn là lựa chọn an toàn và phổ biến nhất vì không phụ thuộc vào đồng hồ hệ thống của nhiều server khác nhau.

### 2. `LockModeType.OPTIMISTIC` vs `OPTIMISTIC_FORCE_INCREMENT`

Đôi khi bạn muốn **chủ động** ép version tăng lên, dù bản thân Entity đó không có field nào bị sửa trực tiếp — ví dụ khi sửa 1 dòng `OrderItem` (con), bạn muốn `Order` (cha) cũng được đánh dấu "vừa có thay đổi" dù không field nào của `Order` bị đổi cả:

```java
// Chỉ kiểm tra version chưa bị đổi, KHÔNG ép tăng version nếu không có gì thay đổi
Order order = em.find(Order.class, orderId, LockModeType.OPTIMISTIC);

// Ép tăng version NGAY CẢ KHI Order không có field nào bị sửa trực tiếp
Order order = em.find(Order.class, orderId, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
```

| `LockModeType` | Hành vi |
| :--- | :--- |
| **`OPTIMISTIC`** | Chỉ kiểm tra `version` chưa đổi tại thời điểm `flush`/`commit`, không ép tăng nếu Entity không dirty |
| **`OPTIMISTIC_FORCE_INCREMENT`** | Luôn tăng `version` khi Transaction kết thúc, kể cả khi Entity gốc không có field nào bị sửa (hữu ích khi muốn lan truyền "dấu hiệu thay đổi" từ con lên cha) |

---

## VII. Cạm bẫy thực chiến (Pitfalls)

### 1. Bulk Update bằng JPQL "lách qua mặt" Version

Đây là bẫy cực kỳ phổ biến: khi bạn dùng câu `UPDATE` hàng loạt bằng JPQL (`@Modifying @Query`), Hibernate **KHÔNG** tự động tăng `version` như khi bạn sửa qua Entity + Dirty Checking:

```java
// ❌ NGUY HIỂM: version KHÔNG được tự động tăng!
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - 1 WHERE p.id = :id")
void decreaseStock(@Param("id") Long id);
```

Vì câu lệnh này chạy thẳng xuống DB, bỏ qua hoàn toàn vòng đời `Dirty Checking`/`flush()` của Persistence Context (xem lại [Persistence Context & L1 Cache, Mục V](../hibernate%20%26%20jpa%20internals/1_PERSISTENCE_CONTEXT_AND_L1_CACHE.md)) — nên cơ chế `@Version` tự động của Hibernate **không có cơ hội can thiệp**. Nếu 1 luồng khác đang giữ Entity này trong RAM với `version` cũ, nó sẽ không hề hay biết dữ liệu dưới DB đã đổi.

**Giải pháp**: tự tay tăng version trong chính câu JPQL nếu bắt buộc phải Bulk Update:

```java
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - 1, p.version = p.version + 1 WHERE p.id = :id")
void decreaseStock(@Param("id") Long id);
```

### 2. Retry Pattern: Xử lý `OptimisticLockException` đúng cách

Vì bản chất Optimistic Locking là "cứ thử trước, sai thì báo lỗi", tầng Service **bắt buộc** phải có chiến lược xử lý khi Exception xảy ra — không đơn giản như Pessimistic Locking (chỉ cần chờ là xong). Cách phổ biến nhất: **Retry** — tải lại dữ liệu mới nhất và thử áp lại thay đổi:

```java
@Service
public class StockService {

    private static final int MAX_RETRY = 3;

    public void decreaseStock(Long productId) {
        int attempt = 0;
        while (true) {
            try {
                doDecreaseStock(productId);
                return; // thành công, thoát vòng lặp
            } catch (OptimisticLockException e) {
                attempt++;
                if (attempt >= MAX_RETRY) {
                    throw new RuntimeException("Cập nhật thất bại sau nhiều lần thử, vui lòng thử lại", e);
                }
                // Vòng lặp tiếp theo sẽ load lại Entity MỚI NHẤT rồi thử lại
            }
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void doDecreaseStock(Long productId) {
        Product product = productRepository.findById(productId).orElseThrow();
        product.setStock(product.getStock() - 1);
        // Hibernate tự sinh UPDATE ... WHERE id=? AND version=? khi flush/commit
    }
}
```

> **Lưu ý**: `doDecreaseStock()` cần `REQUIRES_NEW` (xem lại [Propagation](../consistency%20%26%20transaction%20management/3_PROPAGATION_AND_CONNECTION_MANAGEMENT.md)) để mỗi lần retry là 1 Physical Transaction hoàn toàn mới — tải lại Entity với `version` mới nhất từ DB, chứ không tiếp tục dùng Entity cũ (đã "hỏng") từ Persistence Context trước đó. Ở tầng Spring, bạn cũng có thể dùng `@Retryable` (thư viện Spring Retry) để khai báo retry gọn hơn thay vì viết `while` thủ công.

### 3. Đừng bao giờ đưa `version` vào `equals()`/`hashCode()`

Một lỗi tinh vi khi Entity tự viết `equals()`/`hashCode()` dựa trên **toàn bộ field** (kể cả `version`):

```java
// ❌ SAI: version thay đổi liên tục sau mỗi lần UPDATE
@Override
public boolean equals(Object o) {
    Product other = (Product) o;
    return Objects.equals(this.id, other.id)
        && Objects.equals(this.version, other.version); // ❌ Đừng làm vậy!
}
```

Vì `version` đổi liên tục theo từng lần `UPDATE`, việc đưa nó vào `equals()` khiến 2 object đại diện cho **cùng 1 dòng dữ liệu logic** (cùng `id`) nhưng ở 2 thời điểm khác nhau lại bị coi là "khác nhau" — phá vỡ hợp đồng ổn định mà các cấu trúc như `HashSet`/`HashMap` yêu cầu. Quy tắc chuẩn: chỉ nên dùng **Business Key** (hoặc `id` sau khi Entity đã Managed) cho `equals()`/`hashCode()`, tuyệt đối không đụng tới `version`.

---

## VIII. Bảng So Sánh: Optimistic Locking vs Pessimistic Locking

| Tiêu chí | Optimistic Locking | Pessimistic Locking |
| :--- | :--- | :--- |
| **Cơ chế** | Không khóa gì cả, kiểm tra `version` tại thời điểm ghi | `SELECT ... FOR UPDATE` — khóa Row thật sự dưới DB |
| **Ai chặn ai?** | Không ai bị chặn, cứ đọc/sửa tự do | Transaction khác phải **chờ** tới khi Lock được nhả |
| **Chi phí khi tải cao (High Contention)** | Nhiều Transaction bị `rollback`, phải retry nhiều lần | Hàng đợi (queue) chờ Lock phình to, tăng Latency |
| **Chi phí khi tải thấp (Low Contention)** | **Gần như miễn phí** — không tốn tài nguyên giữ Lock | Vẫn tốn chi phí quản lý Lock dù ít người tranh chấp |
| **Giữ Connection lâu?** | Không — mỗi thao tác là 1 Transaction ngắn gọn | **Có** — Connection bị giữ suốt thời gian chờ user thao tác |
| **Phù hợp "Long UI Conversation"?** | **Có** — không sợ user để tab mở hàng chục phút | **Không** — Lock treo lơ lửng gây cạn Connection Pool |
| **Độ phức tạp xử lý lỗi** | Cần code Retry/thông báo "Dữ liệu đã bị thay đổi, vui lòng tải lại" | Đơn giản hơn — cứ chờ là xong, hiếm khi lỗi |
| **Ví dụ điển hình** | CMS, quản lý hồ sơ, form sửa thông tin, blog, e-commerce trung bình | Hệ thống ngân hàng, ví điện tử, đặt vé máy bay/flash sale |

---

## IX. Trade-off: Khi nào chọn cái nào?

* **Optimistic Locking là lựa chọn mặc định hợp lý cho đa số ứng dụng Web CRUD** — tỷ lệ 2 người cùng sửa đúng 1 dòng dữ liệu trong cùng khoảnh khắc thường rất thấp, nên "đặt cược" vào việc hiếm khi xung đột là hoàn toàn hợp lý, đổi lại Throughput cao hơn hẳn vì không ai phải xếp hàng chờ Lock.
* **Pessimistic Locking phù hợp khi mức độ tranh chấp (Contention) cực cao trên cùng 1 dòng** — ví dụ hàng nghìn người cùng bấm mua 1 vé concert cuối cùng, hoặc trừ tiền trên cùng 1 tài khoản ví điện tử ở tốc độ cao. Lúc này, nếu dùng Optimistic Locking, phần lớn Transaction sẽ liên tục bị `rollback` và phải `retry`, tạo ra vòng lặp lãng phí CPU còn tệ hơn cả việc đứng xếp hàng chờ Lock đàng hoàng.
* **Có thể kết hợp cả hai**: dùng Optimistic Locking làm "lưới an toàn" mặc định cho toàn hệ thống, chỉ chuyển sang Pessimistic Locking (`@Lock(LockModeType.PESSIMISTIC_WRITE)`) cho đúng những nghiệp vụ cực kỳ nhạy cảm, tần suất tranh chấp cao đã được xác định rõ ràng — không nên áp dụng Pessimistic tràn lan "cho chắc" vì cái giá phải trả về Connection Pool và Throughput là rất lớn (xem lại hệ quả `REQUIRES_NEW`/giữ Connection lâu ở [Chương 3 - Propagation](../consistency%20%26%20transaction%20management/3_PROPAGATION_AND_CONNECTION_MANAGEMENT.md)).

---

## X. Cheat Sheet Phỏng vấn (Interview Q&A)

### Q1: Vì sao gọi là "Optimistic" (Lạc quan)? Bản chất nó có tạo Lock dưới Database không?
> **Trả lời**: Gọi là "lạc quan" vì hệ thống giả định xung đột dữ liệu hiếm khi xảy ra. Nó **không hề tạo bất kỳ khóa vật lý nào** dưới Database — không `LOCK`, không `FOR UPDATE`, không Transaction nào bị block chờ đợi. Toàn bộ cơ chế chỉ là sự phối hợp giữa Hibernate (sinh câu SQL có điều kiện `WHERE version = ?`) và JDBC Driver (trả về update count) ở tầng Application, Database hoàn toàn không biết khái niệm Optimistic Locking.

### Q2: Trình bày cơ chế hoạt động của `@Version` khi Hibernate thực hiện UPDATE.
> **Trả lời**: Khi Entity có field `@Version`, mỗi lần UPDATE, Hibernate tự động sinh câu lệnh dạng `UPDATE table SET ..., version = version_cũ + 1 WHERE id = ? AND version = version_cũ`. Điều kiện `AND version = version_cũ` đóng vai trò "chốt chặn" — chỉ khi dữ liệu dưới DB vẫn còn đúng phiên bản mà Entity đã đọc lên, câu UPDATE mới thực sự khớp và chạy thành công.

### Q3: Hibernate phát hiện có xung đột dữ liệu bằng cách nào, trong khi Database không hề báo lỗi?
> **Trả lời**: Dựa vào **update count** — giá trị `int` mà hàm `executeUpdate()` của `PreparedStatement` (JDBC) luôn trả về, đại diện cho số dòng thực sự bị tác động. Nếu điều kiện `WHERE id=? AND version=?` không khớp bất kỳ dòng nào (vì version đã bị người khác đổi trước), Database không báo lỗi, nó chỉ âm thầm trả về `updateCount = 0`. Hibernate coi số 0 này là dấu hiệu xung đột, lập tức ném `OptimisticLockException` và rollback Transaction.

### Q4: Vì sao Optimistic Locking giải quyết tốt bài toán "Long UI Conversation" hơn Pessimistic Locking?
> **Trả lời**: Vì Optimistic Locking không cần giữ bất kỳ Transaction/Connection nào "sống" trong lúc User đang thao tác trên UI (điền form, suy nghĩ, để tab mở...) — mỗi lần đọc và mỗi lần ghi là các Transaction ngắn, độc lập, chỉ kiểm tra `version` tại đúng khoảnh khắc bấm Save. Ngược lại, Pessimistic Locking (`SELECT FOR UPDATE`) buộc phải giữ Row Lock (và cả Connection) suốt thời gian User thao tác trên UI — với những phiên làm việc dài (hàng chục phút), điều này gây cạn kiệt Connection Pool gần như ngay lập tức.

### Q5: Tại sao Bulk Update bằng JPQL (`@Modifying @Query`) lại là một cạm bẫy với Optimistic Locking?
> **Trả lời**: Vì Bulk Update JPQL chạy thẳng câu SQL xuống DB, bỏ qua hoàn toàn cơ chế Dirty Checking/flush tự động của Persistence Context — nơi Hibernate mới thực sự chèn logic tăng `version` và thêm điều kiện `WHERE version = ?`. Nếu không tự tay viết `p.version = p.version + 1` trong câu JPQL, cột `version` sẽ không được cập nhật, khiến cơ chế Optimistic Locking bị "lách qua mặt" hoàn toàn cho những dòng bị Bulk Update.

### Q6: Khi gặp `OptimisticLockException`, ứng dụng nên xử lý như thế nào?
> **Trả lời**: Cách phổ biến nhất là **Retry** — bắt Exception, tải lại Entity mới nhất từ DB (với `version` mới), áp lại thay đổi nghiệp vụ, rồi thử lưu lại lần nữa, thường giới hạn số lần thử tối đa. Mỗi lần retry nên chạy trong 1 Transaction mới (`Propagation.REQUIRES_NEW`) để đảm bảo lấy đúng dữ liệu mới nhất, không tiếp tục dùng Entity "đã hỏng" từ lần thử trước. Nếu retry vượt quá ngưỡng cho phép, nên trả về thông báo rõ ràng cho người dùng kiểu "Dữ liệu vừa bị người khác thay đổi, vui lòng tải lại trang".

### Q7: `LockModeType.OPTIMISTIC` khác `OPTIMISTIC_FORCE_INCREMENT` ở điểm nào?
> **Trả lời**: `OPTIMISTIC` chỉ kiểm tra `version` chưa bị ai đổi tại thời điểm flush/commit, không ép tăng version nếu bản thân Entity không có field nào bị sửa. `OPTIMISTIC_FORCE_INCREMENT` luôn ép tăng `version` khi Transaction kết thúc dù Entity gốc không đổi field nào — hữu ích khi cần "lan truyền" tín hiệu thay đổi từ Entity con lên Entity cha (ví dụ sửa `OrderItem` thì `Order` cha cũng cần được đánh dấu là vừa có cập nhật).

### Q8: So sánh nhanh: khi nào nên chọn Optimistic Locking, khi nào nên chọn Pessimistic Locking?
> **Trả lời**: Optimistic Locking phù hợp cho phần lớn ứng dụng Web CRUD thông thường (CMS, quản lý hồ sơ, e-commerce tầm trung) — nơi tỷ lệ tranh chấp trên cùng 1 dòng dữ liệu thấp, ưu tiên Throughput cao. Pessimistic Locking phù hợp cho các nghiệp vụ có mức độ tranh chấp cực cao trên cùng 1 dòng và không chấp nhận sai sót (ví dụ trừ tiền ví điện tử, đặt vé số lượng giới hạn) — ở đây, số lần Optimistic Locking phải `rollback`/`retry` liên tục sẽ còn tốn kém hơn việc xếp hàng chờ Lock đàng hoàng.

---

> **Tài liệu tham khảo:**
> - [Optimistic locking with JPA and Hibernate - Vlad Mihalcea](https://vladmihalcea.com/optimistic-locking-version-property-jpa-hibernate/)
> - [How to address the OptimisticLockException in JPA and Hibernate - Vlad Mihalcea](https://vladmihalcea.com/an-entity-modeling-strategy-for-scaling-optimistic-locking/)
> - [Optimistic Locking in JPA - Baeldung](https://www.baeldung.com/jpa-optimistic-locking)
> - [Pessimistic vs Optimistic Locking: Khi nào nên "khóa chặt", khi nào nên "thả lỏng"? - Viblo](https://viblo.asia/p/pessimistic-vs-optimistic-locking-khi-nao-nen-khoa-chat-khi-nao-nen-tha-long-ym4008OW491)
> - [Optimistic lock và Pessimistic lock - Viblo](https://viblo.asia/p/009-optimistic-lock-va-pessimistic-lock-L4x5xr7aZBM)
