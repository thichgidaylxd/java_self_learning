# Tài liệu tự học lại: Spring Core → JPA Auditing → JWT → Spring Security

> Viết cho người **không biết gì**, đọc lại từ đầu vẫn hiểu được. Theo
> đúng thứ tự nên học — mỗi phần dựa vào phần trước, đừng nhảy cóc.

---

## PHẦN 0 — HTTP/HTTPS (nền tảng trước tất cả)

### HTTP là gì
Giao thức (protocol) quy định cách 2 máy nói chuyện qua mạng, theo mô
hình **Request → Response**: client hỏi trước, server luôn trả lời sau.
Server không tự "gọi" client trước.

### Cấu trúc 1 Request — nhớ tắt "M-P-H-B"

| Ký hiệu | Tên | Ý nghĩa | Ví dụ |
|---|---|---|---|
| M | Method | Hành động | GET / POST / PUT / DELETE |
| P | Path | Địa chỉ tài nguyên | /api/persons |
| H | Headers | Thông tin phụ | Authorization: Bearer xxx |
| B | Body | Dữ liệu chính | {"fullName": "A"} |

**JWT luôn nằm ở Headers**, không nằm ở Body hay URL.

### Response
Gồm **Status code** (3 số) + **Body**.

- `2xx` = OK
- `4xx` = LỖI DO CLIENT
  - `401` = "bạn là ai?" — chưa xác định được danh tính (token sai/thiếu)
  - `403` = "tôi biết bạn, nhưng không đủ quyền"
  - `404` = không tìm thấy
- `5xx` = LỖI DO SERVER (500 = lỗi bất ngờ)

### HTTPS = HTTP + mã hoá (TLS/SSL) — 3 mục tiêu "BÍ - TOÀN - XÁC"
1. **BÍ mật (Confidentiality)** — mã hoá, ai chặn giữa đường không đọc được
2. **TOÀN vẹn (Integrity)** — phát hiện được nếu dữ liệu bị sửa giữa đường
3. **XÁC thực server (Authentication)** — chứng chỉ SSL chứng minh đúng server thật

**Liên hệ nhớ lâu:** SePay yêu cầu cả HTTPS lẫn HMAC-SHA256 vì đây là
**2 việc khác nhau**: HTTPS chống **đọc trộm**, HMAC chống **sửa nội
dung**. Không cái nào thay được cái kia.

---

## PHẦN 1 — Nền tảng Spring: IoC & Dependency Injection

### Vấn đề cần giải quyết
Không có Spring, muốn `PersonController` dùng `PersonService` (mà
`PersonService` lại cần `PersonRepository`), phải tự tay:
```java
PersonRepository repo = new PersonRepository();
PersonService service = new PersonService(repo);
PersonController controller = new PersonController(service);
```
Với hàng trăm object phụ thuộc chéo nhau, tự ghép nối bằng tay cực kỳ
dễ sai và khó bảo trì.

### IoC (Inversion of Control) — đảo ngược quyền kiểm soát
Thay vì `PersonController` tự tạo `PersonService`, bạn chỉ **khai báo**
"tôi cần 1 PersonService", còn **Spring lo việc tạo ra và đưa cho bạn**:

```java
@RestController
public class PersonController {
    private final PersonService personService;

    public PersonController(PersonService personService) { // Spring tự đưa vào
        this.personService = personService;
    }
}
```

Spring giữ 1 "kho" chứa sẵn mọi object đã tạo, gọi là **ApplicationContext**.

### Dependency Injection (DI) — cơ chế cụ thể hiện thực IoC
"Dependency" = sự phụ thuộc (Controller phụ thuộc Service để hoạt động).
"Injection" = hành động Spring **đưa (tiêm)** object đó vào, thường qua
constructor. `@RequiredArgsConstructor` (Lombok) tự sinh constructor
nhận các field `final` để Spring có chỗ tiêm vào.

**"Bean"** = 1 object được Spring tạo ra và quản lý vòng đời.

