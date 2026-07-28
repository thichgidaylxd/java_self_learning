# Tự học: i18n (Internationalization) trong Spring Boot

> Viết cho người chưa biết gì. Áp dụng trực tiếp vào project của bạn —
> đã có sẵn `MessageKeys.java`, `MessageService.java`, `LocaleConfig.java`,
> `MessageConfig.java`, `messages_vi.properties`, `messages_en.properties`.

---

## 1. Vấn đề cần giải quyết

Ứng dụng gia phả của bạn phục vụ người dùng Việt Nam, nhưng có thể sau
này mở rộng ra kiều bào nói tiếng Anh. Thay vì viết chết (hardcode)
chuỗi thông báo trong code:

```java
// KHÔNG nên — chuỗi tiếng Việt viết cứng trong code
throw new RuntimeException("Không tìm thấy người này");
```

Bạn muốn: **cùng 1 đoạn code**, nhưng thông báo trả về **tự đổi ngôn
ngữ** tuỳ theo người dùng đang dùng app bằng tiếng gì. Đây chính là bài
toán **i18n** (Internationalization — "quốc tế hoá", viết tắt vì có 18
chữ cái giữa "i" và "n").

## 2. `Locale` — khái niệm gốc, đại diện cho "1 vùng ngôn ngữ"

`Locale` là 1 class có sẵn trong Java (`java.util.Locale`), đại diện
cho tổ hợp **ngôn ngữ + vùng miền**:

```java
Locale.forLanguageTag("vi");    // tiếng Việt
Locale.forLanguageTag("en");    // tiếng Anh
Locale.forLanguageTag("en-US"); // tiếng Anh, vùng Mỹ (có thể khác en-GB)
```

Mọi cơ chế i18n của Spring đều xoay quanh: **xác định `Locale` nào cho
request hiện tại**, rồi **tra đúng bản dịch tương ứng**.

## 3. File `messages_xx.properties` — nơi chứa bản dịch

```
i18n/
├── messages_vi.properties
└── messages_en.properties
```

Nội dung dạng key-value, giống hệt cấu trúc `.properties` đã học ở bài
`@ConfigurationProperties`:

```properties
# messages_vi.properties
error.person_not_found=Không tìm thấy người này
error.internal_server_error=Đã có lỗi xảy ra, vui lòng thử lại
success=Thành công

# messages_en.properties
error.person_not_found=Person not found
error.internal_server_error=Something went wrong, please try again
success=Success
```

**Quy ước đặt tên bắt buộc:** `{basename}_{languageTag}.properties` —
`messages` là **basename** (tên gốc, bạn tự đặt, phải khớp với cấu
hình `MessageConfig`), `_vi`/`_en` là hậu tố ngôn ngữ. Đây chính là quy
ước Java `ResourceBundle` (có từ rất lâu trước Spring, Spring chỉ dùng
lại cơ chế có sẵn của Java, không phát minh mới).

## 4. `MessageSource` — interface trung tâm của toàn bộ cơ chế

Spring định nghĩa interface này để **tra cứu bản dịch theo key + locale**:

```java
public interface MessageSource {
    String getMessage(String code, Object[] args, Locale locale);
    // code = key ("error.person_not_found")
    // args = tham số chèn vào chuỗi (giải thích ở mục 6)
    // locale = ngôn ngữ muốn lấy
}
```

Spring Boot **tự động tạo sẵn 1 Bean `MessageSource`** (thường là
`ResourceBundleMessageSource` hoặc `ReloadableResourceBundleMessageSource`)
nếu bạn có file `messages.properties` (hoặc biến thể ngôn ngữ) đúng
chỗ mặc định (`src/main/resources/messages.properties`). Vì file của
bạn nằm trong thư mục con `i18n/`, cần **khai báo tường minh** bean này
— đây chính là vai trò của `MessageConfig.java`:

```java
@Configuration
public class MessageConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource source =
                new ReloadableResourceBundleMessageSource();

        source.setBasenames("classpath:i18n/messages"); // trỏ đúng thư mục
        source.setDefaultEncoding("UTF-8");              // BẮT BUỘC cho tiếng Việt có dấu
        source.setCacheSeconds(3600);                    // xem mục 9
        source.setFallbackToSystemLocale(false);         // xem mục 8

        return source;
    }
}
```

