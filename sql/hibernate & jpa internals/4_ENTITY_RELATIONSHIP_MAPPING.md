# Chương 4: Entity Relationship Mapping - Ánh Xạ Quan Hệ Trong JPA

![Status](https://img.shields.io/badge/Outline-yellow) ![Topic](https://img.shields.io/badge/OneToMany_ManyToMany_Cascade-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

> **Ghi chú**: Đây là bản **khung sườn (outline)** — liệt kê các ý chính cần triển khai, chưa phải bản đầy đủ.

## Mục lục (dự kiến)

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
- [II. `@ManyToOne` \& `@OneToMany` — Cặp Quan Hệ Phổ Biến Nhất](#ii-manytoone--onetomany--cặp-quan-hệ-phổ-biến-nhất)
- [III. `@OneToOne` — Ít Dùng Nhưng Nhiều Bẫy](#iii-onetoone--ít-dùng-nhưng-nhiều-bẫy)
- [IV. `@ManyToMany` — Vì Sao Nên Tránh Dùng Trực Tiếp](#iv-manytomany--vì-sao-nên-tránh-dùng-trực-tiếp)
- [V. `CascadeType` — Lan Truyền Thao Tác Giữa Các Entity](#v-cascadetype--lan-truyền-thao-tác-giữa-các-entity)
- [VI. `orphanRemoval` — Khác Gì Với `CascadeType.REMOVE`?](#vi-orphanremoval--khác-gì-với-cascadetyperemove)
- [VII. Cạm Bẫy Bidirectional Relationship](#vii-cạm-bẫy-bidirectional-relationship)
- [VIII. Cheat Sheet Phỏng vấn (Interview Q\&A)](#viii-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

* Quan hệ ở tầng DB (Foreign Key, 1 chiều, không có khái niệm "sở hữu") khác quan hệ ở tầng OOP (2 Object có thể tham chiếu qua lại) như thế nào — JPA phải "dịch" giữa 2 thế giới này.
* Giới thiệu khái niệm cốt lõi sẽ xuyên suốt cả chương: **Owning Side** (phía giữ cột FK thật sự, quyết định SQL được sinh ra) vs **Inverse Side** (`mappedBy`, chỉ để đọc, không ảnh hưởng SQL).

## II. `@ManyToOne` & `@OneToMany` — Cặp Quan Hệ Phổ Biến Nhất

* Ví dụ kinh điển: `Department` (1) - `Employee` (N).
* `@ManyToOne` luôn là **Owning Side** (Employee giữ cột `department_id`); `@OneToMany(mappedBy = "department")` phía Department là Inverse Side.
* Bẫy hay gặp: chỉ set 1 chiều (`department.getEmployees().add(emp)`) mà quên set `emp.setDepartment(department)` → Insert xong `department_id` vẫn `NULL` vì Inverse Side không quyết định SQL.
* Nhắc lại `fetch = FetchType.LAZY` mặc định cho `@OneToMany` — liên kết ngược tới [N+1 Select Problem](2_N_PLUS_1_SELECT_PROBLEM.md).

## III. `@OneToOne` — Ít Dùng Nhưng Nhiều Bẫy

* 2 cách triển khai: **Shared Primary Key** (`@MapsId`) vs **Foreign Key riêng có `@JoinColumn(unique = true)`**.
* Bẫy nổi tiếng: `@OneToOne(fetch = FetchType.LAZY)` ở phía **không giữ FK** (Inverse Side) thực chất vẫn load **EAGER** trong nhiều trường hợp — vì Hibernate không thể tạo Proxy khi không biết chắc bản ghi liên kết có tồn tại hay không (phải `SELECT` để kiểm tra `NULL`/not-null trước).
* Cách khắc phục: bytecode enhancement (`hibernate.enhance.enableLazyInitialization`) hoặc thiết kế lại thành `@OneToMany` với ràng buộc unique.

## IV. `@ManyToMany` — Vì Sao Nên Tránh Dùng Trực Tiếp

* Cách khai báo cơ bản với `@JoinTable`, bảng trung gian tự sinh không có Entity riêng.
* Vấn đề thực chiến: bảng trung gian gần như luôn cần thêm cột riêng theo thời gian (ví dụ `enrolled_at`, `role_in_project`) — `@ManyToMany` thuần **không hỗ trợ** thêm cột tùy ý vào bảng nối.
* Giải pháp khuyến nghị: tách hẳn thành 1 Entity trung gian tường minh (ví dụ `Enrollment` với `@ManyToOne` tới cả `Student` lẫn `Course`, dùng Composite Key hoặc `@EmbeddedId`) — kiểm soát hoàn toàn, dễ mở rộng.
* Nhắc lại rủi ro `MultipleBagFetchException`/Cartesian Product khi `JOIN FETCH` nhiều `@ManyToMany` cùng lúc — đã phân tích ở [N+1 Select Problem, Mục III](2_N_PLUS_1_SELECT_PROBLEM.md).

## V. `CascadeType` — Lan Truyền Thao Tác Giữa Các Entity

* Liệt kê 6 loại: `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL`.
* Ví dụ áp `CascadeType.PERSIST` trên `@OneToMany` để chỉ cần `save()` Entity cha, các Entity con tự động được `persist()` theo mà không cần gọi tay từng cái.
* Cảnh báo thực chiến: `CascadeType.REMOVE`/`ALL` trên quan hệ có khối lượng dữ liệu con lớn có thể vô tình xóa hàng loạt (ví dụ xóa `Department` cascade luôn toàn bộ `Employee`) — cần cân nhắc kỹ ý định nghiệp vụ trước khi dùng `ALL` cho tiện.

## VI. `orphanRemoval` — Khác Gì Với `CascadeType.REMOVE`?

* `CascadeType.REMOVE` chỉ kích hoạt khi **Entity cha bị xóa tường minh** (`remove(parent)`).
* `orphanRemoval = true` kích hoạt ngay cả khi **chỉ gỡ liên kết** (`parent.getChildren().remove(child)`) mà không xóa cha — Hibernate hiểu "con mồ côi" này cần bị xóa luôn khỏi DB.
* Ví dụ minh họa khác biệt bằng đúng 1 dòng code duy nhất thay đổi hành vi.

## VII. Cạm Bẫy Bidirectional Relationship

* Nguyên tắc: luôn viết **helper method** (`addEmployee()`/`removeEmployee()`) để đồng bộ **cả 2 chiều** trong 1 lần gọi, tránh quên 1 phía như Mục II.
* Bẫy `equals()`/`hashCode()`/`toString()` mặc định của Lombok (`@Data`) trên Entity có quan hệ 2 chiều → `StackOverflowError` do đệ quy vô hạn (A gọi `toString()` của B, B gọi lại `toString()` của A).
* Khuyến nghị: dùng `@ToString(exclude = ...)`/`@EqualsAndHashCode(exclude = ...)` của Lombok, hoặc tự viết `equals()`/`hashCode()` chỉ dựa trên Business Key — liên kết ngược lại lưu ý tương tự đã nói ở [Optimistic Locking, Mục VII.3](../locking%20%26%20concurrency/1_OPTIMISTIC_LOCKING.md#3-đừng-bao-giờ-đưa-version-vào-equalshashcode) (đừng đưa field dễ đổi vào `equals()`).

## VIII. Cheat Sheet Phỏng vấn (Interview Q&A)

*(sẽ bổ sung ở bản đầy đủ — dự kiến 7-8 câu xoay quanh: Owning Side, bẫy @OneToOne Lazy, vì sao tránh @ManyToMany thuần, CascadeType.REMOVE vs orphanRemoval, StackOverflow do bidirectional)*

---

> **Tài liệu tham khảo dự kiến:** Vlad Mihalcea blog (Best way to map @OneToOne / @ManyToMany), Baeldung JPA Relationship mapping, Hibernate User Guide.
