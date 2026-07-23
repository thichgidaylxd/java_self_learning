# Spring Core Concepts — Học từ gốc đi lên

> Sắp xếp theo **5 tầng phụ thuộc**: tầng dưới là điều kiện bắt buộc để hiểu tầng trên. Ôn đúng thứ tự này để không bị hổng kiến thức giữa chừng.

```
Tầng 1: Nền tảng ngôn ngữ
        ↓
Tầng 2: Vùng nhớ & runtime
        ↓
Tầng 3: Cơ chế Java nâng cao
        ↓
Tầng 4: Cơ chế lõi Spring
        ↓
Tầng 5: Ứng dụng thực tế
```

Hiểu hết 5 tầng thì khoảng 80% Spring không còn là "phép thuật" — chỉ còn là ráp các mảnh ghép đã biết lại với nhau.

---

# TẦNG 1 — Nền tảng ngôn ngữ

## 1.1 Object và Reference

Biến trong Java **không chứa object**. Nó chỉ chứa một **tham chiếu (reference)** — có thể hiểu như địa chỉ.

```java
UserService service = new UserService();
```

Tưởng tượng sai (nhiều người mới học nghĩ vậy):

```
service
┌────────────┐
│UserService │   ← SAI, biến không "chứa" object
└────────────┘
```

Thực tế trong JVM:

```
Stack                      Heap
┌─────────┐               ┌────────────────┐
│service  │──────────────▶│ UserService    │
│ 0x100   │               └────────────────┘
└─────────┘
```

`service` = địa chỉ nhà. `UserService object` = căn nhà.

### Hai biến cùng trỏ một object

```java
UserService s1 = new UserService();
UserService s2 = s1;
```

```
s1 ─────┐
        ▼
     UserService   (chỉ có 1 object)
        ▲
s2 ─────┘
```

```java
s2.setName("ABC");
s1.getName(); // → "ABC", vì cùng trỏ tới 1 object
```

**Đây chính là bản chất của Dependency Injection** (sẽ gặp lại ở Tầng 4): Controller không "chứa" Service, Controller chỉ **chứa địa chỉ** của Service.

## 1.2 Primitive type vs Reference type

### Primitive type

Biến primitive **luôn là chính ô nhớ chứa giá trị** — không có khái niệm địa chỉ. Nhưng ô nhớ đó nằm ở đâu tùy ngữ cảnh:

- **Biến local (trong method)** → ô nhớ nằm trên **Stack frame**.
```java
void process() {
    int age = 20;   // ô nhớ trên Stack
}
```
- **Field (trong class)** → ô nhớ nằm **ngay trong object, trên Heap**.
```java
class User { int age; }  // age nằm trong object, trên Heap

User u = new User();
u.age = 20;
```

### Reference type

- Copy giữa 2 biến reference (`u2 = u1`) → chỉ **copy địa chỉ**, không copy nội dung object.
- `new` là thứ **DUY NHẤT** tạo object mới trên Heap — bất kể gán cho biến nào:
```java
User u1 = new User();  // (1) tạo object mới (2) trả địa chỉ (3) gán cho u1
User u2 = u1;           // KHÔNG new → không tạo object mới, chỉ copy địa chỉ
```

### Bảng tổng kết

| | Biến local (trong method) | Field (trong object) |
|---|---|---|
| **Primitive** | Ô nhớ trên Stack, biến = giá trị thật | Ô nhớ trong object, trên Heap |
| **Reference** | Ô nhớ trên Stack, chứa địa chỉ trỏ Heap | Ô nhớ trong object (Heap), chứa địa chỉ trỏ object khác trên Heap |

**Điểm chung:** Dù primitive hay reference, local hay field — biến luôn là 1 ô nhớ. Khác biệt chỉ ở chỗ ô nhớ đó **chứa gì**: giá trị thật hay địa chỉ.

## 1.3 Pass by Value vs Pass by Reference

**Sự thật quan trọng nhất: Java LUÔN LUÔN pass-by-value. Không có pass-by-reference.**

### Trường hợp 1 — truyền kiểu nguyên thủy

```java
void increase(int x) { x = x + 1; }

int age = 20;
increase(age);
System.out.println(age); // vẫn 20
```

`age` được **copy giá trị** sang `x` — 2 ô nhớ hoàn toàn khác nhau.

### Trường hợp 2 — truyền object (sửa nội dung)

```java
void rename(User u) { u.setName("Bob"); }

User user = new User("Alice");
rename(user);
System.out.println(user.getName()); // in ra "Bob"!
```