### `@Component` — tín hiệu để Spring tạo Bean
Gắn lên class để nói: *"class này, hãy tạo 1 instance và quản lý nó"*.
Lúc khởi động, Spring **quét (component scan)** toàn bộ package, tìm
class có `@Component` (hoặc annotation "họ hàng") thì tạo Bean.

### `@Service`, `@Repository`, `@Controller` — chỉ là `@Component` đội lốt tên khác
```java
@Component public @interface Service { }
@Component public @interface Repository { }
@Component public @interface Controller { }
```
Về kỹ thuật, cả 3 làm đúng 1 việc như `@Component`. Khác nhau ở:
1. **Ý nghĩa cho người đọc code** (semantic) — không có tác dụng kỹ thuật
2. **`@Repository` có xử lý riêng thật sự** — tự "dịch" exception thô
   của JDBC/Hibernate thành exception chuẩn của Spring
   (`DataIntegrityViolationException`)

`@RestController` = `@Controller` + `@ResponseBody` — tự động biến giá
trị trả về thành JSON trong body response.

### `@Configuration` + `@Bean` — khi KHÔNG viết ra class đó
`@Component` chỉ dùng được khi **bạn là người viết ra class**. Với các
object từ thư viện ngoài (`PasswordEncoder`, `SecurityFilterChain`...),
dùng:

```java
@Configuration          // "class này CHỨA các khai báo Bean thủ công"
public class SecurityConfig {

    @Bean               // "kết quả trả về của method, đưa vào kho Bean"
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**Phân biệt cốt lõi:**
| | `@Component` | `@Configuration` + `@Bean` |
|---|---|---|
| Dùng cho | Class của bạn | Class không phải của bạn, hoặc cần đặt tên Bean tường minh |
| Cách tạo | Spring tự `new` khi quét thấy | Bạn tự `new`/tạo trong thân method |

**Ví dụ khi CẦN `@Bean` dù class đó do bạn viết:** nested class (class
lồng bên trong class khác) không nên/không tiện dùng `@Component` (quy
ước code, không phải Spring cấm), và khi cần **đặt tên Bean chính xác**
để nơi khác tham chiếu bằng string (ví dụ
`@EnableJpaAuditing(auditorAwareRef = "auditorAware")` cần đúng 1 Bean
tên `"auditorAware"`).

### Dependency Injection + Singleton + nhiều Thread — cơ chế thật

**Wiring (chỉ 1 lần, lúc app khởi động):** Spring tạo hết Bean
singleton, rồi **nối chúng lại** ngay lúc đó — gán tham chiếu vào field,
xong 1 lần, không lặp lại:
```java
PersonService instance = new PersonService(...);      // tạo 1 lần
PersonController controller = new PersonController(instance); // tiêm 1 lần
```

**Xử lý request (liên tục, nhiều thread cùng lúc):** Tomcat giao mỗi
request cho 1 thread. Thread gọi `controller.createPerson(...)` — chỉ
là **truy cập field đã gán sẵn từ bước Wiring**, không có bước "tìm lại"
nào chạy ở đây. Về bản chất là truy cập field Java bình thường.

**Vì sao singleton + nhiều thread AN TOÀN nếu field là `final`,
immutable (Repository, Service khác)** — nhiều thread cùng đọc 1 giá
trị không đổi thì không tranh chấp.

**NGUY HIỂM nếu Bean có field mutable lưu trạng thái riêng theo từng
request** — 2 thread cùng đọc/ghi 1 field dùng chung → race condition,
dữ liệu lẫn lộn. **Quy tắc:** dữ liệu theo từng request phải truyền qua
tham số/biến local, không lưu vào field của Bean singleton.

---

## PHẦN 2 — JPA Auditing

### Vấn đề cần giải quyết
Không có JPA Auditing, mỗi lần tạo/sửa Entity phải tự tay gán
`createdAt`, `createdBy`, `updatedAt` — dễ quên ở đâu đó, dữ liệu sai
âm thầm.

### Cơ chế nền: JPA Entity Lifecycle Events
Chuẩn JPA (Hibernate hiện thực) cho phép "móc" (hook) code vào các thời
điểm trong vòng đời Entity:
```
Entity mới tạo → @PrePersist (trước khi INSERT) → INSERT vào DB
Entity sửa     → @PreUpdate (trước khi UPDATE)  → UPDATE vào DB
```
`@PrePersist`/`@PreUpdate` thuộc **chuẩn JPA**, không riêng của Spring.

### `@EntityListeners(AuditingEntityListener.class)`
`AuditingEntityListener` là class có sẵn của Spring Data, bên trong đã
viết sẵn method gắn `@PrePersist`/`@PreUpdate`. Annotation này nói với
Hibernate: *"mỗi khi entity sắp insert/update, gọi thêm các method
trong AuditingEntityListener"*.

Bên trong (ý tưởng): dùng **reflection** tìm field có
`@CreatedDate`/`@CreatedBy`/`@LastModifiedDate`/`@LastModifiedBy`, tự
gán giá trị vào — đây là lý do bạn chỉ cần gắn đúng annotation lên
đúng field, không cần viết logic thêm.

### `@CreatedDate` vs `@CreatedBy` — khác nhau ở nguồn dữ liệu
- `@CreatedDate` — chỉ cần `LocalDateTime.now()`, tự lấy được
- `@CreatedBy` — cần biết "ai đang thao tác" — **JPA/Hibernate không
  thể tự biết** (chỉ biết về DB, không biết gì về HTTP/JWT/đăng nhập)
  → Spring bắt bạn tự cung cấp câu trả lời qua `AuditorAware<T>`

### `@MappedSuperclass` — vì sao không phải `@Entity`
`@Entity` = "class này có bảng riêng trong DB". `BaseEntity` chỉ là
khuôn field dùng chung, không phải khái niệm nghiệp vụ có bảng riêng
→ dùng `@MappedSuperclass`: field của nó "trộn vào" bảng của class con
extends nó, không tạo bảng `base_entity` riêng.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @Id @GeneratedValue
    private UUID id;

    @CreatedBy    private UUID createdBy;
    @LastModifiedBy private UUID updatedBy;
    private UUID deletedBy;

    @CreatedDate  private LocalDateTime createdAt;
    @LastModifiedDate private LocalDateTime updatedAt;
    private LocalDateTime deletedAt;
}
```

