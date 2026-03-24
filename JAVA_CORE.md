# JAVA CORE

![Status](https://img.shields.io/badge/JavaCore-orange) ![Topic](https://img.shields.io/badge/Topic-Learn-blue)

## 4 Tính Chất Quan Trọng Của OOP

> Nguồn: [JavaHighlight](https://javahighlight.com/java/lap-trinh-huong-doi-tuong-trong-java)

### 1. Tính đóng gói (Encapsulation)

Là việc đóng gói dữ liệu (các biến) và các phương thức thao tác trên dữ liệu đó trong một lớp, nhằm che giấu thông tin bên trong lớp và chỉ cung cấp những gì cần thiết thông qua các phương thức công khai (public methods).

---

### 2. Tính kế thừa (Inheritance)

Kế thừa trong java là sự liên quan giữa hai class với nhau, trong đó có class cha (superclass) và class con (subclass). Khi kế thừa class con được hưởng tất cả các phương thức và thuộc tính của class cha. Tuy nhiên, nó chỉ được truy cập các thành viên **public** và **protected** của class cha. Nó không được phép truy cập đến thành viên **private** của class cha.

#### Các kiểu kế thừa trong java

- Các kiểu kế thừa theo lớp

    - *Đơn kế thừa (Single Inheritance)*: Một lớp chỉ có thể kế thừa từ một lớp cha duy nhất. Đây là kiểu kế thừa phổ biến nhất trong Java.

    - *Kế thừa đa cấp (Multilevel Inheritance)*: Một lớp kế thừa từ một lớp cha, lớp cha này lại kế thừa từ một lớp cha khác và cứ thế. Tạo thành một chuỗi kế thừa nhiều cấp.

    - *Kế thừa thứ bậc (Hierarchical Inheritance)*: Nhiều lớp con kế thừa từ cùng một lớp cha.

#### Tại sao Java không hỗ trợ đa kế thừa trực tiếp qua lớp?

Một trong những vấn đề nổi bật nhất là "Diamond Problem" (Vấn đề kim cương):

- Giả sử bạn có 4 lớp: A, B, C, và D.

- Lớp B và C đều kế thừa từ lớp A.

- Lớp D kế thừa cả lớp B và lớp C.

Trong trường hợp này, lớp D sẽ có hai bản sao của tất cả các phương thức và biến từ lớp A (thông qua lớp B và lớp C). Nếu lớp A có một phương thức, lớp D sẽ không biết chính xác phải sử dụng phiên bản nào của phương thức đó - phiên bản từ B hay từ C? Điều này dẫn đến sự mơ hồ và xung đột.

![alt text](image/diamond_problem.png)

**=> Giải pháp:** 

- Đa kế thừa qua Interface

- Sử dụng các lớp trừu tượng

---

### 3. Tính đa hình (Polymorphism)

Tính đa hình cho phép một đối tượng có thể thực hiện các hành động (phương thức) theo nhiều cách khác nhau. Điều này có nghĩa là cùng một giao diện hoặc phương thức, nhưng các đối tượng khác nhau có thể triển khai và thực hiện hành động đó theo cách riêng của mình.

---

### 4. Tính trừu tượng (Abstraction)

Trừu tượng (Abstraction) là quá trình ẩn đi chi tiết triển khai và chỉ hiển thị chức năng cho người dùng. Nói cách khác, nó chỉ cho người dùng thấy những phần quan trọng và giấu đi các chi tiết bên trong. Trừu tượng giúp bạn tập trung vào những gì đối tượng làm thay vì cách nó làm.

---

## String Class

> Nguồn: [JavaHighlight](https://javahighlight.com/java/lop-string-trong-java)

### StringBuffer

`StringBuffer` là một lớp trong Java đại diện cho một chuỗi ký tự có thể thay đổi. Khác với lớp String là bất biến (không thể thay đổi), `StringBuffer` cho phép bạn sửa đổi nội dung của chuỗi mà không cần tạo ra một đối tượng mới mỗi lần.

#### Đặc điểm

- An toàn cho đa luồng: Các phương thức của `StringBuffer` được đồng bộ hóa, đảm bảo rằng các hoạt động trên cùng một đối tượng `StringBuffer` từ nhiều luồng khác nhau diễn ra theo một thứ tự nhất quán.

- Có thể thay đổi: Bạn có thể thêm, xóa hoặc thay thế các ký tự trong một đối tượng `StringBuffer`.

- Có dung lượng: Mỗi `StringBuffer` có một dung lượng. Nếu chuỗi vượt quá dung lượng, bộ nhớ sẽ tự động được mở rộng.

#### So sánh `String` & `StringBuffer`

| Tiêu chí | `String` | `StringBuffer` |
|-------|-----------|------------|
| **Tính chất** | Bất biến (immutable) | Có thể thay đổi (mutable) |
| **Hiệu suất thay đổi** |Kém hiệu quả với các thao tác thay đổi nhiều lần vì tạo ra đối tượng mới mỗi lần thay đổi. | Hiệu quả hơn với các thao tác thay đổi nhiều lần vì thay đổi trực tiếp trên đối tượng hiện tại. |
| **An toàn trong đa luồng** | An toàn (thread-safe) vì là bất biến. | An toàn (thread-safe) vì các phương thức được đồng bộ hóa. |
| **Khả năng mở rộng** | Không mở rộng kích thước (đối tượng không thay đổi). | Tự động mở rộng kích thước khi cần. |
| **Phương thức thay đổi** | Không có phương thức để thay đổi nội dung. | Cung cấp các phương thức như append(), insert(), delete(), replace(). |
| **Sử dụng trong đa luồng** | Thích hợp khi làm việc với chuỗi không thay đổi. | Thích hợp khi làm việc với chuỗi thay đổi trong môi trường đa luồng. |
| **Khả năng tối ưu hóa** | Tối ưu cho các thao tác với chuỗi bất biến. | Tối ưu cho các thao tác thay đổi chuỗi. |
| **Khả năng điều chỉnh kích thước** | Không hỗ trợ điều chỉnh kích thước. | Hỗ trợ điều chỉnh kích thước tự động. |
| **Tạo đối tượng mới** | 	Tạo đối tượng mới khi thực hiện thao tác thay đổi. | Không tạo đối tượng mới khi thay đổi nội dung. |

#### Các phương thức của `StringBuffer`

- `append()`: thêm dữ liệu vào cuối chuỗi của lớp `StringBuffer`.

- `delete()`: xóa ký tự hoặc chuỗi của lớp `StringBuffer`.

- `insert()`: chèn dữ liệu vào một vị trí cụ thể của lớp `StringBuffer`.

- `replace(startIndex, endIndex, replacement)` / `setCharAt(int index, char ch)`: thay đổi đoạn / kí tự.

- `reverse()`: Đảo ngược toàn bộ thứ tự các ký tự trong chuỗi `StringBuffer`.

---

### StringBuilder

`StringBuilder` là một lớp trong Java đại diện cho một chuỗi ký tự có thể thay đổi. Nó tương tự như `StringBuffer` nhưng **không đảm bảo tính đồng bộ**, điều này làm cho nó **nhanh hơn** trong môi trường đơn luồng.

#### Đặc điểm

- Có thể thay đổi: Bạn có thể thêm, xóa hoặc thay thế các ký tự trong một đối tượng `StringBuilder`.

- Không an toàn cho đa luồng: Các phương thức của `StringBuilder` không được đồng bộ hóa, vì vậy không nên sử dụng nó trong môi trường đa luồng.

- Có dung lượng: Mỗi `StringBuilder` có một dung lượng. Nếu chuỗi vượt quá dung lượng, bộ nhớ sẽ tự động được mở rộng.

#### So sánh `StringBuffer` & `StringBuilder`

**Đồng bộ hóa (Synchronization):**

- `StringBuffer`: Là một lớp đồng bộ hóa (synchronized). Điều này có nghĩa là các phương thức của `StringBuffer` được thiết kế để an toàn khi sử dụng trong môi trường đa luồng. Khi nhiều luồng cùng truy cập và sửa đổi một đối tượng `StringBuffer`, các phương thức sẽ được thực thi một cách tuần tự, tránh xung đột dữ liệu.

- `StringBuilder`: Không đồng bộ. Điều này làm cho `StringBuilder` hiệu quả hơn `StringBuffer` trong các môi trường đơn luồng, nhưng không an toàn khi sử dụng trong môi trường đa luồng. Nếu nhiều luồng cùng truy cập và sửa đổi một đối tượng `StringBuilder`, có thể dẫn đến các vấn đề về đồng bộ hóa và lỗi không mong muốn.

## Phương Thức `equals()` & `hashCode()` trong Java

> Nguồn: [Viblo](https://viblo.asia/p/phuong-thuc-equals-hashcode-trong-java-tim-hieu-chi-tiet-K9Vy8XyaLQR)

Trong lập trình **Java**, phương thức `equals()` và phương thức `hashCode()` là hai phương thức quan trọng thuộc lớp **Object**, đóng vai trò thiết yếu trong việc so sánh các đối tượng và quản lý dữ liệu trong các cấu trúc như `HashMap`, `HashSet`, hay `Hashtable`.

---

### Phương thức `equals()` trong Java là gì?

Phương thức `equals()` được định nghĩa trong lớp Object và được sử dụng để so sánh xem hai đối tượng có "bằng nhau" hay không. 

Theo mặc định, `equals()` trong lớp Object so sánh **tham chiếu (reference)** của hai đối tượng, tức là kiểm tra xem chúng có cùng **địa chỉ bộ nhớ** hay không.

Tuy nhiên, trong thực tế, bạn thường cần **ghi đè (override)** phương thức `equals()` để so sánh dựa trên nội dung của đối tượng thay vì tham chiếu. 

Ví dụ:

```java
public class Student {
    private int id;
    private String name;

    public Student(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Student student = (Student) obj;    // Ép kiểu cho obj về Student
        return id == student.id && name.equals(student.name);
    }
}
```

---

### Phương thức `hashCode()` trong Java là gì?

Phương thức `hashCode()` cũng thuộc lớp Object và mặc định trả về một giá trị *số nguyên* (hash code) đại diện cho đối tượng. Giá trị này thường được sử dụng trong các cấu trúc dữ liệu dựa trên bảng băm như `HashMap` hay `HashSet` để xác định vị trí lưu trữ của đối tượng.

Cú pháp: `public native int hashCode();` 

Khi bạn ghi đè `equals()`, bạn cũng cần ghi đè `hashCode()` để đảm bảo tính nhất quán theo **hợp đồng hashCode-equals** trong Java:

- Nếu hai đối tượng được coi là bằng nhau qua `equals()`, thì chúng phải có cùng giá trị `hashCode()`.

- Nếu hai đối tượng có cùng `hashCode()`, chúng không nhất thiết phải bằng nhau qua `equals()`.

- `hashCode()` nên trả về giá trị cố định cho một đối tượng không thay đổi trong suốt vòng đời của nó.

Ví dụ:

```java
@Override
public int hashCode() {
    int result = 17; // Số nguyên tố ban đầu
    result = 31 * result + id; // Kết hợp id
    result = 31 * result + name.hashCode(); // Kết hợp name
    return result;
}
```

---

### Tại sao cần ghi đè `equals()` và `hashCode()`?

Việc ghi đè hai phương thức này là cần thiết khi bạn sử dụng các đối tượng trong các cấu trúc dữ liệu như `HashMap`, `HashSet`, hoặc khi bạn muốn so sánh các đối tượng dựa trên nội dung thay vì tham chiếu. Nếu không ghi đè:

- `equals()` chỉ so sánh tham chiếu, dẫn đến kết quả không mong muốn.

- `hashCode()` không nhất quán với `equals()`, gây ra lỗi khi sử dụng `HashMap` hoặc `HashSet`.

Ví dụ, nếu bạn không ghi đè `hashCode()` mà chỉ ghi đè `equals()`, hai đối tượng bằng nhau theo `equals()` có thể được lưu ở các vị trí khác nhau trong `HashMap`, làm mất dữ liệu hoặc gây lỗi truy xuất.

---

### Cách triển khai `equals()` và `hashCode()` hiệu quả

- Kiểm tra tham chiếu trong `equals()`: Sử dụng `this == obj` để tăng hiệu suất khi hai đối tượng cùng trỏ đến một địa chỉ bộ nhớ.

    ```java
    if (this == obj) return true;
    ```

- Kiểm tra `null` và kiểu đối tượng: Đảm bảo đối tượng không null và thuộc cùng lớp trước khi so sánh.

    ```java
    if (obj == null || getClass() != obj.getClass()) return false;
    ```
    
- Sử dụng **số nguyên tố** trong `hashCode()`: Sử dụng các số nguyên tố như **17** và **31** để giảm xung đột hash. Tận dụng thư viện: Sử dụng `Objects.hash()` (Java 7 trở lên) để tạo `hashCode()` nhanh chóng:

    ```java
    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
    ```

---

### Các lỗi phổ biến khi triển khai `equals()` và `hashCode()`

- Quên ghi đè cả hai phương thức: Nếu chỉ ghi đè `equals()` mà không ghi đè `hashCode()`, bạn sẽ vi phạm hợp đồng **hashCode-equals**.

- Sử dụng thuộc tính thay đổi trong `hashCode()`: Nếu sử dụng các thuộc tính có thể thay đổi (như `List` hoặc `Array`), giá trị `hashCode()` sẽ thay đổi, gây lỗi trong **HashMap**.

- Hiệu suất kém: Tạo `hashCode()` phức tạp hoặc sử dụng thuật toán không tối ưu có thể làm giảm hiệu suất.

---

### Ứng dụng thực tế của `equals()` và `hashCode()` 

- `HashMap`/`HashSet`: Hai phương thức này được sử dụng để xác định khóa (key) hoặc phần tử duy nhất.

- So sánh đối tượng: Dùng trong các ứng dụng cần kiểm tra sự giống nhau của dữ liệu, như kiểm tra thông tin người dùng.

- Tối ưu hóa hiệu suất: Một `hashCode()` tốt giúp giảm xung đột trong bảng băm, cải thiện tốc độ truy xuất.

## Collections