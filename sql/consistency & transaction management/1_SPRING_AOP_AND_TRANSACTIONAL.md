# Chương 1: Spring AOP và Bản chất của `@Transactional`

![Status](https://img.shields.io/badge/Spring_AOP-green) ![Topic](https://img.shields.io/badge/Transactional_Internals-blue) ![Level](https://img.shields.io/badge/Level-Deep_Dive-red)

## Mục lục

- [I. Tổng quan \& Đặt vấn đề](#i-tổng-quan--đặt-vấn-đề)
  - [1. Quản lý Transaction trong Java/Spring](#1-quản-lý-transaction-trong-javaspring)
  - [2. Vấn đề cốt lõi: Bản chất thực sự của @Transactional](#2-vấn-đề-cốt-lõi-bản-chất-thực-sự-của-transactional)
- [II. Cơ chế tạo Proxy (JDK Dynamic Proxy vs CGLIB)](#ii-cơ-chế-tạo-proxy-jdk-dynamic-proxy-vs-cglib)
  - [1. Khi nào Spring dùng JDK Proxy vs CGLIB?](#1-khi-nào-spring-dùng-jdk-proxy-vs-cglib)
  - [2. Phân tích chi tiết cơ chế hoạt động \& Bytecode Manipulation](#2-phân-tích-chi-tiết-cơ-chế-hoạt-động--bytecode-manipulation)
  - [3. So sánh hiệu năng \& Hạn chế kỹ thuật của từng loại](#3-so-sánh-hiệu-năng--hạn-chế-kỹ-thuật-của-từng-loại)
  - [4. Luồng tạo Proxy trong vòng đời Spring Bean (BeanPostProcessor)](#4-luồng-tạo-proxy-trong-vòng-đời-spring-bean-beanpostprocessor)
- [III. TransactionInterceptor \& Luồng đi thực tế của Request](#iii-transactioninterceptor--luồng-đi-thực-tế-của-request)
  - [1. Các thành phần tham gia vào luồng xử lý](#1-các-thành-phần-tham-gia-vào-luồng-xử-lý)
  - [2. Luồng thực thi chi tiết từ Request đến Database](#2-luồng-thực-thi-chi-tiết-từ-request-đến-database)
  - [3. Sơ đồ tuần tự (Sequence Diagram)](#3-sơ-đồ-tuần-tự-sequence-diagram)
- [IV. TransactionSynchronizationManager \& ThreadLocal Storage](#iv-transactionsynchronizationmanager--threadlocal-storage)
  - [1. Đặt vấn đề: Bài toán chia sẻ Connection trong chuỗi gọi hàm](#1-đặt-vấn-đề-bài-toán-chia-sẻ-connection-trong-chuỗi-gọi-hàm)
  - [2. Cơ chế ThreadLocal của TransactionSynchronizationManager](#2-cơ-chế-threadlocal-của-transactionsynchronizationmanager)
  - [3. Cách Repository/DAO sử dụng Connection từ ThreadLocal](#3-cách-repositorydao-sử-dụng-connection-từ-threadlocal)
- [V. Vấn đề Self-Invocation \& Giải pháp triệt để](#v-vấn-đề-self-invocation--giải-pháp-triệt-để)
  - [1. Nguyên nhân sâu xa: Con trỏ `this` và cơ chế Bypass Proxy](#1-nguyên-nhân-sâu-xa-con-trỏ-this-và-cơ-chế-bypass-proxy)
  - [2. Giải pháp 1: Self-Injection (Tiêm chính Proxy vào Bean)](#2-giải-pháp-1-self-injection-tiêm-chính-proxy-vào-bean)
  - [3. Giải pháp 2: AspectJ Weaving (Compile-time / Load-time Weaving)](#3-giải-pháp-2-aspectj-weaving-compile-time--load-time-weaving)
  - [4. Giải pháp 3: Refactor tách Service (Architectural Best Practice)](#4-giải-pháp-3-refactor-tách-service-architectural-best-practice)
- [VI. Luật Rollback mặc định \& Triết lý thiết kế (Rollback Rules)](#vi-luật-rollback-mặc-định--triết-lý-thiết-kế-rollback-rules)
  - [1. Triết lý thiết kế: Unchecked vs Checked Exception](#1-triết-lý-thiết-kế-unchecked-vs-checked-exception)
  - [2. Cách cấu hình ghi đè với `rollbackFor` và `noRollbackFor`](#2-cách-cấu-hình-ghi-đè-với-rollbackfor-và-norollbackfor)
  - [3. Bẫy nuốt Exception (Try-Catch Suppression \& UnexpectedRollbackException)](#3-bẫy-nuốt-exception-try-catch-suppression--unexpectedrollbackexception)
- [VII. Cheat Sheet Phỏng vấn (Interview Q\&A)](#vii-cheat-sheet-phỏng-vấn-interview-qa)

---

## I. Tổng quan & Đặt vấn đề

### 1. Quản lý Transaction trong Java/Spring

Trong phát triển phần mềm doanh nghiệp (Enterprise Software), tính toàn vẹn dữ liệu tuân theo nguyên tắc **[ACID](https://vietnix.vn/acid-la-gi/)** (Atomicity, Consistency, Isolation, Durability) là yêu cầu bắt buộc.

Spring Framework cung cấp hai cách tiếp cận chính để quản lý Database Transaction:

#### a. Programmatic Transaction Management (Quản lý bằng code)
Lập trình viên chủ động điều khiển luồng transaction bằng code thủ công:
* **JDBC Thuần**: Gọi `connection.setAutoCommit(false)`, `connection.commit()`, `connection.rollback()`.
* **Spring `TransactionTemplate`**: Dùng callback template để thực thi business logic.

```java
// Ví dụ với TransactionTemplate
transactionTemplate.execute(status -> {
    try {
        userRepository.save(user);
        auditRepository.save(audit);
        return true;
    } catch (Exception e) {
        status.setRollbackOnly();
        throw e;
    }
});
```
* **Ưu điểm**: Kiểm soát chính xác từng dòng code, dễ debug.
* **Nhược điểm**: Boilerplate code nhiều, vi phạm nguyên lý **Separation of Concerns** (trộn lẫn logic nghiệp vụ với logic hạ tầng).

#### b. Declarative Transaction Management (Quản lý bằng khai báo)
Sử dụng annotation `@Transactional` để đánh dấu method hoặc class cần được bao bọc bởi transaction.

```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        auditRepository.save(audit);
    }
}
```
* **Ưu điểm**: Code cực kỳ ngắn gọn, tách biệt hoàn toàn nghiệp vụ và hạ tầng.
* **Nhược điểm**: Nếu không hiểu bản chất bên dưới, lập trình viên rất dễ gặp phải các bug "ngầm" nguy hiểm (không rollback, tự gọi method không chạy transaction,...).

---

### 2. Vấn đề cốt lõi: Bản chất thực sự của `@Transactional`

Annotation `@Transactional` **chỉ là một Metadata Marker** (thẻ đánh dấu dữ liệu). Bản thân annotation này không chứa bất kỳ dòng code thực thi nào về SQL hay JDBC `Connection`.

> **Câu hỏi phỏng vấn kinh điển**: Khi một method được đánh dấu `@Transactional`, điều gì thực sự xảy ra trong JVM khi client gọi tới method đó?

Câu trả lời nằm ở hai trụ cột chính của Spring Framework: **Spring AOP (Aspect-Oriented Programming)** và **Dynamic Proxy Pattern**.

---

## II. Cơ chế tạo Proxy (JDK Dynamic Proxy vs CGLIB)

### 1. Khi nào Spring dùng JDK Proxy vs CGLIB?

Spring AOP quyết định loại Proxy được tạo ra dựa trên cấu hình và bản chất của Target Class:

```
                      [Spring Bean Initialization]
                                   │
                    Is spring.aop.proxy-target-class=true?
                                   │
                      ┌────────────┴────────────┐
                     YES                        NO
                      │                         │
               [Use CGLIB]             Does Class Implement 
                                            Interface?
                                                │
                                    ┌───────────┴───────────┐
                                   YES                     NO
                                    │                       │
                            [Use JDK Proxy]            [Use CGLIB]
```

1. **Mặc định của Spring Framework (Legacy)**:
   - Nếu Target Class **implement ít nhất 1 Interface** $\rightarrow$ Dùng **JDK Dynamic Proxy**.
   - Nếu Target Class **không implement Interface nào** $\rightarrow$ Dùng **CGLIB Proxy**.
2. **Mặc định của Spring Boot (Từ Spring Boot 2.x trở đi)**:
   - Spring Boot tự động bật thuộc tính: `spring.aop.proxy-target-class=true`.
   - Do đó, **CGLIB luôn được ưu tiên sử dụng làm mặc định**, bất kể class đó có implement Interface hay không.

> **Tại sao Spring Boot đổi mặc định sang CGLIB?**
> Khi dùng JDK Dynamic Proxy, Spring Bean đăng ký vào ApplicationContext dưới dạng kiểu Interface (`UserService`), chứ không phải Class triển khai (`UserServiceImpl`). Nếu lập trình viên dùng `@Autowired private UserServiceImpl userService;` (inject bằng concrete class), ứng dụng sẽ bị nổ lỗi `BeanNotOfRequiredTypeException` hoặc `ClassCastException`. CGLIB tạo subclass nên hỗ trợ ép kiểu về cả Class gốc lẫn Interface.

---

### 2. Phân tích chi tiết cơ chế hoạt động & Bytecode Manipulation

#### a. JDK Dynamic Proxy (`java.lang.reflect.Proxy`)
* **Cách thức**: Tạo ra một class mới trong bộ nhớ runtime triển khai cùng các Interface với Target Object.
* **Cơ chế intercept**: Mọi cuộc gọi method thông qua Proxy đều được chuyển hướng tới một `InvocationHandler` duy nhất (trong Spring AOP là `JdkDynamicAopProxy`).

```java
// Mô phỏng cơ chế JDK Dynamic Proxy
public class JdkDynamicAopProxy implements InvocationHandler {
    private final Object target;

    public JdkDynamicAopProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. Logic mở transaction
        System.out.println("Begin Transaction");
        try {
            // 2. Gọi method trên Target Object gốc bằng Reflection
            Object result = method.invoke(target, args);
            // 3. Commit
            System.out.println("Commit Transaction");
            return result;
        } catch (InvocationTargetException e) {
            // 4. Rollback
            System.out.println("Rollback Transaction");
            throw e.getCause();
        }
    }
}
```

#### b. CGLIB Proxy (Code Generation Library)
* **Cách thức**: Sử dụng thư viện can thiệp Bytecode (như **ASM** hoặc **ByteBuddy**) để sinh ra một lớp con (Subclass) kế thừa trực tiếp từ Target Class tại thời điểm runtime.
* **Cơ chế intercept**: Mọi method trong Proxy subclass sẽ override method của class cha và chuyển hướng cuộc gọi qua `MethodInterceptor` (trong Spring AOP là `CglibAopProxy.DynamicAdvisedInterceptor`).
* **Cơ chế FastClass**: CGLIB tạo ra các index cho các method thay vì dùng Reflection thuần túy, giúp tốc độ gọi hàm nhanh hơn.

```java
// Mô phỏng cơ chế CGLIB Proxy (Subclassing)
public class UserServiceImpl$$EnhancerBySpringCGLIB extends UserServiceImpl {
    private MethodInterceptor interceptor;

    @Override
    public void createUser(User user) {
        // Chuyển hướng qua Interceptor xử lý Transaction
        interceptor.intercept(this, methodRef, new Object[]{user}, methodProxy);
    }
}
```

---

### 3. So sánh hiệu năng & Hạn chế kỹ thuật của từng loại

| Tiêu chí | JDK Dynamic Proxy | CGLIB Proxy |
| :--- | :--- | :--- |
| **Bản chất kỹ thuật** | Tạo Proxy class implement Interface qua Java Reflection API. | Sinh bytecode tạo Subclass kế thừa Target Class (ASM/ByteBuddy). |
| **Yêu cầu Class** | Target Class **bắt buộc phải có Interface**. | Target Class **không cần Interface**. |
| **Khả năng Proxy Method** | Chỉ proxy được các method nằm trong Interface. | Proxy được các method public/protected của class cha. |
| **Tốc độ tạo Proxy (Initialization)** | **Nhanh hơn** (Chỉ tạo proxy class bằng Reflection cơ bản). | **Chậm hơn một chút** (Tốn CPU cycle để sinh bytecode & nạp ClassLoader). |
| **Tốc độ thực thi Method (Runtime)** | Lịch sử chậm hơn (do Reflection); trên JVM hiện đại đã được JIT tối ưu tiệm cận CGLIB. | **Nhanh hơn** (Sử dụng cơ chế `FastClass` index để gọi hàm trực tiếp, giảm thiểu Reflection overhead). |
| **Hạn chế kỹ thuật** | - Không proxy được class không có interface.<br>- Không proxy được method ngoài interface. | - **Không thể proxy `final` class** (không kế thừa được).<br>- **Không thể proxy `final` method** (không override được).<br>- Không proxy được `private` method.<br>- Yêu cầu Target Class phải có Default Constructor (mặc dù các phiên bản hiện đại dùng Objenesis để lách qua). |

---

### 4. Luồng tạo Proxy trong vòng đời Spring Bean (BeanPostProcessor)

Việc chuyển đổi một Bean thường thành Proxy Bean diễn ra tự động thông qua `BeanPostProcessor` đặc biệt của Spring:

```
[1. Instantiate Bean] ──► [2. Populate Dependencies] ──► [3. PostProcessAfterInitialization]
                                                                      │
                                                        quét @Transactional advisor
                                                                      │
                                                                      ▼
                                                       [4. Wrap with Proxy Object]
                                                                      │
                                                                      ▼
                                                      [5. Register Proxy to IoC Container]
```

* **Lớp chịu trách nhiệm**: `InfrastructureAdvisorAutoProxyCreator` (hoặc `AnnotationAwareAspectJAutoProxyCreator`).
* **Thời điểm**: Trong giai đoạn `postProcessAfterInitialization()` của vòng đời Spring Bean.
* **Kết quả**: Instance gốc của `UserServiceImpl` nằm ẩn bên trong bộ nhớ, còn Bean nằm trong IoC Container mà các controller khác `@Autowired` vào chính là **Proxy Object** (`UserServiceImpl$$EnhancerBySpringCGLIB`).

---

## III. TransactionInterceptor & Luồng đi thực tế của Request

### 1. Các thành phần tham gia vào luồng xử lý

Để điều khiển một giao dịch hoàn chỉnh, Proxy phối hợp chặt chẽ với 4 lớp hạ tầng chính:

1. **`TransactionInterceptor`**: AOP Advice nhận cuộc gọi từ Proxy, bao bọc logic trong khối try-catch-finally.
2. **`TransactionAttributeSource`**: Đọc và parse metadata của `@Transactional` (Propagation, Isolation, Rollback rules).
3. **`PlatformTransactionManager`**: Thực hiện các thao tác hạ tầng với DB Driver (`getTransaction`, `commit`, `rollback`).
4. **`TransactionSynchronizationManager`**: Quản lý `ThreadLocal` lưu trữ `Connection`.

---

### 2. Luồng thực thi chi tiết từ Request đến Database

Khi Client gọi method `userService.createUser(user)`:

#### Bước 1: Tiếp nhận cuộc gọi (Proxy Interception)
Client gọi method $\rightarrow$ Đích đến là **Proxy Object** $\rightarrow$ Proxy chuyển tiếp cuộc gọi đến `TransactionInterceptor.invoke()`.

#### Bước 2: Chuẩn bị Transaction Context
`TransactionInterceptor` gọi `TransactionAspectSupport.invokeWithinTransaction()`:
1. Đọc metadata `@Transactional` từ `TransactionAttributeSource`.
2. Xác định `PlatformTransactionManager` tương ứng (ví dụ `DataSourceTransactionManager`).
3. Gọi `transactionManager.getTransaction(txDefinition)` để mở hoặc tham gia vào giao dịch.

#### Bước 3: Lấy Connection & Disable Auto-Commit
Bên trong `PlatformTransactionManager`:
1. Xin một JDBC `Connection` từ Connection Pool (HikariCP).
2. Tắt chế độ tự động commit: **`connection.setAutoCommit(false)`**.
3. Đặt Isolation Level & Timeout nếu có cấu hình.

#### Bước 4: ThreadLocal Binding
`PlatformTransactionManager` gọi:
`TransactionSynchronizationManager.bindResource(dataSource, connectionHolder)`
-> Gắn `Connection` này vào **`ThreadLocal`** của Thread hiện tại.

#### Bước 5: Thực thi Business Logic gốc
`TransactionInterceptor` gọi `invocation.proceedWithInvocation()` $\rightarrow$ Chuyển quyền điều khiển cho **Target Object** thực hiện code nghiệp vụ.
* Trong quá trình này, khi `UserRepository` hoặc `JdbcTemplate` thực hiện các câu lệnh `INSERT`/`UPDATE`, chúng sẽ dùng `DataSourceUtils.getConnection(dataSource)` để **lấy lại đúng `Connection` đang nằm trong `ThreadLocal`**.

#### Bước 6: Xử lý Kết quả (Commit / Rollback) & Cleanup
* **Trường hợp 1: Thành công (No Exception)**
  1. `TransactionInterceptor` nhận kết quả trả về.
  2. Gọi `transactionManager.commit(status)` $\rightarrow$ Thực thi **`connection.commit()`** xuống Database Engine.
* **Trường hợp 2: Thất bại (Exception xảy ra)**
  1. Khối `catch` trong `TransactionInterceptor` bắt được Exception.
  2. Kiểm tra xem Exception có thỏa mãn quy tắc rollback không (`ex instanceof RuntimeException || ex instanceof Error` hoặc khớp `rollbackFor`).
  3. Nếu thỏa mãn: Gọi `transactionManager.rollback(status)` $\rightarrow$ Thực thi **`connection.rollback()`**.
* **Bước cuối (Finally Block)**:
  1. Phục hồi trạng thái `autoCommit = true` nếu cần.
  2. Unbind connection khỏi `ThreadLocal`: `TransactionSynchronizationManager.unbindResource(dataSource)`.
  3. **Trả Connection về HikariCP Pool** (không đóng kết nối vật lý).

---

### 3. Sơ đồ tuần tự (Sequence Diagram)

```
[Client]       [Proxy Object]   [TransactionInterceptor]  [TransactionManager]   [ThreadLocal / Connection]   [Target Service]
   │                 │                     │                       │                         │                      │
   │──1. createUser()─►                    │                       │                         │                      │
   │                 │──2. invoke()────────►                       │                         │                      │
   │                 │                     │──3. getTransaction()─►                          │                      │
   │                 │                     │                       │──4. Get Conn from Pool──►                      │
   │                 │                     │                       │──5. setAutoCommit(false)►                      │
   │                 │                     │                       │──6. bindResource()─────► [ThreadLocal]         │
   │                 │                     │                       │                         │                      │
   │                 │                     │──7. proceed() (Execute Target Method)─────────────────────────────────►│
   │                 │                     │                                                 │                      │
   │                 │                     │                                                 │◄──8. Repository SQL  │
   │                 │                     │                                                 │   (uses bound Conn)  │
   │                 │                     │                                                 │                      │
   │                 │                     │◄──9. Return Success────────────────────────────────────────────────────│
   │                 │                     │                       │                         │                      │
   │                 │                     │──10. commit()────────►                          │                      │
   │                 │                     │                       │──11. connection.commit()► [DB Engine]          │
   │                 │                     │                       │──12. unbindResource()──► [ThreadLocal]         │
   │                 │                     │                       │──13. Return Conn to Pool► [HikariCP]           │
   │◄──14. Return────│◄────────────────────│                       │                         │                      │
```

---

## IV. TransactionSynchronizationManager & ThreadLocal Storage

### 1. Đặt vấn đề: Bài toán chia sẻ Connection trong chuỗi gọi hàm

Hãy xét một chuỗi gọi hàm điển hình:
`UserService.createUser()` $\rightarrow$ `UserRepository.save()` $\rightarrow$ `AuditRepository.log()`.

Làm thế nào để `UserRepository` và `AuditRepository` biết và dùng chung **đúng một JDBC Connection** mà `UserService` đã mở, mà lập trình viên **không cần phải truyền tham số `Connection conn` thủ công** qua từng hàm?

Spring giải quyết bài toán này thông qua **`TransactionSynchronizationManager`**.

---

### 2. Cơ chế ThreadLocal của TransactionSynchronizationManager

`TransactionSynchronizationManager` là một utility class của Spring, sử dụng các biến `ThreadLocal` để đóng vai trò làm **Storage giữ Context** cho từng Thread đang chạy.

```java
public abstract class TransactionSynchronizationManager {

    // ThreadLocal chính: Map chứa Key là DataSource, Value là ConnectionHolder
    private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<>("Transactional resources");

    // ThreadLocal lưu danh sách các Callback / Synchronization (như @TransactionalEventListener)
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
            new NamedThreadLocal<>("Transaction synchronizations");

    // ThreadLocal lưu tên của Transaction hiện tại
    private static final ThreadLocal<String> currentTransactionName =
            new NamedThreadLocal<>("Current transaction name");

    // ThreadLocal lưu trạng thái Read-Only
    private static final ThreadLocal<Boolean> currentTransactionReadOnly =
            new NamedThreadLocal<>("Current transaction read-only");

    // ThreadLocal lưu Isolation Level
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
            new NamedThreadLocal<>("Current transaction isolation level");
    
    // ...
}
```

* **Cấu trúc bộ nhớ `resources`**:
  `ThreadLocal<Map<Object, Object>>`
  - Key: Instance của `DataSource` (hoặc `EntityManagerFactory`).
  - Value: Instance của `ConnectionHolder` (chứa JDBC `Connection` vật lý + reference count).

---

### 3. Cách Repository/DAO sử dụng Connection từ ThreadLocal

Khi các framework ORM/DAO (như `JdbcTemplate`, `JpaRepository`, `Hibernate`) cần thực thi câu lệnh SQL, chúng **không bao giờ gọi trực tiếp `dataSource.getConnection()`**.

Thay vào đó, chúng gọi qua class tiện ích của Spring: **`DataSourceUtils.getConnection(dataSource)`**.

```java
// Mô phỏng logic bên trong DataSourceUtils.getConnection()
public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
    // 1. Kiểm tra xem ThreadLocal của Thread hiện tại có giữ Connection nào cho DataSource này không
    ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);

    if (holder != null && holder.hasConnection()) {
        // 2. NẾU CÓ: Tái sử dụng ngay Connection đang nằm trong ThreadLocal (Cùng Transaction!)
        return holder.getConnection();
    }

    // 3. NẾU KHÔNG (Không nằm trong @Transactional): Lấy Connection mới từ Pool (Non-transactional)
    Connection conn = dataSource.getConnection();
    return conn;
}
```

> **Kết luận**: Nhờ `ThreadLocal`, miễn là các thao tác diễn ra trên **cùng một Thread**, mọi câu lệnh SQL trong chuỗi gọi hàm sẽ luôn dùng chung một JDBC Connection duy nhất.

---

## V. Vấn đề Self-Invocation & Giải pháp triệt để

### 1. Nguyên nhân sâu xa: Con trỏ `this` và cơ chế Bypass Proxy

#### Ca bệnh điển hình:
```java
@Service
public class OrderService {

    // Method A: KHÔNG có @Transactional
    public void placeOrder(Order order) {
        System.out.println("Processing order...");
        // Gọi nội bộ method B
        this.payOrder(order); 
    }

    // Method B: CÓ @Transactional
    @Transactional
    public void payOrder(Order order) {
        paymentRepository.save(new Payment(order));
    }
}
```

* **Hiện tượng**: Khi client gọi `orderService.placeOrder(order)`, method `payOrder()` được thực thi **nhưng KHÔNG CÓ TRANSACTION KHỞI TẠO**. Nếu có exception nổ ra trong `payOrder()`, dữ liệu **KHÔNG ROLLBACK**.

#### Giải thích bản chất bằng hình ảnh:

```
[Client Call]
     │
     ▼
┌────────────────────────────────────────────────────────┐
│                   PROXY OBJECT                         │
│  (UserServiceImpl$$EnhancerBySpringCGLIB)              │
│                                                        │
│   ┌────────────────────────────────────────────────┐   │
│   │                TARGET OBJECT                   │   │
│   │                                                │   │
│   │   placeOrder() {                               │   │
│   │       // Con trỏ 'this' trỏ trực tiếp tới      │   │
│   │       // Target Object nằm trong RAM           │   │
│   │       this.payOrder(); ───► payOrder()         │   │
│   │   }                         (BYPASS PROXY!)    │   │
│   └────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

1. Client `@Autowired OrderService` $\rightarrow$ Client nhận được **Proxy Object**.
2. Client gọi `orderService.placeOrder()` $\rightarrow$ Cuộc gọi đi qua Proxy. Nhưng `placeOrder()` không có `@Transactional`, Proxy chuyển tiếp thẳng vào `TargetObject.placeOrder()`.
3. Bên trong `placeOrder()`, câu lệnh `this.payOrder()` sử dụng **con trỏ `this` của Java**.
4. Trong Java runtime, con trỏ `this` **luôn trỏ trực tiếp tới bản thân Target Object instance trong bộ nhớ Heap**, chứ không trỏ ngược ra Proxy Object bọc bên ngoài.
5. Do cuộc gọi `payOrder()` bị nhảy trực tiếp nội bộ trong Target Object, nó **hoàn toàn Bypass (bỏ qua) Proxy Object** $\rightarrow$ `TransactionInterceptor` không bao giờ được kích hoạt.

---

### 2. Giải pháp 1: Self-Injection (Tiêm chính Proxy vào Bean)

Cho phép Bean tự inject chính Proxy của nó từ IoC Container thông qua `@Autowired` kết hợp với `@Lazy` (để tránh vòng lặp khởi tạo `BeanCurrentlyInCreationException`).

```java
@Service
public class OrderService {

    // Inject chính Proxy của bản thân từ ApplicationContext
    @Autowired
    @Lazy
    private OrderService self;

    public void placeOrder(Order order) {
        System.out.println("Processing order...");
        // Gọi qua biến 'self' (chính là Proxy Object!)
        self.payOrder(order); 
    }

    @Transactional
    public void payOrder(Order order) {
        paymentRepository.save(new Payment(order));
    }
}
```
* **Cơ chế**: Vì `self` là Proxy Object, cuộc gọi `self.payOrder()` sẽ đi ngược ra ngoài Proxy $\rightarrow$ kích hoạt `TransactionInterceptor` bình thường.
* **Cách khác tương tự**: Dùng `AopContext.currentProxy()` (Yêu cầu bật `@EnableAspectJAutoProxy(exposeProxy = true)`).

---

### 3. Giải pháp 2: AspectJ Weaving (Compile-time / Load-time Weaving)

Nếu hệ thống đòi hỏi gọi hàm nội bộ vẫn phải chạy Transaction mà không muốn dùng Proxy, ta phải từ bỏ cơ chế Spring AOP Proxy và chuyển sang **AspectJ Native**.

AspectJ không sử dụng Proxy Object. Nó can thiệp và **sửa đổi trực tiếp Bytecode** của Target Class:

1. **Compile-Time Weaving (CTW)**: Sử dụng trình biên dịch của AspectJ (`ajc`) để chèn trực tiếp bytecode xử lý transaction vào file `.class` ngay khi build ứng dụng.
2. **Load-Time Weaving (LTW)**: Sử dụng một Java Agent (`-javaagent:aspectjweaver.jar`) để sửa đổi bytecode của class ngay khi ClassLoader nạp class vào JVM Memory.

```java
// Khi dùng AspectJ, Bytecode của method payOrder() sau khi weave sẽ tương đương:
public void payOrder(Order order) {
    // AspectJ chèn thẳng code Transaction vào đây!
    TransactionStatus status = TransactionManager.begin();
    try {
        paymentRepository.save(new Payment(order));
        TransactionManager.commit(status);
    } catch(Exception e) {
        TransactionManager.rollback(status);
    }
}
```
* **Kết quả**: Vì code transaction nằm trực tiếp bên trong method, nên dù gọi qua con trỏ `this.payOrder()`, transaction vẫn hoạt động hoàn hảo.

---

### 4. Giải pháp 3: Refactor tách Service (Architectural Best Practice)

Dù Self-Injection hay AspectJ đều giải quyết được bài toán, nhưng giải pháp chuẩn mực nhất về mặt kiến trúc phần mềm (Clean Code & Single Responsibility Principle) là **Refactor tách riêng Service**.

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService; // Tách sang Service riêng

    public void placeOrder(Order order) {
        System.out.println("Processing order...");
        paymentService.payOrder(order); // Gọi qua Service khác -> Đi qua Proxy của PaymentService
    }
}

@Service
public class PaymentService {

    @Transactional
    public void payOrder(Order order) {
        paymentRepository.save(new Payment(order));
    }
}
```

---

## VI. Luật Rollback mặc định & Triết lý thiết kế (Rollback Rules)

### 1. Triết lý thiết kế: Unchecked vs Checked Exception

> **Câu hỏi phỏng vấn**: Tại sao mặc định `@Transactional` chỉ rollback khi nổ `RuntimeException` (Unchecked) và `Error`, mà lại COMMIT khi nổ `Checked Exception` (như `IOException`, `SQLException`, Custom Exception)?

#### Triết lý ngôn ngữ Java & EJB Specification:
* **Unchecked Exception (`RuntimeException`)**: Đại diện cho các **System Failures / Technical Errors** (ví dụ `NullPointerException`, `IndexOutOfBoundsException`, `DataAccessException`, `OutofMemoryError`). Đây là những lỗi không lường trước, ứng dụng bị sai hỏng trạng thái và **bắt buộc phải Abort / Rollback** toàn bộ giao dịch để bảo toàn dữ liệu.
* **Checked Exception (`Exception`)**: Trong thiết kế của Java, Checked Exception đại diện cho các **Application / Business Conditions có thể phục hồi (Recoverable Conditions)** (ví dụ `InsufficientBalanceException`, `FileNotFoundException`, `UserNotFoundException`). Ngôn ngữ Java bắt buộc lập trình viên phải `try-catch` hoặc `throws` để xử lý các tình huống này. Do đó, Spring coi đây là một kết quả kiểm soát nghiệp vụ bình thường và **mặc định thực hiện COMMIT**.

---

### 2. Cách cấu hình ghi đè với `rollbackFor` và `noRollbackFor`

Để thay đổi hành vi mặc định, Spring cung cấp các thuộc tính cấu hình trong annotation `@Transactional`:

#### a. Buộc Rollback với Checked Exception (`rollbackFor`)
```java
// Rollback với tất cả Exception (bao gồm cả Checked Exception)
@Transactional(rollbackFor = Exception.class)
public void processOrder() throws Exception {
    orderRepository.save(order);
    throw new CustomCheckedException("Lỗi nghiệp vụ"); // -> Sẽ ROLLBACK!
}
```

#### b. Cấu hình cụ thể nhiều loại Exception
```java
@Transactional(rollbackFor = {IOException.class, SQLException.class})
public void exportData() throws IOException { ... }
```

#### c. Loại trừ không Rollback cho một số Unchecked Exception (`noRollbackFor`)
```java
// Không rollback nếu gặp NotificationFailedException
@Transactional(noRollbackFor = NotificationFailedException.class)
public void registerUser(User user) {
    userRepository.save(user);
    // Nếu gửi email lỗi -> Nổ NotificationFailedException nhưng DB vẫn SAVE user!
    notificationService.sendWelcomeEmail(user); 
}
```

---

### 3. Bẫy nuốt Exception (Try-Catch Suppression & UnexpectedRollbackException)

Một lỗi cực kỳ phổ biến trong thực tế production:

```java
@Transactional
public void createProduct(Product product) {
    productRepository.save(product);

    try {
        // Method này nổ RuntimeException (ví dụ DataIntegrityViolationException)
        auditService.log(product); 
    } catch (Exception e) {
        // Lập trình viên cố tình nuốt Exception để method chính không bị dừng
        log.error("Lưu audit thất bại nhưng vẫn tiếp tục", e);
    }
}
```

* **Kịch bản lỗi xảy ra**:
  1. `auditService.log(product)` nổ `RuntimeException`.
  2. Spring Transaction Aspect bắt được Exception này ở tầng dưới và lập tức đánh dấu vào `TransactionStatus`: **`setRollbackOnly(true)`**.
  3. Tầng Service của bạn dùng `try-catch` nuốt Exception và tiếp tục chạy hết hàm `createProduct()`.
  4. Khi hàm `createProduct()` kết thúc, `TransactionInterceptor` chuẩn bị commit. Nhưng nó phát hiện `TransactionStatus.isRollbackOnly() == true`.
  5. Spring không thể commit một transaction đã bị đánh dấu hư hỏng. Nó lập tức rollback toàn bộ và nổ ngoại lệ:
     **`org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only`**.

#### Giải pháp khắc phục:
Nếu muốn thao tác audit chạy độc lập và không làm hư hỏng transaction chính khi bị lỗi, hãy khai báo `Propagation.REQUIRES_NEW` cho method audit:

```java
@Service
public class AuditService {
    // Mở transaction hoàn toàn mới, độc lập với transaction gọi nó
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(Product product) {
        auditRepository.save(new AuditLog(product));
    }
}
```

---

## VII. Cheat Sheet Phỏng vấn (Interview Q&A)

### Q1: Phân biệt cơ chế tạo Proxy giữa JDK Dynamic Proxy và CGLIB trong Spring.
> **Trả lời**: JDK Dynamic Proxy sử dụng Java Reflection API để tạo Proxy Class triển khai cùng Interface với Target Class (yêu cầu Target Class phải có Interface). CGLIB sử dụng thư viện can thiệp bytecode (ASM/ByteBuddy) để tạo Subclass kế thừa từ Target Class (không cần Interface). Từ Spring Boot 2.x+, CGLIB được sử dụng làm mặc định (`spring.aop.proxy-target-class=true`).

---

### Q2: Luồng đi thực tế của một cuộc gọi `@Transactional` qua Proxy diễn ra như thế nào?
> **Trả lời**: Client gọi method $\rightarrow$ Proxy tiếp nhận $\rightarrow$ chuyển cho `TransactionInterceptor`. Interceptor gọi `PlatformTransactionManager` lấy `Connection` từ Pool, set `autoCommit = false`, và bind `Connection` vào `ThreadLocal` qua `TransactionSynchronizationManager`. Sau đó, Target Method thực thi. Nếu thành công, Proxy gọi `connection.commit()`. Nếu nổ Unchecked Exception, Proxy gọi `connection.rollback()`. Cuối cùng, unbind `ThreadLocal` và trả `Connection` về Pool.

---

### Q3: Giải thích vấn đề Self-Invocation và nguyên nhân tại sao `@Transactional` không hoạt động?
> **Trả lời**: Self-Invocation xảy ra khi một method không có `@Transactional` gọi một method có `@Transactional` trong cùng một class. Nguyên nhân là do cú pháp `this.methodB()` sử dụng con trỏ `this` trỏ trực tiếp đến Target Object instance trong RAM, hoàn toàn Bypass (bỏ qua) Proxy Object bọc bên ngoài. Do không qua Proxy, `TransactionInterceptor` không được kích hoạt.

---

### Q4: Làm thế nào để giải quyết vấn đề Self-Invocation?
> **Trả lời**: Có 3 giải pháp chính: 
> 1) Refactor tách method sang một Service mới (Best Practice).
> 2) Self-Injection (inject chính Proxy của class vào bản thân bằng `@Autowired @Lazy`).
> 3) Chuyển từ Spring AOP sang AspectJ Weaving (Compile-time hoặc Load-time weaving) để chèn code transaction trực tiếp vào bytecode.

---

### Q5: Lớp `TransactionSynchronizationManager` có vai trò gì trong Spring Transaction?
> **Trả lời**: Lớp này chịu trách nhiệm quản lý Context của Transaction theo luồng thực thi bằng các biến `ThreadLocal`. Nó lưu trữ Map chứa JDBC `Connection` (key là `DataSource`). Nhờ đó, các tầng DAO/Repository trong cùng một luồng có thể lấy lại đúng Connection này để dùng chung một transaction mà không cần truyền tham số Connection thủ công.

---

### Q6: Mặc định `@Transactional` rollback khi nào? Tại sao Checked Exception lại không được rollback?
> **Trả lời**: Mặc định Spring chỉ rollback đối với `RuntimeException` (Unchecked) và `Error`. Theo triết lý EJB/Java, Unchecked Exception đại diện cho lỗi hệ thống không thể phục hồi (bắt buộc rollback), trong khi Checked Exception đại diện cho tình huống nghiệp vụ có thể dự đoán và phục hồi (nên COMMIT). Để rollback với Checked Exception, ta phải cấu hình `@Transactional(rollbackFor = Exception.class)`.