Cái được truyền vào là **giá trị của reference** (địa chỉ) — bản copy của "chìa khóa", không phải bản copy "cái tủ". Cả 2 chìa khóa cùng mở 1 tủ, sửa nội dung tủ thì thấy ở cả 2 nơi.

### Trường hợp 3 — gán lại biến object bên trong method

```java
void reassign(User u) { u = new User("Charlie"); }

User user = new User("Alice");
reassign(user);
System.out.println(user.getName()); // vẫn "Alice"!
```

`u` bên trong chỉ là **bản copy của reference**. Gán lại `u` chỉ đổi hướng bản copy đó, không đổi hướng biến `user` gốc.

### Bảng tóm tắt

| | Truyền cái gì | Sửa nội dung object? | Gán lại = object mới? |
|---|---|---|---|
| Kiểu nguyên thủy | copy giá trị | N/A | Không ảnh hưởng ngoài |
| Object | copy của reference | **Có** ảnh hưởng (cùng trỏ 1 object) | Không ảnh hưởng ngoài |

**Cách gọi chính xác nhất:** Java là *"pass by value of the reference"*.

---

# TẦNG 2 — Vùng nhớ & runtime

## 2.1 Stack và Heap

```java
public void process() {
    int age = 20;
    User user = new User();
}
```

```
Stack Frame                Heap
┌──────────┐               ┌──────────┐
│age = 20  │               │ 0x200    │
│user=0x200│──────────────▶│ User     │
└──────────┘               └──────────┘
```

**Vì sao quan trọng?** Vì rất nhiều bug trong Spring liên quan tới việc không hiểu Stack và Heap.

```java
@Service
public class UserService {
    private User currentUser;   // field, KHÔNG nằm trên Stack
}
```

Field `currentUser` nằm **trong object**, mà object nằm trên **Heap** — vùng nhớ **dùng chung** cho toàn bộ chương trình.

```
Thread A ──▶ UserService
Thread B ──▶ UserService   (cùng đọc/ghi 1 field)
```

→ Đây là nguồn gốc của **race condition**.

## 2.2 Java Memory Model (JMM) và `volatile`

```java
boolean running = true;

// Thread A:
while (running) { }

// Thread B:
running = false;
```

Bạn nghĩ Thread A dừng ngay. **Không chắc.** Mỗi CPU core có **cache riêng**. Thread B ghi `false`, nhưng Thread A có thể vẫn đọc `true` từ cache riêng của core mình — vòng lặp chạy vô tận.

`volatile` giải quyết 2 vấn đề:

1. **Visibility**: đảm bảo ghi ở thread này → thấy ngay ở thread khác (không kẹt trong CPU cache).
2. **Ngăn reorder lệnh**: JVM/CPU thường được phép sắp xếp lại thứ tự lệnh để tối ưu; `volatile` cấm điều đó với biến đó.

```java
volatile boolean running;
```

**`volatile` KHÔNG đảm bảo atomic** cho phép toán phức tạp như `count++` (thực chất là 3 bước: đọc – cộng – ghi). Hai thread có thể xen vào giữa làm mất 1 lần cộng. Muốn an toàn phải dùng `AtomicInteger` hoặc `synchronized`.

`volatile` là nền tảng của: `ConcurrentHashMap`, `AtomicInteger`, `ThreadLocal`, Transaction Manager, Tomcat Thread Pool.

---

# TẦNG 3 — Cơ chế Java nâng cao

## 3.1 Reflection — Spring "nhìn thấy" code của bạn

Bình thường bạn phải biết trước class để dùng `new`. Reflection cho phép **hỏi 1 class về chính nó** mà không cần biết trước.

```java
Class<?> clazz = UserService.class;

clazz.getMethods();       // Có method nào?
clazz.getFields();        // Có field nào?
clazz.getAnnotations();   // Có annotation nào?
```

Spring khởi động → quét (scan) toàn bộ package → với mỗi class, hỏi bằng Reflection:

```java
clazz.getAnnotation(Service.class)
```

Nếu thấy `@Service` → Spring tự gọi tương đương `new UserService()` (thực chất qua `clazz.getDeclaredConstructor().newInstance()`) → **bean được sinh ra**.

**Không có Reflection thì Spring không tồn tại** — vì đây là cách Spring biết `@Service`, `@Component`, `@Repository`, `@RestController` là gì và tạo bean từ chúng.

## 3.2 Chain of Responsibility (nền cho Servlet Filter ở Tầng 5)

```java
public interface Filter {
    void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        // xử lý gì đó trước
        chain.doFilter(req, res);  // chuyển cho filter tiếp theo
        // xử lý gì đó sau
    }
}
```

