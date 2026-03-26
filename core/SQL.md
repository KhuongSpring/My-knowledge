# Learn SQL

## Các câu hỏi thường gặp về SQL

### 1. SQL là gì?

SQL là viết tắt của *Structured Query Language* - ngôn ngữ truy vấn có cấu trúc. Nó được thiết kế để quản lý dữ liệu trong một hệ quản trị cơ sở dữ liệu quan hệ (*RDBMS*). SQL được dùng để tạo, xóa, truy vấn và sửa đổi dữ liệu.

### 2. Định nghĩa `JOIN` và các loại `JOIN`

Từ khóa `JOIN` được dùng để nạp dữ liệu từ 2 hay nhiều bảng liên quan. Sử dụng `JOIN` khi cần truy vấn các cột dữ liệu từ nhiều bảng khác nhau để trả về trong cùng một tập kết quả.

Các loại `JOIN`:

- **INNER JOIN**
  - Lấy các bản ghi trùng khớp ở cả hai bảng theo điều kiện `JOIN`.
  - Nếu không có giá trị trùng khớp, dòng đó sẽ không xuất hiện.
  - Đây là loại `JOIN` phổ biến nhất.
  - Phù hợp khi chỉ quan tâm đến dữ liệu khớp nhau giữa các bảng.

- **LEFT OUTER JOIN**
  - Lấy toàn bộ bản ghi từ bảng bên trái và các bản ghi khớp ở bảng bên phải.
  - Nếu không có bản ghi khớp, các cột từ bảng bên phải sẽ là `NULL`.
  - Phù hợp khi cần tất cả dữ liệu từ bảng chính và thêm thông tin bổ sung nếu có.

- **RIGHT OUTER JOIN**
  - Ngược lại với `LEFT JOIN`.

- **FULL OUTER JOIN**
  - Kết hợp cả `LEFT JOIN` và `RIGHT JOIN`.
  - Lấy tất cả bản ghi từ cả hai bảng, ghép nếu có khớp, nếu không thì điền `NULL`.
  - Không được hỗ trợ trực tiếp bởi một số hệ quản trị như MySQL, thường phải mô phỏng bằng `UNION`.

- **CROSS JOIN**
  - Lấy tích Descartes của hai bảng.
  - Mỗi bản ghi của bảng A sẽ kết hợp với tất cả bản ghi của bảng B.
  - Không có điều kiện `ON`.
  - Cần cẩn thận khi bảng lớn vì số lượng kết quả có thể tăng rất mạnh.

- **SELF JOIN**
  - `JOIN` một bảng với chính nó.
  - Hữu ích khi làm việc với quan hệ phân cấp.
  - Cần dùng alias để phân biệt các bản ghi của cùng một bảng.

### 3. `CHECK Constraint`

`CHECK` là một ràng buộc dùng để giới hạn các giá trị hoặc kiểu dữ liệu có thể được lưu trữ trong một cột. Nếu bản ghi không đáp ứng điều kiện thì sẽ không được lưu vào bảng.

### 4. `BOOLEAN`

Đối với trường dữ liệu `BOOLEAN`, thường có 2 giá trị: `TRUE` và `FALSE`. Một số hệ quản trị có thể biểu diễn tương ứng bằng `1/0` hoặc `-1/0` tùy hệ thống.

### 5. DML và DDL

- **DML** (*Data Manipulation Language*): `INSERT`, `UPDATE`, `DELETE`
- **DDL** (*Data Definition Language*): `CREATE`, `ALTER`, `DROP`, `RENAME`

### 6. Thứ tự của `SQL SELECT`

Thứ tự các mệnh đề khi viết thường là:

`SELECT -> FROM -> WHERE -> GROUP BY -> HAVING -> ORDER BY`

Nhưng thứ tự thực thi thực tế thường là:

1. `FROM`: lấy bảng
2. `ON`: lấy điều kiện `JOIN`
3. `JOIN`: kết hợp bảng
4. `WHERE`: lọc theo điều kiện
5. `GROUP BY`: gom nhóm dữ liệu
6. `HAVING`: lọc dữ liệu sau khi đã nhóm
7. `SELECT`: lấy dữ liệu cần thiết
8. `DISTINCT`: loại bỏ dữ liệu trùng lặp
9. `ORDER BY`: sắp xếp dữ liệu
10. `LIMIT / OFFSET`: giới hạn dữ liệu đầu ra

### 7. Sự khác biệt giữa `TRUNCATE`, `DELETE` và `DROP`

- **DELETE**: xóa một hoặc tất cả các hàng từ một bảng dựa trên điều kiện, thường có thể rollback nếu đang trong transaction phù hợp.
- **TRUNCATE**: xóa tất cả các hàng của bảng nhanh hơn `DELETE`, thường không dùng điều kiện `WHERE`.
- **DROP**: xóa hoàn toàn bảng khỏi cơ sở dữ liệu.

### 8. Các thuộc tính của một giao dịch

Được gọi là **ACID**, bao gồm:

- Atomicity: tính nguyên tử
- Consistency: tính nhất quán
- Isolation: tính cô lập
- Durability: độ bền

### 9. `UNION`, `MINUS`, `UNION ALL`, `INTERSECT` là gì?

Đây là các *set operations* (phép toán tập hợp), thường dùng để kết hợp hoặc so sánh kết quả của nhiều câu truy vấn `SELECT`.

- **UNION**
  - Kết hợp hai hoặc nhiều câu lệnh `SELECT`.
  - Loại bỏ các bản ghi trùng lặp.
  - Các cột của các truy vấn con phải tương ứng về số lượng và kiểu dữ liệu.
  - Có thể tốn tài nguyên vì phải xử lý loại bỏ dữ liệu trùng.

- **UNION ALL**
  - Tương tự `UNION` nhưng giữ lại các bản ghi trùng lặp.
  - Thường nhanh hơn `UNION` vì không thực hiện lọc trùng.

- **INTERSECT**
  - Trả về các bản ghi chung giữa hai tập kết quả `SELECT`.
  - Tương đương phép giao trong toán học.
  - Một số hệ quản trị như MySQL không hỗ trợ trực tiếp.

- **MINUS** hoặc **EXCEPT**
  - Trả về bản ghi từ câu truy vấn đầu tiên nhưng không xuất hiện trong câu truy vấn thứ hai.
  - Oracle dùng `MINUS`.
  - PostgreSQL và SQL Server dùng `EXCEPT`.
  - MySQL không hỗ trợ trực tiếp, thường phải mô phỏng bằng `LEFT JOIN ... WHERE ... IS NULL`.

### 10. Sự khác biệt giữa `WHERE` và `HAVING`

Cả hai mệnh đề đều chỉ định điều kiện tìm kiếm:

- `WHERE` lọc dữ liệu trước khi nhóm.
- `HAVING` lọc dữ liệu sau khi đã `GROUP BY`.

Nếu không có `GROUP BY`, `HAVING` vẫn có thể dùng trong một số hệ quản trị, nhưng thông thường `WHERE` là lựa chọn phù hợp hơn khi chỉ cần lọc dữ liệu gốc.

## Kiến thức tối ưu và quan trọng

### 1. Các cách tối ưu truy vấn chung

- Thay vì `SELECT * FROM`, hãy chọn các cột cụ thể cần dùng.
- Nếu biết chỉ có một kết quả truy vấn, hãy dùng `LIMIT 1` để tránh quét tài nguyên không cần thiết.
- Hạn chế sử dụng `OR` trong mệnh đề `WHERE` vì có thể làm giảm hiệu quả của **index**.
- Có thể cân nhắc dùng `UNION ALL` thay thế trong một số trường hợp.
- Sử dụng `WHERE` để:
  - giới hạn dữ liệu, tránh trả lại row thừa
  - thay thế `HAVING` khi không cần lọc sau bước gom nhóm
  - ưu tiên điều kiện rõ ràng và sớm để DB tối ưu tốt hơn
- Có thể dùng `>` hoặc `<` thay vì `>=` hoặc `<=` trong một số bài toán nếu phù hợp nghiệp vụ.
- Dùng `BETWEEN` khi cần so sánh khoảng giá trị.
- Tránh dùng `!=` hoặc `<>` nếu có thể vì đôi khi làm giảm hiệu quả của **index**.
- Dùng `DISTINCT` cẩn trọng:
  - chỉ nên dùng khi thực sự cần loại trùng
  - đôi khi có thể thay bằng thiết kế truy vấn rõ hơn
- Sử dụng giá trị mặc định thay vì `NULL` trong một số trường hợp để giảm phức tạp truy vấn.

  ```sql
  SELECT * FROM user WHERE age > 0;
  ```

- Có thể dùng `ENUM` thay cho `VARCHAR` nếu bài toán phù hợp và hệ quản trị hỗ trợ.
- Giảm thiểu số lượng truy vấn phụ nếu không cần thiết.
- `EXISTS` và `IN`:
  - `EXISTS`: phù hợp khi muốn kiểm tra sự tồn tại và điều kiện lọc chính nằm ở query ngoài
  - `IN`: phù hợp khi so sánh một giá trị với một tập giá trị
- Đánh **index** ở các cột thường dùng trong `WHERE`, `ORDER BY`, `GROUP BY`.
- Tránh dùng `LIKE '%keyword'` nếu có thể vì thường khó tận dụng index.
- Tránh áp dụng **function** trực tiếp lên cột đã đánh **index** vì có thể làm mất hiệu lực của index.
- Tối ưu schema:
  - chọn kiểu dữ liệu gọn, đúng mục đích
  - tránh `NULL` nếu không cần thiết

### 2. Index trong SQL

- Khái niệm: `Index` là cấu trúc dữ liệu giúp tăng tốc truy vấn bằng cách giảm số dòng phải quét trong bảng.

#### Các loại `Index` phổ biến

