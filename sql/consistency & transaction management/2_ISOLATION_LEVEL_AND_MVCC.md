# Chương 2: Isolation Levels & Kiến trúc MVCC (MySQL vs PostgreSQL)

![Status](https://img.shields.io/badge/MVCC-green) ![Topic](https://img.shields.io/badge/Isolation_Levels-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
  - [1. Bài toán Concurrency trong Database](#1-bài-toán-concurrency-trong-database)
  - [2. Ba Hiện Tượng Dị Thường (Read Phenomena)](#2-ba-hiện-tượng-dị-thường-read-phenomena)
- [II. Bốn Mức Isolation Level Chuẩn SQL-92](#ii-bốn-mức-isolation-level-chuẩn-sql-92)
  - [1. Bảng định nghĩa \& mức mặc định của từng DBMS](#1-bảng-định-nghĩa--mức-mặc-định-của-từng-dbms)
  - [2. Repeatable Read của MySQL "mạnh" hơn chuẩn SQL-92](#2-repeatable-read-của-mysql-mạnh-hơn-chuẩn-sql-92)
- [III. MVCC - Multi-Version Concurrency Control](#iii-mvcc---multi-version-concurrency-control)
  - [1. Định nghĩa \& Nguyên lý "Readers Don't Block Writers"](#1-định-nghĩa--nguyên-lý-readers-dont-block-writers)
  - [2. So sánh Throughput: Locking (2PL) vs MVCC](#2-so-sánh-throughput-locking-2pl-vs-mvcc)
- [IV. Kiến trúc lưu trữ phiên bản: MySQL InnoDB](#iv-kiến-trúc-lưu-trữ-phiên-bản-mysql-innodb)
  - [1. Cột ẩn hệ thống (DB\_TRX\_ID, DB\_ROLL\_PTR, DB\_ROW\_ID)](#1-cột-ẩn-hệ-thống-db_trx_id-db_roll_ptr-db_row_id)
  - [2. Undo Log \& Version Chain](#2-undo-log--version-chain)
  - [3. Read View \& Thuật toán xác định Visibility](#3-read-view--thuật-toán-xác-định-visibility)
  - [4. Khác biệt Read View giữa Read Committed và Repeatable Read](#4-khác-biệt-read-view-giữa-read-committed-và-repeatable-read)
- [V. Kiến trúc lưu trữ phiên bản: PostgreSQL](#v-kiến-trúc-lưu-trữ-phiên-bản-postgresql)
  - [1. Cột ẩn hệ thống (xmin, xmax, ctid)](#1-cột-ẩn-hệ-thống-xmin-xmax-ctid)
  - [2. Heap Tuple \& Cơ chế UPDATE = INSERT + set xmax](#2-heap-tuple--cơ-chế-update--insert--set-xmax)
  - [3. Snapshot \& Visibility Rules](#3-snapshot--visibility-rules)
- [VI. Dọn dẹp rác (Garbage Collection)](#vi-dọn-dẹp-rác-garbage-collection)
  - [1. MySQL: Purge Thread \& History List Length](#1-mysql-purge-thread--history-list-length)
  - [2. PostgreSQL: VACUUM, Autovacuum \& Bloat](#2-postgresql-vacuum-autovacuum--bloat)
  - [3. Transaction ID Wraparound - "Quả bom hẹn giờ" của Postgres](#3-transaction-id-wraparound---quả-bom-hẹn-giờ-của-postgres)
- [VII. Ngăn chặn Phantom Read: Next-Key Lock vs Snapshot Isolation](#vii-ngăn-chặn-phantom-read-next-key-lock-vs-snapshot-isolation)
  - [1. MySQL: Record Lock + Gap Lock = Next-Key Lock](#1-mysql-record-lock--gap-lock--next-key-lock)
  - [2. Lưu ý: Next-Key Lock KHÔNG áp dụng cho Plain SELECT](#2-lưu-ý-next-key-lock-không-áp-dụng-cho-plain-select)
  - [3. PostgreSQL: Snapshot Isolation \& giới hạn Write Skew](#3-postgresql-snapshot-isolation--giới-hạn-write-skew)
  - [4. Serializable Snapshot Isolation (SSI) trong PostgreSQL](#4-serializable-snapshot-isolation-ssi-trong-postgresql)
- [VIII. Bảng So Sánh Tổng Hợp MySQL vs PostgreSQL](#viii-bảng-so-sánh-tổng-hợp-mysql-vs-postgresql)
- [IX. Trade-off: Chọn Isolation Level nào cho nghiệp vụ nào?](#ix-trade-off-chọn-isolation-level-nào-cho-nghiệp-vụ-nào)
- [X. Cheat Sheet Phỏng vấn (Interview Q\&A)](#x-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

### 1. Bài toán Concurrency trong Database

Một Database Engine tại cùng một thời điểm phải phục vụ hàng trăm, hàng nghìn Transaction chạy song song, cùng đọc/ghi lên chung một tập bản ghi vật lý. Nếu để các Transaction "dẫm chân" lên nhau một cách tự do, dữ liệu trả về cho Client sẽ mất tính toàn vẹn.

**Isolation** (chữ **I** trong ACID) chính là cam kết: *"Kết quả của các Transaction chạy song song phải giống như thể chúng chạy tuần tự, lần lượt từng cái một"*. Nhưng chạy tuần tự thật thì Throughput sẽ tụt thảm hại. Do đó, các DB Engine đưa ra 4 **Isolation Level** như 4 mức "nới lỏng" cam kết đó, đánh đổi giữa **Consistency** và **Performance**.

---

### 2. Ba Hiện Tượng Dị Thường (Read Phenomena)

Đây là 3 "kẻ thù" mà Isolation Level sinh ra để đối phó — mỗi khi nhiều Transaction cùng đọc/ghi chung một dữ liệu song song:

* **Dirty Read (Đọc bẩn)**: Transaction A đọc phải dữ liệu mà Transaction B vừa thay đổi nhưng **chưa Commit**. Nếu B sau đó `ROLLBACK`, coi như A vừa dựa vào một "sự thật ảo" chưa từng tồn tại.

  | Thời điểm | Transaction A | Transaction B |
  | :---: | :--- | :--- |
  | T1 | | `BEGIN;`<br>`UPDATE accounts SET balance = balance - 500000 WHERE id = 1;` (chưa commit) |
  | T2 | `SELECT balance FROM accounts WHERE id = 1;` → thấy số dư **đã bị trừ** | |
  | T3 | | `ROLLBACK;` (giao dịch bị huỷ, số dư quay lại như cũ) |

  *Hậu quả*: hệ thống thanh toán chấp nhận một giao dịch dựa trên số dư ví **chưa bao giờ thực sự tồn tại**.

* **Non-repeatable Read (Đọc không lặp lại)**: Trong **cùng một Transaction**, đọc một dòng dữ liệu 2 lần nhưng ra 2 kết quả khác nhau, vì một Transaction khác đã `UPDATE` và `COMMIT` xen giữa.

  | Thời điểm | Transaction A | Transaction B |
  | :---: | :--- | :--- |
  | T1 | `BEGIN;`<br>`SELECT balance FROM accounts WHERE id = 1;` → `1.000.000` | |
  | T2 | | `UPDATE accounts SET balance = 1.500.000 WHERE id = 1;`<br>`COMMIT;` |
  | T3 | `SELECT balance FROM accounts WHERE id = 1;` → `1.500.000` (khác lần đọc đầu, dù A **chưa hề Commit**) | |

  *Hậu quả*: một tác vụ kiểm toán (audit) hoặc xuất báo cáo tài chính bị sai lệch số liệu vì dữ liệu thay đổi ngay trong lúc đang đọc.

* **Phantom Read (Đọc bóng ma)**: Chạy lại **cùng một câu truy vấn thống kê** (COUNT/SUM theo điều kiện) 2 lần trong cùng Transaction, nhưng số dòng trả về thay đổi vì một Transaction khác vừa `INSERT` hoặc `DELETE`.

  | Thời điểm | Transaction A | Transaction B |
  | :---: | :--- | :--- |
  | T1 | `BEGIN;`<br>`SELECT COUNT(*) FROM orders WHERE status = 'PENDING';` → `10` | |
  | T2 | | `INSERT INTO orders (status) VALUES ('PENDING');`<br>`COMMIT;` |
  | T3 | `SELECT COUNT(*) FROM orders WHERE status = 'PENDING';` → `11` (xuất hiện thêm 1 dòng "bóng ma") | |

  *Hậu quả*: bạn vừa xác nhận kho còn đúng 1 sản phẩm cuối cùng để bán, nhưng thực tế một Transaction khác đã kịp thêm/xoá dòng đó, khiến logic xử lý sau đó bị gãy.

> **Điểm mấu chốt**: Dirty Read & Non-repeatable Read xảy ra trên **cùng một dòng dữ liệu đã đọc** bị sửa đổi. Phantom Read xảy ra do có **dòng hoàn toàn mới xuất hiện/biến mất** nằm ngoài những dòng đã đọc trước đó — đây là lý do kỹ thuật chống 2 hiện tượng đầu (khoá/versioning theo từng dòng) khác hẳn kỹ thuật chống Phantom Read (khoá theo khoảng/range hoặc snapshot toàn cục), sẽ phân tích kỹ ở Mục VII.

---

## II. Bốn Mức Isolation Level Chuẩn SQL-92

### 1. Bảng định nghĩa & mức mặc định của từng DBMS

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Mặc định |
| :--- | :---: | :---: | :---: | :--- |
| **Read Uncommitted** | Có thể bị | Có thể bị | Có thể bị | — |
| **Read Committed** | Ngăn chặn | Có thể bị | Có thể bị | **PostgreSQL**, Oracle, SQL Server |
| **Repeatable Read** | Ngăn chặn | Ngăn chặn | *Tùy Engine* | **MySQL (InnoDB)** |
| **Serializable** | Ngăn chặn | Ngăn chặn | Ngăn chặn | — (chi phí rất cao) |

* **Read Uncommitted**: nhanh nhất, chấp nhận đọc bẩn — dùng cho analytics/dashboard không cần tuyệt đối chính xác.
* **Read Committed**: chỉ đọc dữ liệu đã commit — điểm cân bằng cho phần lớn ứng dụng Web CRUD thông thường.
* **Repeatable Read**: dữ liệu đã đọc trong Transaction không đổi trong suốt vòng đời Transaction đó.
* **Serializable**: giả lập thực thi tuần tự — loại bỏ toàn bộ anomaly, đổi lại Throughput giảm mạnh vì lock/conflict-check tăng vọt.

### 2. Repeatable Read của MySQL "mạnh" hơn chuẩn SQL-92

Theo đúng chuẩn SQL-92, **Repeatable Read** *vẫn có thể* dính **Phantom Read** — chỉ có `Serializable` mới đảm bảo chặn hoàn toàn. Tuy nhiên, MySQL InnoDB triển khai `REPEATABLE READ` mạnh hơn lý thuyết:
* Với **truy vấn đọc thuần túy (plain SELECT, consistent non-locking read)**: Phantom Read gần như được loại bỏ nhờ **Snapshot/Read View** (xem Mục IV.3) — giống cơ chế Snapshot Isolation của PostgreSQL.
* Với **truy vấn có khóa (`SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`)**: InnoDB dùng thêm **Next-Key Lock** (Mục VII) để khóa cả khoảng trống, chặn INSERT xen vào.

Đây chính là lý do MySQL chọn `REPEATABLE READ` làm mặc định thay vì `READ COMMITTED` như đa số RDBMS khác — độ an toàn gần tương đương `SERIALIZABLE` trong phần lớn tình huống thực tế, nhưng chi phí thấp hơn nhiều.

---

## III. MVCC - Multi-Version Concurrency Control

### 1. Định nghĩa & Nguyên lý "Readers Don't Block Writers"

**MVCC (Multi-Version Concurrency Control)** là kỹ thuật cô lập Transaction **không dựa vào Lock** để giải quyết xung đột đọc/ghi, mà dựa vào việc **lưu giữ nhiều phiên bản (version)** của cùng một dòng dữ liệu. Mỗi Transaction, thay vì thấy "dữ liệu hiện tại duy nhất", sẽ thấy **một phiên bản dữ liệu phù hợp với thời điểm nó bắt đầu (snapshot)**.

Nguyên lý cốt lõi: **"Người đọc không chặn người ghi, người ghi không chặn người đọc"**.
* Khi Transaction Ghi (Writer) đang sửa một Row, nó tạo ra một **phiên bản mới**, không xóa phiên bản cũ ngay lập tức.
* Transaction Đọc (Reader) khác vẫn có thể đọc **phiên bản cũ** (chưa bị ảnh hưởng bởi Writer) mà không cần chờ Writer commit/rollback, và không cần xin Lock nào cả.
* Ngược lại, Writer cũng không cần chờ Reader "buông" Lock đọc, vì Reader chưa bao giờ khóa dòng dữ liệu đó — Reader chỉ đọc bản snapshot của riêng mình.

```
Locking (2PL) truyền thống                MVCC
─────────────────────────                 ─────────────────────────
Writer đang UPDATE row X                  Writer đang UPDATE row X
        │ (giữ Exclusive Lock)                    │ (tạo version mới, version cũ giữ nguyên)
        ▼                                          ▼
Reader muốn SELECT row X                  Reader muốn SELECT row X
        │                                          │
   ⛔ BLOCKED – phải xếp hàng chờ            ✅ Đọc ngay version cũ (snapshot)
   Writer commit/rollback xong                     không cần chờ, không cần Lock
```

### 2. So sánh Throughput: Locking (2PL) vs MVCC

Với cơ chế **Locking thuần túy (Two-Phase Locking)**, mọi Transaction muốn đọc một Row đang bị Transaction khác ghi phải **xếp hàng (queue) chờ Lock được giải phóng**. Trong hệ thống có tỷ lệ đọc/ghi lệch nhiều về phía đọc (Read-heavy, tỷ lệ phổ biến trong Web app là 80/20 hoặc 90/10), việc để hàng loạt Reader phải chờ một vài Writer sẽ tạo ra **Lock Contention** — hàng đợi (queue) phình to, Latency tăng vọt, CPU tốn nhiều để quản lý Lock Manager (Lock table, deadlock detection).

MVCC loại bỏ hoàn toàn hàng đợi đó cho luồng đọc: Reader **luôn có một phiên bản để đọc ngay lập tức**, không phải chờ ai. Kết quả:
* **Reader không còn là "khách hàng phải xếp hàng"** — hàng nghìn Read Transaction chạy song song mà không tranh chấp với Write Transaction.
* **Writer chỉ còn tranh chấp với Writer khác** trên cùng 1 dòng (write-write conflict) — vốn dĩ là tập giao nhỏ hơn nhiều so với read-write conflict.
* Vì vậy trong workload Read-heavy điển hình, Throughput hệ thống dùng MVCC có thể cao hơn **gấp nhiều lần** so với hệ thống Lock 2PL thuần túy dưới cùng mức tải, vì phần lớn Traffic (các câu SELECT) hoàn toàn không phải chờ đợi.
* **Cái giá phải trả** không biến mất, nó chỉ **dịch chuyển**: từ "chi phí chờ Lock lúc runtime" sang "chi phí lưu trữ & dọn dẹp version cũ" (xem Mục VI) — đây chính là điều Mục V và VI phân tích sâu.

---

## IV. Kiến trúc lưu trữ phiên bản: MySQL InnoDB

### 1. Cột ẩn hệ thống (DB_TRX_ID, DB_ROLL_PTR, DB_ROW_ID)

Mỗi Row trong InnoDB (Clustered Index B+Tree, sắp xếp theo Primary Key) mang theo 3 cột ẩn không hiển thị khi `SELECT *`:

| Cột ẩn | Kích thước | Vai trò |
| :--- | :---: | :--- |
| `DB_TRX_ID` | 6 byte | ID của Transaction **gần nhất** đã INSERT/UPDATE row này. |
| `DB_ROLL_PTR` | 7 byte | Con trỏ (Rollback Pointer) trỏ tới bản ghi **Undo Log** ngay trước đó — hình thành một **Version Chain**. |
| `DB_ROW_ID` | 6 byte | Chỉ tồn tại nếu bảng **không có Primary Key/Unique Key** nào — InnoDB tự sinh để làm Clustered Index. |

### 2. Undo Log & Version Chain

Điểm khác biệt cốt lõi so với PostgreSQL: **bản ghi chính (trong bảng/clustered index) luôn luôn là phiên bản MỚI NHẤT**. Các phiên bản CŨ không nằm trong bảng, mà bị "đá" sang một vùng lưu trữ riêng gọi là **Undo Log** (nằm trong Rollback Segment, thuộc System Tablespace hoặc Undo Tablespace riêng).

```
                     Bảng chính (Clustered Index) — luôn là bản MỚI NHẤT
┌──────────────────────────────────────────────────────────────────────┐
│ id=5 | balance=300 | DB_TRX_ID=105 | DB_ROLL_PTR ──────────────┐     │
└────────────────────────────────────────────────────────────────┼─────┘
                                                                 │
                                                                 ▼
                                            ┌── Undo Log Record (trx=102, balance=200) ──┐
                                            │  ROLL_PTR ─────────────────────────┐       │
                                            └────────────────────────────────────┼───────┘
                                                                                 ▼
                                            ┌── Undo Log Record (trx=98,  balance=100) ──┐
                                            │  ROLL_PTR = NULL (bản gốc, hết chuỗi)      │
                                            └────────────────────────────────────────────┘

     "Version Chain": 300 (hiện tại) ──► 200 (cũ hơn) ──► 100 (cũ nhất)
```

* Mỗi lần `UPDATE`, InnoDB: (1) ghi giá trị **cũ** vào một bản ghi Undo Log mới, (2) sửa **tại chỗ** giá trị trong bảng chính thành giá trị **mới**, (3) cập nhật `DB_ROLL_PTR` của row trỏ sang bản ghi Undo Log vừa tạo.
* Muốn dựng lại một phiên bản cũ hơn (để phục vụ Reader đang có snapshot cũ, hoặc để `ROLLBACK`), InnoDB chỉ cần **đi ngược theo chuỗi con trỏ `ROLL_PTR`** trong Undo Log, áp ngược từng thay đổi (giống "patch ngược").
* Có 2 loại Undo Log: **Insert Undo Log** (chỉ cần cho Rollback, được xoá ngay sau khi Transaction commit vì không ai khác cần đọc bản ghi trước-khi-insert) và **Update Undo Log** (cần giữ lại lâu hơn để phục vụ MVCC cho các Transaction khác đang đọc — chỉ bị xoá khi **Purge Thread** xác nhận không còn Read View nào cần tới).

### 3. Read View & Thuật toán xác định Visibility

**Read View** là một "ảnh chụp" (snapshot) trạng thái các Transaction đang hoạt động tại một thời điểm, dùng để quyết định phiên bản nào trong Version Chain là "nhìn thấy được" đối với Transaction hiện tại. Cấu trúc rút gọn:

```java
class ReadView {
    long[] activeTrxIds;     // m_ids: danh sách trx đang active (chưa commit) khi View được tạo
    long   lowLimitId;       // trx_id kế tiếp sẽ được cấp phát (mọi trx >= giá trị này => "sinh sau" View)
    long   upLimitId;        // = min(activeTrxIds), mọi trx < giá trị này => chắc chắn đã commit trước View
    long   creatorTrxId;     // id của chính Transaction tạo ra View này
}
```

Thuật toán xác định 1 phiên bản trong Version Chain có **visible** với Read View hay không, xét theo `DB_TRX_ID` của phiên bản đó:

1. Nếu `DB_TRX_ID == creatorTrxId` → **Visible** (đọc chính thay đổi của bản thân Transaction).
2. Nếu `DB_TRX_ID < upLimitId` → **Visible** (đã commit từ trước khi View được tạo).
3. Nếu `DB_TRX_ID >= lowLimitId` → **Not Visible** (Transaction này sinh ra *sau* thời điểm View được tạo) → nhảy xuống phiên bản cũ hơn trong Version Chain (`ROLL_PTR`), lặp lại bước 1.
4. Nếu `upLimitId <= DB_TRX_ID < lowLimitId`: kiểm tra `DB_TRX_ID` có nằm trong `activeTrxIds` không:
   - Có → **Not Visible** (Transaction đó vẫn đang chạy dở/chưa commit tại thời điểm View tạo) → lùi tiếp Version Chain.
   - Không → **Visible** (đã commit trong khoảng thời gian đó).

Kết quả: mỗi Transaction "đi ngược" theo Version Chain cho đến khi tìm được phiên bản đầu tiên thoả điều kiện Visible — đó chính là dữ liệu nó sẽ thấy, hoàn toàn không cần khóa row.

### 4. Khác biệt Read View giữa Read Committed và Repeatable Read

| | Read Committed | Repeatable Read |
| :--- | :--- | :--- |
| **Thời điểm tạo Read View** | **Tạo mới cho MỖI câu SELECT** | **Tạo MỘT LẦN duy nhất** ở câu SELECT đầu tiên của Transaction |
| **Hệ quả** | Đọc 2 lần trong cùng Transaction có thể thấy 2 kết quả khác nhau (Non-repeatable Read) vì mỗi lần đều "chụp ảnh" trạng thái mới nhất | Đọc bao nhiêu lần trong Transaction cũng ra cùng 1 kết quả, vì luôn dùng chung 1 Read View "đóng băng" từ đầu |

---

## V. Kiến trúc lưu trữ phiên bản: PostgreSQL

### 1. Cột ẩn hệ thống (xmin, xmax, ctid)

PostgreSQL **không có Undo Log riêng biệt**. Thay vào đó, mọi phiên bản (cũ và mới) của một dòng logic đều là các **tuple vật lý độc lập nằm chung trong Heap** (bảng dữ liệu chính). Mỗi tuple mang theo các cột hệ thống ẩn:

| Cột ẩn | Vai trò |
| :--- | :--- |
| `xmin` | Transaction ID **đã tạo ra** tuple này (do INSERT hoặc do UPDATE sinh bản mới). |
| `xmax` | Transaction ID **đã "xoá"/thay thế** tuple này (do DELETE hoặc do UPDATE tạo bản mới hơn). Bằng `0`/`null` nếu tuple vẫn còn hiệu lực. |
| `ctid` | Vị trí vật lý (page, offset) của tuple — dùng để trỏ sang phiên bản kế tiếp trong chuỗi update (HOT chain). |

### 2. Heap Tuple & Cơ chế UPDATE = INSERT + set xmax

```
Heap Page (8KB) — nhiều PHIÊN BẢN của CÙNG 1 logical row nằm CHUNG 1 bảng

┌────────────────────────────────────────────────────────────────────┐
│ Tuple #1: id=5, balance=100   xmin=98   xmax=102   (đã bị thay thế)│──┐
│ Tuple #2: id=5, balance=200   xmin=102  xmax=105   (đã bị thay thế)│  │ ctid trỏ
│ Tuple #3: id=5, balance=300   xmin=105  xmax=NULL  (BẢN HIỆN TẠI)  │◄─┘ tới nhau
└────────────────────────────────────────────────────────────────────┘
        ▲ "dead tuple" – rác, chờ VACUUM         "dead tuple" – rác
```

Khi chạy `UPDATE`, PostgreSQL **không sửa tại chỗ**. Nó thực hiện 2 bước nguyên tử:
1. **INSERT** một tuple hoàn toàn mới với giá trị mới, gán `xmin = transaction hiện tại`.
2. Set `xmax = transaction hiện tại` trên tuple **cũ** để đánh dấu "đã hết hiệu lực kể từ đây" — tuple cũ **vẫn nằm nguyên trong bảng**, không bị xoá vật lý ngay.

Đây chính là lý do bảng PostgreSQL có thể "phình to" nhanh chóng dưới tải UPDATE cao: mỗi UPDATE đều để lại một "dead tuple" ngay trong chính trang dữ liệu, khác hẳn InnoDB (nơi bảng chính luôn giữ đúng 1 bản mới nhất, phần "rác" bị đẩy hẳn sang vùng Undo Log riêng).

### 3. Snapshot & Visibility Rules

Khi một Transaction bắt đầu (hoặc mỗi câu lệnh, tùy Isolation Level), PostgreSQL chụp một **Snapshot** gồm: `xmin` (trx nhỏ nhất còn active), `xmax` (trx kế tiếp sẽ cấp phát), và danh sách các `xip` (trx đang in-progress tại thời điểm chụp). Một tuple được coi là **visible** với Snapshot nếu — tương tự về bản chất với thuật toán Read View của InnoDB ở Mục IV.3:
* `xmin` của tuple đã **commit** và **nhỏ hơn** `snapshot.xmax`, và không nằm trong danh sách `xip` (tức là đã hoàn tất trước khi Snapshot được chụp), **và**
* `xmax` của tuple là **rỗng**, hoặc **chưa commit**, hoặc **lớn hơn/nằm trong `xip`** của Snapshot (tức là "sự xoá" đó xảy ra sau/đồng thời, Snapshot chưa nên thấy nó).

Nhờ vậy, nhiều Transaction với Snapshot khác nhau có thể cùng lúc nhìn thấy các phiên bản khác nhau của cùng một logical row, hoàn toàn không cần Lock để đọc.

---

## VI. Dọn dẹp rác (Garbage Collection)

### 1. MySQL: Purge Thread & History List Length

Vì "rác" (các bản ghi cũ) của InnoDB nằm tách biệt trong Undo Log, **bảng chính không hề bị phình to** dù có bao nhiêu UPDATE xảy ra. Việc dọn dẹp được thực hiện **ngầm, tự động** bởi một hoặc nhiều **Purge Thread** (cấu hình qua `innodb_purge_threads`):
* Purge Thread định kỳ xác định **Read View cũ nhất đang tồn tại** trong toàn hệ thống (tức Transaction đang chạy dở lâu nhất).
* Bất kỳ bản ghi Undo Log nào có `trx_id` nhỏ hơn ngưỡng đó (không còn Read View nào cần tham chiếu tới nữa) sẽ được **xoá vật lý** khỏi Rollback Segment, đồng thời purge luôn các row đã bị `DELETE` (delete-marked) trong bảng chính.
* Chỉ số `History List Length` (xem qua `SHOW ENGINE INNODB STATUS`) đo lượng Undo Log records đang chờ Purge — nếu chỉ số này tăng liên tục không giảm, dấu hiệu cho thấy có **Long-running Transaction** đang giữ Read View cũ, chặn Purge Thread dọn dẹp, khiến Undo Tablespace phình to.

### 2. PostgreSQL: VACUUM, Autovacuum & Bloat

PostgreSQL **không có tiến trình dọn dẹp hoàn toàn tự động, ngầm ẩn** như Purge Thread — nó **bắt buộc** phải chạy lệnh/tiến trình `VACUUM`:
* `VACUUM` quét Heap, tìm các "dead tuple" (tuple có `xmax` đã commit và không còn Snapshot nào tham chiếu tới), đánh dấu vùng nhớ đó là **có thể tái sử dụng** (ghi vào Free Space Map — FSM) để các INSERT/UPDATE tiếp theo tái dùng chỗ trống **trong cùng file**, chứ **không trả lại dung lượng cho hệ điều hành** (muốn trả thật sự phải chạy `VACUUM FULL`, khoá bảng và ghi lại toàn bộ file).
* **Autovacuum daemon** chạy nền, tự kích hoạt `VACUUM`/`ANALYZE` khi số dead tuple trong bảng vượt ngưỡng cấu hình (`autovacuum_vacuum_scale_factor`, mặc định thêm 20% row bị sửa/xoá thì kích hoạt).
* **Nếu Autovacuum không chạy kịp** (bị chặn bởi Transaction dài, bị tắt cấu hình, hoặc tải I/O quá lớn), dead tuple tích tụ ngày càng nhiều → hiện tượng **Table Bloat**: file vật lý phình to dù dữ liệu "logic" không đổi, Index phải quét qua nhiều entry rác hơn, `SELECT`/Index Scan chậm dần dù **không hề có tranh chấp Lock nào** — đây chính là "hidden cost" MVCC mà bài viết tham khảo đã cảnh báo.

### 3. Transaction ID Wraparound - "Quả bom hẹn giờ" của Postgres

Một hệ quả kỹ thuật sâu hơn chỉ riêng PostgreSQL gặp phải: `xmin`/`xmax` là số nguyên **32-bit**, không gian ID hữu hạn (~4 tỷ, dùng vòng tròn/modulo). Nếu một hệ thống cực kỳ bận rộn tạo ra hàng tỷ Transaction ID mà **VACUUM không kịp "đóng băng" (freeze)** các tuple cũ (gán một `FrozenXID` đặc biệt để không cần so sánh tuổi nữa), bộ đếm `xid` có thể **quay vòng (wraparound)** — khiến các tuple *cũ* trông như thể được tạo ra *trong tương lai*, dữ liệu "biến mất" khỏi kết quả truy vấn dù vẫn còn trên đĩa. PostgreSQL tự bảo vệ bằng cách **buộc hệ thống vào chế độ chỉ-đọc (read-only)** khi số Transaction ID còn lại xuống dưới một ngưỡng an toàn, buộc admin phải chạy `VACUUM FREEZE` thủ công. Đây là lý do Autovacuum **không phải là tùy chọn**, mà là yêu cầu vận hành bắt buộc đối với PostgreSQL.

---

## VII. Ngăn chặn Phantom Read: Next-Key Lock vs Snapshot Isolation

### 1. MySQL: Record Lock + Gap Lock = Next-Key Lock

**Next-Key Lock** = **Record Lock** (khóa đúng bản ghi index tìm thấy) **+ Gap Lock** (khóa luôn "khoảng trống" ngay trước bản ghi đó trên B+Tree Index), giúp chặn Transaction khác `INSERT` một bản ghi mới lọt vào đúng khoảng đang bị khóa.

```
B+Tree Index (cột `age`)

   ... ──[age=10]──(gap)──[age=20]──(gap)──[age=30]──(gap)──[age=40]──...
                     ▲                        ▲
         Next-Key Lock trên [age=20]:   Gap Lock chặn INSERT
         khóa record 20 + khoảng        age=25, age=28... — mọi giá trị
         trống (10,20]                  rơi vào khoảng (10,20]

Query:  SELECT * FROM users WHERE age = 20 FOR UPDATE;
Transaction khác: INSERT INTO users (age) VALUES (15);  -- ⛔ BỊ CHẶN (rơi vào Gap Lock)
```

Next-Key Lock được InnoDB áp dụng khi Transaction thực hiện **locking read** (`SELECT ... FOR UPDATE`/`FOR SHARE`) hoặc **DML** (`UPDATE`/`DELETE` theo điều kiện range) ở mức `REPEATABLE READ` — mục đích kép: (1) chặn Phantom Row chen vào giữa 2 lần đọc-có-khóa trong cùng Transaction, và (2) đảm bảo tính đúng đắn khi replicate theo `STATEMENT-based binlog` (nếu không khóa gap, một INSERT xen giữa có thể khiến Slave và Master lệch dữ liệu).

### 2. Lưu ý: Next-Key Lock KHÔNG áp dụng cho Plain SELECT

Đây là điểm rất hay bị hiểu nhầm: câu `SELECT` **thuần túy, không khóa** (`SELECT * FROM users WHERE age = 20;`, không có `FOR UPDATE`) ở `REPEATABLE READ` **hoàn toàn không xin bất kỳ Lock nào** — nó dựa 100% vào **Read View/MVCC** (Mục IV.3) để đọc snapshot nhất quán, y hệt tinh thần Snapshot Isolation của PostgreSQL. Next-Key Lock/Gap Lock **chỉ phát sinh khi có ý định GHI hoặc khóa tường minh** trên phạm vi range đó. Nói cách khác: MySQL Repeatable Read chống Phantom Read cho *đọc thuần* bằng **snapshot** (giống Postgres), và chống Phantom Read cho *đọc-để-ghi* bằng **Next-Key Lock** (khác Postgres).

### 3. PostgreSQL: Snapshot Isolation & giới hạn Write Skew

PostgreSQL **không có khái niệm Gap Lock**. Ở mức `REPEATABLE READ`, nó dùng thuần **Snapshot Isolation**: Transaction chụp đúng 1 Snapshot tại câu lệnh đầu tiên, và **mọi thứ xảy ra sau đó — dù là UPDATE, DELETE hay INSERT của Transaction khác — đều vô hình hoàn toàn** với Snapshot đó (dựa theo quy tắc Visibility ở Mục V.3), kể cả khi Transaction hiện tại có chạy `SELECT ... FOR UPDATE` đi nữa (PostgreSQL sẽ tự kiểm tra tuple có bị Transaction khác sửa trước không, và báo lỗi serialization thay vì âm thầm khóa khoảng trống).

Tuy nhiên, Snapshot Isolation **không tương đương 100% với Serializable** về lý thuyết — nó vẫn có thể dính hiện tượng **Write Skew**: 2 Transaction cùng đọc một tập dữ liệu chung (ví dụ tổng số bác sĩ trực ca), cùng dựa vào điều kiện đó để quyết định ghi vào 2 dòng *khác nhau* (2 bác sĩ khác nhau cùng xin nghỉ), khiến ràng buộc nghiệp vụ (phải có ít nhất 1 bác sĩ trực) bị vi phạm dù không Transaction nào ghi đè lên nhau.

### 4. Serializable Snapshot Isolation (SSI) trong PostgreSQL

Để đạt `SERIALIZABLE` thật sự (chặn cả Write Skew), PostgreSQL (từ bản 9.1+) triển khai **SSI — Serializable Snapshot Isolation**: bên cạnh Snapshot thông thường, Engine theo dõi thêm các **cạnh phụ thuộc đọc-ghi (rw-conflict)** giữa các Transaction đang chạy song song. Nếu phát hiện một "chu trình nguy hiểm" (dangerous structure) có thể dẫn tới kết quả không tương đương chạy tuần tự, PostgreSQL sẽ **chủ động abort** một trong các Transaction liên quan, trả về lỗi `could not serialize access due to read/write dependencies among transactions` — buộc Application phải **retry**. Đây là cách Postgres đạt Serializable mà **không cần Lock bi quan (pessimistic lock) trên toàn bộ range**, khác hẳn cách tiếp cận "khóa trước" của MySQL.

---

## VIII. Bảng So Sánh Tổng Hợp MySQL vs PostgreSQL

| Tiêu chí | MySQL (InnoDB) | PostgreSQL |
| :--- | :--- | :--- |
| **Mức Isolation mặc định** | Repeatable Read | Read Committed |
| **Vị trí lưu bản mới nhất** | Ngay trong Clustered Index (B+Tree theo PK) | Ngay trong Heap (lẫn chung với bản cũ) |
| **Vị trí lưu bản CŨ** | Tách riêng — **Undo Log** (Rollback Segment) | **Chung bảng** — tuple cũ vẫn nằm trong Heap |
| **Cột ẩn version** | `DB_TRX_ID`, `DB_ROLL_PTR` (con trỏ tới Undo Log) | `xmin`, `xmax` (trực tiếp trên tuple) |
| **Cơ chế UPDATE vật lý** | Sửa tại chỗ (in-place) + ghi Undo Log | INSERT tuple mới + set `xmax` bản cũ |
| **Dọn rác** | **Tự động, ngầm** — Purge Thread | **Cần chủ động** — (Auto)VACUUM |
| **Hệ quả nếu dọn rác trễ** | Undo Tablespace phình to (bảng chính KHÔNG phình) | **Bảng chính phình to** (Table/Index Bloat) |
| **Rủi ro đặc thù** | History List Length tăng cao do Long Transaction | Transaction ID Wraparound (buộc read-only) |
| **Chặn Phantom Read ở RR** | Snapshot (đọc thuần) + **Next-Key/Gap Lock** (đọc-ghi) | Thuần **Snapshot Isolation**, không Gap Lock |
| **Serializable thật sự** | Locking hoàn toàn (khóa mọi row đọc được) | **SSI** — phát hiện conflict, abort + retry |

---

## IX. Trade-off: Chọn Isolation Level nào cho nghiệp vụ nào?

* **Analytics / Dashboard / Report tổng hợp**: `Read Committed` (hoặc thậm chí `Read Uncommitted` nếu chấp nhận sai số nhỏ) — ưu tiên tốc độ, không cần snapshot cố định xuyên suốt.
* **Ứng dụng Web CRUD thông thường (Social, E-commerce catalog)**: `Read Committed` là đủ — người dùng chấp nhận thấy dữ liệu "hơi trễ" một nhịp, đổi lại Latency thấp và ít Contention.
* **Nghiệp vụ tài chính, Ví điện tử, tính toán tồn kho theo lô**: `Repeatable Read` trở lên — cần đảm bảo các lần đọc trong cùng 1 Transaction nhất quán tuyệt đối để tính toán chính xác.
* **Nghiệp vụ có ràng buộc chéo giữa nhiều dòng/bảng (constraint xuyên dòng, kiểu "luôn phải có ít nhất N bản ghi thỏa điều kiện X")**: cân nhắc `Serializable` (chấp nhận chi phí retry) hoặc bổ sung **Application-level Lock** (`SELECT FOR UPDATE`, Optimistic Lock bằng cột `version`) thay vì chỉ trông cậy vào Isolation Level — vì như Read Phenomena đã chỉ ra, Isolation Level **không giải quyết được bài toán Lost Update ở tầng Application** (đọc-tính toán-ghi qua nhiều round-trip).

---

## X. Cheat Sheet Phỏng vấn (Interview Q&A)

### Q1: Phân biệt Dirty Read, Non-repeatable Read và Phantom Read.
> **Trả lời**: Dirty Read là đọc dữ liệu **chưa commit** của Transaction khác. Non-repeatable Read là đọc **cùng một dòng dữ liệu** 2 lần trong 1 Transaction nhưng kết quả khác nhau do dòng đó bị UPDATE/DELETE và commit giữa chừng. Phantom Read là chạy lại **cùng một truy vấn range** 2 lần nhưng số lượng dòng trả về thay đổi do có Transaction khác INSERT/DELETE dòng mới nằm trong phạm vi điều kiện, nằm ở block/index-entry khác với các block đã đọc trước đó.

### Q2: MVCC là gì? Vì sao MVCC giúp tăng Throughput so với Locking truyền thống?
> **Trả lời**: MVCC (Multi-Version Concurrency Control) là cơ chế lưu giữ nhiều phiên bản của cùng một dòng dữ liệu, cho phép Transaction đọc luôn thấy một snapshot nhất quán mà không cần xin Lock. Nhờ đó Reader không bao giờ bị Writer chặn và ngược lại — loại bỏ hoàn toàn hàng đợi Lock Contention giữa đọc và ghi, vốn là điểm nghẽn lớn nhất trong workload Read-heavy của hệ thống dùng Locking (2PL) thuần túy.

### Q3: Trình bày sự khác biệt kiến trúc lưu trữ MVCC giữa MySQL InnoDB và PostgreSQL.
> **Trả lời**: InnoDB luôn giữ bản ghi **mới nhất** ngay trong bảng chính (Clustered Index), các phiên bản **cũ** bị đẩy sang vùng riêng gọi là **Undo Log**, liên kết bằng con trỏ ẩn `DB_ROLL_PTR`. Ngược lại, PostgreSQL không có Undo Log — mọi phiên bản (cũ lẫn mới) đều là các tuple vật lý độc lập nằm **chung trong Heap** của bảng, phân biệt bằng 2 cột ẩn `xmin` (transaction tạo ra) và `xmax` (transaction xoá/thay thế).

### Q4: Tại sao PostgreSQL cần VACUUM còn MySQL thì không cần thao tác tương tự?
> **Trả lời**: Vì PostgreSQL lưu bản ghi cũ ngay trong bảng chính (dead tuple lẫn với live tuple), nên cần `VACUUM`/`Autovacuum` chủ động quét và đánh dấu lại vùng nhớ có thể tái sử dụng, nếu không bảng sẽ phình to (Table Bloat). MySQL InnoDB dọn rác **ngầm tự động** bằng Purge Thread ngay trong Undo Log tách biệt, nên bảng chính không hề bị ảnh hưởng bởi lượng UPDATE/DELETE.

### Q5: Ở mức Repeatable Read, MySQL và PostgreSQL chặn Phantom Read khác nhau như thế nào?
> **Trả lời**: MySQL dùng **2 lớp**: với truy vấn đọc thuần (plain SELECT) thì dựa vào Read View/snapshot (giống Postgres); còn với truy vấn có khóa hoặc DML thì dùng thêm **Next-Key Lock** (Record Lock + Gap Lock) để khóa cả khoảng trống trên B+Tree Index, chặn INSERT mới chen vào. PostgreSQL không có Gap Lock, chỉ dựa hoàn toàn vào **Snapshot Isolation** — Transaction giữ nguyên 1 snapshot từ đầu, mọi thay đổi của Transaction khác sau thời điểm đó đều vô hình, kể cả INSERT.

### Q6: Snapshot Isolation có tương đương Serializable không? PostgreSQL giải quyết khoảng trống đó bằng cách nào?
> **Trả lời**: Không hoàn toàn — Snapshot Isolation vẫn có thể dính hiện tượng **Write Skew** (2 Transaction cùng đọc 1 tập dữ liệu chung rồi ghi vào 2 dòng khác nhau, vi phạm ràng buộc nghiệp vụ dù không ghi đè trực tiếp lên nhau). PostgreSQL giải quyết bằng **SSI (Serializable Snapshot Isolation)** — theo dõi các cạnh phụ thuộc đọc-ghi giữa các Transaction, phát hiện chu trình nguy hiểm và chủ động abort một Transaction, buộc Application retry, để đảm bảo Serializable thật sự mà không cần khóa bi quan trên toàn bộ range.

### Q7: Read View trong InnoDB khác nhau thế nào giữa Read Committed và Repeatable Read?
> **Trả lời**: Ở `Read Committed`, InnoDB tạo một **Read View mới cho mỗi câu SELECT**, nên mỗi lần đọc đều thấy trạng thái mới nhất đã commit — dẫn tới Non-repeatable Read. Ở `Repeatable Read`, Read View chỉ được tạo **một lần duy nhất** tại câu SELECT đầu tiên của Transaction và dùng lại cho toàn bộ Transaction đó, nên mọi lần đọc sau đều cho cùng một kết quả nhất quán.

---

> **Tài liệu tham khảo:**
> - [Isolation Levels: 4 Cấp Độ Giúp Database Vừa Nhanh, Vừa Chính Xác - Database System Design P10](https://viblo.asia/p/isolation-levels-4-cap-do-giup-database-vua-nhanh-vua-chinh-xac-database-system-design-p10-gdJzvBWjJz5)
> - [Isolation Level of MySQL](https://viblo.asia/p/isolation-level-of-mysql-63vKjRmAK2R)
