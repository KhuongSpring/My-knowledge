# Spring 
## 1.`@RestController`
### Dùng để thay thế cho `@Controller`:
- `@Controller` dùng để *prepare Model* và chọn *view* để hiện thị *data*. 
- Khi chỉ gọi *anotation* `@Controller` thì *view* sẽ hiện thị toàn bộ *data*. 
- Vì còn có nhiệm vụ là ghi dữ liệu trực tiếp vào luồng *response* -> `@ResponseBody`  
  + Lúc này kết quả sẽ trả về đúng cái mình *return (JSON, XML, ....)*.  
  + Từ đây, `@RestController` ra đời.
- Dùng để thuận tiện trong việc phát triển các dự án **RESTful Web Service**.
## 2. Các loại HTTP Request
### Các hành động CRUD:
- Create: *POST* -> `@PostMapping`
- Read: *GET* -> `@GetMapping`
- Update: *PUT* -> `@PutMapping`
- Delete: *DELETE* -> `@DeleteMapping`
- `@RequestMapping`: Chỉ định *endpoint* bắt buộc. Được dùng ở bên trên class **Controller**.
### Nhận request:
- `@RequestParam`:
    ```java
      public ResponseEntity<?> getAllUsers(
          @RequestParam("age") int age,
          @RequestParam("name") string name
      ){}
    ```
  *Request param* còn có `defaultValue`.
- `@PathVariable`:  
Là một phần trong *url*.
  ```java
    @GetMapping("/users/{id}/info")
    public ResponseEntity<?> getUserInfo(
        @PathVariable(value = "id", defaultValue = "0") int userId
    ){}
  ```
  Có các thuộc tính như `@RequestParam`.
- `@RequestBody`:  
  + Cần dùng khi sử dụng `@PostMapping` hay `@PutMapping`  
  + Đây là nơi chứa *data* chính để gửi lên. Thường ở dạng **JSON**, *class Controller* sẽ tự động
  map về **Object** (thường là **DTO** - Data Transfer Object).
  ```java
    @GetMapping("/users")
    public ResponseEntity<?>(@RequestBody UserDTO userDto){}
  ```
### `ResponseEntity<T>`:  
- Ưu điểm: Không code dài dòng, tự *builder*, giải quyết trả về *status*.  
- *Return* về dạng `ResponseEntity.ok() / .notFound()`
### `@ModelAttribute`: 
Đánh dấu một *model* được gửi lên từ *form*.
## 3. Các loại DATA
### DTO (Data Transfer Object):
Là các class đóng gói để tách biệt giữa *response/request* với *entity*. Được dùng để trao đổi
  thông tin từ phía **Client-Sever** hoặc giữa các **Service** trong **MicroService**. Mục đích là để giảm
  bớt lượng info gửi đi.  
-> Dùng phổ biến ở **Web Layer**.
### Domain Layer:  
Là class đảm nhiệm làm các class tham số đầu vào để tính toán hoặc là kết quả tính toán.
### Entity:  
Là các class thao tác trực tiếp từ DB và dùng để Map sang DB.  
-> Dùng phổ biến ở **Repository Layer**.
### Model Mapping: 
Là việc chuyển dữ liệu từ các dạng *data* với nhau.
## 4. Dependency Injection
### Module Coupling: 
Luôn đưa mqh giữa các *module* về dạng **Loose Coupling**. 
### Nguyên lý Dependency Inversion:
- Ý thứ 5 của **SOLID**: **DI principle (Dependecy Inversion Principle)**  
  + Các *module* cấp cao không được phụ thuộc trực tiếp vào các *module* cấp thấp. Cả 2 nên phụ thuộc
  vào **Abstraction** (thường là interface).  
      => Bằng các lợi dụng tính đa hình, chúng ta nên để các class phụ thuộc vào **Abstraction**.
  + **Abstraction** không nên phụ thuộc vào chi tiết, và ngược lại.
### IoC (Inversion of Control):  
- **IoC** nhằm mục đích đơn giản hóa quá trình tạo đối tượng và liên kết giữa chúng, bằng cách tuân
theo nguyên tắc: **Không tạo đối tượng, chỉ mô tả cách chúng sẽ được tạo ra**.  
- **IoC** sẽ quản lý, phân tích các mối phụ thuộc, tạo các đối tượng theo thứ tự hợp lý và liên kết
chúng lại theo mô tả của lập trình viên.  
- Cách dùng: Đánh dấu một class là *module* bằng *annotation* `@Component`/`@Service`/`@Repository`/..., 
  sau đó dùng `@Autowired` để tìm object tương ứng và *inject* (chèn) nó vào. 
