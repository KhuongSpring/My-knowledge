# Spring Boot

## 1. `@RestController`

### Dùng để thay thế cho `@Controller`

- `@Controller` dùng để *prepare model* và chọn *view* để hiển thị *data*.
- Khi chỉ dùng annotation `@Controller`, *view* sẽ hiển thị toàn bộ *data*.
- Vì còn có nhiệm vụ ghi dữ liệu trực tiếp vào luồng *response* nên có `@ResponseBody`.
  - Lúc này kết quả sẽ trả về đúng cái mình `return` như JSON, XML, ...
  - Từ đây, `@RestController` ra đời.
- Dùng để thuận tiện trong việc phát triển các dự án **RESTful Web Service**.

## 2. Các loại HTTP Request

### Các hành động CRUD

- Create: *POST* -> `@PostMapping`
- Read: *GET* -> `@GetMapping`
- Update: *PUT* -> `@PutMapping`
- Delete: *DELETE* -> `@DeleteMapping`
- `@RequestMapping`: Chỉ định *endpoint* bắt buộc, thường dùng phía trên class **Controller**.

### Nhận request

- `@RequestParam`

  ```java
  public ResponseEntity<?> getAllUsers(
      @RequestParam("age") int age,
      @RequestParam("name") String name
  ) {}
  ```

  *Request param* còn có `defaultValue`.

- `@PathVariable`

  Là một phần trong *URL*.

  ```java
  @GetMapping("/users/{id}/info")
  public ResponseEntity<?> getUserInfo(
      @PathVariable(value = "id", defaultValue = "0") int userId
  ) {}
  ```

  Có các thuộc tính tương tự `@RequestParam`.

- `@RequestBody`

  - Cần dùng khi sử dụng `@PostMapping` hay `@PutMapping`.
  - Đây là nơi chứa *data* chính để gửi lên, thường ở dạng **JSON**.
  - *Controller* sẽ tự động map về **Object**, thường là **DTO** (*Data Transfer Object*).

  ```java
  @GetMapping("/users")
  public ResponseEntity<?> getUsers(@RequestBody UserDTO userDto) {}
  ```

### `ResponseEntity<T>`

- Ưu điểm: không cần code dài dòng, có thể dùng *builder*, đồng thời xử lý trả về *status*.
- Thường `return` ở dạng `ResponseEntity.ok()` hoặc `ResponseEntity.notFound().build()`.

### `@ModelAttribute`

Đánh dấu một *model* được gửi lên từ *form*.

## 3. Các loại Data

### DTO (*Data Transfer Object*)

Là các class đóng gói để tách biệt giữa *response/request* với *entity*. Được dùng để trao đổi thông tin từ phía **Client-Server** hoặc giữa các **Service** trong **Microservice**. Mục đích là để giảm bớt lượng dữ liệu gửi đi.

-> Dùng phổ biến ở **Web Layer**.

### Domain Layer

Là class đảm nhiệm làm các class tham số đầu vào để tính toán hoặc là kết quả tính toán.

### Entity

Là các class thao tác trực tiếp với DB và dùng để map sang DB.

-> Dùng phổ biến ở **Repository Layer**.

### Model Mapping

Là việc chuyển dữ liệu giữa các dạng *data* với nhau.

## 4. Dependency Injection

### Module Coupling

Luôn đưa mối quan hệ giữa các *module* về dạng **Loose Coupling**.

### Nguyên lý Dependency Inversion

- Ý thứ 5 của **SOLID**: **DIP** (*Dependency Inversion Principle*).
  - Các *module* cấp cao không được phụ thuộc trực tiếp vào các *module* cấp thấp. Cả hai nên phụ thuộc vào **Abstraction** (thường là interface).
  - Bằng cách tận dụng tính đa hình, ta nên để các class phụ thuộc vào **Abstraction**.
  - **Abstraction** không nên phụ thuộc vào chi tiết, và ngược lại.

### IoC (*Inversion of Control*)

- **IoC** nhằm mục đích đơn giản hóa quá trình tạo đối tượng và liên kết giữa chúng, bằng cách tuân theo nguyên tắc: **Không tạo đối tượng, chỉ mô tả cách chúng sẽ được tạo ra**.
- **IoC** sẽ quản lý, phân tích các mối phụ thuộc, tạo các đối tượng theo thứ tự hợp lý và liên kết chúng lại theo mô tả của lập trình viên.
- Cách dùng:
  - Đánh dấu một class là *module* bằng annotation như `@Component`, `@Service`, `@Repository`, ...
  - Sau đó dùng `@Autowired` để tìm object tương ứng và *inject* vào.
