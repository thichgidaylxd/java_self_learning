# Tổng hợp đầy đủ: HTTP/HTTPS → Spring → JWT → Spring Security → OAuth2

> File này giữ nguyên độ chi tiết như lúc học trong hội thoại — không tóm tắt gạch đầu dòng, mà giữ lại cả phần "vì sao", ví dụ, và mạch suy luận. Đọc từ trên xuống dưới đúng thứ tự roadmap.

---

## 0. HTTP/HTTPS — nền tảng trước khi vào bất kỳ giai đoạn nào

Đây là lớp nền của mọi thứ học sau (JWT nằm trong header HTTP, filter chain xử lý HTTP request, OAuth2 redirect qua HTTP...) — nên phải nắm chắc trước.

### HTTP là gì — bản chất

HTTP (HyperText Transfer Protocol) là quy ước (protocol) về cách 2 máy tính nói chuyện với nhau qua mạng, theo mô hình **request → response**:

```
Client (browser/app/Postman)  --- Request --->  Server
Client (browser/app/Postman)  <-- Response ---  Server
```

Mỗi lần bấm nút trên web/app, trình duyệt gửi đi 1 request, server xử lý xong trả về 1 response. **Không có khái niệm server tự "gọi lại" client trước** — đây là lý do webhook (ví dụ SePay) phải gọi ngược vào server mình: bản chất lúc đó bên gửi webhook đóng vai "client", server mình đóng vai "server". Không phải server "chủ động" — chỉ là đổi vai ai gửi request trước.

### Cấu trúc 1 HTTP Request

```
POST /api/persons HTTP/1.1              <- dòng đầu: method + path + version
Host: api.giapha.com                    <- headers
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{"fullName": "Nguyễn Văn A", "gender": "nam"}   <- body
```

4 thành phần:

- **Method** — hành động muốn làm: `GET` (lấy dữ liệu), `POST` (tạo mới), `PUT`/`PATCH` (sửa), `DELETE` (xoá). Đây chính là những gì `@GetMapping`/`@PostMapping` trong Spring map tới.
- **Path/URL** — địa chỉ tài nguyên: `/api/persons`, `/api/persons/{id}`. Đây là những gì `@RequestMapping("/api/persons")` khai báo.
- **Headers** — thông tin phụ, không phải nội dung chính. `Authorization` chứa JWT, `Content-Type` báo body đang là JSON. **Đây chính là nơi JWT sống** — không nằm trong body, không nằm trong URL, nằm trong header `Authorization: Bearer <token>`.
- **Body** — dữ liệu chính gửi kèm (thường chỉ có ở POST/PUT/PATCH, GET thường không có body). Đây là những gì `@RequestBody` trong Spring đọc vào.

### Cấu trúc 1 HTTP Response

```
HTTP/1.1 200 OK                          <- status line
Content-Type: application/json

{"code": 200, "success": true, "data": {...}}   <- body
```

**Status code** — con số 3 chữ số nói kết quả:

- **2xx** — thành công (200 OK, 201 Created)
- **4xx** — lỗi do client (400 Bad Request, **401 Unauthorized** — chưa đăng nhập/token sai, **403 Forbidden** — đã đăng nhập nhưng không đủ quyền, 404 Not Found)
- **5xx** — lỗi do server (500 Internal Server Error — đúng loại `GlobalExceptionHandler` bắt)

**Điểm quan trọng cho sau này**: 401 và 403 khác nhau — Spring Security filter chain (giai đoạn 4) trả **401** khi không xác định được bạn là ai (JWT thiếu/sai), trả **403** khi biết bạn là ai nhưng không đủ quyền làm việc đó. Đây là phân biệt sẽ dùng lại liên tục ở các giai đoạn sau.

### HTTPS = HTTP + mã hoá (TLS/SSL)

HTTP thuần gửi mọi thứ dạng văn bản trần (plain text) — ai chặn được gói tin giữa đường (wifi công cộng, ISP...) đều đọc được `Authorization: Bearer <token>`, coi như mất luôn JWT.

HTTPS thêm 1 lớp mã hoá (TLS) bao ngoài toàn bộ request/response, gồm 3 mục tiêu:

1. **Confidentiality (bí mật)** — nội dung bị mã hoá, ai chặn giữa đường chỉ thấy dữ liệu vô nghĩa.
2. **Integrity (toàn vẹn)** — phát hiện được nếu dữ liệu bị sửa giữa đường (liên hệ trực tiếp tới lý do SePay dùng HMAC-SHA256 ký payload — cùng mục tiêu, khác cơ chế).
3. **Authentication (xác thực server)** — chứng chỉ SSL (certificate) chứng minh `api.giapha.com` đúng là server thật, không phải ai đó giả mạo domain (man-in-the-middle).

**TLS handshake** — trước khi gửi request thật, client/server "bắt tay" thống nhất 1 khoá mã hoá tạm thời (session key) chỉ dùng cho phiên kết nối đó. Không cần nhớ chi tiết thuật toán, chỉ cần hiểu: mọi request/response sau khi handshake xong đều được mã hoá bằng khoá tạm này, kể cả nếu ai chặn được gói tin cũng không giải mã được nếu không có khoá.

**Vì sao HTTPS bắt buộc dù đã có HMAC-SHA256 (ví dụ SePay webhook)**: thiếu HTTPS, dù có ký HMAC-SHA256, ai đó vẫn đọc được nội dung giao dịch (số tiền, mã thanh toán) khi truyền qua mạng — HMAC chỉ chống **sửa**, không chống **đọc trộm**, 2 việc khác nhau, cần HTTPS để giải quyết vế còn lại.

---

## Giai đoạn 1 — Nền tảng Spring (bắt buộc trước tất cả)

**Mục tiêu**: hiểu Spring "quản lý object hộ mình" nghĩa là gì.

### IoC Container & Dependency Injection

Khái niệm gốc của mọi thứ trong Spring. Câu hỏi cốt lõi: **tại sao không tự `new PersonService()` mà để Spring tạo hộ?**

Nếu tự `new`, mỗi chỗ cần `PersonService` phải tự tay tạo, tự tay truyền các dependency nó cần (repository, config...) — khi 1 dependency thay đổi cách khởi tạo, phải sửa ở mọi nơi gọi `new`. Spring đảo ngược việc đó: bạn chỉ khai báo "tôi cần 1 `PersonService`", Spring (IoC Container) tự lo việc tạo, tự lo việc truyền đúng dependency vào — đây gọi là **Inversion of Control**: quyền kiểm soát việc tạo object bị đảo ngược, từ code của bạn sang cho framework.

### @Component vs @Service vs @Repository vs @Controller

Về **bản chất kỹ thuật giống hệt nhau** — tất cả đều là `@Component` (3 cái kia chỉ là annotation "con", được đánh dấu thêm `@Component` bên trong). Khác nhau ở:

- **Ý nghĩa ngữ nghĩa cho người đọc code** — nhìn `@Service` biết ngay đây là tầng business logic, nhìn `@Repository` biết ngay đây là tầng truy cập dữ liệu.
- **1 vài xử lý đặc biệt** — ví dụ `@Repository` tự động dịch (translate) exception JDBC gốc (khó đọc, đặc thù driver DB) thành `DataAccessException` chuẩn của Spring, dễ xử lý hơn.

### @Configuration + @Bean

Khác với `@Component` ở chỗ:

- `@Component` = "tự đánh dấu lên class do mình viết" — bạn có source code của class đó, thêm annotation vào là xong.
- `@Bean` = "khai báo thủ công 1 object mà mình không viết ra (thư viện ngoài) để Spring quản lý" — dùng khi object đến từ thư viện bên ngoài (không sửa được source), bạn viết 1 method trong class `@Configuration`, method đó trả về object, đánh dấu `@Bean` để Spring biết "đây cũng là 1 object cần quản lý y như `@Component`".