### `@EnableJpaAuditing` — công tắc tổng
```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorAware")
public class JpaAuditingConfig { ... }
```
Không có annotation này ở đâu đó trong app, toàn bộ cơ chế trên **không
hoạt động**, dù đã gắn đủ annotation lên `BaseEntity`.

### `ThreadLocal` — cơ chế nền của `SecurityContextHolder`

**Vấn đề:** "ai đang đăng nhập" thay đổi theo từng request (thread khác
nhau), nhưng Bean lại singleton (1 instance dùng chung) → không thể lưu
vào field thường (sẽ bị ghi đè lẫn nhau giữa các thread, đúng race
condition đã học ở Phần 1).

**Giải pháp:** `ThreadLocal` — 1 biến duy nhất, nhưng **mỗi thread nhìn
vào lại thấy giá trị khác nhau**:
```java
ThreadLocal<String> currentUser = new ThreadLocal<>();
// Thread A: currentUser.set("User X"); currentUser.get() -> "User X"
// Thread B (song song): currentUser.set("User Y"); currentUser.get() -> "User Y"
```
Mỗi `Thread` trong Java tự mang theo 1 map riêng (`Map<ThreadLocal, Object>`).
`.set()`/`.get()` thao tác trên map riêng của thread đang chạy dòng
code đó.

`SecurityContextHolder` chỉ là 1 lớp bọc quanh `ThreadLocal`:
```java
public class SecurityContextHolder {
    private static ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();
    public static SecurityContext getContext() { return contextHolder.get(); }
}
```

### `AuditorAware<T>` — hợp đồng trả lời "ai đang thao tác"
```java
public interface AuditorAware<T> {
    Optional<T> getCurrentAuditor();
}
```
Implementation đọc từ `SecurityContextHolder` (ThreadLocal, riêng theo
thread):
```java
public Optional<UUID> getCurrentAuditor() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

    if (authentication == null || !authentication.isAuthenticated()
            || "anonymousUser".equals(authentication.getPrincipal())) {
        return Optional.empty();
    }

    Object principal = authentication.getPrincipal();
    try {
        return Optional.of(UUID.fromString(principal.toString()));
    } catch (IllegalArgumentException e) {
        return Optional.empty();
    }
}
```