**`setBasenames("classpath:i18n/messages")`** — không viết đuôi
`_vi`/`.properties`, Spring tự thêm phần đó dựa theo `Locale` đang cần.
Nếu file thật là `i18n/messages_vi.properties`, basename khai đúng là
`i18n/messages` (phần chung, chưa có hậu tố ngôn ngữ).

## 5. `LocaleResolver` — xác định request này dùng ngôn ngữ nào

Đây là câu hỏi khác: **biết được có file dịch rồi, nhưng làm sao biết
1 request cụ thể nên lấy bản `vi` hay `en`?** — đây là việc của
`LocaleResolver`, khai trong `LocaleConfig.java`.

### Cách 1 — Dựa vào header `Accept-Language` (trình duyệt tự gửi)

```java
@Bean
public LocaleResolver localeResolver() {
    AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
    resolver.setDefaultLocale(Locale.forLanguageTag("vi")); // fallback nếu header không có/không hiểu
    resolver.setSupportedLocales(List.of(
            Locale.forLanguageTag("vi"),
            Locale.forLanguageTag("en")
    ));
    return resolver;
}
```

Trình duyệt/app tự gửi header `Accept-Language: en-US,en;q=0.9` mỗi
request (đúng cấu trúc HTTP Header đã học ở phần đầu) — resolver này
đọc header đó, chọn locale khớp nhất trong danh sách hỗ trợ.

### Cách 2 — Cho phép client tự chỉ định qua query param (`?lang=en`)

```java
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    resolver.setDefaultLocale(Locale.forLanguageTag("vi"));
    return resolver;
}

@Bean
public LocaleChangeInterceptor localeChangeInterceptor() {
    LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
    interceptor.setParamName("lang");   // ?lang=en trong URL
    return interceptor;
}
```

**Khác biệt quan trọng cho API thuần (không phải web app truyền
thống):** `SessionLocaleResolver` dựa vào **session** — nhưng hệ thống
của bạn đã chọn **stateless (JWT, không session)** ở phần Security đã
học! Dùng `SessionLocaleResolver` ở đây **sẽ không nhất quán về kiến
trúc** — mỗi request không session sẽ luôn dùng lại default locale, mất
tác dụng "nhớ" lựa chọn ngôn ngữ.

**Khuyến nghị cho API stateless như hệ thống này:** dùng
`AcceptHeaderLocaleResolver` (Cách 1) — client tự gửi header mỗi
request, không cần server "nhớ" gì cả, đúng tinh thần stateless đã
chọn xuyên suốt.

## 6. Tham số chèn vào chuỗi (`args`) — vì sao `getMessage` nhận `Object[]`

Nhiều thông báo cần **chèn giá trị động** vào giữa câu:

```properties
error.min_length=Trường {0} phải có tối thiểu {1} ký tự
```

```java
messageSource.getMessage(
    "error.min_length",
    new Object[]{"Họ tên", 3},   // {0} -> "Họ tên", {1} -> 3
    locale
);
// Kết quả: "Trường Họ tên phải có tối thiểu 3 ký tự"
```

Đây dùng cú pháp `MessageFormat` của Java (`{0}`, `{1}`...) — không
phải cú pháp riêng của Spring.

## 7. `MessageService.java` — vì sao cần lớp bọc riêng, không gọi thẳng `MessageSource`

Nhìn lại `ApiResponseFactory` đã dùng trước đó:
```java
messageService.get(MessageKeys.SUCCESS)
```

Không gọi thẳng `messageSource.getMessage(...)` vì 2 lý do:

1. **`getMessage()` gốc luôn bắt bạn tự truyền `Locale`** — lặp lại ở
   mọi nơi gọi, dễ quên, dễ sai. `MessageService` nên **tự lấy Locale
   hiện tại** (qua `LocaleContextHolder.getLocale()` — cùng ý tưởng
   "tự động lấy context hiện tại" như `SecurityContextHolder` đã học,
   cũng dựa trên `ThreadLocal`) để người gọi không cần truyền tay.

