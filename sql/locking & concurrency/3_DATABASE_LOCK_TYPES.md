# Chương 3: Database Lock Types - Từ Row Lock Đến Metadata Lock

![Status](https://img.shields.io/badge/Outline-yellow) ![Topic](https://img.shields.io/badge/Lock_Granularity_&_MDL-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

> **Ghi chú**: Đây là bản **khung sườn (outline)** — liệt kê các ý chính cần triển khai, chưa phải bản đầy đủ.

## Mục lục (dự kiến)

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
- [II. Phân Loại Theo Granularity (Độ Chi Tiết)](#ii-phân-loại-theo-granularity-độ-chi-tiết)
- [III. Shared Lock (S-Lock) vs Exclusive Lock (X-Lock)](#iii-shared-lock-s-lock-vs-exclusive-lock-x-lock)
- [IV. Intention Lock — Tại Sao DB Không Cần Quét Toàn Bảng Để Biết Có Xung Đột?](#iv-intention-lock--tại-sao-db-không-cần-quét-toàn-bảng-để-biết-có-xung-đột)
- [V. Metadata Lock (MDL) — Kẻ Thù Giấu Mặt Của `ALTER TABLE`](#v-metadata-lock-mdl--kẻ-thù-giấu-mặt-của-alter-table)
- [VI. Gap Lock \& Next-Key Lock — Nhắc Lại Nhanh](#vi-gap-lock--next-key-lock--nhắc-lại-nhanh)
- [VII. Bảng Tổng Hợp Toàn Bộ Các Loại Lock](#vii-bảng-tổng-hợp-toàn-bộ-các-loại-lock)
- [VIII. Cheat Sheet Phỏng vấn (Interview Q\&A)](#viii-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

* Nhắc lại: [Chương 2 - Pessimistic Locking](2_PESSIMISTIC_LOCKING.md) mới chỉ khai thác **Row Lock** (qua `FOR UPDATE`/`FOR SHARE`) — nhưng đó chỉ là 1 trong nhiều "tầng" Lock mà Database Engine thực sự vận hành bên dưới.
* Mục tiêu chương này: nhìn toàn cảnh các loại Lock theo 2 trục — **Granularity** (khóa to hay nhỏ) và **Access Mode** (khóa để đọc hay để ghi) — để hiểu vì sao đôi khi Database "đứng hình" dù bạn chỉ chạy 1 câu `ALTER TABLE` tưởng như vô hại.

## II. Phân Loại Theo Granularity (Độ Chi Tiết)

* **Row-level Lock**: khóa đúng 1 dòng — đã học ở Pessimistic Locking, độ tương tranh (concurrency) cao nhất.
* **Table-level Lock**: khóa nguyên bảng (`LOCK TABLES ... WRITE` trong MySQL/MyISAM) — hiếm dùng ở tầng ứng dụng hiện đại, chủ yếu gặp khi thao tác DDL hoặc storage engine cũ không hỗ trợ row lock.
* **Page-level Lock**: khóa theo từng trang dữ liệu (phổ biến ở SQL Server) — cân bằng giữa Row Lock (quá mịn, tốn overhead quản lý) và Table Lock (quá thô, chặn hết mọi người).
* Nguyên tắc chung: **Granularity càng nhỏ → Concurrency càng cao, nhưng chi phí quản lý Lock (bộ nhớ, CPU) càng lớn** — đây là lý do các Engine hiện đại (InnoDB) ưu tiên tối đa Row Lock.

## III. Shared Lock (S-Lock) vs Exclusive Lock (X-Lock)

* **Shared Lock (S)**: nhiều Transaction có thể cùng giữ trên 1 tài nguyên (tương ứng `PESSIMISTIC_READ`/`FOR SHARE`).
* **Exclusive Lock (X)**: chỉ 1 Transaction được giữ, chặn tất cả S-Lock và X-Lock khác (tương ứng `PESSIMISTIC_WRITE`/`FOR UPDATE`).
* Bảng **Compatibility Matrix** kinh điển hay hỏi phỏng vấn:

  | | S-Lock đang giữ | X-Lock đang giữ |
  | :--- | :---: | :---: |
  | Xin thêm S-Lock | ✅ Được | ⛔ Chặn |
  | Xin thêm X-Lock | ⛔ Chặn | ⛔ Chặn |

## IV. Intention Lock — Tại Sao DB Không Cần Quét Toàn Bảng Để Biết Có Xung Đột?

* Đặt vấn đề: nếu Transaction A đang giữ Row Lock trên 1 dòng, làm sao Transaction B muốn xin **Table Lock** trên cả bảng biết ngay lập tức là "có xung đột" mà không phải quét từng dòng để kiểm tra?
* Giới thiệu **Intention Shared (IS)** / **Intention Exclusive (IX)** — Lock đặt ở cấp bảng, chỉ mang tính "báo hiệu": "bên trong bảng này đang có ai đó giữ Row Lock (S hoặc X) ở tầng thấp hơn".
* Cơ chế: trước khi xin Row Lock, Transaction phải xin Intention Lock tương ứng ở cấp bảng trước — giúp việc kiểm tra xung đột Table-level trở thành phép so sánh **O(1)** thay vì phải quét toàn bộ Row Lock đang tồn tại.

## V. Metadata Lock (MDL) — Kẻ Thù Giấu Mặt Của `ALTER TABLE`

* Định nghĩa: MySQL tự động giữ 1 Metadata Lock (đọc) trên mọi bảng đang tham gia 1 Transaction đang mở, suốt vòng đời Transaction đó — kể cả khi Transaction chỉ gọi 1 câu `SELECT` đơn giản.
* Vấn đề kinh điển production: `ALTER TABLE` (xin MDL ghi, cần **độc quyền**) trên 1 bảng đang có Transaction dài chạy dở (dù chỉ đang `SELECT`, chưa `COMMIT`) → `ALTER TABLE` bị treo chờ; tệ hơn, mọi Query **mới** sau đó nhắm vào bảng này (kể cả `SELECT` thường) cũng bị xếp hàng phía sau `ALTER TABLE` đang chờ → cả bảng "đơ" toàn bộ dù không ai chủ động khóa gì.
* Giải pháp thực chiến: giới hạn thời gian chờ MDL (`lock_wait_timeout`), hoặc dùng công cụ Online Schema Change (`pt-online-schema-change` của Percona, `gh-ost` của GitHub) để đổi schema mà không cần giữ MDL độc quyền kéo dài.

## VI. Gap Lock & Next-Key Lock — Nhắc Lại Nhanh

* Chỉ tóm tắt 2-3 dòng vì đã phân tích sâu ở [Isolation Level & MVCC, Mục VII](../consistency%20%26%20transaction%20management/2_ISOLATION_LEVEL_AND_MVCC.md) — mục đích của phần này chỉ là **định vị lại** 2 loại Lock đó vào đúng bức tranh tổng thể của chương (chúng thuộc nhóm Row-level Lock mở rộng, dùng để chặn Phantom Read).

## VII. Bảng Tổng Hợp Toàn Bộ Các Loại Lock

* Bảng tổng hợp cuối chương: liệt kê tất cả Lock đã học xuyên suốt 3 chương của folder `locking & concurrency/` — Row Lock, Table Lock, S/X-Lock, Intention Lock, MDL, Gap Lock/Next-Key Lock — kèm cột "Granularity", "Do ai chủ động xin", "Ví dụ SQL kích hoạt".

## VIII. Cheat Sheet Phỏng vấn (Interview Q&A)

*(sẽ bổ sung ở bản đầy đủ — dự kiến 6-7 câu xoay quanh: Compatibility Matrix, vai trò Intention Lock, vì sao ALTER TABLE có thể làm treo cả hệ thống, cách né MDL khi migrate schema)*

---

> **Tài liệu tham khảo dự kiến:** MySQL InnoDB Locking documentation, Percona Blog (Metadata Locks), high-performance-mysql (O'Reilly).