**Vì sao cần `@Bean` (không `@Component`) cho implementation này:**
1. Là nested class (lồng trong `JpaAuditingConfig`) — quy ước code
   không nên `@Component` cho nested class
2. `@EnableJpaAuditing(auditorAwareRef = "auditorAware")` cần đúng 1
   Bean **tên** là `"auditorAware"` — `@Bean` cho phép đặt tên qua tên
   method, `@Component` tự sinh tên khác (theo tên class), sẽ lệch

### Toàn bộ luồng khi tạo 1 `Person` mới
```
1. personRepository.save(newPerson)
2. Hibernate chuẩn bị INSERT → kích hoạt @PrePersist
3. @EntityListeners → gọi AuditingEntityListener.touchForCreate()
4. Soi field, thấy @CreatedDate → gán now()
5. Thấy @CreatedBy → hỏi Bean "auditorAware" → lấy UUID từ SecurityContext
   (ThreadLocal riêng theo thread)
6. Gán UUID vào createdBy
7. Hibernate chạy INSERT với đầy đủ dữ liệu
```
Cần **đủ cả 4 mảnh**: `@MappedSuperclass` + `@EntityListeners` +
`@EnableJpaAuditing` + `AuditorAware` Bean — thiếu 1, cơ chế im lặng
không báo lỗi, field chỉ là `null`.

---

## PHẦN 3 — JWT (JSON Web Token)

### Vấn đề cần giải quyết
Hệ thống **stateless** (không session) — mỗi request phải tự chứng
minh "tôi là ai" mà không ai giả mạo được.

### Cấu trúc — 3 phần cách nhau bởi dấu chấm
```
eyJhbGci....eyJzdWIi....SflKxw...
Header      .Payload    .Signature
```
Mỗi phần là JSON được **Base64Url encode** (không phải mã hoá bảo mật —
**ai cũng giải mã đọc được**).

**Header:** `{"alg": "HS256", "typ": "JWT"}` — thuật toán ký.

**Payload:** `{"sub": "123", "exp": 1730000000, "iat": ...}` — claims.
Payload **KHÔNG được mã hoá**, ai chặn được token đều đọc được nội
dung → **không bao giờ nhét mật khẩu/dữ liệu nhạy cảm vào đây**.

**Signature — phần bảo mật thật sự:**
```
signature = HMAC-SHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secretKey
)
```
Cùng nguyên lý HMAC-SHA256 mà SePay dùng ký webhook.

### HMAC-SHA256 — lõi cơ chế
Hàm nhận 2 đầu vào (dữ liệu + secretKey), cho ra 1 hash cố định. Tính
chất:
1. **1 chiều** — không suy ngược từ hash ra dữ liệu gốc
2. **Nhạy cảm cực độ** — đổi 1 ký tự, hash khác hoàn toàn
3. **Không biết secretKey thì không tính ra đúng hash**

### Vì sao ký chống được giả mạo — kịch bản tấn công
```
1. Kẻ tấn công chặn được JWT, sửa payload "123" -> "999"
2. Không biết secretKey -> không tính lại được signature đúng
3. Gửi lên: header.newPayload.oldSignature (chữ ký cũ, không khớp payload mới)
4. Server tự tính lại signature từ payload mới, so sánh -> KHÔNG khớp
5. Server từ chối -> "invalid signature"
```

### Confidentiality vs Integrity — phân biệt quan trọng
| | Confidentiality (bí mật) | Integrity (toàn vẹn) |
|---|---|---|
| Câu hỏi | Ai đọc được nội dung? | Nội dung có bị sửa không? |
| JWT payload | KHÔNG đảm bảo (Base64 không phải mã hoá) | CÓ đảm bảo (nhờ signature) |