- Thứ tự ưu tiên: Type -> Name -> `@Primary` / `@Qualifier`
- **IoC Container**: `ApplicationContext`
- Các *module* trong **IoC Container**: `Bean`

### Dependency Injection sẽ thực hiện

- Tìm và *inject* `Bean` tương ứng thông qua `@Autowired`.
- Tạo tiếp *module* cấp cao; **IoC** sẽ tìm *module* cấp thấp mà nó phụ thuộc và *inject* vào.

-> Khi chạy dự án, **IoC** sẽ quét tất cả class được đánh dấu là `Bean`, tạo một đối tượng duy nhất (**Singleton**) và đưa vào **IoC Container**. Khi cần sẽ lấy ra để sử dụng.

-> Nếu khi tạo `Bean` mới mà `Bean` đó cần phụ thuộc vào `Bean` khác, **IoC** sẽ tìm để *inject* vào. Nếu chưa có thì tạo mới, sau đó *inject*. Việc *inject* tự động này được gọi là **Dependency Injection**.

### Các loại Injection

- **Constructor-based Injection**: *inject* các *module* bắt buộc; các *module* được *inject* nằm trong *constructor*.
- **Setter-based Injection**: *inject* các *module* tùy chọn; mỗi *module* sẽ được *inject* thông qua *setter*.

## 5. Bean và `ApplicationContext`

### Bean

Là những *module* chính của chương trình được **IoC Container** quản lý. Chúng có thể phụ thuộc lẫn nhau.

### `ApplicationContext`

Là khái niệm để chỉ **IoC Container**. Tương tự `BeanFactory` nhưng ở mức cao hơn.

### Inject `Bean` vào `Bean` khác

- `@Autowired`: Tự động tìm và *inject* `Bean` phù hợp.
- Thông qua **constructor** hoặc **setter**:
  - Gọi *constructor* truyền trực tiếp, không cần annotation.
  - Dùng phương thức *set* để inject. Có thể thêm `@Required` để *setter* luôn được gọi.
  - Cách dùng *setter* để *inject* thường dùng trong trường hợp phụ thuộc vòng. **Spring Boot** sẽ không biết nên khởi tạo *constructor* nào trước. Vì vậy một `Bean` có thể dùng *constructor* và `Bean` còn lại dùng *setter*.
  - Khi có nhiều `Bean` phù hợp và **Spring Boot** không biết chọn cái nào:
    - Dùng `@Primary`: Chỉ định thứ tự ưu tiên của `Bean`.
    - Dùng `@Qualifier`: Chỉ định rõ tên `Bean` (thường là tên class), đặt trước tên field.

### Bean Life Cycle

- **IoC Container** tạo `Bean`.
- Gọi các *setter method* để *inject* các `Bean`.
- `@PostConstruct` được gọi.
- *Init method* được gọi.
- Sau khi không dùng nữa sẽ bị hủy thông qua `@PreDestroy`.

`@PostConstruct` là sau khi `Bean` đã khởi tạo xong, dùng để thực hiện một số task khi khởi tạo `Bean`.

`@PreDestroy` là trước khi `Bean` bị phá hủy, dùng để dọn dẹp `Bean`.

Hai annotation này được đánh dấu lên method để method tự động được gọi khi vòng đời `Bean` thay đổi.

### Các loại Bean (*Scope*)

- **Singleton** (mặc định): **IoC Container** chỉ tạo duy nhất 1 *object* từ class `Bean` này.
- **Prototype**: trả về một `Bean` riêng biệt cho mỗi lần sử dụng. `@Scope("prototype")`
- **Request**: tạo mỗi `Bean` cho mỗi *request*.
- **Session**: tạo mỗi `Bean` cho mỗi *session*.
- **Global Session**: tạo mỗi `Bean` cho mỗi *global session*.

### Dùng `@Bean` bên trong `@Configuration`

- Cách này dùng cho trường hợp `Bean` cần nhiều thao tác phức tạp để khởi tạo hoặc có nhiều `Bean` liên quan tới nhau.
- Thay vì khởi tạo riêng lẻ, ta gom các `Bean` cần khởi tạo vào một class chứa.
- Thường class đó được đánh dấu `@Configuration` và có hậu tố **Config**.
- Các thao tác xử lý được đặt trong *constructor* và khai báo các *module* bằng `@Bean`.