2. **Tên method ngắn gọn hơn**, và là chỗ tốt để xử lý thêm (ví dụ: key
   không tồn tại thì trả về chính `key` đó thay vì ném exception, tránh
   sập app chỉ vì thiếu 1 dòng dịch).

```java
@Service
@RequiredArgsConstructor
public class MessageService {

    private final MessageSource messageSource;

    public String get(String key) {
        return get(key, null);
    }

    public String get(String key, Object[] args) {
        Locale locale = LocaleContextHolder.getLocale();  // tự lấy, không cần truyền tay
        try {
            return messageSource.getMessage(key, args, locale);
        } catch (NoSuchMessageException e) {
            return key;  // fallback an toàn — trả về key thay vì crash
        }
    }
}
```

`LocaleContextHolder` hoạt động đúng cơ chế `ThreadLocal` đã học ở bài
JPA Auditing/SecurityContext — Spring **tự set** locale vào đây ở đầu
mỗi request (dựa vào `LocaleResolver` đã cấu hình), rồi bất kỳ đâu
trong cùng thread xử lý request đó đều đọc lại được, không cần truyền
tay qua nhiều tầng method.

## 8. `setFallbackToSystemLocale(false)` — vì sao nên tắt

Mặc định, nếu 1 key **không tồn tại** trong file locale đang yêu cầu
(`messages_en.properties` thiếu key nào đó), Spring sẽ **tự động** thử
tìm trong **Locale hệ điều hành của server** (`Locale.getDefault()`) —
điều này **cực kỳ khó đoán trong production**: server dev của bạn chạy
Windows tiếng Việt sẽ ra kết quả khác server production chạy Linux
tiếng Anh, dù code y hệt. Tắt hành vi này (`false`) để đảm bảo: thiếu
bản dịch thì **rõ ràng báo lỗi/fallback về giá trị mặc định bạn tự
định nghĩa** (như `MessageService` ở mục 7 đã làm), không phụ thuộc
môi trường server đang chạy ở đâu.

## 9. `setCacheSeconds` — đánh đổi giữa hiệu năng và cập nhật nhanh

`ReloadableResourceBundleMessageSource` (khác với
`ResourceBundleMessageSource` cơ bản) hỗ trợ **đọc lại file mỗi X
giây** thay vì chỉ đọc 1 lần lúc khởi động — hữu ích khi dev: sửa file
`.properties`, không cần restart cả app mới thấy thay đổi.

```java
source.setCacheSeconds(3600);  // production: cache 1 giờ, đỡ đọc file liên tục
// hoặc
source.setCacheSeconds(0);     // dev: đọc lại MỌI LẦN gọi, thấy thay đổi ngay (chậm hơn)
// hoặc
source.setCacheSeconds(-1);    // chỉ đọc 1 lần duy nhất lúc khởi động, không bao giờ reload
```

## 10. `MessageKeys.java` — vì sao dùng hằng số thay vì gõ tay chuỗi key

```java
public class MessageKeys {
    public static final String SUCCESS = "success";
    public static final String PERSON_NOT_FOUND = "error.person_not_found";
}
```

Thay vì gõ tay `"error.person_not_found"` ở mọi nơi cần dùng (dễ gõ sai
chính tả, IDE không tự phát hiện lỗi cho tới khi chạy thử) — dùng hằng
số giúp: **IDE tự autocomplete**, gõ sai tên biến sẽ báo lỗi biên dịch
ngay (thay vì lỗi ẩn lúc chạy mới phát hiện thiếu bản dịch), và **đổi
tên key dễ dàng** (chỉ sửa 1 chỗ, IDE tự tìm hết nơi dùng).

## 11. Liên kết với `GlobalExceptionHandler` đã học — Bean Validation message

Nhớ lại `GlobalExceptionHandler.handleValidationException` đã xem
trước đó:
```java
List<String> messages = ex.getBindingResult()
        .getFieldErrors()
        .stream()
        .map(DefaultMessageSourceResolvable::getDefaultMessage)
        .toList();
```