**Vì sao vẫn an toàn chứa `userId`:** biết `userId` không giúp kẻ tấn
công làm được gì — họ vẫn không có `secretKey` để **tự tạo** token mới
mà server chấp nhận. Đọc được ≠ giả mạo được. **Quy tắc:** trước khi
cho dữ liệu vào payload, tự hỏi "nếu ai đó đọc được (không sửa), có
hại gì không?" — không hại thì OK (`userId`, `email`, `role`), có hại
thì không bao giờ (`password`, số thẻ...).

### Access Token vs Refresh Token
| | Access Token | Refresh Token |
|---|---|---|
| Dùng để | Gọi API trực tiếp | CHỈ để xin access token mới |
| Thời hạn | Ngắn (15 phút) | Dài (7 ngày) |
| Rủi ro nếu lộ | Thấp — tự hết hạn nhanh | Cao hơn nhưng ít dùng, ít lộ hơn |

**Luồng:** login → nhận cả 2 token → dùng access gọi API → hết hạn
(401) → gọi `/api/auth/refresh` kèm refresh token → nhận access token
mới, không phải đăng nhập lại → refresh cũng hết hạn → bắt đăng nhập
lại thật.

**JWT không thể "thu hồi" giữa chừng** (vì stateless, server không lưu
gì) — token hợp lệ tới khi tự hết hạn. Đây là lý do access token nên
sống ngắn.

### Public Key / Private Key (RSA) — khi nào dùng thay vì HMAC

**Vấn đề của HMAC:** chỉ 1 khoá cho cả ký và verify — nếu nhiều bên
khác nhau cần verify (như hàng triệu app verify token của Google),
phải chia sẻ secretKey cho tất cả → rủi ro lộ khoá cao, lộ là ký giả
được luôn.

**Giải pháp — cặp khoá bất đối xứng:**
```
Private Key → CHỈ để KÝ    → chỉ bên phát hành giữ (vd: Google)
Public Key  → CHỈ để VERIFY → công khai, ai cũng lấy được
```
Ký bằng Private Key chỉ verify đúng được bằng Public Key tương ứng.
Không thể suy ngược Public Key ra Private Key.

| | HMAC (đối xứng) | RSA/ECDSA (bất đối xứng) |
|---|---|---|
| Số khoá | 1 (dùng chung) | 2 (private ký, public verify) |
| Ai verify được | Chỉ ai có secretKey | Bất kỳ ai có Public Key |
| Lộ khoá verify | Nghiêm trọng (ký giả được) | Không sao (Public Key vốn công khai) |
| Phù hợp khi | 1 hệ thống tự ký + tự verify (JWT nhà mình) | Nhiều bên khác nhau cần verify (Google ID token) |

**Google công bố Public Key** tại URL cố định (JWKS):
`https://www.googleapis.com/oauth2/v3/certs`. Luồng verify: đọc claim
`kid` (key ID) trong header token → lấy đúng public key tương ứng từ
JWKS → verify signature bằng key đó.

**Không nhầm với BCrypt:** BCrypt (hash mật khẩu) là hash 1 chiều để
**so sánh**, không có khoá để giải mã ngược — khác hoàn toàn cơ chế ký
số bằng Public/Private key.

---

## PHẦN 4 — CSRF, Cookie vs Header

### CSRF là gì — cơ chế tấn công
Lợi dụng việc **cookie tự động gửi kèm** mỗi request tới đúng domain,
kể cả khi request đó do 1 trang độc hại khác tạo ra:
```
1. Bạn đăng nhập app-that.com -> cookie chứa session/JWT được lưu
2. Bạn (vẫn đăng nhập) vô tình vào trang xau.com
3. xau.com có form ẩn, tự submit tới app-that.com/api/transfer-money
4. Trình duyệt TỰ ĐỘNG gắn cookie vào request đó (vì gửi TỚI đúng domain)
5. Server verify -> hợp lệ (cookie thật) -> xử lý, dù bạn không chủ ý
```
**Chữ ký JWT không giúp được gì ở đây** — token hoàn toàn thật, chỉ là
bị gửi không theo ý muốn của chủ nhân. CSRF không phải "giả mạo token",
mà là "lừa trình duyệt tự gửi request thay bạn".