- **B-Tree Index**: mặc định trong nhiều DBMS như MySQL, PostgreSQL; tốt cho `=`, `>`, `<`, `BETWEEN`
- **Hash Index**: nhanh cho `=`, nhưng không hỗ trợ range query tốt
- **Full-text Index**: hỗ trợ tìm kiếm văn bản
- **Composite Index**: chỉ mục kết hợp nhiều cột
- **Unique Index**: đảm bảo giá trị duy nhất
- **Bitmap Index**: phù hợp với dữ liệu có ít giá trị phân biệt

#### Cách hoạt động

- Duyệt theo cấu trúc như **B-Tree** để tìm giá trị.
- Nếu có index phù hợp, DBMS sẽ dùng `index scan` thay vì `full table scan`.
- Nếu index không phù hợp thì hệ thống có thể bỏ qua.

#### Lưu ý

- `INSERT` hoặc `UPDATE` nhiều có thể chậm hơn do phải cập nhật index.
- Không nên tạo quá nhiều index vì chi phí bảo trì cao.
- Dùng `EXPLAIN` để biết truy vấn có dùng index hay không.

#### Khi nào nên sử dụng `Index`?

Có một số quy luật thường thấy khi chọn một cột hoặc tập cột để tạo index:

1. **Khóa và cột unique**: database thường tự tạo index, nên tránh tạo thêm nếu không cần.
2. **Tần suất được sử dụng**: truy vấn càng chạy nhiều, lợi ích của index càng rõ.
3. **Số lượng bản ghi lớn**: bảng càng lớn thì index càng có giá trị.
4. **Dữ liệu tăng trưởng nhanh**: bảng cập nhật liên tục nên hạn chế quá nhiều index vì sẽ làm chậm ghi dữ liệu.
5. **Không gian bộ nhớ**: index tiêu tốn bộ nhớ, nên cần chọn lọc.
6. **Độ đa dạng dữ liệu**: cột có tính phân biệt tốt thường tận dụng index tốt hơn.

#### Cách tạo `Index` trong PostgreSQL

```sql
CREATE INDEX index_name ON table_name (column1, column2, ...);
```

### 3. Phân tích truy vấn với `EXPLAIN` / `EXPLAIN ANALYZE`

- Khái niệm: cung cấp kế hoạch thực thi của câu lệnh `SELECT`, cho biết DB sẽ:
  - quét bao nhiêu dòng
  - dùng index nào
  - có đang full scan bảng hay không

- Ví dụ:

  ```sql
  EXPLAIN SELECT * FROM users WHERE age > 30;
  ```

- Thường trả về các thông tin như:
  - `type`: kiểu truy vấn như `ALL`, `index`, `range`, `ref`, `const`, `eq_ref`
  - `possible_keys`: các index có thể dùng
  - `key`: index thực sự được dùng
  - `rows`: số dòng ước tính cần quét

- `EXPLAIN ANALYZE` trong PostgreSQL cho thêm thời gian thực thi thực tế của từng bước.

- Một số lưu ý:
  - `type = ALL`: full table scan, thường cần tối ưu
  - `Using where`: truy vấn có lọc dữ liệu nhưng có thể chưa dùng index
  - `Using index`: dùng covering index, không cần đọc thêm table

### 4. Tối ưu `LIMIT` và `OFFSET`

- Vấn đề: Khi dùng `LIMIT 10 OFFSET 10000`, DB vẫn phải quét 10010 dòng rồi mới trả về 10 dòng cuối.
- Giải pháp: dùng **seek method** hoặc **keyset pagination** dựa trên ID.

```sql
SELECT * FROM products
WHERE id > last_seen_id
ORDER BY id
LIMIT 10;
```

- Cách này tận dụng index để tăng tốc phân trang.
- Không nên dùng `OFFSET` quá lớn nếu không thực sự cần thiết.

### 5. Partition

- Khái niệm: chia bảng lớn thành nhiều phần (*partition*), giúp tối ưu truy vấn, đặc biệt với dữ liệu lớn.

#### Các loại partition

- **Range**: theo khoảng, ví dụ năm hoặc tháng
- **List**: theo danh sách giá trị
- **Hash**: theo hàm băm
- **Composite**: kết hợp nhiều kiểu

#### Khi nào nên dùng partition

- Bảng rất lớn, ví dụ hàng triệu dòng.
- Truy vấn thường xuyên theo mốc thời gian như ngày, tháng, năm.

### 6. Caching ở tầng database

- Khái niệm: lưu tạm kết quả truy vấn ở RAM để dùng lại nhanh hơn.

#### Các hình thức caching

- **Query Cache (DB)**: lưu kết quả truy vấn `SELECT`
- **Materialized View**: bảng lưu kết quả truy vấn theo chu kỳ
- **Application Cache**: cache ở tầng ứng dụng như Redis, Memcached
- **Prepared Statement**: cache kế hoạch thực thi truy vấn

#### Khi nào nên dùng cache

- Hệ thống có nhiều câu lệnh `SELECT` hơn `UPDATE`.
- Câu truy vấn nặng và lặp lại nhiều.
- Dữ liệu không thay đổi thường xuyên.
- Cần giảm tải DB và tăng tốc API.
