# Tự học: `@ConfigurationProperties` — đọc cấu hình vào 1 class riêng

> Viết cho người chưa biết gì. Đọc xong hiểu được: properties là gì,
> `@ConfigurationProperties` hoạt động thế nào bên trong, và nếu KHÔNG
> dùng annotation này thì làm cách nào khác.

---

## 1. "Properties" là gì — khái niệm gốc

Mọi ứng dụng đều cần **cấu hình** — những giá trị có thể thay đổi tuỳ
môi trường (dev/prod), không nên viết cứng (hardcode) vào code:
cổng chạy server, thông tin kết nối DB, JWT secret...

Spring Boot đọc cấu hình từ nhiều **nguồn** (property sources), xếp
theo thứ tự ưu tiên (nguồn sau đè nguồn trước nếu trùng key):

```
1. application.yaml / application.properties (trong resources/)
2. application-{profile}.yaml (vd: application-dev.yaml)
3. Biến môi trường hệ điều hành (environment variables)
4. Tham số dòng lệnh (--server.port=9090)
```

Dù là `.yaml` hay `.properties`, cuối cùng Spring đều gộp lại thành
**1 tập hợp cặp key-value phẳng**, kiểu:
```
jwt.secret = abc123...
jwt.access-token-expiration-ms = 900000
```
File `.yaml` chỉ là cách viết **lồng cấp** cho gọn mắt — về bản chất,
`.yaml` này:
```yaml
jwt:
  secret: abc123
  access-token-expiration-ms: 900000
```
tương đương hệt `.properties` này:
```properties
jwt.secret=abc123
jwt.access-token-expiration-ms=900000
```

## 2. Cách đơn giản nhất để đọc 1 giá trị — `@Value`

Bạn đã dùng cách này ở `JwtTokenProvider`:
```java
public JwtTokenProvider(@Value("${jwt.secret}") String secret) { ... }
```

`@Value("${jwt.secret}")` nói với Spring: *"tìm trong tất cả property
sources đã gộp, key nào tên `jwt.secret`, lấy giá trị đó tiêm vào
tham số này"*.

**Vấn đề khi có NHIỀU giá trị liên quan tới nhau** (như JWT có cả
`secret`, `access-token-expiration-ms`, `refresh-token-expiration-ms`)
— nếu dùng `@Value` cho từng cái, thông tin bị **rải rác** khắp nơi
trong constructor, khó nhìn tổng quan "JWT có bao nhiêu cấu hình", khó
tái sử dụng ở nhiều class khác nhau (mỗi class cần lại phải khai
`@Value` lại từ đầu).

## 3. `@ConfigurationProperties` — gom nhóm cấu hình thành 1 class

```java
@Getter
@Setter
@Component
@ConfigurationProperties(prefix = "app.jwt")
public class JwtProperties {
    private String secret;
    private long expirationMs;
}
```

Ý tưởng: thay vì tiêm từng giá trị lẻ, bạn tạo **1 class đại diện cho
cả nhóm cấu hình**, Spring tự **map (ánh xạ)** toàn bộ các key có cùng
tiền tố (`prefix`) vào các field tương ứng.

### Giải nghĩa từng annotation

**`@ConfigurationProperties(prefix = "app.jwt")`** — nói Spring: *"mọi
key bắt đầu bằng `app.jwt.`, hãy map phần còn lại của tên key vào field
cùng tên trong class này"*.

```yaml
app:
  jwt:
    secret: abc123           # -> field "secret"
    expiration-ms: 900000    # -> field "expirationMs"
```

**`@Component`** — bắt buộc phải có (hoặc cách khác thay thế, xem mục
5) — vì `@ConfigurationProperties` **chỉ là annotation đánh dấu class
này CẦN được bind cấu hình vào**, nó **không tự khiến Spring tạo
Bean**. Nếu thiếu `@Component`, Spring không quét thấy class này, class
này **sẽ không tồn tại** trong ApplicationContext — dù bạn có
`@ConfigurationProperties` cũng vô nghĩa vì không ai tạo instance để
bind vào.