### Header (`Authorization: Bearer`) miễn nhiễm CSRF vì sao
Header **phải được code chủ động gắn vào** (JS tự thêm) — trang độc hại
không có cách nào ra lệnh cho JS của app bạn tự thêm header đó vào
request nó tự tạo.

### `SameSite` cookie — cơ chế trình duyệt hiện đại tự chống CSRF
```
Set-Cookie: token=xxx; HttpOnly; SameSite=Strict
```
`SameSite=Strict`: cookie chỉ gửi kèm khi request xuất phát từ **chính
domain đó** — trang khác tạo request tới domain bạn sẽ không kèm cookie
nữa.

### CSRF vs Token Theft (đánh cắp token) — 2 loại tấn công khác nhau
| | CSRF | Token Theft |
|---|---|---|
| Attacker cần biết giá trị token? | Không | Có — phải lấy được token thật trước |
| Cơ chế | Lợi dụng cookie tự động gắn | Đọc trộm qua XSS, log lộ, sniffing |
| Header có miễn nhiễm? | Có | **Không** — XSS đọc được biến lưu token thì header cũng dính |
| Phòng chống | SameSite, CSRF token, dùng header | HTTPS, httpOnly cookie, token sống ngắn |

**Bảng tổ hợp thực tế:**
| Tổ hợp | CSRF risk? |
|---|---|
| Session + Cookie thường | Có |
| JWT + Header | Không |
| JWT + Cookie (hybrid) | Có (vẫn là cookie, dù nội dung là JWT) |
| Session + Header (hiếm) | Không |

**Nơi lưu JWT phía client:**
| Nơi lưu | Tránh CSRF? | Rủi ro |
|---|---|---|
| localStorage/sessionStorage | Có | Dễ bị đọc trộm nếu có lỗ hổng XSS |
| Cookie httpOnly + SameSite=Strict | Có | JS không đọc được, nhưng vẫn cần đúng 2 cờ |
| Cookie thường | Không | Vừa dính CSRF, vừa dính XSS |

**Kết luận cho hệ thống JWT qua header (đang dùng):**
`.csrf(csrf -> csrf.disable())` là **an toàn**, vì CSRF chỉ khai thác
được cơ chế tự động gắn của cookie — JWT qua header không có cơ chế
đó.

---

## PHẦN 5 — Spring Security Filter Chain

### Servlet Filter — cơ chế nền, có trước cả Spring
Request đi qua **chuỗi Filter** nối tiếp nhau trước khi tới
`DispatcherServlet` (điều hướng tới `@Controller`):
```
Request → Filter 1 → Filter 2 → Filter 3 → DispatcherServlet → Controller
```
Mỗi Filter có 2 quyền: **chặn lại** (tự trả response, không gọi tiếp)
hoặc **cho đi tiếp** (`chain.doFilter(...)`).

### Spring Security = 1 chuỗi Filter đặc biệt, không phải cơ chế riêng
`SecurityFilterChain` là **cấu hình, lắp ráp** nhiều Filter con có sẵn
(CSRF Filter, CORS Filter, Authorization Filter...) theo thứ tự hợp lý
đã định sẵn — không phải bạn viết if/else thủ công.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
                .requestMatchers(PUBLIC_PATHS).permitAll()
                .anyRequest().authenticated()
        )
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

`HttpSecurity http` (tham số) là ví dụ khác của DI — Spring tự tạo sẵn,
tiêm vào để bạn cấu hình, không tự `new HttpSecurity()`.

### Giải nghĩa từng dòng

**`sessionCreationPolicy(STATELESS)`** — nói Spring: đừng bao giờ tự
tạo `HttpSession`, mỗi request phải tự đủ thông tin chứng minh danh
tính (qua JWT), không dựa vào trạng thái server nhớ.

**`authorizeHttpRequests`** — chính là **Authorization Filter**, Filter
**cuối** trong chuỗi Security. Nhìn path, đối chiếu quy tắc theo **thứ
tự khai báo** (quy tắc đầu tiên khớp sẽ áp dụng), quyết định cho qua
hay chặn.