- (Thứ tự ưu tiên: Type -> Name -> `@Primary` / `@Qualifier`)  
- **IoC Container**: `ApplicationContext`
- Các *module* trong **IoC Container**: `Bean`
### Dependency Injection sẽ thực hiện:
- Tìm và *inject* `Bean` tương ứng (`@Autowired`).
- Tạo tiếp *module* cấp cao, *IoC* sẽ tìm *module* cấp thấp mà *module* đó phụ thuộc và *inject* vào *module*
cấp cao đó.  
-> Khi run dự án, *IoC* sẽ quét tất cả class được đánh dấu là `Bean`, tạo một đối tượng duy nhất **(Singleton)**
và bỏ vào **IoC Container**, lúc nào cần sẽ lấy ra sử dụng. Do đó, các *module* được đảm bảo *IoC* chỉ tạo ra
một *object* duy nhất.  
-> Nếu khi tạo `Bean` mới, mà Bean đó cần phụ thuộc vào `Bean` khác, *IoC* sẽ tìm kiếm để *inject*
vào. Nếu `Bean` đó chưa có thì tạo mới, sau đó *inject* vào. Việc *inject* tự động này dược gọi là
**Dependency Injection**.
### Các loại Injection:
- **Constructor-based Injection**: *inject* các *module* bắt buộc, các *module* được *inject* nằm trong *Constructor*.
- **Setter-based Injection**: *inject* các *module* tùy chọn. Mỗi *module* sẽ được *inject* thông qua *Setter*.
## 5. Bean và ApplicationContext
### Bean: 
Là những *module* chính của chương trình được **IoC Container** quản lý. Có thể phụ thuộc lẫn nhau. 
### Application Context: 
Là khái niệm để chỉ **IoC Container**. Tương tự `BeanFactory` nhưng ở mức cao hơn.
### Inject Bean vào Bean khác:
- `@Autowired`: Sẽ tự động tìm và *inject* `Bean` phù hợp.
- Thông qua **Constructor** hoặc **Setter**:
  + Gọi *Constructor* truyền trực tiếp không cần *annotation*.
  + Dùng phương thức *Set* để inject (thêm annotation `@Required` để *Setter* luôn được gọi để inject).
  + Cách dùng *Setter* để *inject* thường dùng trong trường hợp phụ thuộc vòng. **Spring Boot** sẽ không biết
  nên khởi tạo *Constructor* nào trước. Nên một `Bean` sẽ dùng *Constructor* và một `Bean` dùng *Setter*.
  + Khi có nhiều `Bean` phù hợp và **Spring Boot** không biết chọn `Bean` nào:  
  -> Dùng *annotation* `@Primary`: Chỉ định thứ tự ưu tiên của `Bean`.  
  -> Dùng *annotation* `@Qualifier`: Chỉ định rõ tên `Bean` (Tên class). Đặt trước tên field.