**Bài test nhỏ để biết đã hiểu chưa**: đọc lại `SecurityConfig.java` — chỉ ra được từng `@Bean` ở đó tạo ra loại object gì, dùng để làm gì. Nếu không giải thích được từng `@Bean` là gì, nghĩa là mục 1-3 chưa vững, không nên nhảy sang Security/JWT vội.

---

## Giai đoạn 2 — JPA Auditing (dễ nhất trong 4 phần, học trước để có đà)

### @MappedSuperclass

Khác `@Entity` ở chỗ: **không tự thành 1 bảng riêng** trong DB, chỉ là "khuôn field dùng chung" để entity khác kế thừa. Ví dụ `BaseEntity` có `createdAt`, `createdBy` — các entity khác `extends BaseEntity` thì tự có sẵn các field này mà không cần khai báo lại, và cũng không có bảng riêng tên `base_entity` nào được tạo ra.

### @EntityListeners(AuditingEntityListener.class)

Cơ chế Hibernate **"nghe sự kiện"** — trước khi insert, trước khi update — để tự động điền field. Bạn không tự tay gọi hàm điền `createdAt = now()`, mà đánh dấu annotation này lên entity, Hibernate tự "nghe" thấy sự kiện sắp insert và tự điền hộ.

### AuditorAware<T>

Chỉ là 1 interface có **1 hàm duy nhất** `getCurrentAuditor()`, trả lời câu hỏi **"ai đang thao tác lúc này"** — Spring gọi hàm này mỗi khi cần điền `@CreatedBy`. Bản thân interface này không biết "ai" — bạn phải tự implement, thường là đọc từ `SecurityContextHolder` (sẽ học kỹ ở phần Authentication bên dưới).

**Thực hành gợi ý**: tạo 1 entity đơn giản (ví dụ `Note` có `title`, `extends BaseEntity` đã có) → lưu thử → xem `created_at` tự điền chưa (chưa cần quan tâm `created_by` ở bước này, vì lúc này chưa có ai đăng nhập để trả lời "ai đang thao tác").

---

## Giai đoạn 3 — JWT (khái niệm, CHƯA cần Spring Security)

Học phần này tách biệt hoàn toàn khỏi Spring trước — **JWT là 1 chuẩn chung**, không phải riêng của Java, không phải riêng của bất kỳ framework nào.

### Cấu trúc: Header.Payload.Signature

Thử decode 1 JWT mẫu tại `jwt.io` để thấy tận mắt payload chứa gì (chưa cần biết ký ra sao vội, chỉ cần thấy payload là dữ liệu gì).

### Vì sao cần "ký" (sign)

Để server biết token **không bị ai sửa nội dung giữa đường** — ví dụ nếu không ký, ai đó chặn được token có thể tự sửa payload từ `{"userId": "123"}` thành `{"userId": "999"}` để giả mạo thành user khác, mà server không cách nào phát hiện. Chữ ký giải quyết đúng vấn đề đó: chỉ cần đổi 1 ký tự trong payload, chữ ký cũ sẽ không còn khớp nữa, server verify sẽ báo lỗi ngay.

### Access token vs Refresh token

- **Access token** sống ngắn (vd 15 phút) — **đỡ rủi ro nếu bị lộ**, vì kẻ lấy được cũng chỉ dùng được trong thời gian ngắn.
- **Refresh token** sống dài (vd 7 ngày) — **chỉ dùng để xin access token mới**, không dùng gọi API trực tiếp. Tách riêng 2 loại token để giới hạn phạm vi rủi ro: access token lộ thì thiệt hại nhỏ (hết hạn nhanh), refresh token ít khi đi kèm mỗi request nên ít bị lộ hơn.

### Luồng tổng quát

```
login thành công
  → server trả 2 token (access + refresh)
  → client lưu lại
  → mỗi request gắn access token vào header: Authorization: Bearer <token>
  → access token hết hạn
  → dùng refresh token xin cặp mới, không bắt người dùng đăng nhập lại
```

**Thực hành gợi ý**: không cần code Spring — dùng trang jwt.io, tự tạo 1 JWT giả với payload `{"userId": "123"}`, thử sửa 1 ký tự trong phần signature, xem web báo "invalid signature" — cảm nhận trực quan vì sao ký giúp chống giả mạo.

---

## Giai đoạn 4 — Spring Security Filter Chain

Đây là phần dễ rối nhất — chỉ nên học sau khi giai đoạn 1 + 3 đã chắc, vì nó ghép khái niệm Spring (giai đoạn 1) với khái niệm JWT (giai đoạn 3) vào cùng 1 bức tranh.

### Vấn đề cần giải quyết: chặn request TRƯỚC KHI vào Controller

`GlobalExceptionHandler` chỉ chạy **sau khi** Controller đã xử lý xong và văng exception. Nhưng bảo mật cần chặn **sớm hơn thế** — 1 request không có JWT hợp lệ cần bị từ chối **trước khi** chạm vào bất kỳ dòng code nghiệp vụ nào (Controller/Service/Repository), tránh lãng phí tài nguyên và tránh rủi ro lộ logic nội bộ qua exception.

### Servlet Filter — cơ chế nền tảng, có trước cả Spring

Đây là khái niệm thuộc chuẩn Servlet (Java EE/Jakarta EE gốc), **không phải Spring phát minh ra**. Ý tưởng: 1 request HTTP đi qua **1 chuỗi các Filter xếp hàng nối tiếp nhau**, trước khi tới được Servlet cuối cùng (trong Spring MVC, servlet đó chính là `DispatcherServlet` — thứ điều hướng request tới đúng `@Controller`):

```
Request  →  Filter 1  →  Filter 2  →  Filter 3  →  DispatcherServlet  →  Controller
Response ←  Filter 1  ←  Filter 2  ←  Filter 3  ←  DispatcherServlet  ←  Controller
```

Mỗi Filter có 2 quyền:

- **Chặn lại, không cho đi tiếp** — tự trả response luôn (ví dụ trả 401), không gọi tới Filter kế tiếp.
- **Cho đi tiếp** — gọi `chain.doFilter(request, response)` để chuyển sang Filter kế tiếp trong chuỗi.

```java
// Ý tưởng 1 Filter thô, chưa phải Spring
public class SimpleFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        if (điều kiện không hợp lệ) {
            // Chặn lại, không gọi chain.doFilter() -> request KHÔNG đi tiếp
            ((HttpServletResponse) res).setStatus(401);
            return;
        }
        chain.doFilter(req, res);  // Cho đi tiếp tới Filter kế
    }
}
```

Response cũng đi ngược lại qua đúng chuỗi Filter đó (theo thứ tự ngược), nên Filter còn có thể **sửa response** trước khi trả về client (ví dụ CORS filter thêm header `Access-Control-Allow-Origin` vào response).

### Spring Security không viết lại cơ chế Filter — nó CHỈ LÀ 1 chuỗi Filter đặc biệt

