# Chương 2: Bài toán N+1 Select - "Sát Thủ" Giấu Mặt

![Status](https://img.shields.io/badge/N%2B1_Problem-green) ![Topic](https://img.shields.io/badge/JOIN_FETCH_&_EntityGraph-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Tổng quan \& Đặt vấn đề: "Sát Thủ" Giấu Mặt Là Ai?](#i-tổng-quan--đặt-vấn-đề-sát-thủ-giấu-mặt-là-ai)
  - [1. Kịch bản phát sinh: In danh sách Phòng ban kèm Nhân viên](#1-kịch-bản-phát-sinh-in-danh-sách-phòng-ban-kèm-nhân-viên)
  - [2. Cội nguồn: Lazy Loading và vòng lặp `for`](#2-cội-nguồn-lazy-loading-và-vòng-lặp-for)
  - [3. Vì sao không dùng Eager Loading toàn cục để né N+1?](#3-vì-sao-không-dùng-eager-loading-toàn-cục-để-né-n1)
- [II. Bộ Vũ Khí Chống N+1: JOIN FETCH, @EntityGraph \& @BatchSize](#ii-bộ-vũ-khí-chống-n1-join-fetch-entitygraph--batchsize)
  - [1. JOIN FETCH (JPQL/HQL)](#1-join-fetch-jpqlhql)
  - [2. @EntityGraph (Spring Data JPA)](#2-entitygraph-spring-data-jpa)
  - [3. @BatchSize / hibernate.default_batch_fetch_size](#3-batchsize--hibernatedefault_batch_fetch_size)
- [III. Gót Chân Achilles: Cartesian Product \& MultipleBagFetchException](#iii-gót-chân-achilles-cartesian-product--multiplebagfetchexception)
  - [1. Tích Đề-các (Cartesian Product) là gì?](#1-tích-đề-các-cartesian-product-là-gì)
  - [2. MultipleBagFetchException — Lưới an toàn của Hibernate](#2-multiplebagfetchexception--lưới-an-toàn-của-hibernate)
  - [3. Cạm bẫy "lách luật" bằng Set thay vì List](#3-cạm-bẫy-lách-luật-bằng-set-thay-vì-list)
- [IV. Cạm Bẫy Khi Kết Hợp JOIN FETCH với Phân Trang (Pagination)](#iv-cạm-bẫy-khi-kết-hợp-join-fetch-với-phân-trang-pagination)
  - [1. Cảnh báo HHH000104: "firstResult/maxResults specified with collection fetch"](#1-cảnh-báo-hhh000104-firstresultmaxresults-specified-with-collection-fetch)
  - [2. Vì sao Hibernate buộc phải Phân Trang trên RAM (In-memory Pagination)?](#2-vì-sao-hibernate-buộc-phải-phân-trang-trên-ram-in-memory-pagination)
  - [3. Giải pháp triệt để: Truy vấn 2 bước (Query ID trước, JOIN FETCH/EntityGraph sau)](#3-giải-pháp-triệt-để-truy-vấn-2-bước-query-id-trước-join-fetchentitygraph-sau)
  - [4. Giải pháp thay thế: Tách truy vấn thủ công 100% bằng Java (trả về DTO)](#4-giải-pháp-thay-thế-tách-truy-vấn-thủ-công-100-bằng-java-trả-về-dto)
- [V. OSIV (Open Session In View) — "Kẻ Sát Nhân Thầm Lặng" \& LazyInitializationException](#v-osiv-open-session-in-view--kẻ-sát-nhân-thầm-lặng--lazyinitializationexception)
  - [1. Vì sao Entity Lazy vẫn "chạy được" ngoài Controller?](#1-vì-sao-entity-lazy-vẫn-chạy-được-ngoài-controller)
  - [2. Tắt OSIV để lộ nguyên hình LazyInitializationException](#2-tắt-osiv-để-lộ-nguyên-hình-lazyinitializationexception)
  - [3. Nguyên tắc kiến trúc: Không để Entity rò rỉ khỏi tầng Service](#3-nguyên-tắc-kiến-trúc-không-để-entity-rò-rỉ-khỏi-tầng-service)
- [VI. Bảng So Sánh Toàn Diện Các Giải Pháp Chống N+1](#vi-bảng-so-sánh-toàn-diện-các-giải-pháp-chống-n1)
- [VII. Cheat Sheet Phỏng vấn (Interview Q\&A)](#vii-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề: "Sát Thủ" Giấu Mặt Là Ai?

### 1. Kịch bản phát sinh: In danh sách Phòng ban kèm Nhân viên

Bài toán N+1 là lỗi hiệu năng kinh điển và phổ biến bậc nhất khi làm việc với ORM (Hibernate/JPA nói riêng). Nó đáng sợ ở chỗ **hoàn toàn "vô hình"** trong lúc code và test với dữ liệu mẫu ít ỏi — chỉ phát nổ khi lên Production với dữ liệu thật.

Giả sử bạn có quan hệ 1-N: 1 `Department` (Phòng ban) chứa nhiều `Employee` (Nhân viên). Yêu cầu: in ra danh sách 10 Phòng ban kèm tên các Nhân viên trong từng phòng.

```java
List<Department> departments = departmentRepository.findAll();
// (1) Hibernate bắn: SELECT * FROM departments  →  trả về 10 dòng

for (Department dept : departments) {
    System.out.println(dept.getName());
    for (Employee emp : dept.getEmployees()) {   // quan hệ @OneToMany mặc định LAZY
        System.out.println(" - " + emp.getName());
    }
}
```

Đoạn code trông hoàn toàn vô hại — chỉ là 1 vòng lặp `for` bình thường. Nhưng hãy đếm số câu SQL thực sự chạy dưới nền:

```
Câu lệnh gốc (Số 1):
  SELECT * FROM departments                              → 10 Department

Vòng lặp chạy 10 lần, mỗi lần chạm vào dept.getEmployees() (Số N):
  SELECT * FROM employees WHERE department_id = 1
  SELECT * FROM employees WHERE department_id = 2
  SELECT * FROM employees WHERE department_id = 3
  ...
  SELECT * FROM employees WHERE department_id = 10        → 10 câu SELECT rời rạc

Tổng cộng: 1 + 10 = 11 câu SQL cho một tác vụ tưởng chừng đơn giản!
```

Nếu 10 Phòng ban là 1.000 Phòng ban (hoàn toàn bình thường ở hệ thống thật), con số này vọt lên **1.001 câu SQL** — tương đương 1.001 lần round-trip mạng riêng biệt tới Database chỉ để phục vụ **một** request của người dùng. Đây chính là lý do N+1 được gọi là "sát thủ giấu mặt": môi trường dev với vài dòng dữ liệu mẫu chạy mượt như không có gì bất thường, nhưng lên Production, hệ thống có thể sập nguồn hoặc timeout hàng loạt chỉ vì 1 màn hình danh sách tưởng như vô hại.

### 2. Cội nguồn: Lazy Loading và vòng lặp `for`

Nguyên nhân sâu xa nằm ở chiến lược tải dữ liệu **mặc định** của các quan hệ Collection (`@OneToMany`, `@ManyToMany`) trong JPA: **`FetchType.LAZY`**.

* Khi bạn `findAll()` lấy `Department`, Hibernate **không** lập tức lấy kèm `Employee`. Nó chỉ lấy đúng dữ liệu của bảng `departments` (câu SQL số 1).
* Với field `employees`, Hibernate gắn vào đó một **Proxy** (một Collection "rỗng vỏ", chưa có dữ liệu thật bên trong).
* Chỉ khi code thực sự "chạm" vào Proxy đó — ở đây là lệnh `dept.getEmployees()` bên trong vòng lặp — Hibernate mới lật đật mở connection, bắn 1 câu SELECT xuống DB để "đổ đầy" Proxy bằng dữ liệu thật.

Vì vòng lặp `for` chạm vào Proxy này **N lần riêng biệt** (N = số Phòng ban), Hibernate cũng bắn N câu SELECT riêng biệt tương ứng — không có bất kỳ sự "gộp lại" thông minh nào cả, vì tại thời điểm chạy câu SELECT gốc, Hibernate hoàn toàn chưa biết bạn có ý định duyệt tiếp vào `employees` của toàn bộ 10 Phòng ban hay không.

### 3. Vì sao không dùng Eager Loading toàn cục để né N+1?

Phản xạ đầu tiên của nhiều người mới học Hibernate: đổi hẳn `fetch = FetchType.EAGER` ngay trên annotation `@OneToMany` để Hibernate luôn tự động lấy kèm `Employee` mỗi khi lấy `Department`. N+1 biến mất thật — nhưng bạn vừa đánh đổi lấy 2 vấn đề còn nghiêm trọng hơn:

* **Lãng phí băng thông & bộ nhớ ở những nơi không cần**: Có hàng chục màn hình trong hệ thống chỉ cần hiển thị *tên* Phòng ban (dropdown, breadcrumb, báo cáo tổng quan...). Với `EAGER` toàn cục, **mọi** truy vấn `Department` — dù bạn có nhu cầu hay không — đều âm thầm `JOIN` sang bảng `employees` và kéo theo toàn bộ Nhân viên lên RAM. Nếu 1 phòng ban có hàng chục nghìn nhân viên, một màn hình đơn giản như dropdown chọn phòng ban có thể kéo sập ứng dụng vì `OutOfMemoryError`.
* **Cartesian Product (Tích Đề-các)**: EAGER dùng cơ chế `JOIN` vật lý dưới DB, dữ liệu Phòng ban sẽ bị lặp lại theo đúng số dòng Nhân viên tương ứng (phân tích chi tiết ở Mục III) — vừa tốn băng thông, vừa tốn công "lọc trùng" (deduplicate) ở tầng ứng dụng.

Nói cách khác: `EAGER` toàn cục là con dao 2 lưỡi — bạn đổi "nhiều câu SQL nhỏ, chạy tùy nhu cầu" lấy "1 câu SQL to, chạy vô điều kiện dù cần hay không". Giải pháp đúng đắn không phải là chọn 1 trong 2 chiến lược tải dữ liệu cố định cho **toàn bộ** hệ thống, mà là **giữ nguyên LAZY làm mặc định an toàn**, rồi **chủ động ép EAGER đúng lúc, đúng chỗ, đúng nhu cầu của từng use-case cụ thể** — đây chính là tinh thần của các giải pháp ở Mục II.

---

## II. Bộ Vũ Khí Chống N+1: JOIN FETCH, @EntityGraph & @BatchSize

### 1. JOIN FETCH (JPQL/HQL)

`JOIN FETCH` là từ khóa đặc biệt trong JPQL/HQL, khác hẳn `JOIN` thông thường. `JOIN` chỉ dùng để lọc điều kiện (không kéo theo dữ liệu quan hệ vào Object trả về), còn **`JOIN FETCH` ép Hibernate JOIN 2 bảng ngay ở tầng DB và nạp luôn dữ liệu quan hệ vào Object Java, gói gọn trong đúng 1 câu SELECT duy nhất**:

```java
@Query("SELECT DISTINCT d FROM Department d JOIN FETCH d.employees")
List<Department> findAllWithEmployees();
```

SQL sinh ra tương ứng:

```sql
SELECT DISTINCT d.*, e.*
FROM departments d
LEFT OUTER JOIN employees e ON d.id = e.department_id
```

Chỉ **1 câu SQL** cho toàn bộ dữ liệu — N+1 biến mất hoàn toàn. Vài lưu ý quan trọng khi dùng `JOIN FETCH`:

* **Bắt buộc dùng `DISTINCT`** trong JPQL khi `JOIN FETCH` vào 1 quan hệ Collection: vì phép `JOIN` vật lý sẽ lặp lại dữ liệu Phòng ban theo đúng số Nhân viên (1 Phòng ban có 5 Nhân viên → 5 dòng kết quả, trong đó thông tin Phòng ban bị trùng lặp cả 5 lần). `DISTINCT` trong JPQL giúp Hibernate lọc trùng ở tầng Object Java sau khi nhận kết quả (bản thân câu SQL vật lý dưới DB vẫn trả về đủ số dòng lặp, `DISTINCT` chỉ có tác dụng loại trùng khi Hibernate dựng lại danh sách Entity).
* **Không kết hợp được với Phân trang (`Pageable`/`setMaxResults`)** một cách an toàn — xem cạm bẫy chi tiết ở Mục IV.
* **Không JOIN FETCH quá 1 quan hệ Collection cùng lúc** — dễ dính Cartesian Product hoặc `MultipleBagFetchException` (Mục III).

### 2. @EntityGraph (Spring Data JPA)

Nếu hệ thống có nhiều kịch bản tải dữ liệu khác nhau cho cùng 1 Entity (lúc chỉ cần `Department` trơn, lúc cần kèm `employees`, lúc cần kèm cả `employees` lẫn `budget`...), viết tay hàng loạt câu `@Query` JPQL riêng biệt sẽ khiến Repository phình to và trùng lặp logic. `@EntityGraph` giải quyết việc này một cách thanh lịch hơn: bạn khai báo ngay trên method của Repository *những field nào cần Eager Loading cho riêng lần gọi này*, Spring Data JPA sẽ tự dịch thành `LEFT JOIN` tương ứng — hoàn toàn không cần viết JPQL thủ công.

**Cách 1 — Khai báo trực tiếp (ad-hoc) trên Repository:**

```java
public interface DepartmentRepository extends JpaRepository<Department, Long> {

    @EntityGraph(attributePaths = {"employees"})
    List<Department> findAll();
}
```

**Cách 2 — Khai báo tái sử dụng bằng `@NamedEntityGraph` ngay trên Entity:**

```java
@Entity
@NamedEntityGraph(
    name = "Department.withEmployees",
    attributeNodes = @NamedAttributeNode("employees")
)
public class Department {
    @Id
    private Long id;

    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY) // vẫn giữ LAZY mặc định
    private List<Employee> employees;
}
```

```java
public interface DepartmentRepository extends JpaRepository<Department, Long> {

    @EntityGraph(value = "Department.withEmployees", type = EntityGraphType.FETCH)
    List<Department> findByNameContaining(String keyword);
}
```

**Bản chất**: `@EntityGraph` là cơ chế "ghi đè" (override) chiến lược tải dữ liệu từ `LAZY` (mặc định khai báo trên Entity) sang `EAGER`, nhưng chỉ áp dụng **cục bộ cho đúng lần gọi method đó** — các nơi khác trong code gọi `Department` theo cách thông thường vẫn giữ nguyên hành vi `LAZY` ban đầu. Đây chính là câu trả lời cho vấn đề "EAGER toàn cục" ở Mục I.3: bạn có toàn quyền quyết định *khi nào* cần Eager, mà không phải đánh đổi cho *mọi lúc*.

Tham số `type` đáng chú ý có 2 giá trị:

| `EntityGraphType` | Ý nghĩa |
| :--- | :--- |
| **`FETCH`** (mặc định) | Các thuộc tính **có trong** `attributePaths` → tải `EAGER`. Các thuộc tính **không được liệt kê** → ép về `LAZY`, **bất kể** cấu hình gốc trên Entity là gì (kể cả khi field đó vốn khai báo `EAGER` sẵn). |
| **`LOAD`** | Các thuộc tính **có trong** `attributePaths` → tải `EAGER`. Các thuộc tính **không được liệt kê** → giữ nguyên đúng chiến lược fetch **gốc** đã khai báo trên Entity (không ép đổi). |

### 3. @BatchSize / hibernate.default_batch_fetch_size

`JOIN FETCH` và `@EntityGraph` đều dựa trên nguyên lý "ép Eager tường minh" — rất mạnh nhưng dễ dính Cartesian Product nếu bạn có nhiều hơn 1 quan hệ Collection cần tải cùng lúc (Mục III). `@BatchSize` chọn một triết lý hoàn toàn khác: **giữ nguyên `LAZY`**, nhưng thay vì để Hibernate bắn N câu SELECT rời rạc từng-cái-một, nó **gom nhóm (batch)** nhiều lần gọi Lazy lại thành các câu `SELECT ... WHERE id IN (...)`.

Cơ chế hoạt động, với `@BatchSize(size = 20)`:

```
1. Query gốc: SELECT * FROM departments               → 100 Department (LAZY employees)

2. Vòng lặp for chạm vào department.getEmployees()
   Hibernate KHÔNG bắn ngay 100 câu SELECT rời rạc.
   Nó gom các Department đang "chờ" Lazy load thành từng lô 20 ID:

   SELECT * FROM employees WHERE department_id IN (1,2,...,20)
   SELECT * FROM employees WHERE department_id IN (21,22,...,40)
   ...
   (5 lô cho 100 Department)

Tổng cộng: 1 (query gốc) + 5 (lô batch) = 6 câu SQL, thay vì 101 câu!
```

**Kích hoạt cục bộ** (ngay trên quan hệ của Entity):

```java
@Entity
public class Department {
    @Id
    private Long id;

    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    @BatchSize(size = 20)   // annotation riêng của Hibernate (không thuộc chuẩn JPA)
    private List<Employee> employees;
}
```

**Kích hoạt toàn cục** (áp dụng cho mọi quan hệ Lazy trong toàn ứng dụng, không cần sửa từng Entity — thường là lựa chọn mặc định an toàn, khuyến nghị bật ngay từ đầu dự án):

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

**Điểm mạnh của `@BatchSize` so với `JOIN FETCH`/`@EntityGraph`**: không cần sửa bất kỳ dòng code Query nào đang có sẵn, không có nguy cơ Cartesian Product (vì không hề dùng `JOIN`, chỉ dùng `IN`), và an toàn tuyệt đối khi kết hợp với Phân trang. Đánh đổi: vẫn tốn nhiều hơn 1 câu SQL (số câu = 1 + số lô), không tối ưu bằng `JOIN FETCH` (chỉ 1 câu duy nhất) trong trường hợp bạn *chắc chắn* luôn cần dữ liệu quan hệ ngay từ đầu.

---

## III. Gót Chân Achilles: Cartesian Product & MultipleBagFetchException

### 1. Tích Đề-các (Cartesian Product) là gì?

Đây là cái giá phải trả khi dùng `JOIN FETCH`/`@EntityGraph` để kéo **nhiều hơn 1** quan hệ Collection cùng lúc. Giả sử `Department` vừa có danh sách `employees`, vừa có danh sách `projects` (dự án đang triển khai), và bạn muốn tối ưu bằng cách tải cả hai cùng lúc:

```java
@EntityGraph(attributePaths = {"employees", "projects"})
List<Department> findAll();
```

Câu SQL sinh ra sẽ chứa **2 mệnh đề `LEFT JOIN`** — DB sẽ thực hiện phép **nhân chéo (cross join)** giữa 2 bảng con. Giả sử 1 Phòng ban có 40 Nhân viên và 10 Dự án, kết quả trả về **không phải** 50 dòng gộp lại, mà là:

```
1 (Department) × 40 (employees) × 10 (projects) = 400 dòng!
```

Thông tin của chính Phòng ban đó (tên, id...) bị lặp lại đúng **400 lần** trong resultset thô. Hậu quả: băng thông mạng bị tắc nghẽn bởi dữ liệu dư thừa khổng lồ, và Hibernate phải tốn thêm chi phí CPU/RAM đáng kể để "lọc trùng" (deduplicate) hàng trăm nghìn dòng thô đó về lại đúng số Entity gốc — dữ liệu càng lớn, nguy cơ `OutOfMemoryError` càng cao.

### 2. MultipleBagFetchException — Lưới an toàn của Hibernate

Nhận thức được sự nguy hiểm trên, nếu bạn khai báo **từ 2 quan hệ Collection kiểu `List` (Bag — danh sách không đảm bảo thứ tự và cho phép trùng)** trở lên cùng bị `JOIN FETCH`/`@EntityGraph` trong 1 câu truy vấn, Hibernate sẽ **chủ động từ chối chạy**, ném thẳng ra:

```
org.hibernate.loader.MultipleBagFetchException:
cannot simultaneously fetch multiple bags
```

Đây là cơ chế **bảo vệ chủ động** — Hibernate biết chắc chắn kết quả sẽ méo mó (dữ liệu bị nhân bản sai) nên thà chặn đứng ngay từ đầu, còn hơn để ứng dụng chạy ngầm với dữ liệu sai lệch mà không ai hay biết.

### 3. Cạm bẫy "lách luật" bằng Set thay vì List

Một mẹo phổ biến để né `MultipleBagFetchException`: đổi kiểu dữ liệu Collection từ `List<Employee>` sang `Set<Employee>`. Vì `Set` không cho phép phần tử trùng lặp, Hibernate cho phép bạn `JOIN FETCH` nhiều `Set` cùng lúc mà **không** ném exception.

**Nhưng đây là cái bẫy rất hay bị hỏi trong phỏng vấn**: đổi sang `Set` chỉ giúp Hibernate *chấp nhận chạy* câu query, chứ **không hề loại bỏ** bản chất Cartesian Product ở tầng DB — con số `400 dòng` dữ liệu thô ở ví dụ trên **vẫn bị truyền qua mạng y hệt**, `Set` chỉ đảm nhiệm việc lọc trùng ở bước cuối cùng khi Hibernate dựng Object. Tệ hơn, nếu Entity của bạn **không override đúng `equals()`/`hashCode()`** (mặc định dùng địa chỉ bộ nhớ), việc lọc trùng vào `Set` có thể cho ra kết quả sai lệch âm thầm (mất phần tử hoặc trùng phần tử) mà không có bất kỳ cảnh báo nào. Do đó, "lách" bằng `Set` chỉ nên áp dụng khi số lượng bản ghi con thực sự nhỏ và ổn định — với dữ liệu lớn, giải pháp đúng đắn là tách truy vấn (Mục IV.3), không phải đổi kiểu dữ liệu.

---

## IV. Cạm Bẫy Khi Kết Hợp JOIN FETCH với Phân Trang (Pagination)

### 1. Cảnh báo HHH000104: "firstResult/maxResults specified with collection fetch"

Đây là cạm bẫy thực chiến cực kỳ phổ biến: bạn viết 1 API phân trang danh sách Phòng ban, đồng thời muốn tối ưu N+1 bằng `JOIN FETCH`/`@EntityGraph` kèm `employees`:

```java
@Query("SELECT d FROM Department d JOIN FETCH d.employees")
Page<Department> findAllWithEmployees(Pageable pageable);
```

Chạy đoạn code này, log sẽ hiện ngay dòng cảnh báo:

```
WARN: HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```

### 2. Vì sao Hibernate buộc phải Phân Trang trên RAM (In-memory Pagination)?

Nguyên nhân chính là mâu thuẫn vật lý giữa `JOIN` và `LIMIT`/`OFFSET`. Như đã phân tích ở Mục III, `JOIN FETCH` vào 1 quan hệ Collection khiến dữ liệu dòng cha bị lặp lại theo số dòng con — giả sử Phòng ban #1 có 5 Nhân viên, kết quả JOIN thô sẽ trả về 5 dòng, trong đó Phòng ban #1 lặp lại cả 5 lần. Nếu Hibernate áp `LIMIT 20` thẳng xuống câu SQL vật lý này, nó có thể chỉ cắt trọn vẹn được 3-4 Phòng ban (vì mỗi phòng chiếm nhiều dòng khác nhau tùy số nhân viên) — phá vỡ hoàn toàn ngữ nghĩa "20 Phòng ban" mà bạn mong muốn.

Để tránh trả kết quả sai lệch (một Phòng ban bị cắt cụt danh sách Nhân viên giữa chừng), Hibernate đưa ra quyết định "chữa cháy" khá nguy hiểm: **âm thầm bỏ hẳn `LIMIT`/`OFFSET` ở tầng SQL**, kéo **toàn bộ** kết quả JOIN thô (có thể là hàng triệu dòng nếu dữ liệu lớn) lên RAM ứng dụng, rồi mới cắt trang **bằng Java, trong bộ nhớ**. Với bảng dữ liệu nhỏ, bạn sẽ không nhận ra vấn đề gì — nhưng với bảng có hàng trăm nghìn/hàng triệu dòng, việc "phân trang 20 bản ghi" thực chất đã âm thầm kéo tuột **toàn bộ Database** qua mạng và nhồi vào RAM trước, khiến ứng dụng đứng hình hoặc sập vì `OutOfMemoryError` dù bạn chỉ xin đúng 1 trang dữ liệu nhỏ.

> **Ghi nhớ khi đi phỏng vấn**: Không bao giờ dùng `JOIN FETCH`/`@EntityGraph` trực tiếp trên 1 quan hệ Collection kết hợp cùng `Pageable`/`setFirstResult()`/`setMaxResults()`. Cảnh báo `HHH000104` không phải chuyện nhỏ — nó là dấu hiệu ứng dụng đang phân trang sai cách, tiềm ẩn nguy cơ tải nguyên bảng lên RAM.

### 3. Giải pháp triệt để: Truy vấn 2 bước (Query ID trước, JOIN FETCH/EntityGraph sau)

Giải pháp chuẩn mực, tận dụng chính **First-Level Cache** (Identity Map — đã phân tích ở [Chương 1](1_PERSISTENCE_CONTEXT_AND_L1_CACHE.md)) để Hibernate tự "lắp ráp" dữ liệu hộ bạn:

```java
public interface DepartmentRepository extends JpaRepository<Department, Long> {

    // Bước 1: Phân trang THUẦN TÚY dưới DB — tuyệt đối KHÔNG fetch kèm collection
    @Query("SELECT d.id FROM Department d ORDER BY d.name")
    Page<Long> findDepartmentIds(Pageable pageable);

    // Bước 2: JOIN FETCH đúng tập ID đã xác định ở Bước 1 — không còn LIMIT/OFFSET nào nữa
    @Query("SELECT DISTINCT d FROM Department d JOIN FETCH d.employees WHERE d.id IN :ids")
    List<Department> findByIdsWithEmployees(@Param("ids") List<Long> ids);
}
```

```java
Page<Long> idPage = departmentRepository.findDepartmentIds(pageable);
// (1) SELECT id FROM departments ORDER BY name LIMIT 20 OFFSET 0
//     → Phân trang CHUẨN XÁC ở tầng DB, vì không có JOIN nào cả

List<Department> departments =
    departmentRepository.findByIdsWithEmployees(idPage.getContent());
// (2) SELECT DISTINCT d.*, e.* FROM departments d
//     LEFT JOIN employees e ON d.id = e.department_id
//     WHERE d.id IN (...)
//     → JOIN FETCH tự do, KHÔNG kèm LIMIT nên không còn bị Hibernate "chữa cháy" trên RAM
```

**Chỉ 2 câu SQL tường minh**, phân trang chính xác tuyệt đối, và không có bất kỳ dòng dữ liệu thừa nào bị kéo qua mạng ngoài đúng phạm vi trang hiện tại đang cần.

Với bài toán phức tạp hơn — vừa cần phân trang, vừa cần Eager nhiều hơn 1 quan hệ Collection cùng lúc (`employees` và `projects`) — nguyên tắc "chia để trị" được nâng cấp thêm 1 bước: tách hẳn từng quan hệ ra 1 câu Query "làm giàu" (enrich) riêng biệt theo cùng tập `ids`, rồi để Persistence Context tự ghép nối:

```java
// Bước 1: Lấy trang Department "trơn" theo ID (không kèm bất kỳ collection nào)
Page<Department> page = departmentRepository.fetchPageBareBones(pageable);
List<Long> ids = page.getContent().stream().map(Department::getId).toList();

// Bước 2: "Làm giàu" tập hợp employees cho đúng các Department này
departmentRepository.enrichWithEmployees(ids);
// SELECT d FROM Department d JOIN FETCH d.employees WHERE d.id IN :ids

// Bước 3: "Làm giàu" tập hợp projects cho đúng các Department này
departmentRepository.enrichWithProjects(ids);
// SELECT d FROM Department d JOIN FETCH d.projects WHERE d.id IN :ids

// page.getContent() giờ đây đã có ĐẦY ĐỦ cả employees lẫn projects,
// vì ở Bước 2 và 3, Hibernate nhận ra các Department này ĐÃ NẰM SẴN
// trong Identity Map (Persistence Context) từ Bước 1, nên KHÔNG tạo Object mới,
// mà tái sử dụng đúng instance cũ và "đổ" thêm dữ liệu Collection vào nó.
```

Nhờ nguyên lý Identity Map (Mục II, [Chương 1](1_PERSISTENCE_CONTEXT_AND_L1_CACHE.md)), 3 câu Query độc lập này tự động "hội tụ" đúng vào cùng 1 tập Object Java trên RAM, không cần bạn tự tay ghép nối bằng vòng lặp/`Map`.

### 4. Giải pháp thay thế: Tách truy vấn thủ công 100% bằng Java (trả về DTO)

Nếu 2 giải pháp trên vẫn còn dựa một phần vào "phép thuật" ngầm của Hibernate (JOIN vật lý, hoặc Identity Map tự ghép nối), thì đây là cách tiếp cận **kiểm soát hoàn toàn bằng tay**: tự Query, tự gom nhóm bằng Java Stream, và trả thẳng ra DTO — không có bất kỳ Entity Lazy nào "sống sót" ra khỏi tầng Repository. Cách này tỏa sáng khi bạn muốn tối ưu bộ nhớ tối đa (không load thừa field nào ngoài DTO cần), hoặc khi hệ thống lớn tới mức dữ liệu cha/con bị phân mảnh ở nhiều nguồn khác nhau (ví dụ kiến trúc Microservices).

4 bước chuẩn: (1) Query lấy danh sách đối tượng cha → (2) trích xuất ID của cha → (3) Query riêng dùng `IN` để lấy đối tượng con → (4) gom nhóm bằng `Collectors.groupingBy` rồi map sang DTO.

```java
// 1. Lấy danh sách Phòng ban
List<Department> departments = departmentRepository.findAll();

// 2. Trích xuất danh sách departmentId
List<Long> departmentIds = departments.stream()
    .map(Department::getId)
    .toList();

// 🛡️ Chặn lỗi cú pháp SQL: hầu hết RDBMS (MySQL, PostgreSQL, Oracle...) coi
// "WHERE department_id IN ()" (danh sách rỗng) là cú pháp KHÔNG hợp lệ,
// sẽ ném SQL Syntax Error khiến API sập ngay nếu departmentIds rỗng.
if (departmentIds.isEmpty()) {
    return List.of();
}

// 3. Truy vấn Nhân viên theo tập departmentIds
List<Employee> employees = employeeRepository.findByDepartmentIdIn(departmentIds);

// 4. Gom nhóm Nhân viên theo departmentId bằng Java Stream
Map<Long, List<Employee>> employeesByDeptId = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartmentId));

// 5. Ánh xạ (map) sang DTO trả về cho tầng Controller
List<DepartmentDto> result = departments.stream()
    .map(d -> new DepartmentDto(
        d.getId(),
        d.getName(),
        employeesByDeptId.getOrDefault(d.getId(), List.of())
    ))
    .toList();
```

Chi tiết nhỏ nhưng dễ gây sập API nếu bỏ sót: luôn chặn bằng `if (ids.isEmpty()) return ...;` trước khi bắn Query 2 — ranh giới giữa code "chạy được" và code chuẩn mực, chịu lỗi tốt (fault-tolerant) trong môi trường thực tế chính là ở những chi tiết như vậy.

---

## V. OSIV (Open Session In View) — "Kẻ Sát Nhân Thầm Lặng" & LazyInitializationException

### 1. Vì sao Entity Lazy vẫn "chạy được" ngoài Controller?

Đã học ở [Chương 1](1_PERSISTENCE_CONTEXT_AND_L1_CACHE.md) rằng Persistence Context đóng ngay khi Transaction ở tầng Service kết thúc (Entity trở thành Detached), vậy tại sao đoạn code sau — gọi field Lazy ở tận Controller, ngoài phạm vi `@Transactional` — vẫn chạy êm ru, không hề văng lỗi?

```java
@GetMapping("/departments/{id}")
public Department getDepartment(@PathVariable Long id) {
    Department dept = departmentService.findById(id); // @Transactional đã kết thúc ở đây
    dept.getEmployees().size(); // gọi field LAZY... mà vẫn chạy bình thường?!
    return dept;
}
```

Thủ phạm là **OSIV — Open Session In View** (`spring.jpa.open-in-view`), một tính năng **mặc định BẬT (`true`)** trong Spring Boot. OSIV âm thầm giữ Session/Connection Database **mở kéo dài** từ lúc request đi vào tầng Web cho tới khi toàn bộ response (kể cả bước serialize JSON) hoàn tất — dài hơn hẳn phạm vi 1 `@Transactional` thông thường. Nhờ vậy field Lazy vẫn "load được" dù đã ra khỏi Service, tạo cảm giác an toàn giả tạo.

### 2. Tắt OSIV để lộ nguyên hình LazyInitializationException

Cái giá của "ảo giác an toàn" đó: OSIV dung túng cho việc viết code sai kiến trúc (để Entity + field Lazy rò rỉ tự do ra tận Controller), đồng thời **vắt kiệt Connection Pool** khi tải cao — vì mỗi Request giữ 1 Connection mở suốt cả vòng đời xử lý View/Serialize, thay vì trả lại Pool ngay khi Service kết thúc.

Khuyến nghị chuẩn mực (và cũng là câu rất hay bị hỏi khi phỏng vấn vị trí Senior): **tắt OSIV** ngay từ đầu dự án:

```yaml
# application.yml
spring:
  jpa:
    open-in-view: false
```

Khi tắt, Persistence Context đóng đúng ngay khi `@Transactional` của Service kết thúc. Đoạn code gọi `dept.getEmployees()` ở Controller phía trên giờ đây sẽ ném thẳng:

```
org.hibernate.LazyInitializationException:
failed to lazily initialize a collection of role: ...Department.employees,
could not initialize proxy - no Session
```

Lỗi này **không phải là dấu hiệu xấu** — ngược lại, nó buộc bug kiến trúc "Entity rò rỉ ra ngoài Service" phải lộ diện **ngay lúc code, trên máy dev**, thay vì âm thầm ẩn náu rồi phát nổ ngẫu nhiên khi hệ thống chịu tải cao ở Production (do Connection Pool cạn kiệt).

### 3. Nguyên tắc kiến trúc: Không để Entity rò rỉ khỏi tầng Service

Nguyên tắc tối thượng để triệt tiêu `LazyInitializationException`: **không bao giờ để Entity (đặc biệt là các field Lazy) rò rỉ ra khỏi tầng Service** — mọi field Lazy cần thiết phải được "đánh thức" và ánh xạ (map) sang DTO **trong khi** Persistence Context còn mở. Vài chiến thuật thực chiến phổ biến:

* **Mapping chủ động ngay trong vùng `@Transactional`**: gọi getter để "đánh thức" field Lazy, rồi map ngay sang DTO thuần túy trước khi return khỏi Service — thường dùng kèm thư viện như MapStruct/ModelMapper để tự động hóa việc sao chép field, tránh viết tay hàng chục dòng setter lặp lại.
* **DTO Projection**: nếu mục tiêu cuối cùng chỉ là đọc dữ liệu (read-only) để hiển thị, có thể "lấy thẳng" DTO ngay từ tầng Query (Spring Data JPA hỗ trợ Interface-based hoặc Class-based Projection) — không cần dựng Entity đầy đủ rồi mới chuyển đổi, tối ưu cả tốc độ lẫn bộ nhớ.
* **Tách truy vấn thủ công trả DTO** (Mục IV.4) — vốn dĩ đã không đụng tới Entity Lazy nào ngoài phạm vi Repository, nên an toàn tuyệt đối trước `LazyInitializationException` dù có tắt OSIV hay không.

> **Ghi nhớ khi đi phỏng vấn**: OSIV không "sửa" N+1 hay `LazyInitializationException` — nó chỉ **che giấu** triệu chứng bằng cách kéo dài vòng đời Connection. Best practice trong các dự án Production nghiêm túc là **tắt OSIV** (`spring.jpa.open-in-view=false`) và xử lý triệt để việc chuyển đổi Entity → DTO ngay trong tầng Service.

---

## VI. Bảng So Sánh Toàn Diện Các Giải Pháp Chống N+1

| Tiêu chí | `JOIN FETCH` | `@EntityGraph` | `@BatchSize` | Truy vấn 2 bước (ID trước, Fetch sau) | Tách thủ công bằng Java (DTO) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Số câu SQL** | 1 | 1 | 1 + số lô batch | 2 (hoặc nhiều hơn nếu tách nhiều Collection) | 2 |
| **Dữ liệu trả về** | Entity (đã Eager) | Entity (đã Eager) | Entity (LAZY, tự động batch khi chạm tới) | Entity nguyên vẹn (đã nhồi đủ dữ liệu con) | DTO thuần (chỉ đúng field cần) |
| **An toàn với Pagination?** | **Không** (HHH000104) | **Không** (cùng cơ chế) | **Có** — không dùng JOIN nên không có Cartesian Product | **Có** — chính là giải pháp chuẩn cho Pagination | **Có** |
| **Nguy cơ Cartesian Product?** | Có, nếu Fetch >1 Collection | Có, nếu Fetch >1 Collection | **Không** — dùng `IN`, không `JOIN` | Không — mỗi bước chỉ Fetch đúng 1 Collection | Không |
| **An toàn khi tắt OSIV?** | Có (đã Eager sẵn trong Transaction) | Có (đã Eager sẵn trong Transaction) | **Dễ dính `LazyInitializationException`** nếu code đọc tiếp Lazy field khác ngoài Service | Có (Entity đã đủ dữ liệu) | **Có — an toàn tuyệt đối**, không đụng Entity Lazy nào |
| **Độ phức tạp code** | Trung bình — viết JPQL | Thấp — chỉ thêm annotation | **Thấp nhất** — cấu hình 1 lần, không sửa code hiện có | Trung bình — viết 2 method Repository | Cao — tự quản lý Stream/Map |
| **Phù hợp nhất khi nào?** | Biết chắc luôn cần dữ liệu quan hệ, không phân trang | Nhiều kịch bản Eager khác nhau cho cùng Entity | Muốn 1 giải pháp mặc định an toàn, áp dụng toàn cục, ít sửa code | Vừa cần Pagination, vừa cần Eager nhiều quan hệ, chấp nhận trả Entity | Cần tối ưu RAM tối đa, trả DTO, hoặc dữ liệu phân mảnh nhiều nguồn |

---

## VII. Cheat Sheet Phỏng vấn (Interview Q&A)

### Q1: Bài toán N+1 là gì? Vì sao nó nguy hiểm?
> **Trả lời**: N+1 là hiện tượng ORM chạy 1 câu SELECT để lấy N bản ghi cha, sau đó chạy thêm N câu SELECT riêng biệt (mỗi bản ghi cha 1 câu) để lấy dữ liệu con — do quan hệ Collection mặc định `LAZY`, chỉ khi code thực sự truy cập vào field quan hệ (ví dụ trong vòng lặp `for`) thì Hibernate mới bắn SELECT. Nó nguy hiểm vì hoàn toàn "vô hình" với dữ liệu test nhỏ, nhưng với N lớn (hàng nghìn bản ghi), số round-trip mạng tăng tuyến tính theo N, dễ khiến hệ thống timeout hoặc sập khi lên Production.

### Q2: Vì sao không nên đổi `FetchType.EAGER` toàn cục để né N+1?
> **Trả lời**: `EAGER` toàn cục khiến **mọi** truy vấn Entity đó — kể cả các màn hình chỉ cần dữ liệu cha — đều bị bắt buộc `JOIN` và kéo theo toàn bộ dữ liệu con lên RAM, dù không có nhu cầu. Ngoài lãng phí tài nguyên, nó còn dễ dính Cartesian Product khi Entity có nhiều hơn 1 quan hệ Collection. Giải pháp đúng là giữ `LAZY` làm mặc định, rồi ép Eager có chủ đích cho từng use-case cụ thể bằng `JOIN FETCH`/`@EntityGraph`.

### Q3: Phân biệt `JOIN FETCH` và `@EntityGraph`.
> **Trả lời**: Cả hai đều dịch thành `LEFT JOIN` dưới DB và giải quyết N+1 theo cùng nguyên lý ép Eager trong 1 câu SELECT. Khác biệt: `JOIN FETCH` viết trong JPQL/HQL tường minh — phù hợp khi logic Query đã phức tạp sẵn. `@EntityGraph` khai báo bằng annotation ngay trên method Repository (hoặc tái sử dụng qua `@NamedEntityGraph` trên Entity) — phù hợp khi cùng 1 Entity có nhiều kịch bản Eager khác nhau, tránh phải viết nhiều câu JPQL trùng lặp.

### Q4: Cartesian Product xảy ra khi nào? Vì sao Hibernate ném `MultipleBagFetchException`?
> **Trả lời**: Xảy ra khi `JOIN FETCH`/`@EntityGraph` được dùng để tải **từ 2 quan hệ Collection trở lên** cùng lúc — DB sẽ nhân chéo dữ liệu giữa các bảng con (ví dụ 40 Nhân viên × 10 Dự án = 400 dòng cho đúng 1 Phòng ban), gây lãng phí băng thông và rủi ro `OutOfMemoryError`. Hibernate chủ động ném `MultipleBagFetchException` khi phát hiện bạn cố Fetch từ 2 `List` (Bag) trở lên cùng lúc, nhằm chặn đứng nguy cơ dữ liệu bị nhân bản sai trước khi nó âm thầm xảy ra trong Production.

### Q5: Đổi `List` sang `Set` để né `MultipleBagFetchException` có thực sự giải quyết được vấn đề không?
> **Trả lời**: Không triệt để. Đổi sang `Set` chỉ giúp Hibernate *chấp nhận chạy* câu Query mà không ném exception, nhưng bản chất Cartesian Product ở tầng SQL vật lý (số dòng dữ liệu thô bị nhân bản) **vẫn xảy ra y hệt** — `Set` chỉ lọc trùng ở bước cuối khi dựng Object Java. Nếu Entity không override đúng `equals()`/`hashCode()`, việc lọc trùng còn có thể cho ra kết quả sai lệch âm thầm.

### Q6: Cảnh báo `HHH000104` là gì? Vì sao không nên kết hợp `JOIN FETCH` với `Pageable`?
> **Trả lời**: `HHH000104` cảnh báo rằng Hibernate đang phân trang **trong bộ nhớ (in-memory)** thay vì ở tầng DB. Nguyên nhân: khi `JOIN FETCH` một quan hệ Collection, dữ liệu dòng cha bị lặp theo số dòng con, nên áp `LIMIT`/`OFFSET` trực tiếp xuống SQL sẽ cắt sai giữa chừng 1 bản ghi cha. Để tránh sai lệch, Hibernate bỏ hẳn `LIMIT` ở SQL, kéo **toàn bộ** kết quả JOIN lên RAM rồi mới cắt trang bằng Java — với bảng dữ liệu lớn, điều này tương đương tải nguyên bảng lên RAM chỉ để lấy 1 trang nhỏ, cực kỳ nguy hiểm.

### Q7: Giải pháp triệt để cho bài toán vừa cần Phân trang, vừa cần tránh N+1 là gì?
> **Trả lời**: Truy vấn 2 bước. Bước 1: Query thuần túy chỉ lấy danh sách ID theo đúng `Pageable` (không kèm bất kỳ `JOIN FETCH` nào, nên `LIMIT`/`OFFSET` chạy chuẩn xác dưới DB). Bước 2: Dùng tập ID đó, chạy tiếp 1 câu `JOIN FETCH`/`@EntityGraph` theo điều kiện `WHERE id IN (:ids)` — vì không còn `Pageable` ở bước này, Hibernate không cần phân trang trên RAM nữa. Nếu cần Eager nhiều hơn 1 quan hệ Collection, tách tiếp thành nhiều câu Query "làm giàu" theo cùng tập ID — First-Level Cache (Identity Map) sẽ tự động gộp dữ liệu vào đúng Object đã có từ Bước 1.

### Q8: OSIV (Open Session In View) là gì? Vì sao nhiều dự án khuyến nghị tắt nó đi?
> **Trả lời**: OSIV là tính năng mặc định bật (`spring.jpa.open-in-view=true`) trong Spring Boot, giữ Session/Connection Database mở kéo dài từ tầng Web cho tới khi response hoàn tất — dài hơn hẳn phạm vi 1 `@Transactional`. Nhờ vậy field Lazy vẫn "load được" dù Entity đã ra khỏi Service, tạo ảo giác an toàn nhưng dung túng cho việc viết code sai kiến trúc (Entity rò rỉ ra Controller) và vắt kiệt Connection Pool khi tải cao. Khuyến nghị chuẩn mực là tắt OSIV (`open-in-view: false`) để `LazyInitializationException` lộ diện ngay lúc code, buộc lập trình viên phải chuyển đổi Entity sang DTO đúng trong tầng Service.

### Q9: So sánh 3 cách giải quyết N+1 khi vừa cần Pagination, vừa cần tránh N+1: `@BatchSize`, Truy vấn 2 bước (dựa Identity Map), và Tách truy vấn thủ công bằng Java.
> **Trả lời**: `@BatchSize` dễ áp dụng nhất (cấu hình 1 lần, không sửa code Lazy hiện có), trả về Entity, nhưng vẫn có nguy cơ dính `LazyInitializationException` nếu code đọc tiếp field Lazy khác ngoài phạm vi Service khi đã tắt OSIV. Truy vấn 2 bước dựa vào Identity Map trả về Entity nguyên vẹn, an toàn tuyệt đối với Pagination, phù hợp khi cần giữ nguyên Entity để xử lý logic nghiệp vụ phức tạp tiếp theo. Tách truy vấn thủ công bằng Java tốn công viết code nhất (tự quản lý Stream/Map) nhưng tối ưu RAM tối đa vì trả thẳng DTO, không đụng tới Entity Lazy nào — an toàn tuyệt đối dù OSIV bật hay tắt, phù hợp khi hệ thống cần hiệu năng cao hoặc dữ liệu phân mảnh nhiều nguồn.

---

> **Tài liệu tham khảo:**
> - [Tiêu diệt "sát thủ" N+1 Query trong Spring Data JPA: Từ bắt bệnh đến kê đơn triệt để - Viblo](https://viblo.asia/p/tieu-diet-sat-thu-n1-query-trong-spring-data-jpa-tu-bat-benh-den-ke-don-triet-de-kNLr3K8bVgA)
> - [Vấn đề N+1 câu truy vấn trong Hibernate - Viblo](https://viblo.asia/p/van-de-n1-cau-truy-van-trong-hibernate-bWrZn00b5xw)
> - [The best way to fix the Hibernate "firstResult/maxResults specified with collection fetch" warning - Vlad Mihalcea](https://vladmihalcea.com/fix-hibernate-hhh000104-entity-fetch-pagination-warning-message/)