Mỗi `Filter` là một trạm kiểm soát. Nó có quyền: xử lý request, rồi **tự quyết định** có gọi `chain.doFilter()` để chuyển tiếp hay không (nếu không gọi → chặn request, ví dụ trả về 401 Unauthorized).

Đây là pattern nền tảng của Java Servlet API — **có trước Spring**, không phải Spring phát minh ra.

---

# TẦNG 4 — Cơ chế lõi Spring

## 4.1 Bean và Singleton — vì sao dùng chung vẫn an toàn?

Spring mặc định biến mọi `@Service`, `@Component`... thành **singleton** — chỉ tạo 1 lần duy nhất trong `ApplicationContext`.

```
ApplicationContext
┌─────────────┐
│UserService  │  ← chỉ 1 instance
└─────────────┘
     ▲   ▲   ▲
     │   │   │
ControllerA  B  C   (tất cả dùng chung)
```

Khi 2 request tới cùng lúc, mỗi request chạy trên **1 thread riêng**, cùng gọi vào **1 object** UserService:

```java
public User findUser(Long id) {
    User user = repository.findById(id);  // biến LOCAL
    return user;
}
```

`user` không nằm trong object UserService — nó nằm trên **Stack frame riêng của từng thread** (nhắc lại Tầng 2):

```
Thread A → Stack A → user = User1
Thread B → Stack B → user = User2
```

Hoàn toàn độc lập, không đụng nhau, dù chạy chung 1 bean.

| | Dùng chung? |
|---|---|
| Bean (object) | Có |
| Biến local trong method | Không |

→ Đây là lý do Spring khuyến khích viết **Stateless Service** — không lưu trạng thái vào field.

⚠️ Nếu bạn viết field lưu state:
```java
@Service
public class UserService {
    private User currentUser; // field dùng chung → RACE CONDITION
    public void setCurrent(User u) { this.currentUser = u; }
}
```
Thread A và B cùng ghi vào `currentUser` → đè lẫn nhau → sai logic.

## 4.2 Proxy — trái tim của Spring

```java
@Transactional
public void transfer() { }
```

Bạn tưởng `userService.transfer()` gọi thẳng vào method. **Sai.** Spring đưa cho bạn một **Proxy** — object giả mạo bọc quanh object thật, tạo ra bằng Reflection (Tầng 3):

```
Caller
   │
   ▼
Proxy (TransactionProxy)
   │
   ├─ beginTransaction()
   ├─ target.transfer()   ← object THẬT
   └─ commitTransaction()
   │
   ▼
Target (UserService)
```

`@Transactional` **không nằm trong method** — nó nằm **trong Proxy**.

### Bug kinh điển: self-invocation

```java
public void methodA() {
    methodB();          // gọi TRỰC TIẾP this.methodB()
}

@Transactional
public void methodB() { }
```

`methodA()` gọi thẳng `this.methodB()` — **không đi qua Proxy bên ngoài**. Kết quả: `@Transactional` bị bỏ qua hoàn toàn dù annotation vẫn ghi rành rành.

### Nếu không đánh `@Transactional` thì sao?

Không có Proxy bọc transaction → method chạy trần, mỗi câu SQL bên trong tự **auto-commit** độc lập:

```
Bước 1: subtractBalance(A, 100) → chạy xong → auto-commit NGAY
Bước 2: addBalance(B, 100)      → lỗi (mất kết nối, exception...)
```

→ Tiền đã bị **trừ khỏi A và commit vĩnh viễn**, nhưng **không được cộng vào B**. Mất tính **Atomicity**.

Có `@Transactional` → Proxy bắt exception → tự động `rollback()` → hủy luôn bước 1 → tiền quay lại A như chưa có gì xảy ra.

Lưu ý thêm:
- SELECT thuần thường không sao nếu thiếu `@Transactional`, trừ lazy-loading JPA cần transaction đang mở.
- Đánh ở class-level → mọi method public trong class đều được bọc.
- Self-invocation vẫn là bẫy dù đánh đúng annotation.

⚠️ Thiếu `@Transactional` không gây lỗi biên dịch/runtime — nó âm thầm đổi hành vi, chỉ lộ ra khi dữ liệu đã sai trong production.

---

# TẦNG 5 — Ứng dụng thực tế

## 5.1 ThreadLocal — "ngăn kéo riêng" cho từng thread

**Câu hỏi:** Bean là singleton (chỉ 1 instance) — vậy transaction của từng request riêng biệt lưu ở đâu?