### `@Value`

Dùng để đưa giá trị trong `application.properties` vào bằng cách: `@Value("${...}")`

## 6. ModelMapper

- Là một thư viện của **Java**, giúp đơn giản hóa việc convert giữa các **class**.
- Ví dụ: thay vì phải copy từng *field* của class A sang class B thì làm như sau:

```java
User user = new User("john", "123456", "Nguyen Van John", 20);

ModelMapper mapper = new ModelMapper();

UserDto userDto = mapper.map(user, UserDto.class);
```

Trước khi sử dụng thì nên cấu hình cho nó, lý tưởng là biến thành `Bean` và *inject* tự động:

```java
@Configuration
public class ModelMapperConfig {
    @Bean
    public ModelMapper modelMapper() {
        ModelMapper modelMapper = new ModelMapper();
        modelMapper.getConfiguration()
                .setMatchingStrategy(MatchingStrategies.STRICT); // Map chat che
        return modelMapper;
    }
}
```

Lưu ý luôn test và để ý xem đã map đúng chưa.

## 7. `@ControllerAdvice` và `@ExceptionHandler`

### AOP (*Aspect Oriented Programming*)

Ngắt ngang một method để thực hiện method khác trong một điều kiện nào đó.

### `@ExceptionHandler`: xử lý exception

Cho phép xử lý *exception* trong một *controller* cụ thể. Khi một *exception* được ném ra, method đánh dấu với annotation này sẽ được gọi.

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    if (id <= 0) {
        throw new IllegalArgumentException("ID khong hop le!");
    }

    // Gia su day la logic tim user
    return new User(id, "John Doe");
}

@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<String> handleIllegalArgumentException(IllegalArgumentException ex) {
    return new ResponseEntity<>("Loi: " + ex.getMessage(), HttpStatus.BAD_REQUEST);
}
```

### `@ControllerAdvice` / `@RestControllerAdvice`: xử lý exception toàn cục

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleResourceNotFound(ResourceNotFoundException ex) {
        return new ResponseEntity<>("Khong tim thay tai nguyen: " + ex.getMessage(), HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgument(IllegalArgumentException ex) {
        return new ResponseEntity<>("Du lieu khong hop le: " + ex.getMessage(), HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleGlobalException(Exception ex) {
        return new ResponseEntity<>("Da xay ra loi: " + ex.getMessage(), HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> {
            errors.put(error.getField(), error.getDefaultMessage());
        });

        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
}
```

- Có thể tự custom *exception* để xử lý các lỗi do logic gây ra.
- Thường sẽ trả về dạng **JSON** thay vì chuỗi, ví dụ `ErrorResponse`.

## 8. Validation

- Thêm các annotation ràng buộc trên *field*: `@NotNull`, `@NotEmpty`, `@Email`, `@Size`, ...
- Có thể thêm *message* như: `@NotNull(message = "Thieu ...")`
- Trước mỗi *param* cần *valid*, thêm `@Valid` hoặc `@Validated`.
- Cách xử lý *validation fail*:
  - Dùng tham số cuối cùng là `BindingResult`.
  - Hoặc bắt *exception*.

### Cách 1

```java
@PostMapping("/")
public User createUser(
    @RequestBody @Valid UserDto userDto,
    BindingResult bindingResult
) throws Exception {
    // Khi co BindingResult thi loi duoc tam bo qua de xu ly thu cong
    if (bindingResult.hasErrors()) {
        throw new Exception("...");
    }
}
```

### Cách 2

```java
@ExceptionHandler(BindException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public String handleBindException(BindException e) {
    String errorMessage = "Request khong hop le";
    if (e.getBindingResult().hasErrors()) {
        e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
    }
    return errorMessage;
}
```

## 9. Thymeleaf

**Thymeleaf** là một **Java Template Engine** có nhiệm vụ xử lý và generate ra các file *HTML*, *XML*, ...

### Cú pháp

- Sẽ bắt đầu bằng chữ `th`.
- `${...}`: Dùng để lấy *value* theo *key* truyền vào.

  ```html
  <p>Today is: <span th:text="${today}"></span>.</p>
  ```

- `*{...}`: Dùng để lấy giá trị của một biến cho trước bởi `th:object`.

  ```html
  <div th:object="${session.user}">
    <p>Name: <span th:text="*{firstName}"></span>.</p>
  </div>
  ```