**`@Getter` / `@Setter` (Lombok)** — đây là điểm khác biệt lớn nhất so
với `@Value`. Cơ chế bind của `@ConfigurationProperties` (mặc định,
kiểu "JavaBean binding") hoạt động bằng cách:
1. Spring tạo 1 instance rỗng: `new JwtProperties()`
2. Với mỗi key `app.jwt.xxx`, tìm **setter** tương ứng (`setXxx(...)`),
   **gọi** setter đó với giá trị đọc được

→ **Không có setter, Spring không có cách nào gán giá trị vào field**
(trừ khi bạn dùng biến thể khác, xem mục 6). Đây là lý do bắt buộc
`@Setter` (và `@Getter` để nơi khác đọc lại giá trị, dùng `jwtProperties.getSecret()`).

## 4. Quy tắc đặt tên — "Relaxed Binding" (ánh xạ linh hoạt)

Đây là phần hay bị rối nhất — Spring **không** yêu cầu tên key trong
YAML phải khớp 100% tên field trong Java. Nó hỗ trợ nhiều biến thể viết
hoa/thường/dấu gạch khác nhau, tự động coi là **cùng 1 ý nghĩa**:

| Field trong Java | Các cách viết TƯƠNG ĐƯƠNG trong YAML/properties/env |
|---|---|
| `expirationMs` | `expiration-ms` (kebab-case, khuyến nghị dùng trong YAML) |
| | `expirationMs` (camelCase, viết y hệt field) |
| | `expiration_ms` (snake_case) |
| | `EXPIRATION_MS` (khi lấy từ biến môi trường — env var luôn viết hoa + gạch dưới theo quy ước OS) |

**Đây chính là lý do** bạn có thể viết trong `.env`:
```
JWT_SECRET=abc123
```
và trong `application.yaml`:
```yaml
app:
  jwt:
    secret: ${JWT_SECRET}
```
2 chữ `SECRET` (biến môi trường, viết hoa) và `secret` (key YAML, viết
thường) là "tương đương" theo quy tắc Relaxed Binding — không phải
trùng khớp tình cờ.

**Khuyến nghị khi tự viết YAML:** luôn dùng `kebab-case` (chữ thường,
gạch ngang) cho key nhiều từ — đây là style Spring Boot chính thức
khuyến nghị, đọc rõ ràng nhất và tương thích tốt với mọi nguồn cấu
hình.

## 5. Cơ chế bên trong — điều gì thực sự chạy khi app khởi động

Spring Boot có 1 cơ chế tên `ConfigurationPropertiesBindingPostProcessor`
— đây là 1 "hậu xử lý" (`BeanPostProcessor`, cùng họ với
`AuditingEntityListener` đã học — cũng là 1 dạng "móc vào lifecycle",
chỉ khác là lifecycle của **Bean**, không phải của **Entity**):

```
1. Spring quét thấy class có @ConfigurationProperties
2. Sau khi @Component tạo xong instance rỗng (constructor chạy xong)
3. ConfigurationPropertiesBindingPostProcessor can thiệp:
   - Lấy toàn bộ property sources đã gộp
   - Lọc ra key nào bắt đầu bằng prefix khai báo
   - Dùng reflection tìm setter tương ứng (theo Relaxed Binding)
   - Gọi setter, gán giá trị vào (tự convert kiểu: String -> long, -> boolean...)
4. Bean hoàn chỉnh, sẵn sàng dùng
```

Việc **tự convert kiểu dữ liệu** (`"900000"` trong YAML là String, tự
đổi thành `long expirationMs`) cũng là 1 phần của cơ chế này — dùng
`ConversionService` có sẵn của Spring, tương tự cách `@RequestBody`
tự parse JSON thành object Java bạn đã dùng ở Controller.

## 6. Không dùng Lombok `@Setter` — vẫn có cách khác: Constructor Binding

Nếu bạn muốn field là `final` (bất biến, an toàn hơn cho môi trường đa
luồng — đúng nguyên tắc "Bean chỉ nên chứa dữ liệu bất biến" đã học ở
phần Spring cơ bản), có thể dùng **Constructor Binding** thay vì
JavaBean (setter) binding:

```java
@ConfigurationProperties(prefix = "app.jwt")
public record JwtProperties(String secret, long expirationMs) { }
```