`.anyRequest().authenticated()` — kiểm tra `SecurityContext` có
`Authentication` hợp lệ không (đúng object mà `JwtAuthenticationFilter`
set vào) — không có thì `401`.

### Vị trí chèn `JwtAuthenticationFilter`
```java
http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
```
**Phải chèn TRƯỚC Authorization Filter** — vì Authorization Filter cần
đọc `SecurityContext` đã có Authentication rồi mới quyết định. Chèn
sau thì Authorization Filter luôn thấy SecurityContext trống →
luôn 401 dù JWT hợp lệ.

### Toàn bộ luồng 1 request
```
1. Request tới Tomcat
2. Qua chuỗi Filter Servlet, trong đó có FilterChainProxy (điểm vào Spring Security)
3. CORS Filter -> JwtAuthenticationFilter -> Authorization Filter
4. JwtAuthenticationFilter: đọc header, verify JWT -> set Authentication
   vào SecurityContext (ThreadLocal riêng theo thread)
5. Authorization Filter: kiểm tra path + SecurityContext -> qua hoặc 401
6. Qua được -> DispatcherServlet -> đúng @RestController
7. Lúc save() Entity -> AuditingEntityListener đọc lại SecurityContext -> điền created_by
```

---

## PHẦN 6 — `Authentication` object — cấu trúc chi tiết

`Authentication` là **interface**, nhiều implementation khác nhau tuỳ
tình huống:
```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();
    Object getDetails();
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated);
}
```

| Field | Ý nghĩa | Trong hệ thống JWT |
|---|---|---|
| `principal` | Ai đang được xác thực | UUID hoặc `CustomUserDetails` lấy từ claim `sub` |
| `credentials` | Bằng chứng | `null` — JWT signature đã là bằng chứng, verify xong không cần giữ |
| `authorities` | Được phép làm gì (role) | Danh sách `GrantedAuthority`, có thể rỗng ban đầu |
| `authenticated` | Đã xác thực xong chưa | `true` — set khi verify JWT thành công |
| `details` | Thông tin phụ (IP...) | Thường không dùng |

**`getAuthorities()` liên kết với `@PreAuthorize`:**
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteTree(UUID treeId) { ... }
```
Spring đọc `getAuthorities()` của Authentication hiện tại, kiểm tra có
`"ROLE_ADMIN"` không — không có thì `403` (khác `401` — đã biết là ai,
chỉ là không đủ quyền).

**Lưu ý riêng cho hệ thống này:** `role` (`tree_memberships.role`)
**không phải giá trị toàn cục** — phụ thuộc từng `tree_id` (1 user có
thể là `owner` ở tree này, `viewer` ở tree khác). Vì vậy **quyết định:
KHÔNG nhúng role vào JWT**, luôn query `tree_memberships` khi cần biết
quyền — đảm bảo luôn đúng mới nhất (JWT sống 15 phút, role có thể đổi
bất kỳ lúc nào giữa chừng).

**`UsernamePasswordAuthenticationToken`** — implementation có sẵn, dùng
được cho cả JWT dù tên gợi liên tưởng login form:
```java
var authentication = new UsernamePasswordAuthenticationToken(
    userId,   // principal
    null,     // credentials
    List.of() // authorities
);
```

---

## PHẦN 7 — Ráp lại: code thật đã viết

### Quyết định thiết kế đã chốt
```
Access token:  15 phút
Refresh token: 7 ngày
Payload:       { sub: userId, email, type: "access"|"refresh", iat, exp }
Thuật toán:    HS256 (HMAC-SHA256)
Role:          KHÔNG nhúng vào JWT — luôn query tree_memberships khi cần
```

### `JwtTokenProvider` — sign & verify
```java
@Component
public class JwtTokenProvider {

    private final byte[] secretKeyBytes;
    private final long accessTokenExpirationMs;
    private final long refreshTokenExpirationMs;