- `@{...}`: Xử lý và trả ra *URL* theo *context* của máy chủ.

  ```html
  <a href="#" th:href="@{/v1/about}">Link to About me</a>
  ```

Có 2 loại đường dẫn là tuyệt đối và tương đối:

- Tuyệt đối: trước path có `/`
- Tương đối: nối tiếp sau *URL* hiện tại

Ví dụ, nếu đang ở `.../8080/home`:

- `@{/about}` -> `.../8080/about`
- `@{about}` -> nối tiếp theo đường dẫn hiện tại

## 10. Spring Boot JPA

- Là một phần trong hệ sinh thái **Spring Data**, tạo một layer giữa tầng **Service** và *Database*.
- Giúp thao tác với **Database** dễ hơn, tự động *config* và code gọn hơn.
- Trong **Spring Boot**, **Hibernate** là *implementation*, còn **JPA** là *API* tiêu chuẩn.

### Cách dùng

- Khởi tạo một *repository interface* và đánh dấu `Bean`, `extends JpaRepository<T, Type>`.
- *Inject* interface đó vào dự án và gọi các *implementation* để dùng.

### Cơ chế

- **Spring Data JPA** cung cấp một cơ chế cho phép định nghĩa các *method* theo một cấu trúc tên nhất định thì sẽ tự động thực hiện query.
- **Method Name Query Derivation**: tạo truy vấn từ tên phương thức.

### Quy tắc đặt tên

- Tiền tố: `findBy`, `readBy`, `countBy`, `existsBy`, ...
- Tên trường: đặt theo đúng tên field được ánh xạ
- Toán tử nếu có: `And`, `Or`, `GreaterThan`, `LessThan`, `Like`, `Between`, ...
- Ví dụ:

  ```java
  User findFirstByOrderBySttDesc();
  ```

  Tương đương ý tưởng:

  ```sql
  SELECT * FROM user ORDER BY stt DESC LIMIT 1;
  ```

- `@Query`: Nếu không muốn *JPA* tự tạo query thì có thể tự custom.

  ```java
  @Query("select u from User u where u.emailAddress = ?1")
  User myCustomQuery(String emailAddress);
  ```

## 11. Normalize API

- Các API trả về phía **Client** nên có ý nghĩa rõ ràng như `code`, `message`, `result`, ...
- Có thể tạo một class `ApiResponse<T>` để custom format API trả về.
- `@JsonInclude(JsonInclude.Include.NON_NULL)`: Nếu có field là `null` thì API sẽ ẩn field đó đi.

## 12. JWT (*JSON Web Token*)

### JWT là gì

- Là một phương tiện đại diện cho các yêu cầu chuyển giao giữa **Client-Server**.
- Các thông tin trong chuỗi được định dạng bằng **JSON**.
- Một chuỗi *token* có định dạng: `header.payload.signature`
  - **Header**: Algorithm và Token Type
  - **Payload**: Data
  - **Signature**: Verification

### Khi nào nên sử dụng

- **Authentication**: Các *request* phía người dùng sẽ kèm theo *JWT*, cho phép người dùng truy cập vào các *resource* mà *token* đó cho phép.

### Luồng xử lý

- **User** thực hiện thao tác *request* như login.
- **Authentication Server** tạo *JWT* và trả về cho **User** thông qua *response*.
- **User** nhận *JWT* và dùng nó làm chìa khóa cho các request tiếp theo.
- **API** xác thực *JWT* đó và cấp quyền truy cập cho **User**.

### No Sessions

- Với *JWT*, hệ thống không cần giữ *session* trong suốt quá trình xử lý.
- **No session storage**: không cần lưu *session* nên tối ưu bộ nhớ hơn.
- **Truly RESTful Services**: khi không có *session* thì service mới gần với mô hình **RESTful** hơn.

### Cách generate token

```java
@NonFinal
protected static final String SIGNER_KEY = "IOg/flCDdPNoC2+j4/G0m3l8FhX7paEreFwAeO2eFf4YhZabokoyI38afEtUGut7\n";

private String generateToken(String username) {
    JWSHeader header = new JWSHeader(JWSAlgorithm.HS512);
    JWTClaimsSet jwtClaimsSet = new JWTClaimsSet.Builder()
        .subject(username)
        .issuer("demo.com")
        .issueTime(new Date())
        .expirationTime(new Date(Instant.now().plus(1, ChronoUnit.HOURS).toEpochMilli()))
        .claim("Custom claim", "Custom")
        .build();

    Payload payload = new Payload(jwtClaimsSet.toJSONObject());
    JWSObject jwsObject = new JWSObject(header, payload);

    try {
        jwsObject.sign(new MACSigner(SIGNER_KEY.getBytes(StandardCharsets.UTF_8)));
        return jwsObject.serialize();
    } catch (JOSEException e) {
        throw new RuntimeException(e);
    }
}
```