### Bean Life Cycle (Vòng đời của Bean):
- **IoC Container** tạo `Bean`.
- Gọi các *Setter* method để *inject* các `Bean`.
- `@PostConstructor` được gọi.
- Init method được gọi.
- Sau khi không dùng nữa sẽ được hủy thông qua việc gọi `@PreDestroy`  
-> `@PostConstructor`: là sau khi `Bean` đã khởi tạo xong. Dùng để thực hiện một số task khi khởi tạo `Bean`.  
-> `@PreDestroy`: là trước khi `Bean` bị phá hủy. Thực hiện task để dọn dẹp `Bean` trước khi phá hủy.  
-> Dùng 2 *annotation* này để đánh dấu lên method, method đó sẽ tự động gọi khi sự kiện `Bean` xảy ra.
### Các loại Bean (Scope):
- **Singleton** (mặc định): **IoC Container** chỉ tạo đúng duy nhất 1 *object* từ class `Bean` này.
- **Prototype**: *return* một `Bean` *object* riêng biệt cho mỗi lần sử dụng. (`@Scope("prototype")`)
- **Request**: tạo mỗi `Bean` cho mỗi *request*.
- **Session**: tạo mỗi `Bean` cho mỗi *session*.
- **Global Session**: tạo mỗi `Bean` cho mỗi *Global Session*.
### Dùng @Bean bên trong @Configuration
- Cách này dùng cho trường hợp `Bean` cần thực hiện nhiều thao tác phức tạp để khởi tạo, hoặc có nhiều
`Bean` liên quan tới nhau.
- Do đó thay vì khởi tạo riêng lẻ thì gom chung các `Bean` cần khởi tạo lại bỏ vào class chứa. Thường
các class được đánh dấu là `@Configuration` và có hậu tố là **Config**.
- Các thao tác xử lý được đặt trong *Constructor* và khai báo các *module* bằng `@Bean`
### @Value: 
Dùng để đưa giá trị trong *application.properties* vào bằng cách: `@Value("${}")`
## 6. ModelMapper
- Là một thư viện của **Java**, giúp đơn giản hóa việc convert giữa các **Class**.
- Ví dụ: Thay vì phải copy từng *field* của class A sang class B thì làm như sau:
```java
User user = new User("john", "123456", "Nguyễn Văn John", 20);

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
                  .setMatchingStrategy(MatchingStrategies.STRICT); // Map chặt chẽ
          return modelMapper;
      }
  }
```
Lưu ý luôn test và để ý xem đã map đúng chưa!
## 7. @ControllerAdvice và @ExceptionHandler
### AOP (Aspect Oriented Programming): 
Ngắt ngang một method để thực hiện method khác, trong điều kiện gì đó.
### `@ExceptionHandler`: Xử lý exception
Cho phép xử lý *exception* trong một *controller* cụ thể. Khi một *exception* được ném ra,
hàm đánh dấu với *annotation* sẽ được gọi.
```java
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {    
        if (id <= 0) {
            throw new IllegalArgumentException("ID không hợp lệ!");
        }
  // Giả sử đây là logic tìm user
        return new User(id, "John Doe");
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgumentException(IllegalArgumentException ex) {
        return new ResponseEntity<>("Lỗi: " + ex.getMessage(), HttpStatus.BAD_REQUEST);
    }
```
### `@ControllerAdvice` /`@RestControllerAdvice`: Xử lý exception toàn cục.
```java
    @RestControllerAdvice
    public class GlobalExceptionHandler {

        // Xử lý ngoại lệ ResourceNotFoundException
        @ExceptionHandler(ResourceNotFoundException.class)
        public ResponseEntity<String> handleResourceNotFound(ResourceNotFoundException ex) {
            return new ResponseEntity<>("Không tìm thấy tài nguyên: " + ex.getMessage(), HttpStatus.NOT_FOUND);
        }

        // Xử lý IllegalArgumentException hoặc các lỗi không hợp lệ
        @ExceptionHandler(IllegalArgumentException.class)
        public ResponseEntity<String> handleIllegalArgument(IllegalArgumentException ex) {
            return new ResponseEntity<>("Dữ liệu không hợp lệ: " + ex.getMessage(), HttpStatus.BAD_REQUEST);
        }

        // Xử lý lỗi chung bất ngờ (Global Exception)
        @ExceptionHandler(Exception.class)
        public ResponseEntity<String> handleGlobalException(Exception ex) {
            return new ResponseEntity<>("Đã xảy ra lỗi: " + ex.getMessage(), HttpStatus.INTERNAL_SERVER_ERROR);
        }

        // Xử lý lỗi valid data
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
- Thường sẽ trả về dạng **JSON** thay vì chuỗi **(ErrorResponse)**.
## 8. Validation
- Thêm các annotation ràng buộc trên *field* : `@NotNull`, `@NotEmpty`, `@Email`, `@Size`,....
- Có thể thêm *message* sau đó: `@NotNull(message = "Thiếu...")`
- Sau đó trước mỗi *param* cần *valid*, thêm *annotation* `@Valid` / `@Validated`
- Cách xử lý *validation fail*:
  + Dùng tham số cuối cùng là **BindingResult**.
  + Bắt *exception*:  
    C1: 
  ```java
    @PostMapping("/")
    public User createUser(
        @RequestBody @Valid UserDto userDto,
        BindingResult bindingResult) {  // Thêm tham số này
        // Khi có BindingResult thì lỗi được tạm bỏ qua để xử lý thủ công
        // Nếu có lỗi thì chặn lại
        if (bindingResult.hasErrors()) throw new Exception("...");
    }
  ```
    C2:  
  ```java
    @ExceptionHandler(BindException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)  // Nếu validate fail thì trả về 400
    public String handleBindException(BindException e) {
        // Trả về message của lỗi đầu tiên
        String errorMessage = "Request không hợp lệ";
        if (e.getBindingResult().hasErrors())
        e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return errorMessage;
    }
  ```
## 9. Thymeleaf
**Thymeleaf** là một **Java Template Engine** có nhiệm vụ xử lý và generate ra các file *HTML, XML*,..
### Cú pháp:
- Sẽ bắt đầu bằng chữ **th**:
  + `${...}` : Dùng để lấy *value(text)* theo *key* truyền vào.  

  ```html
  <p>Today is: <span th:text="${today}"></span>.</p>
  ``` 
  + `*{...}` : Dùng để lấy giá trị của một biến cho trước bởi `th:object`
  ```html
  <div th:object="${session.user}">
    <p>Name: <span th:text="*{firstName}"></span>.</p>
  </div>
  ```
  + `@{...}` : Xử lý và trả ra *url* theo *context* của máy chủ cho chúng ta.
  ```html
  <a href="#" th:href="@{/v1/about}">Link to About me</a>
  ```
Có 2 loại đường dẫn là tuyệt đối (trước path có /) và tương đối. Tuyệt đối sẽ được dẫn từ
`8080/...` Còn tương đối sẽ chèn tiếp tục sau *url* hiện tại. Ví dụ: `...8080/home` thì nếu là `@{about}`
thì sẽ thành `.../8080/about`
## 10. Spring Boot JPA
- Là một phần trong hệ sinh thái **Spring Data**, nó tạo một layer giữa tầng **Service** và *Database*.
- Giúp ta thao tác với **Database** dễ hơn, tự động *config*, code gọn hơn.
- Trong **Spring Boot**, **Hibernate** là *impl*, **JPA** là *API* tiêu chuẩn.
### Cách dùng:
- Khởi tạo một *interface repository* và đánh dấu `Bean`, `extend JpaRepository<T, Type>`
- *Inject interface* vào dự án và gọi các *impl* để dùng.
### Cơ chế:
- **Spring Data JPA** cung cấp một cơ chế mạnh cho phép định nghĩa các *method* theo một cấu trúc thì sẽ tự động
thực hiện.
- **Method Name Query Derivation**: Tạo truy vấn từ tên phương thức.
### Quy tắc đặt tên:
- Tiền tố: `findBy`.., `readBy`.., `countBy`.., `existsBy`..,...
- Tên trường: chuẩn tên được ánh xạ
- Toán tử (nếu có): `And`, `Or`, `GreaterThan`, `LessThan`, `Like`, `Between`,...
- Ví dụ: `User findFirstByOrderBySttDesc()` = `SELECT * FROM user ORDER BY stt DESC LIMIT 1`
- `@Query`: Nếu không muốn *JPA* tự tạo query tự động thì có thể tự custom
```java
    @Query("select u from User u where u.emailAddress = ?1")
    User myCustomQuery(String emailAddress);