    public JwtTokenProvider(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.access-token-expiration-ms:900000}") long accessTokenExpirationMs,
            @Value("${jwt.refresh-token-expiration-ms:604800000}") long refreshTokenExpirationMs
    ) {
        this.secretKeyBytes = secret.getBytes(); // HS256 cần tối thiểu 256 bit (32 byte)
        this.accessTokenExpirationMs = accessTokenExpirationMs;
        this.refreshTokenExpirationMs = refreshTokenExpirationMs;
    }

    public String generateAccessToken(UUID userId, String email) {
        return generateToken(userId, email, accessTokenExpirationMs, "access");
    }

    public String generateRefreshToken(UUID userId) {
        return generateToken(userId, null, refreshTokenExpirationMs, "refresh");
    }

    private String generateToken(UUID userId, String email, long expirationMs, String tokenType) {
        Instant now = Instant.now();
        JWTClaimsSet.Builder claimsBuilder = new JWTClaimsSet.Builder()
                .subject(userId.toString())
                .claim("type", tokenType)
                .issueTime(Date.from(now))
                .expirationTime(Date.from(now.plusMillis(expirationMs)));
        if (email != null) claimsBuilder.claim("email", email);

        SignedJWT signedJWT = new SignedJWT(new JWSHeader(JWSAlgorithm.HS256), claimsBuilder.build());
        signedJWT.sign(new MACSigner(secretKeyBytes));
        return signedJWT.serialize();
    }

    public JWTClaimsSet validateAndGetClaims(String token, String expectedType) {
        SignedJWT signedJWT = SignedJWT.parse(token);
        if (!signedJWT.verify(new MACVerifier(secretKeyBytes)))
            throw new JwtValidationException("Chữ ký JWT không hợp lệ");

        JWTClaimsSet claims = signedJWT.getJWTClaimsSet();
        if (claims.getExpirationTime() == null || claims.getExpirationTime().before(new Date()))
            throw new JwtValidationException("JWT đã hết hạn");

        String actualType = claims.getStringClaim("type");
        if (!expectedType.equals(actualType))
            throw new JwtValidationException("Sai loại token");

        return claims;
    }
}
```

### `JwtAuthenticationFilter` — nối JWT vào SecurityContext
```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {

        String token = extractToken(request); // đọc header "Authorization: Bearer xxx"

        if (token != null) {
            try {
                JWTClaimsSet claims = jwtTokenProvider.validateAndGetClaims(token, "access");
                UUID userId = jwtTokenProvider.getUserId(claims);

                var authentication = new UsernamePasswordAuthenticationToken(userId, null, List.of());
                SecurityContextHolder.getContext().setAuthentication(authentication);

            } catch (JwtValidationException e) {
                // KHÔNG trả 401 ở đây — để Authorization Filter tự quyết định
                log.debug("JWT không hợp lệ: {}", e.getMessage());
            }
        }

        filterChain.doFilter(request, response); // luôn cho đi tiếp
    }
}
```

**Nguyên tắc quan trọng:** Filter này **chỉ lo xác định danh tính**,
không tự trả lỗi — Authorization Filter (chạy sau) mới quyết định chặn
dựa trên rule `permitAll()`/`authenticated()`.

### `SecurityConfig` — ráp mọi thứ lại
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    private static final String[] PUBLIC_PATHS = {
            "/api/webhooks/**", "/api/auth/**", "/actuator/health"
    };

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(PUBLIC_PATHS).permitAll()
                    .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## Câu hỏi tự kiểm tra (dùng để ôn lại sau này)

1. Vì sao `AuditorAware` cần `@Bean` thay vì `@Component`? (2 lý do)
2. `ThreadLocal` giải quyết vấn đề gì trong ngữ cảnh singleton Bean?
3. JWT payload "ai đọc cũng được" — vì sao vẫn an toàn để chứa `userId`?
4. Vì sao cần cả access token lẫn refresh token, không dùng 1 loại?
5. HMAC và RSA khác nhau ở điểm nào — khi nào dùng cái nào?
6. CSRF khai thác cơ chế gì của cookie? Vì sao header miễn nhiễm?
7. Vì sao `JwtAuthenticationFilter` phải chèn TRƯỚC Authorization Filter?
8. Vì sao hệ thống này quyết định KHÔNG nhúng `role` vào JWT?

*(Tự trả lời được cả 8 câu không cần mở lại tài liệu — nghĩa là đã nắm
vững phần này.)*