Đây là **1 cơ chế i18n riêng, độc lập với `MessageSource` của bạn** —
Bean Validation (`@NotBlank`, `@Min`...) có **hệ thống message riêng**
gọi là `ValidationMessages.properties` (đặt tại
`src/main/resources/ValidationMessages.properties`, không phải
`i18n/messages_xx.properties`):

```java
@NotBlank(message = "{error.full_name_required}")  // {} = tra trong ValidationMessages
private String fullName;
```

```properties
# ValidationMessages.properties (khác file với messages_vi.properties)
error.full_name_required=Họ tên không được để trống
```

**Đây là điểm dễ nhầm nhất của cả bài học:** `MessageSource` (dùng cho
`MessageService`, `ApiResponseFactory`) và cơ chế message của Bean
Validation là **2 hệ thống tách biệt**, đọc từ **2 loại file khác
nhau**, dù cùng mục đích "đa ngôn ngữ". Muốn message lỗi validate cũng
đa ngôn ngữ theo `Locale` giống hệ thống chính, cần cấu hình thêm 1
`LocalValidatorFactoryBean` trỏ về **cùng** `MessageSource` bạn đã tạo,
thay vì để mặc định dùng `ValidationMessages.properties` riêng:

```java
@Bean
public LocalValidatorFactoryBean validator(MessageSource messageSource) {
    LocalValidatorFactoryBean bean = new LocalValidatorFactoryBean();
    bean.setValidationMessageSource(messageSource);  // dùng chung nguồn dịch
    return bean;
}
```

## Toàn bộ luồng — ráp lại 1 request thật

```
1. Client gửi GET /api/persons/123
   Header: Accept-Language: en-US

2. Spring (qua LocaleResolver đã cấu hình) đọc header, xác định
   Locale = en, set vào LocaleContextHolder (ThreadLocal, riêng theo thread)

3. Controller/Service gọi personRepository.findById(id) -> không tìm thấy
   -> throw new AppException(ErrorCode.PERSON_NOT_FOUND)

4. GlobalExceptionHandler.handleAppException() bắt được, gọi
   responseFactory.error(ErrorCode.PERSON_NOT_FOUND)

5. Bên trong đó, messageService.get(errorCode.getMessageKey())
   -> đọc Locale hiện tại từ LocaleContextHolder (= en, từ bước 2)
   -> messageSource.getMessage("error.person_not_found", null, Locale.EN)
   -> tra trong messages_en.properties -> "Person not found"

6. Response trả về: {"code": 404, "success": false, "messages": ["Person not found"]}
```

Nếu client đổi header thành `Accept-Language: vi`, **toàn bộ luồng
giống hệt**, chỉ khác bước 5 tra vào `messages_vi.properties` — đây
chính là giá trị cốt lõi của i18n: code nghiệp vụ (bước 3-4) **không
đổi 1 dòng nào**, chỉ tầng message thay đổi theo request.

---

## Tóm tắt bằng 1 câu cho mỗi thành phần

- **`Locale`** — "ngôn ngữ + vùng miền nào"
- **`messages_xx.properties`** — "kho chứa bản dịch cho từng locale"
- **`MessageSource`** — "tra cứu key + locale ra đúng chuỗi dịch"
- **`LocaleResolver`** — "xác định request này nên dùng locale nào"
- **`LocaleContextHolder`** — "nơi lưu tạm locale hiện tại, riêng theo thread (ThreadLocal)"
- **`MessageService`** — "lớp bọc tiện lợi, tự lấy locale, không phải truyền tay"
- **`MessageKeys`** — "hằng số key, tránh gõ tay sai chính tả"

## Câu hỏi tự kiểm tra

1. Vì sao dự án này (JWT, stateless) nên dùng `AcceptHeaderLocaleResolver`
   thay vì `SessionLocaleResolver`?
2. `setFallbackToSystemLocale(false)` giải quyết rủi ro gì?
3. `MessageService` và `MessageSource` khác nhau ở điểm nào — vì sao
   không gọi thẳng `MessageSource`?
4. Bean Validation message (`ValidationMessages.properties`) và
   `MessageSource` chính (`messages_vi.properties`) có phải cùng 1 cơ
   chế không? Muốn dùng chung nguồn dịch thì làm thế nào?
