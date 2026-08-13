# Chương 3: Transaction Propagation & Quản Lý Kết Nối

![Status](https://img.shields.io/badge/Propagation-green) ![Topic](https://img.shields.io/badge/Connection_Pool-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Physical vs Logical Transaction — Nền Tảng Phải Hiểu Trước](#i-physical-vs-logical-transaction--nền-tảng-phải-hiểu-trước)
- [II. 7 Loại Propagation — Nhìn Một Phát Nhớ Luôn](#ii-7-loại-propagation--nhìn-một-phát-nhớ-luôn)
- [III. REQUIRED: Rollback-Only Lây Lan Trong Cùng 1 Physical Transaction](#iii-required-rollback-only-lây-lan-trong-cùng-1-physical-transaction)
- [IV. REQUIRES_NEW: "Sát Thủ" Connection Pool](#iv-requires_new-sát-thủ-connection-pool)
- [V. Deadlock Kinh Điển: Khi Request Đồng Thời = Max Pool Size](#v-deadlock-kinh-điển-khi-request-đồng-thời--max-pool-size)
- [VI. NESTED: Dùng Savepoint, Không Tốn Thêm Connection](#vi-nested-dùng-savepoint-không-tốn-thêm-connection)
- [VII. REQUIRES_NEW vs NESTED — So Sánh Nhanh](#vii-requires_new-vs-nested--so-sánh-nhanh)
- [VIII. SUPPORTS / NOT_SUPPORTED / MANDATORY / NEVER — Khi Nào Dùng](#viii-supports--not_supported--mandatory--never--khi-nào-dùng)
- [IX. Bảng Tổng Hợp 7 Propagation (Cheat Sheet 1 Trang)](#ix-bảng-tổng-hợp-7-propagation-cheat-sheet-1-trang)
- [X. Cheat Sheet Phỏng Vấn (Q\&A)](#x-cheat-sheet-phỏng-vấn-qa)

---

## I. Physical vs Logical Transaction — Nền Tảng Phải Hiểu Trước

Đây là chìa khóa để hiểu mọi thứ phía sau. Chỉ cần nhớ 1 câu:

> **Physical Transaction = 1 Connection thật, 1 lần COMMIT/ROLLBACK thật dưới Database. 
>Logical Transaction = 1 ranh giới `@Transactional` mà Spring "vẽ" ra trên code — nhiều Logical Transaction có thể dùng CHUNG 1 Physical Transaction.**

Ví dụ dễ hình dung:

```java
@Service
class OrderService {
    @Transactional // Logical TX #1
    public void placeOrder() {
        paymentService.pay();   // gọi sang Service khác
    }
}

@Service
class PaymentService {
    @Transactional // Logical TX #2 — propagation mặc định REQUIRED
    public void pay() { ... }
}
```

* `placeOrder()` chạy trước → mở 1 **Physical Transaction** (lấy 1 Connection, `setAutoCommit(false)`).
* `pay()` được gọi bên trong → thấy đã có Physical Transaction đang chạy → **nhảy vào dùng chung**, không mở Connection mới.
* Kết quả: **2 Logical Transaction, nhưng chỉ 1 Physical Transaction, 1 Connection, 1 lần COMMIT thật**.

Việc "join chung hay tách riêng" chính xác thế nào là do **Propagation** quyết định — đó là chủ đề chính của cả bài này.

---

## II. 7 Loại Propagation — Nhìn Một Phát Nhớ Luôn

Trước khi phân tích chi tiết từng loại, đây là bảng "cầm tay" — chỉ cần nhớ 2 cột **Nếu ĐÃ có TX cha** và **Nếu CHƯA có TX cha**:

| Propagation | Nếu ĐÃ có TX cha | Nếu CHƯA có TX cha | Ghi nhớ 1 câu |
| :--- | :--- | :--- | :--- |
| **REQUIRED** *(mặc định)* | Dùng chung | Tạo mới | Đi ghép xe, không có xe thì tự lái |
| **REQUIRES_NEW** | Treo TX cha, mở TX mới | Tạo mới | Luôn đi xe riêng, không liên quan ai |
| **NESTED** | Tạo Savepoint trong TX cha | Tạo mới | Đi chung xe nhưng có "điểm lùi" riêng |
| **SUPPORTS** | Dùng chung | Chạy **không** transaction | Có thì theo, không thì thôi |
| **NOT_SUPPORTED** | Treo TX cha, chạy **không** transaction | Chạy **không** transaction | Luôn chạy "tay không", TX cha tạm nghỉ |
| **MANDATORY** | Dùng chung | **Ném Exception** | Bắt buộc phải có xe, đi bộ là bị phạt |
| **NEVER** | **Ném Exception** | Chạy **không** transaction | Cấm tiệt transaction, thấy là quăng lỗi |

Chỉ cần thuộc bảng này là trả lời được 80% câu hỏi phỏng vấn về Propagation. Các mục bên dưới giải thích sâu hơn 4 loại hay hỏi nhất: **REQUIRED, REQUIRES_NEW, NESTED**, và nhóm **SUPPORTS/NOT_SUPPORTED/MANDATORY/NEVER**.

---

## III. REQUIRED: Rollback-Only Lây Lan Trong Cùng 1 Physical Transaction

Vì `REQUIRED` dùng **chung Physical Transaction**, nên hệ quả quan trọng nhất là: **1 method con lỗi có thể "kéo sập" toàn bộ, kể cả phần method cha đã làm xong trước đó** — dù method cha có `try-catch` nuốt lỗi đi chăng nữa.

```java
@Transactional // A — REQUIRED
public void placeOrder(Order order) {
    orderRepository.save(order);      // ✅ chạy thành công, save OK
    try {
        paymentService.pay(order);    // ❌ B ném RuntimeException
    } catch (Exception e) {
        log.warn("Thanh toán lỗi, bỏ qua"); // "nuốt" lỗi
    }
    // placeOrder() chạy xong, tưởng chừng OK...
}
```

**Chuyện gì thực sự xảy ra:**
1. `pay()` (method B, cũng `REQUIRED`) lỗi → vì B chỉ là "khách tham gia" (participant) trong Physical Transaction của A, B **không được phép tự rollback connection** (làm vậy sẽ phá luôn phần A đang dùng) → B chỉ **đánh dấu cờ `rollbackOnly = true`** lên Physical Transaction chung rồi ném lại Exception.
2. `placeOrder()` bắt được Exception, log lại, coi như xử lý xong, method chạy hết bình thường.
3. Nhưng khi Spring chuẩn bị `COMMIT` ở cấp A, nó phát hiện cờ `rollbackOnly = true` đã bị bật → **không thể commit một Physical Transaction đã bị đánh dấu hỏng** → Spring buộc phải ROLLBACK toàn bộ (kể cả `orderRepository.save(order)` đã "tưởng như" thành công) và ném ra `UnexpectedRollbackException`.

> **Bài học nhớ đời**: Trong cùng 1 Physical Transaction, **không có khái niệm "commit một phần"**. Nuốt Exception ở tầng cha **không cứu được** dữ liệu — muốn phần B lỗi không ảnh hưởng phần A, phải dùng `REQUIRES_NEW` hoặc `NESTED` (Mục IV, VI).

---

## IV. REQUIRES_NEW: "Sát Thủ" Connection Pool

`REQUIRES_NEW` tách hẳn ra một Physical Transaction độc lập. Cơ chế **treo (suspend)** hoạt động như sau:

```
Thread đang chạy method A (REQUIRED, đã có Connection-A)
        │
        ▼  gọi method B (REQUIRES_NEW)
┌──────────────────────────────────────────────────────────────────────┐
│ 1. "Cất" Connection-A đi (gỡ khỏi ThreadLocal, giữ tạm trong bộ nhớ) │
│ 2. Xin HikariCP một Connection-B HOÀN TOÀN MỚI                       │
│ 3. Chạy method B với Connection-B, commit/rollback riêng độc lập     │
│ 4. Trả Connection-B về Pool                                          │
│ 5. "Đánh thức" Connection-A, gắn lại vào ThreadLocal, A chạy tiếp    │
└──────────────────────────────────────────────────────────────────────┘
```

**Điểm chí mạng cần nhớ**: trong khoảng thời gian method B đang chạy, **1 Thread đang cầm giữ ĐỒNG THỜI 2 Connection** (A đang "đóng băng" chờ + B đang hoạt động). Đây chính là gốc rễ của thảm họa ở Mục V.

Dùng `REQUIRES_NEW` khi nào? Khi bạn **cố tình** muốn 1 phần việc **luôn được lưu xuống DB** dù phần bên ngoài sau đó có rollback — ví dụ ghi Audit Log, gửi lịch sử giao dịch, mà dù nghiệp vụ chính thất bại vẫn cần giữ lại bằng chứng đã cố gắng thực hiện.

---

## V. Deadlock Kinh Điển: Khi Request Đồng Thời = Max Pool Size

Đây là bug kinh điển khiến hệ thống "đứng hình" hoàn toàn dưới tải cao, dù DB không hề có deadlock ở tầng row-lock nào.

**Kịch bản** (giả sử HikariCP cấu hình `maximum-pool-size = 10`):

```
10 Request đến CÙNG LÚC, mỗi Request:
    method A (REQUIRED)  →  giữ 1 Connection
        │
        └─► gọi method B (REQUIRES_NEW) → cần XIN THÊM 1 Connection MỚI

Sau khi cả 10 Request đều vào trong method A:
    Pool: 10/10 Connection đã bị 10 Thread giữ (đứng chờ ở bước gọi B)
    → Không còn Connection nào trống để cấp cho B
    → CẢ 10 THREAD đều bị block chờ Connection cho B
    → Không Thread nào có thể COMMIT method A để TRẢ Connection về Pool
       (vì đang bị kẹt ở giữa, chờ B chạy xong)
    → BẾ TẮC HOÀN TOÀN (Connection Pool Starvation)
```

Sau khoảng thời gian `connectionTimeout` (mặc định HikariCP là 30 giây), toàn bộ 10 Thread đồng loạt nhận:
```
SQLTransientConnectionException: HikariPool-1 - Connection is not available,
request timed out after 30000ms.
```

> **Lưu ý quan trọng**: Đây **không phải** Deadlock ở tầng Database (không có 2 transaction khóa chéo row của nhau) — nó là **Application-level Deadlock**, do chính HikariCP khuyến cáo rõ trong tài liệu của họ: *"Không nên giữ nhiều hơn 1 Connection cho cùng 1 Thread tại cùng một thời điểm, trừ khi bạn hiểu chính xác mình đang làm gì."*

**Cách phòng tránh thực tế**:
* Hạn chế tối đa việc gọi `REQUIRES_NEW` **lồng bên trong** một luồng nghiệp vụ chính đang chịu tải cao — chỉ dùng cho tác vụ phụ, nhẹ, tần suất thấp (audit log, notification).
* Nếu bắt buộc phải tách biệt hoàn toàn, cân nhắc dùng **DataSource/Pool riêng** cho tác vụ `REQUIRES_NEW` đó, hoặc đẩy sang xử lý **bất đồng bộ** (message queue, `@Async`) thay vì transaction lồng đồng bộ.
* Luôn đặt `maximum-pool-size` có dự phòng, và giám sát số Connection active để phát hiện sớm dấu hiệu cạn pool.

---

## VI. NESTED: Dùng Savepoint, Không Tốn Thêm Connection

`NESTED` giải quyết đúng vấn đề của `REQUIRED` (Mục III) nhưng **không tốn thêm Connection** như `REQUIRES_NEW`:

* **Không treo, không mở Connection mới** — vẫn dùng chung đúng 1 Physical Transaction, 1 Connection với method cha.
* Spring chỉ tạo một **JDBC Savepoint** (`connection.setSavepoint()`) ngay tại điểm bắt đầu method con.
* Nếu method con lỗi → Spring chỉ **rollback về đúng Savepoint đó** (`connection.rollback(savepoint)`) — coi như "tẩy" đúng phần việc của method con, **method cha vẫn sống**, có thể tiếp tục chạy và commit bình thường.

```java
@Transactional // cha — REQUIRED
public void placeOrder(Order order) {
    orderRepository.save(order);        // ✅ vẫn được giữ lại
    try {
        paymentService.payNested(order); // NESTED — lỗi thì chỉ rollback riêng nó
    } catch (Exception e) {
        log.warn("Thanh toán lỗi, đơn hàng vẫn được tạo ở trạng thái chờ");
    }
    // Transaction cha COMMIT bình thường, đơn hàng vẫn được lưu!
}

@Service
class PaymentService {
    @Transactional(propagation = Propagation.NESTED)
    public void payNested(Order order) { ... } // lỗi ở đây không kéo sập cha
}
```

> **Lưu ý thực tế**: `NESTED` chạy mượt với `DataSourceTransactionManager` (JDBC thuần, hỗ trợ Savepoint sẵn có). Với JPA/Hibernate (`JpaTransactionManager`), mặc định **không hỗ trợ**, cần cấu hình thêm và nên test kỹ — vì cơ chế flush/dirty-checking của Hibernate không phải lúc nào cũng khớp hoàn hảo với Savepoint của JDBC thuần.

---

## VII. REQUIRES_NEW vs NESTED — So Sánh Nhanh

| Tiêu chí | REQUIRES_NEW | NESTED |
| :--- | :--- | :--- |
| Số Connection cần dùng | **2** (cha bị treo + con mới) | **1** (dùng chung) |
| Cơ chế | Physical Transaction **hoàn toàn độc lập** | **Savepoint** trong cùng Physical Transaction |
| Con lỗi → ảnh hưởng cha? | Không (2 giao dịch tách biệt hoàn toàn) | Không (chỉ rollback tới Savepoint) |
| Cha lỗi/rollback toàn bộ → ảnh hưởng con? | Không (con đã commit độc lập trước đó rồi) | **Có** — con bị cuốn theo rollback (chỉ có 1 lần commit/rollback thật ở connection) |
| Rủi ro Connection Pool | **Cao** (Mục V) | **Không có** (không xin thêm Connection) |
| Yêu cầu hạ tầng | Hoạt động ở mọi Transaction Manager | Cần Driver/TX Manager hỗ trợ Savepoint |

**Ghi nhớ nhanh**: cần tác vụ con **"sống độc lập tuyệt đối"** dù cha có rollback → `REQUIRES_NEW`. Chỉ cần **"cách ly lỗi cục bộ"** mà không quan tâm việc tốn thêm Connection → `NESTED` là lựa chọn nhẹ nhàng và an toàn hơn cho Connection Pool.

---

## VIII. SUPPORTS / NOT_SUPPORTED / MANDATORY / NEVER — Khi Nào Dùng

Nhóm này ít gặp hơn nhưng vẫn hay bị hỏi để kiểm tra độ hiểu sâu:

* **SUPPORTS** — dùng cho các method "tiện ích", linh hoạt: có transaction thì ăn theo, không có cũng chạy được bình thường (không transaction). Hợp cho các hàm đọc dữ liệu dùng chung ở nhiều nơi, không đòi hỏi ràng buộc gì.
* **NOT_SUPPORTED** — dùng cho tác vụ **nặng, không cần transaction**, và **không muốn nó chiếm dụng transaction đang mở**. Ví dụ kinh điển: chạy 1 câu query report/thống kê nặng mất hàng chục giây — nếu để nó chạy chung trong transaction chính, nó sẽ giữ Connection **và** giữ Transaction ID tồn tại lâu dài (nhớ lại [Chương 2](2_ISOLATION_LEVEL_AND_MVCC.md): Transaction càng sống lâu, Undo Log/History List của MySQL càng phình, hoặc Autovacuum của PostgreSQL càng khó dọn dead tuple). `NOT_SUPPORTED` giúp tách hẳn tác vụ report ra khỏi vòng đời transaction chính.
* **MANDATORY** — dùng như một "lời khẳng định kiến trúc": method này **chỉ được phép gọi từ bên trong** một transaction đã có sẵn. Nếu ai gọi trực tiếp mà quên bọc `@Transactional`, hệ thống sẽ báo lỗi ngay (`IllegalTransactionStateException`) thay vì âm thầm chạy sai và gây bug khó phát hiện.
* **NEVER** — hiếm dùng, đảm bảo tuyệt đối method **không bao giờ** được chạy trong bất kỳ transaction nào (có transaction là ném lỗi ngay). Hợp cho các method có tác dụng phụ không nên bị "cuốn" theo vòng đời transaction cha — ví dụ gọi API bên ngoài, gửi email — lỡ transaction cha rollback thì email cũng không thể "rollback" theo được.

---

## IX. Bảng Tổng Hợp 7 Propagation (Cheat Sheet 1 Trang)

| Propagation | Có TX cha → | Không có TX cha → | Tốn Connection thêm? | Con lỗi ảnh hưởng cha? |
| :--- | :--- | :--- | :---: | :---: |
| **REQUIRED** | Dùng chung | Tạo mới | Không | **Có** (rollback-only lan toàn bộ) |
| **REQUIRES_NEW** | Treo cha, tạo mới | Tạo mới | **Có** (2 Connection) | Không |
| **NESTED** | Savepoint trong TX cha | Tạo mới | Không | Không (chỉ rollback tới savepoint) |
| **SUPPORTS** | Dùng chung | Chạy không TX | Không | Tùy trường hợp |
| **NOT_SUPPORTED** | Treo cha, chạy không TX | Chạy không TX | Không | Không |
| **MANDATORY** | Dùng chung | ❌ Exception | Không | **Có** |
| **NEVER** | ❌ Exception | Chạy không TX | Không | — |

---

## X. Cheat Sheet Phỏng Vấn (Q&A)

### Q1: Phân biệt Physical Transaction và Logical Transaction.
> **Trả lời**: Physical Transaction là giao dịch thật ở tầng Database — gắn với đúng 1 Connection, đúng 1 lần COMMIT/ROLLBACK thật sự. Logical Transaction là ranh giới `@Transactional` mà Spring quản lý ở tầng code — nhiều Logical Transaction (nhiều method `@Transactional` gọi lồng nhau) có thể cùng dùng chung 1 Physical Transaction, tùy vào Propagation được cấu hình.

### Q2: Vì sao dùng `try-catch` nuốt Exception vẫn không cứu được dữ liệu khi 2 method cùng `REQUIRED`?
> **Trả lời**: Vì cả 2 method dùng chung 1 Physical Transaction. Khi method con lỗi, nó không tự rollback Connection (vì không có quyền, do chỉ là participant) mà chỉ đánh dấu cờ `rollbackOnly = true` lên Physical Transaction chung. Dù method cha `catch` và nuốt Exception, khi cố `COMMIT`, Spring phát hiện cờ này và bắt buộc rollback toàn bộ, ném `UnexpectedRollbackException` — vì không có khái niệm "commit một phần" trên cùng 1 Connection.

### Q3: `REQUIRES_NEW` hoạt động như thế nào, và vì sao được gọi là "sát thủ Connection Pool"?
> **Trả lời**: `REQUIRES_NEW` sẽ "treo" (suspend) Physical Transaction hiện tại — tạm cất Connection đang dùng, rồi xin một Connection hoàn toàn mới từ Pool để chạy độc lập, commit/rollback riêng. Sau khi xong, Connection mới được trả về Pool và Connection cũ được "đánh thức" lại. Vấn đề là trong lúc đó, 1 Thread giữ đồng thời 2 Connection — nếu số Request đồng thời chạm đúng `maximum-pool-size`, tất cả Thread sẽ cùng bị kẹt chờ xin Connection thứ 2 mà không Thread nào nhả Connection thứ nhất ra, gây cạn kiệt Pool (Connection Pool Starvation/Deadlock).

### Q4: Trình bày kịch bản Deadlock kinh điển khi dùng `REQUIRES_NEW` dưới tải cao.
> **Trả lời**: Nếu số Request đồng thời bằng đúng `maximum-pool-size`, mỗi Request giữ 1 Connection ở method cha (`REQUIRED`) rồi đứng chờ xin thêm Connection cho method con (`REQUIRES_NEW`). Vì Pool đã hết Connection, không Request nào xin được Connection thứ 2, và cũng không Request nào commit được để nhả Connection thứ nhất ra (vì đang bị block giữa chừng) → toàn bộ hệ thống bị treo cho đến khi hết `connectionTimeout`, ném `SQLTransientConnectionException`.

### Q5: `NESTED` khác `REQUIRES_NEW` ở điểm nào? Khi nào nên chọn `NESTED`?
> **Trả lời**: `REQUIRES_NEW` tạo một Physical Transaction hoàn toàn độc lập, cần thêm 1 Connection riêng. `NESTED` vẫn dùng chung đúng 1 Connection/Physical Transaction với method cha, chỉ tạo một JDBC Savepoint — nếu method con lỗi, chỉ rollback về đúng Savepoint đó mà không ảnh hưởng phần cha đã làm. Nên chọn `NESTED` khi chỉ cần cách ly lỗi cục bộ mà không muốn tốn thêm Connection (an toàn hơn cho Connection Pool), và chấp nhận việc nếu cha rollback toàn bộ thì phần con cũng bị cuốn theo (khác với `REQUIRES_NEW` — con đã commit độc lập thì không bị ảnh hưởng dù cha rollback sau đó).

### Q6: `MANDATORY` và `NEVER` khác nhau ở điều kiện nào?
> **Trả lời**: `MANDATORY` yêu cầu **bắt buộc phải có** Transaction cha đang chạy, nếu không sẽ ném Exception — dùng để đảm bảo method chỉ được gọi từ trong ngữ cảnh có transaction. `NEVER` thì ngược lại hoàn toàn: **cấm tuyệt đối** việc chạy trong bất kỳ Transaction nào, nếu phát hiện có Transaction cha đang hoạt động sẽ ném Exception ngay lập tức.

### Q7: Tại sao nên dùng `NOT_SUPPORTED` cho các câu query report/thống kê nặng?
> **Trả lời**: Nếu chạy trong transaction đang mở, câu query nặng sẽ giữ Connection và giữ Transaction ID tồn tại lâu — kéo theo hệ quả ở tầng MVCC: MySQL không purge được Undo Log cũ (History List Length tăng), PostgreSQL khó Vacuum dead tuple (tăng nguy cơ Table Bloat). `NOT_SUPPORTED` tách hẳn câu query đó ra khỏi vòng đời transaction chính, tránh giữ Transaction ID dài hạn không cần thiết.

---

> **Tài liệu tham khảo:**
> - [Quản lý Transaction với Spring và JPA](https://viblo.asia/p/quan-ly-transaction-voi-spring-va-jpa-2oKLnxz14QO)
> - [HikariCP Wiki — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