## 13. Các annotation hữu ích

- `@Builder`: Giúp một class có thể build các field nhanh chóng.

  ```java
  return AuthResponse.builder().token(token).authenticated(true).build();
  ```

- `@FieldDefaults`: Dùng để định nghĩa phạm vi mặc định cho field nếu không khai báo gì thêm.

  ```java
  @FieldDefaults(level = AccessLevel.PRIVATE, makeFinal = true)
  ```

- `@JsonInclude(JsonInclude.Include.NON_NULL)`: Giúp JSON không trả về các field `null`.
- `@RequiredArgsConstructor`: Yêu cầu có một *constructor* đầy đủ, hay dùng trong việc inject `Bean`.
- `@PreAuthorize("hasAuthority('SCOPE_ADMIN')")`: Dùng để phân quyền hiệu quả cùng với `@EnableMethodSecurity`. Trước khi thực hiện method thì sẽ kiểm tra.
- `@PostAuthorize`: Kiểm tra sau khi method thực hiện.

  ```java
  @PostAuthorize("returnObject.username == authentication.name")
  ```

## 14. Spring Security

### Khái niệm

**Spring Security** cung cấp các tính năng xác thực và phân quyền, cũng như hỗ trợ các tiêu chuẩn và giao thức bảo mật như *HTTPS*, *OAuth2*, *JWT*, ...

### Cơ chế hoạt động

Dựa trên cơ chế lọc (*filter*) và sự kiện (*event*) để can thiệp vào quá trình xử lý *request* và *response*.

### Gồm 3 phần

- Authentication
- Authorization
- Authentication Provider

#### Authentication

Là quá trình xác thực xem người dùng có quyền truy cập vào ứng dụng hay không.

Thường dựa trên các thông tin nhận dạng (*identifier*) và thông tin bí mật (*credential*) của người dùng.

- Cơ chế:
  - **Form-based authentication**: xác thực thông qua một form đăng nhập.
  - **HTTP Basic authentication**: xác thực thông qua *header authorization*.
  - **Authentication via a custom login page**: xác thực thông qua một trang đăng nhập tùy chỉnh.
  - **Pre-authenticated authentication**: xác thực thông qua các giá trị được cung cấp từ phía **Client**.
- **Stateful** và **Stateless**:
  - **Stateful**: hệ thống lưu thông tin xác thực của người dùng hoặc ứng dụng trong một *session* trên máy chủ.
  - **Stateless**: hệ thống chỉ sử dụng các mã *token* đã được ký số để xác thực, thường là *JWT token*.

#### Authorization

- Là quá trình xác định quyền truy cập của người dùng đối với các tài nguyên trong ứng dụng.
- Thường dựa trên các thông tin như `role`, `group`, `permission`, `policy`, ...

#### Authentication Provider

- Là một thành phần quan trọng chịu trách nhiệm xác minh thông tin xác thực của người dùng.
- Được sử dụng bởi **Authentication Manager** để xử lý.
- **Authentication Provider** chỉ hỗ trợ một loại **Authentication** cụ thể như:
  - `UsernamePasswordAuthenticationToken`
  - `JwtAuthenticationToken`
  - `PreAuthenticatedAuthenticationToken`

### Các tính năng nâng cao của Spring Security

- **CSRF protection**
- **Session management**
- **Password encoding**

### Ưu và nhược điểm của Spring Security

#### Ưu điểm

- Là một *framework* bảo mật mạnh mẽ và linh hoạt, hỗ trợ nhiều tiêu chuẩn và giao thức bảo mật.
- Được tích hợp sẵn với **Spring Framework**, giúp phát triển ứng dụng web an toàn và hiệu quả.

#### Nhược điểm

- Cấu hình có thể khá phức tạp và khó hiểu, đặc biệt khi làm việc với các tính năng nâng cao.
- Một số tính năng có thể không phù hợp với từng loại ứng dụng web.
- Yêu cầu kiến thức chuyên môn về bảo mật để sử dụng hiệu quả.