(dùng `record` — 1 kiểu class đặc biệt trong Java hiện đại, tự động có
`final` field + constructor + getter, không cần Lombok)

Spring nhận ra class này chỉ có **1 constructor duy nhất**, tự động
hiểu đây là "Constructor Binding" — gọi thẳng constructor với các giá
trị đã đọc được, **không cần setter nào cả**. Đây là cách hiện đại hơn,
khuyến nghị dùng khi có thể — nhưng cách bạn đang học (Getter/Setter +
class thường) **vẫn hoàn toàn đúng và phổ biến**, nhất là với code cũ
hơn hoặc khi cần thêm logic validate phức tạp trong setter.

## 7. Nếu KHÔNG dùng annotation `@ConfigurationProperties` — 3 cách thay thế

Đây là phần bạn hỏi cụ thể. Có annotation chỉ là **tiện lợi**, không
phải bắt buộc duy nhất — dưới đây là các cách "thủ công hơn" để đạt
được kết quả tương tự.

### Cách 1 — `@Value` cho từng field (đã học, cách đơn giản nhất)
```java
@Component
public class JwtProperties {

    @Value("${app.jwt.secret}")
    private String secret;

    @Value("${app.jwt.expiration-ms}")
    private long expirationMs;

    // vẫn cần getter để nơi khác đọc được
    public String getSecret() { return secret; }
    public long getExpirationMs() { return expirationMs; }
}
```
Nhược điểm: phải viết `@Value("${app.jwt.xxx}")` lặp lại cho từng
field, dễ gõ sai tên key (không có gì báo lỗi lúc biên dịch nếu gõ sai
`app.jwt.secrett`), không tự có Relaxed Binding thông minh như
`@ConfigurationProperties` (dù `@Value` vẫn hỗ trợ phần nào).

### Cách 2 — Đọc trực tiếp qua object `Environment`

Đây là cách "thô" nhất, không cần bất kỳ annotation binding nào:

```java
@Component
public class JwtProperties {

    private final String secret;
    private final long expirationMs;

    // Environment là 1 Bean có sẵn của Spring, chứa TOÀN BỘ property
    // sources đã gộp — tiêm vào qua constructor như bình thường (DI đã học)
    public JwtProperties(Environment env) {
        this.secret = env.getProperty("app.jwt.secret");
        this.expirationMs = Long.parseLong(
                env.getProperty("app.jwt.expiration-ms", "900000")); // có default
    }

    public String getSecret() { return secret; }
    public long getExpirationMs() { return expirationMs; }
}
```

`Environment.getProperty(key, defaultValue)` — tự tay gọi, tự tay
convert kiểu (`Long.parseLong`), tự tay xử lý giá trị mặc định nếu
thiếu. Đây chính xác là những việc mà `@ConfigurationProperties` **làm
hộ bạn tự động** — viết cách này giúp hiểu rõ "bên dưới nó đang làm
gì", nhưng thực tế ít ai viết tay kiểu này cho nhiều property, vì lặp
lại code chuyển đổi kiểu nhàm chán.

### Cách 3 — Đọc thẳng biến môi trường qua Java thuần, không qua Spring
```java
public class JwtProperties {
    private final String secret = System.getenv("JWT_SECRET");
}
```
`System.getenv(...)` là API **thuần Java**, không liên quan gì tới
Spring — đọc thẳng biến môi trường hệ điều hành lúc JVM khởi động.
**Nhược điểm lớn:** mất hết lợi ích của Spring Boot property sources
(không đọc được từ YAML, không có profile dev/prod, không có Relaxed
Binding, không tự động fallback giữa nhiều nguồn) — chỉ nên dùng trong
trường hợp rất đặc biệt (ví dụ code chạy trước khi Spring Context khởi
tạo xong), gần như không bao giờ cần trong ứng dụng Spring Boot bình
thường.

## 8. So sánh 4 cách — khi nào dùng cách nào

