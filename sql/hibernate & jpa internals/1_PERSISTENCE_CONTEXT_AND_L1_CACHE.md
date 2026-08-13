# Chương 1: Persistence Context & Bộ đệm cấp 1 (L1 Cache)

![Status](https://img.shields.io/badge/Persistence_Context-green) ![Topic](https://img.shields.io/badge/L1_Cache-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
  - [1. Persistence Context là gì?](#1-persistence-context-là-gì)
  - [2. Vòng đời gắn liền với Session và Transaction](#2-vòng-đời-gắn-liền-với-session-và-transaction)
- [II. Cơ chế Identity Map: Trái tim của First-Level Cache](#ii-cơ-chế-identity-map-trái-tim-của-first-level-cache)
  - [1. Cấu trúc dữ liệu: Key là Class + ID, Value là Reference](#1-cấu-trúc-dữ-liệu-key-là-class--id-value-là-reference)
  - [2. Vì sao gọi find() lần 2 không cần đá xuống DB?](#2-vì-sao-gọi-find-lần-2-không-cần-đá-xuống-db)
  - [3. "Repeatable Read" ở tầng ứng dụng: user1 == user2](#3-repeatable-read-ở-tầng-ứng-dụng-user1--user2)
- [III. Cạm bẫy: JPQL/Query KHÔNG né được câu SELECT](#iii-cạm-bẫy-jpqlquery-không-né-được-câu-select)
  - [1. find() vs Query: khác biệt về hành vi](#1-find-vs-query-khác-biệt-về-hành-vi)
  - [2. Cơ chế "Load rồi vứt" khi bản ghi đã Managed](#2-cơ-chế-load-rồi-vứt-khi-bản-ghi-đã-managed)
- [IV. Vòng đời Entity (Entity Lifecycle States)](#iv-vòng-đời-entity-entity-lifecycle-states)
  - [1. Bốn trạng thái: Transient, Managed, Detached, Removed](#1-bốn-trạng-thái-transient-managed-detached-removed)
  - [2. Bảng chuyển trạng thái qua các method của EntityManager](#2-bảng-chuyển-trạng-thái-qua-các-method-của-entitymanager)
- [V. Dirty Checking \& Cơ chế Flush](#v-dirty-checking--cơ-chế-flush)
  - [1. Snapshot so sánh (Loaded State) là gì?](#1-snapshot-so-sánh-loaded-state-là-gì)
  - [2. Flush xảy ra khi nào? Các FlushModeType](#2-flush-xảy-ra-khi-nào-các-flushmodetype)
  - [3. Write-behind: vì sao gọi setter không bắn SQL ngay lập tức?](#3-write-behind-vì-sao-gọi-setter-không-bắn-sql-ngay-lập-tức)
- [VI. Phân biệt nhanh: First-Level Cache vs Second-Level Cache](#vi-phân-biệt-nhanh-first-level-cache-vs-second-level-cache)
- [VII. Những cạm bẫy thực chiến (Pitfalls)](#vii-những-cạm-bẫy-thực-chiến-pitfalls)
  - [1. OutOfMemoryError khi batch insert/update hàng loạt](#1-outofmemoryerror-khi-batch-insertupdate-hàng-loạt)
  - [2. NonUniqueObjectException khi có 2 object cùng định danh](#2-nonuniqueobjectexception-khi-có-2-object-cùng-định-danh)
  - [3. Persistence Context không Thread-safe, không dùng chung giữa nhiều Request](#3-persistence-context-không-thread-safe-không-dùng-chung-giữa-nhiều-request)
- [VIII. Cheat Sheet Phỏng vấn (Interview Q\&A)](#viii-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

### 1. Persistence Context là gì?

Điều đầu tiên cần khắc sâu: **Persistence Context không phải là Database, cũng không phải là 1 khái niệm trừu tượng mơ hồ** — nó là một **vùng nhớ đệm (Staging Area) nằm ngay trên RAM** của ứng dụng Java, do Hibernate/JPA quản lý.

Trong JPA, Persistence Context được đại diện bởi interface `EntityManager`. Trong Hibernate (implementation phổ biến nhất của JPA), nó được đại diện bởi interface `Session` (chính `Session` cũng `extends EntityManager`). Nói cách khác, mỗi khi bạn cầm trên tay 1 object `EntityManager`/`Session`, bạn đang cầm trên tay 1 Persistence Context.

Nhiệm vụ của Persistence Context:
* Lưu giữ danh sách các Entity đang được "quản lý" (Managed) trong phiên làm việc hiện tại.
* Theo dõi mọi thay đổi trên các Entity đó để tự động đồng bộ xuống DB (**Dirty Checking**, xem Mục V).
* Đóng vai trò **First-Level Cache (Bộ đệm cấp 1)** — tránh việc phải hỏi lại Database những gì đã từng hỏi trong cùng phiên làm việc (xem Mục II).

### 2. Vòng đời gắn liền với Session và Transaction

Mỗi `Session` (hoặc `EntityManager`) sở hữu **một Persistence Context hoàn toàn riêng biệt**, không chia sẻ với `Session` khác. Theo cấu hình mặc định phổ biến nhất (`PersistenceContextType.TRANSACTION`), vòng đời của Persistence Context gắn chặt với vòng đời của **Transaction**:

```
begin Transaction ──► Persistence Context được tạo mới (rỗng)
      │
      │  find(), persist(), setter, ...  (mọi entity được nạp vào PC)
      ▼
commit/rollback Transaction ──► Persistence Context bị hủy, toàn bộ entity trở thành Detached
```

Điều này có nghĩa: trong Spring Boot, mỗi phương thức `@Transactional` (mặc định) sẽ làm việc với **một Persistence Context mới toanh**, khác hoàn toàn với Persistence Context của request trước đó — đây là lý do vì sao 2 Transaction khác nhau **không bao giờ** chia sẻ chung 1 vùng cache L1, và cũng là lý do entity load ra ngoài phạm vi Transaction (ví dụ trả thẳng ra tầng View mà không có OSIV) sẽ rơi vào trạng thái **Detached** (Mục IV) — không còn được Dirty Checking theo dõi nữa.

---

## II. Cơ chế Identity Map: Trái tim của First-Level Cache

### 1. Cấu trúc dữ liệu: Key là Class + ID, Value là Reference

Bên trong, Hibernate quản lý các Entity trong Persistence Context bằng một cấu trúc gọi là **Identity Map** — về bản chất là một `HashMap`:

```java
// Rút gọn từ mã nguồn thật của Hibernate (org.hibernate.engine.internal.StatefulPersistenceContext)
Map<EntityKey, Object> entitiesByKey = new HashMap<>();

class EntityKey {
    String entityName;   // "com.example.User"
    Object identifier;   // 1L
    // equals()/hashCode() dựa trên CẢ HAI field trên
}
```

* **Key** = sự kết hợp giữa **Tên Class** và **Khóa chính (ID)** — ví dụ `"User#1"`.
* **Value** = **địa chỉ tham chiếu (reference)** trực tiếp tới Object Java đang nằm trên RAM.

Ràng buộc quan trọng nhất của Identity Map: **trong một Persistence Context, tại một thời điểm, chỉ tồn tại đúng MỘT instance đại diện cho một dòng dữ liệu (entity name + id)**. Không bao giờ có chuyện 2 object Java khác nhau cùng đại diện cho `User#1` trong cùng 1 Session — nếu để điều đó xảy ra, hệ thống sẽ không biết bản nào là "sự thật" cần đồng bộ xuống DB.

### 2. Vì sao gọi find() lần 2 không cần đá xuống DB?

Khi bạn gọi `entityManager.find(User.class, 1L)`, quy trình bên trong Hibernate diễn ra theo thứ tự ưu tiên sau:

```
find(User.class, 1L)
        │
        ▼
1. Tính Key = EntityKey("User", 1L)
        │
        ▼
2. Tra trong Identity Map (L1 Cache) của Persistence Context hiện tại
        │
   ┌────┴────┐
   │ Có sẵn? │
   └────┬────┘
     Có │            Không có
        ▼                ▼
  Trả về reference   3. Tra Second-Level Cache (nếu bật, xem Mục VI)
  đang có sẵn,                 │
  KHÔNG đụng DB        ┌───────┴────────┐
                    Có │            Không có
                       ▼                ▼
                  Trả về từ L2     4. Bắt buộc SELECT xuống DB,
                                       dựng Object, LƯU vào Identity Map,
                                       rồi mới trả về
```

Áp vào ví dụ cụ thể:

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

User user1 = em.find(User.class, 1L);
// Lần 1: Identity Map rỗng → BẮT BUỘC mở connection, chạy
//   SELECT * FROM users WHERE id = 1
// → ép kiểu ResultSet thành Object User, LƯU vào Identity Map với key "User#1",
//   rồi trả reference đó cho user1

User user2 = em.find(User.class, 1L);
// Lần 2: tra Identity Map, thấy Key "User#1" ĐÃ TỒN TẠI
// → trả THẲNG reference cũ, KHÔNG có bất kỳ round-trip mạng nào tới DB

em.getTransaction().commit();
```

Đây là lý do trong log SQL (bật `show-sql=true`), bạn chỉ thấy **đúng 1 câu SELECT** dù gọi `find()` với cùng ID bao nhiêu lần đi nữa trong cùng Transaction.

### 3. "Repeatable Read" ở tầng ứng dụng: user1 == user2

Vì `user1` và `user2` ở trên trỏ chung tới **một địa chỉ bộ nhớ duy nhất**, phép so sánh `==` (so sánh reference, không phải `equals()`) sẽ trả về `true`:

```java
System.out.println(user1 == user2); // true
```

Điều này quan trọng hơn vẻ ngoài của nó rất nhiều: nó đảm bảo tính chất mà Vlad Mihalcea gọi là **"application-level repeatable reads"** — dù bạn gọi `find()` (hoặc bất kỳ chỗ nào khác trong code đang giữ chung 1 Persistence Context) bao nhiêu lần với cùng 1 ID, bạn luôn nhận về **cùng một Object**, cùng một trạng thái, không lo 2 biến "lệch pha" nhau dù đang đại diện cho cùng 1 dòng DB. Đây chính là điểm khác biệt so với Isolation Level ở tầng Database (đã bàn ở [chương Isolation Level & MVCC](../consistency%20%26%20transaction%20management/2_ISOLATION_LEVEL_AND_MVCC.md)) — Repeatable Read ở đây là cam kết **do tầng Application (Hibernate) đảm bảo**, không phụ thuộc DB Engine đang cấu hình Isolation Level nào.

---

## III. Cạm bẫy: JPQL/Query KHÔNG né được câu SELECT

Đây là điểm rất hay bị hiểu nhầm khi mới học Hibernate: nhiều người nghĩ rằng cứ Entity đã có trong Persistence Context thì **mọi cách truy vấn** (kể cả JPQL, Criteria, native Query) đều sẽ được L1 Cache chặn lại. **Sai.**

### 1. find() vs Query: khác biệt về hành vi

| | `EntityManager.find()` | JPQL / Criteria / native Query |
| :--- | :--- | :--- |
| Có tra Identity Map **trước khi** đụng DB không? | **Có** — nếu thấy Key đã tồn tại, trả về ngay, không có SQL nào chạy | **Không** — Query luôn được dịch thành SQL và gửi thẳng xuống DB |
| Có chắc chắn tránh được round-trip mạng không? | Có (nếu cache hit) | **Không bao giờ** — kể cả khi kết quả trả về đã có sẵn trong Identity Map |

### 2. Cơ chế "Load rồi vứt" khi bản ghi đã Managed

```java
User user1 = em.find(User.class, 1L);
// (1) SELECT * FROM users WHERE id = 1  → lưu vào Identity Map "User#1"

List<User> users = em.createQuery(
        "select u from User u where u.id = :id", User.class)
    .setParameter("id", 1L)
    .getResultList();
// (2) JPQL LUÔN LUÔN bắn: SELECT * FROM users WHERE id = 1
//     Identity Map KHÔNG hề chặn được câu Query này!
//
//     Nhưng: sau khi ResultSet trả về, Hibernate phát hiện Key "User#1"
//     ĐÃ tồn tại trong Persistence Context → nó VỨT BỎ dữ liệu vừa đọc được
//     (chỉ dùng để so sánh/refresh nếu cần), và trả về NGUYÊN object user1 cũ.

System.out.println(user1 == users.get(0)); // true — vẫn cùng 1 reference!
```

Nói cách khác: Identity Map **không giúp giảm số lượng câu SQL khi dùng Query**, nhưng **vẫn giữ đúng tính bất biến "1 ID = 1 Reference"** — nó chỉ khác `find()` ở chỗ vẫn phải trả giá 1 round-trip DB, dù cuối cùng "hàng vừa mua về" bị vứt đi để giữ nguyên "hàng cũ" đang có trên kệ. Đây cũng chính là lý do một Transaction dài, gọi lại Query nhiều lần trên cùng 1 dòng dữ liệu **vẫn không thấy thay đổi từ Transaction khác đã commit xen giữa** — vì Hibernate luôn ưu tiên giữ nguyên state đã Managed từ đầu, đúng tinh thần repeatable read ở Mục II.3.

> **Ghi nhớ khi đi phỏng vấn**: "First-Level Cache ngăn được nhiều lần gọi `find()` cùng ID tạo ra nhiều câu SELECT, nhưng **không thể ngăn** một câu JPQL/SQL load lại bản ghi mới nhất từ DB — nó chỉ đảm bảo đối tượng trả về cho bạn là cùng 1 reference đã Managed."

---

## IV. Vòng đời Entity (Entity Lifecycle States)

### 1. Bốn trạng thái: Transient, Managed, Detached, Removed

Mọi Entity trong JPA/Hibernate tại bất kỳ thời điểm nào cũng nằm ở đúng 1 trong 4 trạng thái sau, và Persistence Context chính là "biên giới" phân định các trạng thái này:

| Trạng thái | Ý nghĩa | Có trong Persistence Context (Identity Map)? | Dirty Checking có áp dụng? |
| :--- | :--- | :---: | :---: |
| **Transient** (New) | Vừa `new User()`, chưa từng gắn với bất kỳ Session nào | Không | Không |
| **Managed** (Persistent) | Đang được 1 Persistence Context quản lý, đại diện đúng 1 dòng DB | **Có** | **Có** — mọi thay đổi field tự động sinh UPDATE khi flush |
| **Detached** | Đã từng Managed, nhưng Session đóng / `clear()` / `evict()` khiến nó bị "văng" ra khỏi Persistence Context | Không | Không — sửa field không tự đồng bộ xuống DB nữa |
| **Removed** | Đã gọi `remove()`, đang chờ câu `DELETE` ở lần flush kế tiếp | Có (tạm thời) | Không cần — số phận đã định là bị xóa |

### 2. Bảng chuyển trạng thái qua các method của EntityManager

```
                        new User()
                            │
                            ▼ 
                     ┌─────────────┐
                     │  TRANSIENT  │
                     └──────┬──────┘
                            │ persist()
                            ▼
   find() ────────►  ┌─────────────┐   remove()   ┌─────────────┐
   getReference()    │   MANAGED   │ ───────────► │   REMOVED   │
   merge() ────────► │ (Persistent)│ ◄─────────── │ (chờ DELETE)│
                     └──────┬──────┘   persist()  └─────────────┘
                            │  detach() / clear() / evict() / close()
                            ▼
                     ┌─────────────┐
                     │  DETACHED   │
                     └──────┬──────┘
                            │ merge() — tạo 1 bản Managed MỚI (copy),
                            │           KHÔNG biến đối tượng detached
                            ▼           thành managed tại chỗ
                  (MANAGED — instance khác)
```

Vài lưu ý dễ gây lỗi thực chiến:
* `merge()` trên 1 object Detached **không** biến chính object đó thành Managed — nó **copy dữ liệu sang một object Managed khác** (đã có sẵn trong Identity Map hoặc vừa load từ DB) rồi trả về object Managed đó. Nếu bạn tiếp tục sửa trên object Detached ban đầu, thay đổi đó **sẽ không được lưu**.
* `persist()` trên 1 object đang ở trạng thái Removed (chưa flush) sẽ đưa nó quay lại Managed — hủy luôn lệnh xóa đang chờ.
* Đóng `EntityManager`/`Session` khiến **toàn bộ** Entity đang Managed trong đó đồng loạt trở thành Detached — không cần gọi `detach()` từng cái.

---

## V. Dirty Checking & Cơ chế Flush

### 1. Snapshot so sánh (Loaded State) là gì?

Ngay tại thời điểm 1 Entity chuyển sang trạng thái Managed (do `find()` load lên, hoặc do `persist()` sau khi đã có ID), Hibernate **chụp lại một bản snapshot** toàn bộ giá trị field của nó tại thời điểm đó, gọi là **loaded state** (bản chất là 1 mảng `Object[]`), lưu song song bên cạnh chính Entity trong Persistence Context.

Khi `flush()` xảy ra, Hibernate duyệt qua **toàn bộ** Entity đang Managed, so sánh **field-by-field** giữa giá trị hiện tại và loaded state cũ:
* Nếu **không có gì khác biệt** → bỏ qua, không sinh SQL.
* Nếu **có field khác biệt** → Entity được đánh dấu "dirty", Hibernate tự sinh câu `UPDATE` tương ứng — **hoàn toàn không cần bạn gọi `save()`/`update()` thủ công**.

### 2. Flush xảy ra khi nào? Các FlushModeType

| FlushModeType | Khi nào flush? | Rủi ro |
| :--- | :--- | :--- |
| **`AUTO`** (mặc định) | Trước mỗi lần **commit**, và trước mỗi **Query** nếu Hibernate phát hiện Query đó có thể bị ảnh hưởng bởi thay đổi chưa đồng bộ trong Persistence Context | An toàn nhất, nhưng có thể flush "thừa" nhiều lần hơn cần thiết |
| **`COMMIT`** | **Chỉ** flush khi Transaction commit | Query chạy giữa chừng Transaction có thể **đọc phải dữ liệu cũ**, nếu trước đó bạn vừa sửa Entity mà chưa flush |
| `MANUAL` *(riêng của Hibernate `Session`, không có trong JPA chuẩn)* | Chỉ flush khi gọi `session.flush()` thủ công | Dễ quên flush → mất dữ liệu tưởng đã lưu, hoặc Query đọc dữ liệu stale |

### 3. Write-behind: vì sao gọi setter không bắn SQL ngay lập tức?

Một hiểu lầm phổ biến khác: gọi `user.setName("Khương")` trên 1 Entity Managed sẽ khiến Hibernate **lập tức** gửi `UPDATE` xuống DB. **Không đúng.** Setter chỉ đơn thuần thay đổi field trên Object Java bình thường trong RAM — không có bất kỳ tương tác nào với DB tại thời điểm đó.

Chỉ đến khi `flush()` được kích hoạt (theo FlushModeType ở trên), Hibernate mới:
1. Duyệt **Action Queue** — hàng đợi nội bộ chứa các thao tác đang chờ (`EntityInsertAction`, `EntityUpdateAction`, `EntityDeleteAction`, `CollectionUpdateAction`...).
2. **Sắp xếp lại thứ tự** thực thi theo nhóm: tất cả `INSERT` chạy trước, rồi tới `UPDATE`, cuối cùng mới tới `DELETE` — nhằm tránh vi phạm ràng buộc khóa ngoại (ví dụ: không thể `UPDATE` tham chiếu tới 1 dòng chưa kịp `INSERT`).
3. Gom nhóm và gửi các câu lệnh SQL cùng loại theo batch (nếu bật `hibernate.jdbc.batch_size`) xuống DB qua JDBC.

Cơ chế trì hoãn này gọi là **Write-behind (Transactional write-behind)** — nó cho phép Hibernate tối ưu số lượng round-trip xuống DB (gom nhiều thay đổi lại thành ít đợt gửi nhất có thể), thay vì gửi ngay từng câu lệnh một cách "ngây thơ" mỗi khi có 1 dòng code thay đổi field.

---

## VI. Phân biệt nhanh: First-Level Cache vs Second-Level Cache

Persistence Context/L1 Cache thường bị nhầm lẫn với Second-Level Cache (L2) — vốn là 1 khái niệm hoàn toàn khác cấp độ:

| Tiêu chí | First-Level Cache (L1) | Second-Level Cache (L2) |
| :--- | :--- | :--- |
| **Phạm vi** | 1 `Session`/`EntityManager` (thường = 1 Transaction) | Toàn bộ `SessionFactory`/`EntityManagerFactory` — **dùng chung** giữa nhiều Session/Transaction/Request |
| **Bật/tắt** | **Luôn luôn bật, không thể tắt** — là thành phần bắt buộc (mandatory) của JPA/Hibernate | **Mặc định TẮT** — phải cấu hình thêm thư viện cache ngoài (Ehcache, Redis, Caffeine, Infinispan...) |
| **Vòng đời** | Ngắn — bị hủy hoàn toàn khi Transaction/Session kết thúc | Dài — tồn tại xuyên suốt vòng đời ứng dụng, sống sót qua nhiều Transaction |
| **Thread-safety** | **Không** thread-safe (1 Session chỉ dùng cho 1 Thread) | **Có** — được thiết kế để nhiều Thread/Request cùng đọc |
| **Mục đích chính** | Identity Map + Dirty Checking + tránh lặp SELECT trong **cùng 1** Transaction | Giảm tải DB **xuyên suốt nhiều** Transaction/Request khác nhau, cho dữ liệu ít thay đổi |

---

## VII. Những cạm bẫy thực chiến (Pitfalls)

### 1. OutOfMemoryError khi batch insert/update hàng loạt

Vì Persistence Context giữ **reference tới mọi Entity** đã từng `find()`/`persist()` trong Session đó, nếu bạn viết 1 batch job duyệt và xử lý hàng trăm nghìn/hàng triệu dòng **trong cùng 1 Transaction/Session duy nhất**, Identity Map sẽ phình to không kiểm soát → **OutOfMemoryError**.

**Giải pháp thường dùng**: định kỳ gọi `entityManager.flush()` (đẩy các thay đổi đang chờ xuống DB) rồi `entityManager.clear()` (dọn sạch Identity Map, đưa mọi Entity đang Managed về Detached) sau mỗi N bản ghi — kỹ thuật này gọi là **batch flushing**. Với các job chỉ cần đọc (không cần Dirty Checking), nên cân nhắc dùng `StatelessSession` của Hibernate — hoàn toàn không có Persistence Context, đọc tới đâu trả về tới đó.

### 2. NonUniqueObjectException khi có 2 object cùng định danh

Lỗi này xảy ra khi bạn cố gắn (`persist()`/`update()`) 1 object **Detached** (hoặc tự tạo tay, gán sẵn ID trùng khớp) vào Session, trong khi Session đó **đã có sẵn** 1 Entity Managed khác với cùng `EntityKey` ("cùng Class + cùng ID"). Hibernate phát hiện điều này vi phạm trực tiếp ràng buộc cốt lõi của Identity Map ("1 Key = 1 Reference duy nhất") và ném `NonUniqueObjectException`.

**Giải pháp**: dùng `merge()` thay vì `persist()`/`update()` khi làm việc với object Detached đã có ID — `merge()` được thiết kế đúng để "hợp nhất" dữ liệu vào Entity Managed hiện có, thay vì cố nhét thêm 1 reference thứ 2 cho cùng 1 ID.

### 3. Persistence Context không Thread-safe, không dùng chung giữa nhiều Request

`EntityManager`/`Session` **không** được thiết kế để nhiều Thread cùng truy cập đồng thời — Identity Map bên trong (`HashMap` thường, không phải `ConcurrentHashMap`) không có bất kỳ cơ chế đồng bộ hóa (synchronization) nào. Trong ứng dụng Spring Boot, mỗi Request/Transaction nên có 1 Persistence Context riêng biệt (Spring tự động tạo mới cho mỗi `@Transactional` theo mặc định `PROPAGATION_REQUIRED`). Tuyệt đối không lưu `EntityManager` như 1 biến static/singleton dùng chung cho nhiều Request — sẽ dẫn tới lỗi dữ liệu ngẫu nhiên (race condition), rất khó tái hiện và debug.

---

## VIII. Cheat Sheet Phỏng vấn (Interview Q&A)

### Q1: Persistence Context là gì? Nó có phải là Database hay không?
> **Trả lời**: Persistence Context **không phải Database** — nó là một vùng nhớ đệm (Staging Area) nằm trên RAM của ứng dụng Java, do Hibernate/JPA quản lý, đại diện bởi `EntityManager` (JPA) hoặc `Session` (Hibernate). Nó lưu trữ và theo dõi trạng thái của các Entity trong 1 phiên làm việc, đồng thời đóng vai trò First-Level Cache. Vòng đời của nó thường gắn liền với 1 Transaction.

### Q2: Vì sao gọi `find()` 2 lần với cùng ID chỉ tốn đúng 1 câu SELECT?
> **Trả lời**: Vì Persistence Context quản lý Entity bằng cấu trúc **Identity Map** (bản chất là `HashMap<EntityKey, Object>`, Key = Class + ID). Lần gọi `find()` đầu tiên tra Map không thấy, buộc phải SELECT xuống DB rồi lưu kết quả vào Map. Lần gọi thứ 2 tra Map thấy Key đã tồn tại, trả thẳng reference đang có sẵn trên RAM, không có round-trip mạng nào xảy ra.

### Q3: Nếu dùng JPQL/Query thay vì `find()` thì First-Level Cache còn tác dụng không?
> **Trả lời**: Có tác dụng một phần nhưng dễ gây hiểu lầm. JPQL/Criteria/native Query **luôn luôn** được dịch thành SQL và gửi xuống DB — Identity Map **không hề chặn được** round-trip này. Tuy nhiên, sau khi có kết quả, nếu Hibernate phát hiện Entity với Key tương ứng **đã Managed** sẵn trong Persistence Context, nó sẽ **vứt bỏ** dữ liệu vừa đọc được và trả về nguyên object cũ đang quản lý — đảm bảo tính bất biến "1 Key = 1 Reference", nhưng không tiết kiệm được chi phí truy vấn.

### Q4: Phân biệt 4 trạng thái Transient, Managed, Detached, Removed.
> **Trả lời**: **Transient** — vừa `new()`, chưa gắn Session nào. **Managed** — đang nằm trong Persistence Context, mọi thay đổi field được Dirty Checking tự động đồng bộ xuống DB khi flush. **Detached** — đã từng Managed nhưng bị gỡ khỏi Persistence Context (đóng Session, `clear()`, `evict()`) — sửa đổi không còn tự động lưu. **Removed** — đã gọi `remove()`, chờ câu `DELETE` ở lần flush tiếp theo.

### Q5: Dirty Checking hoạt động dựa trên cơ chế nào bên trong?
> **Trả lời**: Ngay khi 1 Entity trở thành Managed, Hibernate chụp lại một **snapshot** (loaded state) toàn bộ giá trị field tại thời điểm đó. Khi `flush()` xảy ra, Hibernate so sánh **field-by-field** giữa state hiện tại và snapshot cũ của mọi Entity Managed — nếu phát hiện khác biệt, tự động sinh câu `UPDATE` tương ứng mà không cần gọi `save()`/`update()` thủ công.

### Q6: Tại sao Persistence Context có thể gây `OutOfMemoryError` trong 1 batch job?
> **Trả lời**: Vì Persistence Context giữ reference tới **mọi** Entity đã `find()`/`persist()` trong Session đó, không tự giải phóng cho tới khi Transaction kết thúc. Nếu 1 batch job xử lý hàng loạt bản ghi trong cùng 1 Session, Identity Map sẽ phình to liên tục → hết bộ nhớ. Khắc phục bằng cách định kỳ gọi `flush()` rồi `clear()` sau mỗi N bản ghi, hoặc dùng `StatelessSession` (không có Persistence Context) cho các tác vụ chỉ đọc/ghi thuần túy.

### Q7: Phân biệt First-Level Cache và Second-Level Cache.
> **Trả lời**: First-Level Cache gắn với **1 Session/Transaction**, luôn luôn bật và không thể tắt, không thread-safe, mất đi khi Transaction kết thúc. Second-Level Cache gắn với **toàn bộ SessionFactory**, mặc định tắt (cần cấu hình thêm như Ehcache/Redis), thread-safe, dùng chung và tồn tại xuyên suốt nhiều Transaction/Request khác nhau — mục đích là giảm tải DB ở phạm vi rộng hơn hẳn L1.

---

> **Tài liệu tham khảo:**
> - [The JPA and Hibernate first-level cache - Vlad Mihalcea](https://vladmihalcea.com/jpa-hibernate-first-level-cache/)
> - [JPA/Hibernate Persistence Context - Baeldung](https://www.baeldung.com/jpa-hibernate-persistence-context)
> - [Hibernate Entity Lifecycle - Baeldung](https://www.baeldung.com/hibernate-entity-lifecycle)