Đây là điểm quan trọng nhất để hiểu `SecurityFilterChain`: Spring Security bản chất là **1 tập hợp nhiều Filter con**, được Spring đóng gói và chèn vào đúng vị trí trong chuỗi Filter tổng của Servlet Container (Tomcat). Khi viết:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
                .requestMatchers(PUBLIC_PATHS).permitAll()
                .anyRequest().permitAll()
        );
    return http.build();
}
```

Bạn **không** đang viết logic if/else thủ công như ví dụ `SimpleFilter` ở trên — bạn đang **cấu hình, lắp ráp sẵn** nhiều Filter con có sẵn của Spring Security (CSRF Filter, CORS Filter, Authorization Filter...), theo đúng thứ tự đã được Spring định sẵn là hợp lý (ví dụ: CORS phải chạy trước Authorization, vì cần xử lý preflight request trước khi kiểm tra quyền).

`HttpSecurity http` — object nhận vào tham số (đây chính là Dependency Injection, Spring tự tạo sẵn đưa vào) — có vai trò như 1 "builder", mỗi lời gọi `.csrf(...)`, `.cors(...)`, `.sessionManagement(...)` là thêm/cấu hình 1 Filter con vào chuỗi cuối cùng. `http.build()` ở cuối mới thực sự ráp tất cả lại thành 1 `SecurityFilterChain` hoàn chỉnh.

### Giải nghĩa từng dòng cấu hình — theo đúng cơ chế Filter vừa học

**`.csrf(csrf -> csrf.disable())`**

CSRF (Cross-Site Request Forgery) là kiểu tấn công lợi dụng cookie/session — trình duyệt tự động gửi kèm cookie có sẵn khi bạn vô tình bấm vào 1 link độc hại, khiến server tưởng đây là request hợp lệ từ bạn. Vì hệ thống dùng JWT thuần, không cookie/session (JWT phải gắn thủ công vào header `Authorization` mỗi lần, không tự động gửi kèm như cookie) → không có bề mặt tấn công CSRF → tắt Filter kiểm tra CSRF token đi cho gọn, không cần thiết. (Phân tích chi tiết vì sao xem mục "CSRF — phân tích sâu" bên dưới.)

**`.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))`**

Nói thẳng với Spring Security: "đừng bao giờ tự tạo `HttpSession` cho request nào cả, kể cả khi có code nào đó vô tình gọi `request.getSession()`". Đúng bản chất đã chọn từ đầu (JWT thuần) — mỗi request phải tự đủ thông tin chứng minh danh tính (qua JWT trong header), không dựa vào bất kỳ trạng thái nào server nhớ từ request trước.

**`.authorizeHttpRequests(auth -> auth.requestMatchers(PUBLIC_PATHS).permitAll().anyRequest().permitAll())`**

Đây chính là **Authorization Filter** — Filter cuối cùng trong chuỗi Security, có nhiệm vụ: nhìn vào path của request, đối chiếu với danh sách quy tắc đã khai (theo đúng thứ tự khai báo, quy tắc đầu tiên khớp sẽ áp dụng, các quy tắc sau không xét nữa), quyết định cho qua hay chặn.

- `requestMatchers(PUBLIC_PATHS).permitAll()` — các path trong mảng `PUBLIC_PATHS` (`/api/webhooks/**`, `/api/auth/**`...) → luôn cho qua, bất kể có JWT hay không.
- `anyRequest().permitAll()` — dòng tạm thời — mọi path còn lại cũng cho qua luôn. Đây là lý do trước khi có giai đoạn 5, test API qua Postman không cần gắn JWT vẫn chạy được — vì dòng này đang mở toang. Khi Giai đoạn 5 xong, dòng này đổi thành `.anyRequest().authenticated()` → Authorization Filter sẽ kiểm tra `SecurityContext` có `Authentication` hợp lệ hay không (đúng object mà `JwtAuthenticationFilter` sẽ set vào) — không có thì chặn, trả 401.

### Vị trí JwtAuthenticationFilter sẽ chèn vào đâu trong chuỗi

Khi viết Filter riêng để verify JWT, cần chèn nó vào đúng vị trí trong chuỗi — cụ thể là **trước** Authorization Filter (vì Authorization Filter cần đọc `SecurityContext` đã có `Authentication` rồi mới quyết định cho qua hay không — nếu chèn sau, Authorization Filter sẽ luôn thấy `SecurityContext` trống):

```java
http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
```

`UsernamePasswordAuthenticationFilter` là 1 Filter có sẵn của Spring Security (dùng cho login form truyền thống, không dùng tới vì không có login form HTML) — nhưng nó vẫn tồn tại trong chuỗi mặc định, dùng làm **điểm mốc tham chiếu** để nói "chèn Filter của tôi vào ngay trước vị trí này".

### Tóm tắt toàn bộ luồng 1 request đi qua — ráp lại mọi thứ đã học

```
1. Request tới, Tomcat nhận (biết TCP/IP, HTTP)
2. Đi qua chuỗi Filter của Servlet Container, trong đó có
   FilterChainProxy (điểm vào của Spring Security)
3. Bên trong đó, lần lượt qua các Filter con:
   CORS Filter -> (sau này) JwtAuthenticationFilter -> Authorization Filter
4. JwtAuthenticationFilter (giai đoạn 5): đọc header Authorization,
   verify JWT (giai đoạn 3) -> set Authentication vào SecurityContext
   (ThreadLocal, riêng theo thread)
5. Authorization Filter: kiểm tra path + SecurityContext -> quyết định
   cho qua hay trả 401
6. Nếu qua được, request tới DispatcherServlet -> đúng @RestController
7. Trong Controller/Service, khi save() Entity -> AuditingEntityListener
   đọc lại đúng SecurityContext (giai đoạn 2) -> điền created_by
```

### permitAll() vs authenticated()

Quy tắc "path này không cần đăng nhập" (`permitAll()`) vs "path này bắt buộc đăng nhập" (`authenticated()`). Ví dụ `/api/auth/login` phải `permitAll()` (chưa đăng nhập thì làm sao gọi API cần đăng nhập để login), còn `/api/persons` nên `authenticated()`.

**Thực hành gợi ý**: trong `SecurityConfig` đã viết, thử đổi `PUBLIC_PATHS` thêm/bớt 1 path, chạy app, gọi thử qua Postman xem path đó có bị chặn 401 đúng như mong đợi không.

### CSRF — phân tích sâu: tại sao tắt được, và khi nào không nên tắt

Câu hỏi hay dễ gây nhầm: "session/cookie" và "JWT" **không phải 2 thứ đối lập hoàn toàn** — chúng nằm trên **2 trục độc lập**, có thể trộn lẫn với nhau. Tách ra sẽ dễ hiểu hơn nhiều.

**Trục 1: Nơi lưu "ai đang đăng nhập" — Stateful vs Stateless**

| | Session (Stateful) | JWT (Stateless) |
|---|---|---|
| Server có nhớ gì không | Có — giữ 1 bảng/bộ nhớ sessionId → userInfo | Không — mọi thông tin nằm ngay trong token, server chỉ verify chữ ký |
| Muốn "logout ngay lập tức" | Dễ — xoá session khỏi bộ nhớ server | Khó — token cũ vẫn hợp lệ tới khi tự hết hạn |

**Trục 2: Cách token/session-id "đi kèm" mỗi request — Cookie vs Header**

| | Cookie | Header (`Authorization: Bearer ...`) |
|---|---|---|
| Ai gắn nó vào request | Trình duyệt **tự động** gắn — không cần code JS nào làm việc này | Phải code JS **tự tay** gắn vào mỗi request (fetch, axios interceptor...) |
| Gắn cả khi nào | Cả khi bạn **không chủ ý** gửi request (ví dụ bị dụ bấm vào 1 link/form độc hại từ trang web khác) | Chỉ gắn khi code của chính app bạn **chủ động** thêm vào |

**Đây chính là chỗ mấu chốt — CSRF liên quan tới Trục 2, KHÔNG liên quan tới Trục 1**

Rất nhiều người nhầm rằng "dùng JWT thì không bao giờ bị CSRF" — sai, chỉ đúng khi JWT đi qua **Header**. Nếu JWT đi qua **Cookie** (1 số hệ thống làm vậy, để tận dụng cookie tự động gửi, đỡ phải code JS gắn header thủ công), thì CSRF quay trở lại, dù vẫn đang "dùng JWT":

```
Kịch bản tấn công (JWT lưu trong Cookie):
1. Bạn đăng nhập app-that.com -> server set-cookie chứa JWT
2. Bạn (vẫn đang đăng nhập) vô tình mở tab khác, vào trang xau.com
3. xau.com có 1 form ẩn, tự động submit tới app-that.com/api/transfer-money
4. Trình duyệt "tốt bụng" TỰ ĐỘNG gắn cookie JWT của app-that.com vào
   request đó, vì request đang gửi TỚI đúng domain app-that.com
5. Server app-that.com verify JWT -> hợp lệ (chữ ký đúng, chưa hết hạn)
   -> XỬ LÝ LỆNH CHUYỂN TIỀN, dù chính bạn không hề chủ ý gửi request này
```

Điểm quan trọng: **JWT signature hoàn toàn không giúp gì ở đây** — token đó thật 100%, không bị sửa gì cả, chỉ là nó bị gửi đi không theo ý muốn của chủ nhân. CSRF không phải tấn công "giả mạo token", mà là **"lừa trình duyệt tự gửi request thay bạn"** — đây là lý do CSRF liên quan tới cơ chế tự động gắn của cookie, không liên quan tới JWT hay session.

**4 tổ hợp thực tế:**

| Tổ hợp | Có CSRF risk? | Ghi chú |
|---|---|---|
| Session + Cookie (kiểu truyền thống, PHP/Rails cũ) | **Có** | Kịch bản kinh điển, cần CSRF token để chống |
| JWT + Header (đúng cách hệ thống này đang làm) | **Không** | Trang độc hại không có cách nào "ra lệnh" cho JS của app tự gắn header vào request nó tự tạo |
| JWT + Cookie (hybrid, 1 số hệ thống chọn vì tiện) | **Có** | Như kịch bản trên — vẫn dùng JWT nhưng transport qua cookie, CSRF quay lại |
| Session + Header (hiếm gặp) | **Không** | Ít ai làm vậy vì mất hết lợi ích "tiện" của cookie, nhưng về lý thuyết an toàn CSRF |

**"Chỉ dùng session/cookie thì sao"** — đây chính là ô đầu tiên trong bảng: **luôn có CSRF risk**, vì bản chất cookie tự động gắn là điều kiện đủ để tấn công này xảy ra, không phụ thuộc nội dung cookie là session ID hay gì khác.

**Nếu 1 hệ thống thực sự "kết hợp cả 3" — thường nghĩa là gì trong thực tế**

Thường gặp nhất: web app dùng **cookie `httpOnly`** để lưu refresh token (chống JS đọc trộm token qua XSS), còn **access token vẫn qua header** cho mỗi API call. Đây là pattern khá phổ biến và an toàn hơn so với lưu cả 2 loại token trong `localStorage` (localStorage dễ bị đọc trộm nếu có lỗ hổng XSS). Nhưng vì refresh token đi qua cookie, API nhận refresh token đó (`/api/auth/refresh`) **vẫn cần CSRF protection riêng** — dù các API khác (dùng access token qua header) thì không cần.

**Kết luận cho câu hỏi gốc**: `.csrf().disable()` an toàn trong `SecurityConfig` hiện tại vì hệ thống dùng JWT thuần qua Header — CSRF khai thác cơ chế **tự động gắn** của cookie, còn JWT trong header phải được code **chủ động** gắn vào, nên 1 trang độc hại không có cách nào "ra lệnh" cho JS của app tự thêm header đó vào request nó tự tạo ra. Vô hiệu hoá CSRF Filter là đúng đắn — không phải bỏ qua bước bảo mật cần thiết, mà vì bề mặt tấn công đó không tồn tại trong kiến trúc này. Nếu sau này thêm cookie (ví dụ refresh token qua cookie `httpOnly`), câu trả lời sẽ khác — cần bật lại CSRF protection riêng cho đúng route đó.

---

## Cấu trúc `Authentication` — interface cốt lõi của Spring Security

Trước khi vào giai đoạn 5, cần hiểu rõ `Authentication` — vì đây là object trung tâm mà `JwtAuthenticationFilter` sẽ tạo ra.

### Authentication là 1 interface, không phải 1 class cụ thể

Giống cách `AuditorAware<T>` là interface (giai đoạn 2), `Authentication` cũng chỉ là **hợp đồng (interface)** do Spring Security định nghĩa, có nhiều implementation khác nhau tuỳ tình huống đăng nhập (form login, JWT, OAuth2...). Interface này khai báo các method bắt buộc phải có:

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

### 1. getPrincipal() — "ai" đang được xác thực

Đây chính là field dùng trong `AuditorAware`:

```java
Object principal = authentication.getPrincipal();
```

`principal` **không có kiểu cố định** — tuỳ implementation mà nó chứa gì:

- Với login form truyền thống: thường là `String` (username) hoặc `UserDetails` (object đầy đủ thông tin user).
- Với JWT filter: do chính bạn quyết định đặt gì vào đây khi tạo `Authentication` object — có thể là `String` chứa UUID trần, hoặc `CustomUserDetails` chứa cả id, email, role.

Ví dụ cụ thể:

```java
// Cách đơn giản:
principal = "a3f1c8e0-1234-...";  // String chứa UUID

// Cách đầy đủ hơn (khi có CustomUserDetails):
principal = new CustomUserDetails(userId, email, role);
```

### 2. getCredentials() — "bằng chứng" chứng minh danh tính

- Với **login form**: đây là mật khẩu người dùng gõ vào (chỉ dùng lúc verify ban đầu, sau đó nên xoá đi — không giữ mật khẩu trong bộ nhớ lâu vì lý do bảo mật).
- Với **JWT**: field này gần như luôn để trống (`null`) — vì "bằng chứng" chính là bản thân **JWT signature đã verify** — signature hợp lệ tức là đã chứng minh xong, không cần giữ thêm "credentials" nào nữa sau khi verify. Đây là khác biệt quan trọng so với login form (nơi credentials = password được dùng để **so sánh** với hash trong DB tại thời điểm xác thực — với JWT không có bước "so sánh" nào nữa sau khi verify chữ ký).

### 3. getAuthorities() — "được phép làm gì" (quyền/role)

Trả về 1 tập hợp các `GrantedAuthority` — mỗi phần tử là 1 quyền cụ thể:

```java
public interface GrantedAuthority extends Serializable {
    String getAuthority();   // ví dụ trả về "ROLE_ADMIN", "ROLE_USER"
}
```

Đây chính là nơi liên kết trực tiếp với `@EnableMethodSecurity` (annotation có sẵn trong `SecurityConfig` từ đầu, giờ mới có ý nghĩa thật sự):

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteTree(UUID treeId) { ... }
```

Khi method này được gọi, Spring Security đọc `getAuthorities()` của `Authentication` hiện tại (lấy từ `SecurityContext` — ThreadLocal), kiểm tra trong danh sách đó có `"ROLE_ADMIN"` không — có thì cho chạy, không thì ném `AccessDeniedException` (dẫn tới **403**, đúng phân biệt 401 vs 403 đã học ở phần HTTP).

**Liên hệ với schema thật (`tree_memberships.role`: owner/editor/viewer)**: đây là ứng viên tự nhiên để map thành `GrantedAuthority`, dù phức tạp hơn ví dụ `ROLE_ADMIN` đơn giản ở trên vì role phụ thuộc vào **từng `tree_id`** (1 user có thể là owner ở tree này, viewer ở tree khác) — không phải 1 role cố định toàn cục. Đây là bài toán "phân quyền theo resource cụ thể" — xem phần riêng ở cuối file, sau giai đoạn 6.

### 4. isAuthenticated() — cờ boolean, đã xác thực xong chưa

Field dùng để check trong `AuditorAware`:

```java
if (authentication == null || !authentication.isAuthenticated() || ...)
```

**Điểm hay bị hiểu nhầm**: cờ này **không tự động là true** chỉ vì object `Authentication` tồn tại — bạn (hoặc Spring) phải **chủ động gọi `setAuthenticated(true)`** sau khi verify thành công. Đây là lý do trong `JwtAuthenticationFilter` (giai đoạn 5), sau khi verify JWT hợp lệ, bước tạo `Authentication` object phải đảm bảo cờ này được set đúng.

### UsernamePasswordAuthenticationToken — implementation phổ biến nhất, kể cả cho JWT

Class có sẵn của Spring Security, implement `Authentication` — tên gọi hơi gây nhầm lẫn (nghe như chỉ dùng cho username/password) nhưng thực tế được dùng rộng rãi cho cả JWT, vì nó chỉ là 1 "hộp chứa" tiện lợi:

```java
// Constructor thường dùng cho JWT (đã authenticated sẵn, không cần verify lại credentials)
Authentication auth = new UsernamePasswordAuthenticationToken(
    principal,      // object tạo ra ở bước verify JWT
    null,           // credentials - null vì đã verify qua JWT signature rồi
    authorities     // danh sách GrantedAuthority (có thể để rỗng nếu chưa làm phân quyền)
);
```

Có **2 constructor** khác nhau trong class này — 1 cái tự động set `authenticated = false` (dùng khi chưa verify, ví dụ lúc nhận username/password thô từ login form, chưa biết đúng hay sai), 1 cái tự động set `authenticated = true` (dùng khi đã verify xong, đúng tình huống JWT filter — verify signature xong mới tạo object này).

### getDetails() — thường ít dùng, nhưng nên biết

Chứa thông tin phụ trợ về request, không phải về user — ví dụ IP address, session ID (nếu có). Với hệ thống JWT thuần, field này thường để `null` hoặc không set, vì không cần thiết cho logic nghiệp vụ hiện tại — chỉ nên biết nó tồn tại để không bối rối khi thấy trong IDE autocomplete.

### Tổng hợp — hình dung Authentication như 1 "thẻ căn cước tạm thời"

| Field | Ý nghĩa | Trong hệ thống JWT sẽ là gì |
|---|---|---|
| `principal` | Ai | `CustomUserDetails` (hoặc UUID) lấy từ claim `sub` trong JWT |
| `credentials` | Bằng chứng | `null` — JWT signature đã là bằng chứng, verify xong không cần giữ |
| `authorities` | Được phép gì | Danh sách role — có thể rỗng ban đầu, làm sau khi cần `@PreAuthorize` |
| `authenticated` | Đã xác thực chưa | `true` — set ngay khi tạo, vì JWT filter chỉ tạo object này SAU KHI verify thành công |
| `details` | Thông tin phụ | Thường không dùng tới trong hệ thống này |

---

## Giai đoạn 5 — Ghép JWT vào Spring Security (JwtAuthenticationFilter)

Đây là chỗ giai đoạn 3 + 4 gặp nhau — chỉ học khi cả 2 đã vững. Đây là lúc mọi kiến thức rời rạc (Filter chain, JWT sign/verify, SecurityContext/ThreadLocal, AuditorAware) ráp lại thành code chạy thật.

Filter tự viết (`OncePerRequestFilter`) đọc header `Authorization`, verify JWT (dùng lại kiến thức giai đoạn 3), rồi tự tay `SecurityContextHolder.getContext().setAuthentication(...)` — đây chính là bước biến **"1 JWT hợp lệ"** thành **"Spring Security công nhận đây là user đã đăng nhập"**. Đây cũng là lúc `AuditorAware` (giai đoạn 2) mới thực sự có dữ liệu để điền `created_by` — trước giai đoạn này, không có ai đăng nhập nên không có gì để điền.

### Việc cần làm, theo đúng thứ tự phụ thuộc

```
1. JwtTokenProvider.java        -- sign() lúc login, verify() lúc có request
2. JwtAuthenticationFilter.java -- đọc header, gọi verify(), set SecurityContext
3. Sửa SecurityConfig.java      -- addFilterBefore(...), đổi permitAll -> authenticated
```

### 3 điều cần quyết định trước khi code JwtTokenProvider

`JwtTokenProvider` cần quyết định trước 3 điều (đúng những gì đã học ở Giai đoạn 3):

1. **`secretKey` lấy từ đâu** — hardcode (tuyệt đối không) hay từ biến môi trường (`application.yaml` → `${JWT_SECRET}`, khớp với cách làm với `DB_URL`)?
2. **Access token sống bao lâu, refresh token sống bao lâu** — con số cụ thể (ví dụ 15 phút / 7 ngày như đã bàn ở giai đoạn 3, hay muốn số khác)?
3. **Payload (claims) chứa gì ngoài `sub` (userId)** — có cần thêm `email`, `role` vào token luôn để đỡ query DB mỗi request không, hay giữ tối giản chỉ có `userId`?

### Vấn đề cần giải quyết ở giai đoạn này — trước khi code

4 giai đoạn trước học riêng lẻ: JWT là gì (giai đoạn 3), Filter chain hoạt động sao (giai đoạn 4), `SecurityContext` dựa trên `ThreadLocal` (giai đoạn 2 phần mở rộng). Giờ cần hiểu chúng **nối với nhau ở đâu** — cụ thể là: **ai** là người đọc JWT từ header, verify nó, rồi set vào `SecurityContext` để Authorization Filter (giai đoạn 4) và `AuditorAware` (giai đoạn 2) dùng được?

Câu trả lời: 1 Filter tự viết, đặt tên quy ước là `JwtAuthenticationFilter` — nó là **mảnh ghép còn thiếu** nối giữa "JWT trong header" và "SecurityContext có Authentication".

### OncePerRequestFilter — lớp cha cần hiểu trước

Spring cung cấp sẵn 1 abstract class tên `OncePerRequestFilter`, kế thừa từ Servlet Filter gốc (đã học khái niệm Filter ở giai đoạn 4). Điểm khác biệt duy nhất nó thêm vào: **đảm bảo logic verify chỉ chạy đúng 1 lần cho mỗi request**, dù trong 1 số trường hợp phức tạp (forward/include request nội bộ), Servlet Container có thể vô tình gọi filter nhiều lần cho cùng 1 request gốc.

```java
public abstract class OncePerRequestFilter extends GenericFilterBean {
    // bạn chỉ cần override đúng 1 method:
    protected abstract void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    );
}
```

### Logic bên trong filter này làm gì — từng bước, nối lại kiến thức cũ

**Bước 1 — Đọc header `Authorization`** (đúng cấu trúc HTTP request đã học ở phần đầu, header là 1 trong 4 thành phần request):

```
Authorization: Bearer eyJhbGci...
```

Filter cần: kiểm tra header có tồn tại không, có bắt đầu bằng chữ `"Bearer "` không (đây là quy ước chuẩn, không phải Spring tự đặt ra), rồi cắt bỏ chữ `"Bearer "` để lấy đúng chuỗi token thuần.

**Bước 2 — Verify token** (đúng cơ chế HMAC-SHA256 đã học ở giai đoạn 3 — tính lại signature, so sánh, kiểm tra `exp` chưa hết hạn). Nếu verify thất bại (sai chữ ký, hết hạn, sai định dạng) → filter **không set gì cả**, để request đi tiếp với `SecurityContext` trống — quan trọng: **không tự trả lỗi 401 ngay tại đây**, mà để cho Authorization Filter (giai đoạn 4, chạy sau filter này trong chuỗi) tự quyết định chặn dựa trên rule `authenticated()` — đây là cách phân chia trách nhiệm rõ ràng: filter này chỉ lo **"xác định danh tính"**, không lo **"có được phép truy cập path này không"**.

**Bước 3** — Nếu verify thành công, lấy `userId` ra từ payload (đúng phần payload JWT đã học — claim `sub`).

**Bước 4** — Tạo object `Authentication`, set vào `SecurityContext` — đây chính là hành động ghi vào "ngăn kéo riêng của thread hiện tại" (đúng cơ chế `ThreadLocal` đã học ở giai đoạn 2 mở rộng):

```java
SecurityContextHolder.getContext().setAuthentication(authenticationObject);
```

Từ dòng này trở đi, trong cùng luồng xử lý request đó (cùng 1 thread — Spring MVC xử lý 1 request trọn vẹn trên 1 thread, không nhảy thread giữa chừng), bất kỳ đoạn code nào gọi `SecurityContextHolder.getContext().getAuthentication()` — kể cả `AuditorAware` đã viết từ trước, dù nó được viết trước khi filter này tồn tại — sẽ tự động lấy được đúng thông tin user vừa xác thực. **Đây chính là lý do code `AuditorAware` viết từ giai đoạn 2 không cần sửa gì cả khi giai đoạn 5 hoàn thành** — nó đã sẵn sàng chờ đúng dữ liệu này từ đầu.

**Bước 5** — Gọi `filterChain.doFilter(request, response)` — đúng khái niệm "cho đi tiếp" đã học ở giai đoạn 4, chuyển sang Authorization Filter kế tiếp trong chuỗi.

### Vì sao filter này phải đặt TRƯỚC Authorization Filter — nhắc lại lý do, giờ có đủ ngữ cảnh để thấy rõ

Nếu đặt sau Authorization Filter: Authorization Filter chạy trước, kiểm tra `SecurityContext` lúc đó **còn trống trơn** (vì `JwtAuthenticationFilter` chưa kịp set gì) → mọi request đều bị coi là "chưa đăng nhập" → luôn trả 401, kể cả khi JWT hoàn toàn hợp lệ. **Thứ tự trong chuỗi Filter quan trọng tuyệt đối, không phải chi tiết tuỳ ý.**

### Điều gì xảy ra với các claim khác trong token (email, role...)

Nếu `JwtTokenProvider` nhúng thêm `email`/`role` vào payload lúc ký, thì ở Bước 3, filter có thể đọc luôn các claim đó, đóng gói vào 1 object `CustomUserDetails` thay vì chỉ có `userId` trần — đây chính là điểm cần ghi TODO trong `AuditorAware` từ giai đoạn 2 ("thay bằng cách lấy đúng UUID từ principal thật sau khi có `CustomUserDetails`") — ghi chú đó đang **chờ đúng mảnh ghép này** hoàn thiện.

### JwtTokenProvider — sinh và verify JWT

```java
@Component
public class JwtTokenProvider {

    private final SecretKey key;
    private final long accessTokenValidityMs;

    public JwtTokenProvider(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.access-token-validity-ms}") long accessTokenValidityMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenValidityMs = accessTokenValidityMs;
    }

    public String generateAccessToken(UUID userId, String role) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + accessTokenValidityMs);
        return Jwts.builder()
                .subject(userId.toString())
                .claim("role", role)
                .issuedAt(now)
                .expiration(expiry)
                .signWith(key)
                .compact();
    }

    // Trả về claims nếu hợp lệ, ném exception nếu sai chữ ký/hết hạn
    public Claims validateAndGetClaims(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

Lưu ý: `validateAndGetClaims` chính là bước "kiểm tra signature" đã cảm nhận trực quan trên jwt.io ở giai đoạn 3 — ở đây làm bằng code thay vì bằng tay.

### JwtAuthenticationFilter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {

        String header = request.getHeader("Authorization"); // đúng vị trí JWT sống, đã học ở phần HTTP

        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);

            try {
                Claims claims = jwtTokenProvider.validateAndGetClaims(token);
                UUID userId = UUID.fromString(claims.getSubject());
                String role = claims.get("role", String.class);

                var authorities = List.of(new SimpleGrantedAuthority("ROLE_" + role));

                // principal ở đây chọn UUID cho đơn giản — có thể thay bằng CustomUserDetails
                Authentication authentication = new UsernamePasswordAuthenticationToken(
                        userId,        // principal — đúng field AuditorAware sẽ đọc lại
                        null,          // credentials — null, vì signature đã là bằng chứng
                        authorities    // authorities — dùng cho @PreAuthorize sau này
                );

                SecurityContextHolder.getContext().setAuthentication(authentication);

            } catch (JwtException | IllegalArgumentException e) {
                // Token sai/hết hạn -> không set Authentication -> filter chain sau sẽ tự trả 401
                SecurityContextHolder.clearContext();
            }
        }

        filterChain.doFilter(request, response); // luôn cho đi tiếp, để entry point xử lý 401 nếu cần
    }
}
```

### Gắn vào SecurityConfig

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(PUBLIC_PATHS).permitAll()
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class); // chạy TRƯỚC filter mặc định

    return http.build();
}
```

`addFilterBefore` là chỗ trả lời câu hỏi "filter tự viết chạy ở đâu trong chuỗi filter" từ giai đoạn 4.

---

## Giai đoạn 6 — OAuth2/Google login (id_token)

Học cuối cùng vì cần cả JWT lẫn hiểu request/response HTTP tốt.

### 1. OAuth2 giải quyết vấn đề gì — bắt đầu từ bài toán gốc

Trước khi có OAuth2, nếu muốn "đăng nhập bằng Google", cách duy nhất là bắt người dùng gõ **email + mật khẩu Google** trực tiếp vào app của bạn, app bạn gửi 2 thứ đó cho Google để kiểm tra hộ. Vấn đề: server của bạn **cầm được mật khẩu Google thật** của người dùng — nếu server bạn bị hack, hoặc bạn là kẻ xấu, bạn có toàn quyền vào Gmail, Google Drive... của họ.

OAuth2 sinh ra để cắt đứt việc đó: **ứng dụng bên thứ 3 (app của bạn) không bao giờ nhìn thấy mật khẩu Google**. Thay vào đó, người dùng đăng nhập thẳng trên trang của Google (địa chỉ `accounts.google.com`, không phải trang bạn), Google xác nhận xong thì đưa lại cho app của bạn **một mảnh bằng chứng đã ký** rằng "đúng là user này, tôi (Google) xác nhận rồi" — bạn không cần biết mật khẩu, chỉ cần tin bằng chứng đó.

Đây chính là lý do OAuth2 luôn có bước **redirect qua domain của Google** rồi mới quay lại app bạn — nếu không có bước redirect đó thì mất hết ý nghĩa (vì lúc đó lại thành gõ mật khẩu thẳng vào app bạn).

### 2. id_token là gì — bản chất, không phải khái niệm mới

Điểm quan trọng nhất để không học lại từ đầu: **id_token về bản chất chính là 1 JWT**, y hệt cấu trúc `Header.Payload.Signature` đã học ở giai đoạn 3 — chỉ khác **ai là người ký**.

| | JWT tự bạn cấp (giai đoạn 3, 5) | id_token của Google |
|---|---|---|
| Cấu trúc | Header.Payload.Signature | Header.Payload.Signature — **giống hệt** |
| Ai ký (signature) | Server của bạn (secret key riêng) | Google (private key của Google) |
| Payload chứa gì | `sub` (userId), `role`, `exp`... do bạn tự định nghĩa | `sub` (Google user id), `email`, `email_verified`, `aud`, `exp`... do Google định nghĩa sẵn |
| Ai verify | Server bạn tự verify bằng chính secret key của mình | Server bạn verify bằng **public key của Google** |

Vì vậy nếu đã hiểu "vì sao cần ký" và "verify là kiểm tra chữ ký" ở giai đoạn 3, thì id_token **không phải kiến thức mới**, chỉ là JWT do một bên khác (Google) phát hành thay vì server bạn.

### 3. Vì sao verify id_token khác với verify JWT của chính mình

Ở giai đoạn 5, verify JWT của bạn rất đơn giản: bạn tự giữ 1 `secretKey` (symmetric key), ký bằng key đó, verify cũng bằng đúng key đó — 1 bên duy nhất giữ key.

Với id_token, bạn **không hề có key của Google**, và Google cũng không đưa key bí mật cho bạn (nếu đưa thì ai cũng giả mạo được token). Google dùng **cặp khoá bất đối xứng (public/private key)**:

- Google giữ **private key** để ký id_token (bí mật, không ai có ngoài Google).
- Google công bố công khai **public key** tại 1 URL cố định (`https://www.googleapis.com/oauth2/v3/certs`) — ai cũng tải được.
- Verify nghĩa là: dùng public key đó để kiểm tra chữ ký — nếu khớp, chắc chắn token này do đúng private key của Google ký ra (không ai giả mạo được vì không ai có private key), mà **không cần Google chia sẻ bí mật nào cả**.

Đây là lý do nên dùng thư viện chính thức (`google-api-client`) thay vì tự viết verify — vì Google **xoay vòng (rotate)** các public key này định kỳ, tự tải và cache đúng key hiện hành là việc thư viện lo, tự viết dễ sai.

### 4. Ngoài chữ ký, còn phải kiểm tra những claim gì trong payload

Chữ ký hợp lệ chỉ chứng minh "token này đúng do Google ký", **chưa chứng minh token này dành cho app của bạn**. Vì Google phát hành id_token cho hàng triệu app khác nhau, payload có claim `aud` (audience) — đúng là "token này được ký ra để gửi cho `client_id` nào".

Nếu không kiểm tra `aud`, kẻ xấu có thể lấy 1 id_token hợp lệ (ký thật bởi Google) nhưng vốn được Google phát hành cho **app khác**, rồi gửi sang app của bạn — chữ ký vẫn đúng (vì đúng là Google ký), nhưng token đó không phải "dành cho bạn". Đây chính là lý do dòng code `.setAudience(Collections.singletonList(googleClientId))` bắt buộc phải có — không phải chi tiết thừa.

Ngoài `aud`, còn nên để thư viện tự kiểm tra `exp` (hết hạn chưa) và `iss` (issuer — có đúng là `accounts.google.com` phát hành không) — 3 claim này đúng là bộ 3 điều kiện tối thiểu để 1 id_token được coi là hợp lệ, không chỉ riêng chữ ký.

### 5. Luồng đầy đủ — ai làm gì, ở đâu

```
Frontend                          Backend (server bạn)              Google
   |                                     |                             |
   | 1. Dùng Google SDK, mở popup       |                             |
   |    đăng nhập Google  ------------------------------------------->|
   |                                     |                             |
   |  <---------------------- 2. Google trả về id_token (JWT do Google ký)
   |                                     |                             |
   | 3. Gửi id_token lên backend ------>|                             |
   |    (KHÔNG gọi API nghiệp vụ         |                             |
   |     bằng id_token này)              |                             |
   |                                     | 4. Verify chữ ký bằng       |
   |                                     |    public key Google        |
   |                                     |    + check aud/exp/iss      |
   |                                     |                             |
   |                                     | 5. Tìm user theo email/sub  |
   |                                     |    trong DB, tạo mới nếu    |
   |                                     |    chưa có                  |
   |                                     |                             |
   |                                     | 6. Backend TỰ cấp JWT        |
   |                                     |    của riêng mình (giống    |
   |                                     |    hệt giai đoạn 3, 5)      |
   |                                     |                             |
   | <---- 7. Backend trả access/refresh token (JWT của app bạn) -----|
   |                                     |                             |
   | 8. Từ đây về sau, mọi request      |                             |
   |    dùng JWT của app bạn (bước 6),  |                             |
   |    KHÔNG dùng id_token nữa          |                             |
```

Điểm hay bị hiểu nhầm nhất: **bước 8** — nhiều người tưởng cứ dùng thẳng `id_token` của Google để gọi API sau này cho đỡ phải tự cấp JWT. Sai, vì 2 lý do:

1. `id_token` của Google có `exp` rất ngắn và bạn **không kiểm soát được** thời gian sống, không refresh được theo ý bạn (Google không cấp refresh cho id_token theo kiểu bạn tự quản).
2. Toàn bộ hệ thống filter chain, `AuditorAware`, phân quyền `tree_memberships`... bạn đã xây dựng ở giai đoạn 4-5 đều **dựa trên JWT do chính bạn phát hành và tự định nghĩa claim** (`sub` = userId trong DB bạn, `role` = role trong hệ thống bạn) — id_token của Google không có sẵn field `role` nội bộ của bạn.

→ Vì vậy Google chỉ đóng vai trò **"chứng minh danh tính 1 lần lúc đăng nhập"**, sau đó vòng đời request/response vẫn chạy hoàn toàn trên hệ JWT tự chủ của bạn — đúng kiến trúc đã học ở giai đoạn 3 và 5, không có gì khác biệt về sau.

### Code

**Verify id_token từ Google:**

```java
@Component
public class GoogleIdTokenVerifierService {

    @Value("${google.client-id}")
    private String googleClientId;

    public GoogleIdToken.Payload verify(String idTokenString) {
        try {
            GoogleIdTokenVerifier verifier = new GoogleIdTokenVerifier.Builder(
                    new NetHttpTransport(), GsonFactory.getDefaultInstance())
                    .setAudience(Collections.singletonList(googleClientId)) // check claim "aud"
                    .build();

            GoogleIdToken idToken = verifier.verify(idTokenString);
            if (idToken == null) {
                throw new BadCredentialsException("Invalid Google id_token");
            }
            return idToken.getPayload(); // chứa email, sub (Google user id), name...
        } catch (GeneralSecurityException | IOException e) {
            throw new BadCredentialsException("Cannot verify Google id_token", e);
        }
    }
}
```

**Controller nhận id_token, cấp JWT của hệ thống:**

```java
@PostMapping("/api/auth/google")
public ResponseEntity<AuthResponse> loginWithGoogle(@RequestBody GoogleLoginRequest request) {

    GoogleIdToken.Payload payload = googleIdTokenVerifierService.verify(request.getIdToken());

    String email = payload.getEmail();
    Person user = personRepository.findByEmail(email)
            .orElseGet(() -> personRepository.save(Person.fromGoogle(payload)));

    // Quan trọng: KHÔNG dùng thẳng id_token của Google để gọi API sau này
    String accessToken = jwtTokenProvider.generateAccessToken(user.getId(), user.getRole());
    String refreshToken = refreshTokenService.generate(user.getId());

    return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken));
}
```

Từ đây, mọi request tiếp theo đi qua đúng `JwtAuthenticationFilter` đã code ở giai đoạn 5 — Google không còn liên quan gì nữa.

---

## Phân quyền theo resource cụ thể (nâng cao — vd `tree_memberships`)

Đây là bài toán được nhắc tới khi học phần `getAuthorities()` — nâng cao hơn ví dụ cơ bản `ROLE_ADMIN`, không cần lo ngay khi mới bắt đầu, nhưng cần hiểu rõ **vì sao** nó khác để không bị bối rối khi tới lúc thiết kế thật.

### 1. Vì sao ví dụ ROLE_ADMIN cơ bản không áp dụng được cho bài toán này

`getAuthorities()` trả về 1 danh sách quyền gắn liền với **user**, và danh sách đó được đặt vào `Authentication` **một lần duy nhất, ngay tại filter (giai đoạn 5)** — tức là được xác định **trước khi biết** request này đang thao tác trên tài nguyên nào.

Với `ROLE_ADMIN`, điều đó hợp lý: 1 user hoặc là admin hoặc không, đúng với **mọi** request, không phụ thuộc URL gọi tới cái gì.

Nhưng với `tree_memberships`, câu hỏi "user này có quyền gì" **không có câu trả lời cố định** — nó phụ thuộc vào 1 tham số **chỉ xuất hiện lúc gọi API**, cụ thể là `treeId`:

```
user A → tree 1: role = owner
user A → tree 2: role = viewer
user A → tree 3: (không phải thành viên — không có quyền gì)
```

Nếu cố nhét toàn bộ danh sách này vào `getAuthorities()` lúc login (ví dụ `["tree_1:owner", "tree_2:viewer"]`), sẽ vỡ ngay khi:

- User được thêm vào 1 cây mới **sau khi** đã login — JWT cũ không tự cập nhật được (JWT là stateless, đã ký xong không sửa được nội dung, đúng bản chất đã học ở giai đoạn 3).
- User có hàng trăm cây — payload JWT phình to bất hợp lý, JWT vốn được thiết kế để gọn nhẹ.

→ Kết luận: **`Authentication` (tạo 1 lần lúc verify JWT) không phải chỗ phù hợp để trả lời "được làm gì với resource X"**. Nó chỉ nên trả lời đúng câu hỏi nó vốn được thiết kế để trả lời: "đây là ai" (`getPrincipal`) và "có phải xác thực thành công không" (`isAuthenticated`).

### 2. Vậy câu hỏi này nên được trả lời ở đâu, và khi nào

Câu trả lời đúng: trả lời **tại thời điểm gọi API**, bằng cách **query lại DB** (`tree_memberships`) với 2 tham số cùng lúc mới có: `userId` (đã có sẵn từ `Authentication.getPrincipal()`, do filter giai đoạn 5 set) và `treeId` (chỉ có khi request tới, ví dụ nằm trong path `/api/trees/{treeId}/persons`).

Đây là điểm khác biệt cốt lõi so với mọi thứ học ở giai đoạn 4-5: giai đoạn 4-5 giải quyết **authentication** (bạn là ai), còn bài toán này là **authorization theo resource** (bạn được làm gì với **cái này cụ thể**) — 2 khái niệm khác nhau, và authorization loại này **luôn cần thêm 1 bước query** vì dữ liệu quyền nằm trong DB, thay đổi được theo thời gian thực, không thể đóng băng vào JWT.

### 3. Hai hướng thiết kế — vì sao khác nhau, chọn dựa trên gì

**Hướng A — kiểm tra thủ công trong Service:**

```java
@Service
@RequiredArgsConstructor
public class PersonService {

    private final TreeMembershipRepository membershipRepository;

    public void deletePerson(UUID treeId, UUID personId, UUID currentUserId) {
        TreeMembership membership = membershipRepository
                .findByTreeIdAndUserId(treeId, currentUserId)
                .orElseThrow(() -> new AccessDeniedException("Không phải thành viên cây này"));

        if (membership.getRole() != TreeRole.OWNER && membership.getRole() != TreeRole.EDITOR) {
            throw new AccessDeniedException("Không đủ quyền xoá");
        }
        // ... logic xoá
    }
}
```

Ưu điểm: đơn giản, dễ debug (đọc code là hiểu ngay), không cần học thêm cơ chế Spring nào mới. Nhược điểm: nếu có 20 chỗ cần check quyền, phải lặp lại logic này 20 lần — dễ quên check ở 1 chỗ nào đó.

**Hướng B — PermissionEvaluator + @PreAuthorize:**

```java
@Component
@RequiredArgsConstructor
public class TreePermissionEvaluator implements PermissionEvaluator {

    private final TreeMembershipRepository membershipRepository;

    @Override
    public boolean hasPermission(Authentication authentication, Object targetId,
                                  String targetType, Object permission) {
        UUID userId = (UUID) authentication.getPrincipal(); // đúng field học ở phần Authentication
        UUID treeId = (UUID) targetId;

        return membershipRepository.findByTreeIdAndUserId(treeId, userId)
                .map(m -> m.getRole().name().equals(permission))
                .orElse(false);
    }

    @Override
    public boolean hasPermission(Authentication authentication, Serializable targetId,
                                  String targetType, Object permission) {
        return hasPermission(authentication, (Object) targetId, targetType, permission);
    }
}
```

```java
@PreAuthorize("hasPermission(#treeId, 'tree', 'EDITOR')")
public void deletePerson(UUID treeId, UUID personId) { ... }
```

Ưu điểm: logic phân quyền tập trung 1 chỗ duy nhất (`TreePermissionEvaluator`), khai báo ở method chỉ 1 dòng, khó quên check. Nhược điểm: phải học thêm cơ chế `PermissionEvaluator` mà `@EnableMethodSecurity` (có sẵn trong `SecurityConfig` từ đầu nhưng chưa thật sự dùng tới) mới kích hoạt được.

**Chọn dựa trên gì**: nếu số lượng chỗ cần check quyền theo tree còn ít (vài method), Hướng A đủ dùng và code lộ ra rõ ràng hơn để tự tin đang làm đúng. Khi hệ thống lớn dần, nhiều entity đều cần check theo `treeId` tương tự nhau, chuyển sang Hướng B để không lặp code. Đây không phải quyết định phải chốt ngay — có thể bắt đầu bằng Hướng A, refactor sang Hướng B sau khi thấy lặp lại nhiều.

---

## Cách luyện tập tổng thể — thứ tự ráp lại khi bắt tay code thật

Sau khi qua hết 6 giai đoạn, quay lại đúng roadmap code cũ (`JwtTokenProvider` → `JwtAuthenticationFilter` → `AuthController`...) — lúc đó đọc lại code đã viết sẵn (`SecurityConfig`, `JpaAuditingConfig`) sẽ hiểu được từng dòng vì sao viết vậy, thay vì copy-paste mà không chắc.

Thứ tự ráp nối đúng logic phụ thuộc giữa các phần:

1. **`JwtTokenProvider`** — sinh & verify JWT (giai đoạn 3 + 5)
2. **`JwtAuthenticationFilter`** — đọc header, verify, set `SecurityContextHolder` (giai đoạn 5)
3. **`SecurityConfig`** — khai báo `SecurityFilterChain`, gắn filter vào đúng vị trí (giai đoạn 4 + 5)
4. **`AuditorAware`** — đọc `principal` từ `SecurityContext` để điền `@CreatedBy` (giai đoạn 2 + Authentication)
5. **`GoogleIdTokenVerifierService` + `AuthController`** — luồng Google login, tự cấp JWT riêng sau khi verify (giai đoạn 6)
6. **Phân quyền theo `treeId`** — thêm sau, ở tầng Service hoặc `PermissionEvaluator`, không phải phần bắt buộc để hệ thống JWT cơ bản chạy được