| Cách | Ưu điểm | Nhược điểm | Khi nào dùng |
|---|---|---|---|
| `@ConfigurationProperties` | Gom nhóm rõ ràng, Relaxed Binding tự động, dễ validate (mục 9) | Cần thêm `@Component` + getter/setter (hoặc record) | Nhóm ≥ 2-3 property liên quan (đúng trường hợp JWT của bạn) |
| `@Value` từng field | Đơn giản, nhanh cho 1-2 giá trị | Rải rác nếu nhiều giá trị, dễ gõ sai key | Chỉ cần đúng 1 giá trị đơn lẻ trong 1 class |
| `Environment.getProperty()` | Kiểm soát hoàn toàn, có thể dùng ngoài Bean | Viết tay nhiều, dễ quên convert kiểu | Cần đọc property tại nơi không tiện dùng `@Value`/`@ConfigurationProperties` (ví dụ trong static method) |
| `System.getenv()` | Không phụ thuộc Spring | Mất hết tính năng Spring config | Gần như không bao giờ, trừ trường hợp đặc biệt |

**Với `JwtProperties` cụ thể của bạn (2 field liên quan: secret +
expiration) — `@ConfigurationProperties` là lựa chọn đúng**, đây chính
là use-case điển hình nó sinh ra để giải quyết.

## 9. Thêm validate — vì sao nên làm khi có `@ConfigurationProperties`

Vì secret quá ngắn sẽ làm `MACSigner` (đã học ở `JwtTokenProvider`)
lỗi lúc khởi tạo — có thể **chặn sớm hơn** ngay ở bước đọc cấu hình:

```java
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Min;
import org.springframework.validation.annotation.Validated;

@Getter
@Setter
@Component
@Validated                                    // BẬT validate
@ConfigurationProperties(prefix = "app.jwt")
public class JwtProperties {

    @NotBlank(message = "JWT secret không được để trống")
    private String secret;

    @Min(value = 60000, message = "expirationMs phải tối thiểu 60000 (1 phút)")
    private long expirationMs;
}
```
Thiếu `@Validated`, các annotation `@NotBlank`/`@Min` **chỉ nằm đó cho
đẹp**, không có tác dụng gì — giống hệt bài học "thiếu công tắc tổng
thì cơ chế không chạy" đã gặp ở `@EnableJpaAuditing`.

## 10. Cách dùng sau khi có `JwtProperties` — thay `@Value` cũ trong `JwtTokenProvider`

```java
@Component
@RequiredArgsConstructor   // Lombok tự sinh constructor nhận field final
public class JwtTokenProvider {

    private final JwtProperties jwtProperties;   // tiêm cả object, không tiêm từng giá trị

    public String generateAccessToken(UUID userId, String email) {
        long ttl = jwtProperties.getExpirationMs();
        String secret = jwtProperties.getSecret();
        // ...
    }
}
```
So với cách cũ (`@Value` trực tiếp trong constructor `JwtTokenProvider`),
giờ mọi cấu hình JWT nằm gọn trong `JwtProperties` — nếu sau này thêm
`issuer`, `audience`... chỉ cần thêm field vào `JwtProperties`, không
phải sửa constructor của `JwtTokenProvider`.

---

## Tóm tắt bằng 1 câu cho mỗi annotation

- **`@ConfigurationProperties(prefix=...)`** — "map các key cùng tiền
  tố này vào field cùng tên trong class"
- **`@Component`** — "biến class này thành Bean, không thì không ai
  tạo ra nó để bind vào"
- **`@Getter`/`@Setter`** — "cho phép cơ chế bind (qua setter) và nơi
  khác đọc lại (qua getter)"
- **`@Validated`** — "công tắc bật validate, thiếu thì các rule
  `@NotBlank`/`@Min`... vô tác dụng"

## Câu hỏi tự kiểm tra

1. Vì sao `@ConfigurationProperties` cần thêm `@Component` mới hoạt
   động, tự nó không đủ?
2. `expiration-ms` trong YAML và `expirationMs` trong Java field — vì
   sao Spring hiểu là cùng 1 thứ?
3. Nếu không muốn dùng setter (muốn field `final`), có cách nào khác
   để `@ConfigurationProperties` vẫn bind được?
4. Viết 1 cách đọc `app.jwt.secret` mà KHÔNG dùng bất kỳ annotation
   nào của Spring cả (gợi ý: mục 7, cách 2).