```
## 11. Normalize API
- Các *api* khi trả về phía **Client** phải có ý nghĩa: `code`, `message`, `result`,...   
-> Tạo một class `ApiResponse<T>` để có thể *custom api* trả về.
- `@JsonInclude(JsonInclude.Include.NON_NULL)`: Nếu có *field* là *null* thì *api* sẽ ẩn đi *field* đó.
## 12. JWT (Json Web Token)
### JWT 
- Là một phương tiện đại diện cho các yêu cầu chuyển giao giữa **Client-Server**.
- Các thông tin trong chuỗi được định dạng bằng **JSON**.
- Một chuỗi *token* có định dạng như sau: **header.payload.signature**
  + **HEADER**: Algorithm & Token Type
  + **PAYLOAD**: Data
  + **SIGNATURE**: Verification
- Khi nào nên sử dụng:
  + **Authentication**: Các *request* phía người dùng sẽ kèm theo *JWT*, điều này cho phép người dùng có thể truy
    cập vào các *resource* mà *token* đó cho phép.
- Luồng xử lý:
  + **User** thực hiện thao tác *request* (login,..).
  + **Authentication Server** tạo *JWT* và trả về cho **User** thông qua *response*.
  + **User** nhận *JWT* trả về làm chìa khóa để thực hiện các "lệnh" tiếp theo.
  + **API** xác thực *JWT* đó và cấp quyền truy cập cho **User**.
- NO SESSIONS:
  + Với *JWT* sẽ không cần giữ *sessions* trong suốt quá trình mà luồng xử lý sẽ như trên, thông qua *token*.
  + **No sessions storage**: không có *session* đồng nghĩa không cần lưu *sessions* -> tối ưu bộ nhớ.
  + **Truely RESTful Services**: khi không có *sessions* thì mới có thể tạo nên 1 *service* thuần **RESTful**.
### Cách gen token:
```java
  @NonFinal
  protected static final String SIGNER_KEY = "IOg/flCDdPNoC2+j4/G0m3l8FhX7paEreFwAeO2eFf4YhZabokoyI38afEtUGut7\n";
  private String generateToken(String username) {
    JWSHeader header = new JWSHeader(JWSAlgorithm.HS512);
    JWTClaimsSet jwtClaimsSet = new JWTClaimsSet.Builder().
        subject(username).
        issuer("demo.com").
        issueTime(new Date()).
        expirationTime(new Date(Instant.now().plus(1, ChronoUnit.HOURS).toEpochMilli())).
        claim("Custom claim", "Custom").
        build();
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
## 13. Các annotation hữu ich
- `@Builder`: Dùng để giúp 1 class có thể build các field một cách nhanh chóng.
```java 
    return AuthResponse.builder().token(token).authenticated(true).build();
```
- `@FieldDefaults`: Dùng để định nghĩa 1 *field* nếu không khai báo phạm vi hay bất kì gì khác:
```java
    @FieldDefaults(level = AccessLevel.PRIVATE, makeFinal = true)
```
- `@JsonInclude(JsonInclude.Include.NON_NULL)`: Dùng để **JSON** không trả về các *field null*.
- `@RequiredArgsConstructor`: Dùng để yêu cầu có một *Constructor* đầy đủ (ứng dụng trong việc inject `Bean`).
- `@PreAuthorize("hasAuthority('SCOPE_ADMIN')")`: Dùng để phân quyền hiệu quả cùng với `@EnableMethodSecurity`. Trước khi
  thực hiện *method* thì sẽ kiểm tra.
- `@PostAuthorize`: Kiểm tra sau khi *method* thực hiện.
```java
    @PostAuthorize("returnObject.username == authentication.name"): Kiểm tra điều kiện.
```
## 14. Spring Security
### 1. Khái niệm: 
***Spring Security*** cung cấp các tính năng xác thực và phân quyền, cũng như hỗ trợ các tiêu chuẩn 
và giao thức bảo mật như *HTTPS, OAuth2, JWT, ...*
### Cơ chế hoạt động: 
Dựa trên cơ chế lọc *(filter)* và sự kiện *(event)* để can thiệp vào quá trình xử lý *request* và 
*response*. 
### Gồm 3 phần: Authentication, Authorization, Authentication Provider.
#### Authentication:
Là quá trình xác thực xem người dùng có quyền truy cập vào ứng dụng hay không.  
Thường dựa trên các thông tin nhận dạng *(identifier)* và thông tin bí mật *(credential)* của người dùng.
- **Cơ chế**:
  + **Form-based authentication**: Xác thực thông qua một form đăng nhập.
  + **HTTP Basic authentication**: Xác thực thông qua các *header authorization*.
  + **Authentication via a custom login page**: Xác thực thông qau một trang đăng nhập tùy chỉnh.
  + **Pre-authenticated authentication**: Xác thực thông qua các giá trị được cung cấp phía **Client**. 
- **Stateful** và **Stateless**:
  + **Stateful**: là một các tiếp cận xác thực trong đó hệ thống sẽ lưu trữ thông tin xác
        thực của người dùng hoặc ứng dụng trong một *session* trên máy chủ.
  + **Stateless**: hệ thống sẽ chỉ sử dụng các mã *token* đã được ký số để xác thực thông tin.
        Cách này khá phổ biến bằng cách dùng *JWT token* và mang lại nhiều lợi ích to lớn.
#### Authorization:
- Là quá trình xác định quyền truy cập của người dùng đối với các tài nguyên trong ứng dụng.
- Thường dựa trên các thông tin như: `role`, `group`, `permission`, `policy`,...
#### Authentication Provider:
- Là một thành phần quan trọng chịu trách nhiệm xác minh thông tin xác thực của người dùng.
- Được sử dụng bởi **Authentication Manager** để xử lý.
- **Authentication Provider** chỉ hỗ trợ một loại **Authentication** cụ thể như:  
`UsernamePasswordAuthenticationToken`, `JwtAuthenticationToken`, `PreAuthenticatedAuthenticationToken`,... 
### Các tính năng nâng cao của Spring Security
- **CSRF protection** (Bảo vệ chống lại tấn công CSRF).
- **Session management**
- **Password encoding**
### Ưu và nhược điểm của Spring Security
#### Ưu điểm:
- Là một *framework* bảo mật mạnh mẽ và linh hoạt, hỗ trợ rất nhiều tiêu chuẩn và giao thức bảo
mật.
- Được tích hợp sẵn với **Spring Framework**, giúp việc phát triển ứng dụng web an toàn, hiệu quả.
#### Nhược điểm:
- Cấu hình có thể khá phức tạp và khó hiểu, đặc biệt khi làm việc với các tính năng nâng cao.
- Một số tính năng có thể không phù hợp với loại ứng dụng web
- Yêu cầu kiến thức chuyên môn về bảo mật để sử dụng hiệu quả.