**Trả lời:** KHÔNG lưu trong field của Bean (sẽ đè nhau như 4.1). Spring dùng `ThreadLocal` — giống mỗi thread có 1 ngăn kéo riêng, khóa riêng:

```
Thread A → ngăn kéo riêng → Transaction A
Thread B → ngăn kéo riêng → Transaction B
Thread C → ngăn kéo riêng → Transaction C
```

Dù `UserService` vẫn chỉ có 1 instance duy nhất.

`SecurityContextHolder` (biết ai đang đăng nhập) cũng hoạt động y hệt cơ chế này — bên trong là `ThreadLocal<SecurityContext>`.

## 5.2 Servlet Filter chain & `SecurityFilterChain`

Dựa trên Chain of Responsibility (3.2): request đi qua **một chuỗi Filter** chạy **trước cả khi Controller được gọi tới** — trước cả Reflection tạo Controller, trước cả Proxy.

```
HTTP request
      │
      ▼
Tomcat gán 1 Thread riêng
      │
      ▼
Servlet Filter chain:
  ├─ CORS filter
  ├─ SecurityFilterChain (xác thực)
  ├─ Filter kiểm tra quyền
  └─ Nếu fail → chặn, trả lỗi ngay
      │  (nếu pass hết)
      ▼
Controller (tạo bằng Reflection)
      │
      ▼
Proxy (@Transactional, v.v.)
      │
      ▼
Service, ThreadLocal, ...
```

`SecurityFilterChain` chỉ là **danh sách các Filter bảo mật xếp nối tiếp nhau** (CORS, xác thực JWT, phân quyền...). Đây là lý do Spring Security chặn được request mà Controller **không hề biết gì** — vì Controller chưa từng được gọi tới.

## 5.3 Chiến lược học 1 class Spring mới (áp dụng cho mọi class, không riêng Security)

Có 2 tầng kiến thức khác nhau:

- **Tầng cơ chế** (Object/Reference, Stack/Heap, JMM, Reflection, Proxy, ThreadLocal, Filter chain) — ổn định qua các version, **phải học có hệ thống trước**.
- **Tầng API bề mặt** (tên class, tên method cụ thể như `SecurityFilterChain`, `HttpSecurity`, `.authorizeHttpRequests()`) — thay đổi liên tục theo version, **học theo nhu cầu là đúng chiến lược**, miễn là đã có tầng cơ chế.

Mỗi lần gặp 1 class lạ, hỏi 3 câu theo thứ tự:

1. **Nó implement pattern/interface gốc nào?** (VD: `SecurityFilterChain` → quay về Chain of Responsibility)
2. **Nó đứng ở đâu trong vòng đời request/bean?** (trước hay sau Proxy? trước hay sau Controller?)
3. **Nó là config hay logic thật?** (nhiều class chỉ là khai báo bean, Spring đọc rồi build object thật lúc khởi động)

Trả lời được 3 câu này thì phần còn lại (cú pháp builder cụ thể) chỉ cần tra tài liệu lúc cần, không cần nhớ.

---

# Sơ đồ tổng — luồng 1 request đi qua Spring (đầy đủ)

```
HTTP Request
      │
      ▼
Tomcat gán 1 Thread riêng                    ← Tầng 2 (Thread), Tầng 4 (Singleton)
      │
      ▼
Servlet Filter chain (CORS, Security, ...)   ← Tầng 3 (Chain of Responsibility), Tầng 5
      │  (nếu pass hết)
      ▼
Controller (đã tạo sẵn bằng Reflection)      ← Tầng 3 (Reflection)
      │
      ▼
Proxy — mở transaction (ghi ThreadLocal)     ← Tầng 4 (Proxy), Tầng 5 (ThreadLocal)
      │
      ▼
Service method (biến local trên Stack)       ← Tầng 1 (Reference), Tầng 2 (Stack/Heap)
      │
      ▼
Proxy — đọc lại ThreadLocal, commit          ← Tầng 4 + Tầng 5
      │
      ▼
HTTP Response trả về client
```

## Chuỗi kiến thức tóm gọn

```
Java Object → Reference → Dependency Injection
      ↓
Reflection → Bean Creation → Proxy
      ↓
@Transactional / Security / Caching / AOP
      ↓
Thread → ThreadLocal → Filter chain
      ↓
Transaction Context / Security Context
      ↓
JMM → Thread Safety
```

Hiểu hết chuỗi này → có thể tự giải thích gần như mọi cơ chế của Spring Security, Transaction, Async, Cache, Event, OAuth2 mà không cần học thuộc lòng annotation.
