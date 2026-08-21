# Chương 3: Second-Level Cache (L2 Cache) - Bộ Nhớ Đệm Toàn Cục

![Status](https://img.shields.io/badge/Outline-yellow) ![Topic](https://img.shields.io/badge/L2_Cache_&_Ehcache_Redis-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

> **Ghi chú**: Đây là bản **khung sườn (outline)** — liệt kê các ý chính cần triển khai, chưa phải bản đầy đủ. Dùng để chốt cấu trúc trước khi viết chi tiết (code, diagram, cheat sheet).

## Mục lục (dự kiến)

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
- [II. Kiến trúc L2 Cache trong Hibernate](#ii-kiến-trúc-l2-cache-trong-hibernate)
- [III. Cache Concurrency Strategy](#iii-cache-concurrency-strategy)
- [IV. Query Cache — Khác Gì Với Entity Cache?](#iv-query-cache--khác-gì-với-entity-cache)
- [V. Cache Invalidation \& Bài Toán Nhiều Instance](#v-cache-invalidation--bài-toán-nhiều-instance)
- [VI. Khi Nào Nên / Không Nên Dùng L2 Cache](#vi-khi-nào-nên--không-nên-dùng-l2-cache)
- [VII. Cheat Sheet Phỏng vấn (Interview Q\&A)](#vii-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

* Nhắc lại giới hạn của **First-Level Cache** (đã học ở [Chương 1](1_PERSISTENCE_CONTEXT_AND_L1_CACHE.md), Mục VI): L1 chỉ sống trong phạm vi 1 `Session`/`Transaction`, không chia sẻ được giữa các Request khác nhau.
* Đặt vấn đề: nếu 1.000 Request/giây cùng đọc dữ liệu bảng danh mục (Category, Config) gần như không đổi, mỗi Request lại phải `SELECT` lại từ đầu — có cách nào cache **xuyên suốt nhiều Transaction/Request** không?
* Giới thiệu L2 Cache: gắn ở cấp `SessionFactory`/`EntityManagerFactory`, sống lâu hơn hẳn 1 Transaction, dùng chung cho toàn ứng dụng.

## II. Kiến trúc L2 Cache trong Hibernate

* L2 Cache **mặc định TẮT** — phải khai báo thêm Cache Provider bên ngoài.
* Các Cache Provider phổ biến: **Ehcache** (truyền thống, phổ biến nhất với Hibernate), **Redis** (qua Redisson hoặc `hibernate-redis`), **Caffeine** (in-memory, nhẹ), **Infinispan**.
* Khái niệm **Region** — mỗi Entity/Collection được cache vào 1 "ngăn" riêng, có thể cấu hình TTL/kích thước độc lập cho từng Region.
* Cách bật: `spring.jpa.properties.hibernate.cache.use_second_level_cache=true` + annotation `@Cacheable` (JPA chuẩn) hoặc `@org.hibernate.annotations.Cache` (Hibernate-specific, cho phép chỉ định Concurrency Strategy).
* Code ví dụ tối thiểu: 1 Entity `Category` được đánh `@Cacheable` + cấu hình Ehcache XML/YAML.

## III. Cache Concurrency Strategy

* 4 chiến lược mà Hibernate cung cấp cho từng Entity/Collection được cache — đánh đổi giữa **tốc độ** và **độ chính xác dữ liệu**:
  1. **`READ_ONLY`** — nhanh nhất, dữ liệu không bao giờ đổi (bảng hằng số, danh mục tĩnh).
  2. **`NONSTRICT_READ_WRITE`** — chấp nhận đọc dữ liệu cũ trong khoảnh khắc ngắn ngay sau khi UPDATE (không khóa gì cả khi ghi).
  3. **`READ_WRITE`** — dùng Soft Lock khi ghi, đảm bảo không đọc phải dữ liệu "nửa vời", phổ biến nhất cho dữ liệu hay đổi vừa phải.
  4. **`TRANSACTIONAL`** — mạnh nhất, cần Cache Provider hỗ trợ JTA (thường chỉ Infinispan/Ehcache-JTA), cực ít dùng.
* Bảng so sánh 4 chiến lược: tốc độ, an toàn dữ liệu, use-case điển hình.

## IV. Query Cache — Khác Gì Với Entity Cache?

* Phân biệt 3 tầng cache dễ nhầm: **Entity Cache** (cache theo `id`), **Collection Cache** (cache danh sách con của quan hệ `@OneToMany`), **Query Cache** (cache kết quả của 1 câu Query cụ thể theo tham số).
* Query Cache **phải bật riêng** (`hibernate.cache.use_query_cache=true`) và gọi tường minh `setHint("org.hibernate.cacheable", true)` trên từng Query — không tự động như Entity Cache.
* Cạm bẫy: Query Cache chỉ lưu **danh sách ID**, vẫn phải tra Entity Cache để dựng lại Object — nếu Entity không được cache, Query Cache gần như vô nghĩa (vẫn tốn round-trip).

## V. Cache Invalidation & Bài Toán Nhiều Instance

* Vấn đề **Stale Cache** khi hệ thống scale-out nhiều Instance Backend cùng trỏ vào 1 DB: Instance A sửa dữ liệu, cache local của Instance B không hề hay biết.
* Vì sao Redis-based L2 Cache (cache tập trung, dùng chung giữa các Instance) thường an toàn hơn Ehcache local-heap trong kiến trúc nhiều instance (trừ khi cấu hình Ehcache Clustering/Terracotta).
* Rủi ro khi có ai đó `UPDATE` thẳng xuống DB bằng tay (migration script, tool khác) — Hibernate hoàn toàn không hay biết, cache "lệch" so với DB cho tới khi TTL hết hạn.

## VI. Khi Nào Nên / Không Nên Dùng L2 Cache

* Nên dùng: dữ liệu đọc nhiều — sửa ít (danh mục, cấu hình, dữ liệu tham chiếu), quy mô Entity nhỏ - vừa.
* Cân nhắc kỹ / không nên: dữ liệu đổi liên tục (đơn hàng, tồn kho realtime), hệ thống có nhiều Instance ghi đồng thời mà chưa có Cache Provider tập trung — dễ gây bug tinh vi hơn là lợi ích tốc độ mang lại.

## VII. Cheat Sheet Phỏng vấn (Interview Q&A)

*(sẽ bổ sung ở bản đầy đủ — dự kiến 6-8 câu xoay quanh: phân biệt L1/L2, 4 Concurrency Strategy, Query Cache vs Entity Cache, rủi ro nhiều Instance)*

---

> **Tài liệu tham khảo dự kiến:** Hibernate User Guide (Caching chapter), Vlad Mihalcea blog về L2 Cache, Baeldung Hibernate Second-Level Cache.
