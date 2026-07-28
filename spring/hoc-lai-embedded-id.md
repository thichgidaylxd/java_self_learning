# Tự học: `@EmbeddedId` — khoá chính kết hợp (Composite Primary Key)

> Viết cho người chưa biết gì. Áp dụng trực tiếp vào 2 bảng thật của
> bạn: `tree_memberships` (PK là cặp `tree_id` + `user_id`) và
> `person_albums` (PK là cặp `person_id` + `album_id`).

---

## 1. Vấn đề cần giải quyết — vì sao không dùng `id: UUID` như mọi bảng khác

Nhìn lại `BaseEntity` đã học — mọi entity thường có `id: UUID` là khoá
chính **1 cột duy nhất**. Nhưng với `tree_memberships`, thiết kế DB đã
chốt:

```sql
CREATE TABLE tree_memberships (
  tree_id   UUID NOT NULL REFERENCES trees(id),
  user_id   UUID NOT NULL REFERENCES users(id),
  role      VARCHAR(10) NOT NULL,
  ...
  PRIMARY KEY (tree_id, user_id)   -- KHÔNG có cột "id" riêng
);
```

**Khoá chính là CẶP 2 cột** (`tree_id`, `user_id`) — không phải 1 cột
UUID sinh tự động. Lý do thiết kế: 1 user chỉ nên có **đúng 1 dòng**
membership cho mỗi tree — ràng buộc này diễn tả tự nhiên nhất bằng
composite key, không cần thêm `UNIQUE(tree_id, user_id)` phụ trên 1
cột `id` UUID vô nghĩa.

**Vấn đề cần giải quyết:** JPA/Hibernate quen làm việc với **1 field
`@Id` duy nhất** trên entity. Làm sao khai báo 1 entity có khoá chính
là **2 field gộp lại**?

## 2. Giải pháp — tách riêng 1 class chỉ chứa các cột làm khoá

JPA yêu cầu: muốn khoá chính gồm nhiều cột, phải **gom các cột đó vào
1 class riêng** (gọi là "khoá nhúng" — embedded key), rồi dùng class
đó làm `@Id` cho entity chính.

### Bước 1 — Tạo class `@Embeddable` chứa đúng các cột làm khoá

```java
@Embeddable
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
public class TreeMembershipId implements Serializable {

    @Column(name = "tree_id")
    private UUID treeId;

    @Column(name = "user_id")
    private UUID userId;
}
```

**`@Embeddable`** — đánh dấu: *"class này không phải 1 entity độc lập
(không có bảng riêng), chỉ là 1 'cụm field' để nhúng vào entity khác"*.
Nghe quen không? — đúng tinh thần gần giống `@MappedSuperclass` đã học
(cũng là "khuôn field dùng chung, không có bảng riêng"), nhưng
`@Embeddable` dùng cho mục đích khác: **gom nhóm nhiều cột thành 1
"khối" có ý nghĩa**, ở đây là để làm khoá chính.

### Bước 2 — Dùng class đó làm `@EmbeddedId` trong entity chính

```java
@Entity
@Table(name = "tree_memberships")
@Getter
@Setter
public class TreeMembership {

    @EmbeddedId
    private TreeMembershipId id;

    @MapsId("treeId")                          // (*) giải thích bên dưới
    @ManyToOne
    @JoinColumn(name = "tree_id")
    private Tree tree;

    @MapsId("userId")                          // (*) giải thích bên dưới
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @Column(nullable = false)
    private String role;

    @Column(name = "invited_by")
    private UUID invitedBy;

    @Column(name = "joined_at", nullable = false)
    private LocalDateTime joinedAt;
}
```

## 3. `@MapsId` — điểm dễ rối nhất, giải thích kỹ

Đây là chỗ hay gây bối rối: **tại sao vừa có `treeId: UUID` trong
`TreeMembershipId`, vừa có `tree: Tree` (quan hệ `@ManyToOne`) trong
`TreeMembership`** — không phải trùng lặp dữ liệu sao?

**Không trùng lặp — đây là 2 cách nhìn vào CÙNG 1 cột `tree_id` trong
DB:**
- `TreeMembershipId.treeId` — nhìn `tree_id` như 1 **giá trị UUID
  thuần**, dùng để **xác định danh tính** (identity) của dòng dữ liệu
  này (dùng cho khoá chính, so sánh, hashCode...)
- `TreeMembership.tree` — nhìn **chính cột `tree_id` đó** như 1
  **quan hệ đối tượng**, để bạn viết code kiểu `membership.getTree().getName()`
  thay vì phải tự query `Tree` riêng bằng UUID

`@MapsId("treeId")` nói với Hibernate: *"quan hệ `@ManyToOne` này
KHÔNG tạo thêm cột `tree_id` mới — nó dùng LẠI đúng giá trị đã có trong
`id.treeId`"*. Không có `@MapsId`, Hibernate sẽ hiểu nhầm bạn muốn **2
cột khác nhau** cùng tên `tree_id`, gây lỗi hoặc trùng cột khi tạo
bảng.

## 4. Vì sao BẮT BUỘC override `equals()`/`hashCode()` trên class Embeddable

Đây là **quy tắc bắt buộc**, không phải tuỳ chọn — nếu quên, code vẫn
biên dịch được nhưng sẽ có **bug ẩn rất khó phát hiện**.

**Lý do kỹ thuật:** Hibernate (và cả Java `HashMap`/`HashSet` nói
chung) cần biết **"2 khoá chính này có phải cùng 1 dòng dữ liệu
không"** bằng cách so sánh `equals()`. Với `@Id` là 1 UUID đơn, Java
so sánh 2 UUID rất đơn giản, tự nhiên đúng. Nhưng với khoá **kết hợp
nhiều field**, Java **mặc định** so sánh theo địa chỉ bộ nhớ (`==`,
kế thừa từ `Object.equals()`) — 2 object `TreeMembershipId` có
**cùng giá trị `treeId`/`userId`** nhưng là **2 instance khác nhau**
trong bộ nhớ sẽ bị coi là **"khác nhau"**, dù về mặt dữ liệu chúng đại
diện **cùng 1 dòng trong DB**.

```java
// KHÔNG override equals/hashCode -> lỗi ẩn
TreeMembershipId id1 = new TreeMembershipId(treeUuid, userUuid);
TreeMembershipId id2 = new TreeMembershipId(treeUuid, userUuid);
id1.equals(id2);  // false! (dù treeId, userId giống hệt nhau)

// Hậu quả thực tế: Hibernate dùng equals/hashCode để quản lý cache
// (Persistence Context) - 2 khoá "trông giống nhau" nhưng bị coi khác
// nhau -> Hibernate có thể tạo 2 bản ghi trong bộ nhớ cho CÙNG 1 dòng
// DB, dẫn tới hành vi khó lường khi save/update.
```

`@EqualsAndHashCode` (Lombok, đã có trong code mẫu ở Bước 1) tự sinh
`equals()`/`hashCode()` dựa trên **giá trị tất cả field**, giải quyết
đúng vấn đề này — đây là lý do annotation này **không được bỏ qua**
với bất kỳ `@Embeddable` nào dùng làm khoá chính.

## 5. Vì sao cần `implements Serializable`

Đây là **yêu cầu bắt buộc của chuẩn JPA** (không phải Hibernate tự đặt
ra) cho bất kỳ class nào dùng làm khoá chính (dù đơn hay kết hợp). Lý
do lịch sử: khoá chính có thể cần **serialize** (chuyển thành dạng lưu
trữ/truyền tải được) trong 1 số cơ chế cache/clustering của JPA
provider — dù thực tế nhiều ứng dụng nhỏ không bao giờ dùng tới tính
năng này, chuẩn JPA vẫn yêu cầu khai báo `Serializable` để đảm bảo
tương thích.

## 6. Vì sao cần `@NoArgsConstructor` (constructor rỗng)

**Hibernate luôn cần tạo object bằng reflection** trước khi gán dữ
liệu vào — tức là gọi `new TreeMembershipId()` (không tham số) rồi mới
set từng field sau, không gọi trực tiếp constructor có tham số bạn tự
định nghĩa. Thiếu constructor rỗng, Hibernate **không có cách nào tạo
object** khi đọc dữ liệu từ DB lên, sẽ ném exception lúc query.

*(Đây cũng chính là lý do mọi `@Entity` thông thường — kể cả
`Person`, `Tree` — đều cần có constructor rỗng, dù bạn có thể không để
ý vì Lombok/JPA thường tự đủ điều kiện; với `@Embeddable` thì quy tắc
này lộ rõ hơn vì bạn hay tự viết `@AllArgsConstructor` đi kèm.)*

## 7. Dùng ở tầng Repository — Generic Type là gì

```java
public interface TreeMembershipRepository
        extends JpaRepository<TreeMembership, TreeMembershipId> {
}
```

`JpaRepository<Entity, IdType>` — tham số thứ 2 luôn là **kiểu dữ liệu
của khoá chính**. Với `Person` (khoá đơn), bạn viết
`JpaRepository<Person, UUID>`. Với `TreeMembership` (khoá kết hợp),
tham số thứ 2 chính là **class Embeddable** vừa tạo, không phải `UUID`.

**Cách gọi tìm theo khoá:**
```java
TreeMembershipId id = new TreeMembershipId(treeId, userId);
Optional<TreeMembership> membership = treeMembershipRepository.findById(id);
```

## 8. Áp dụng cho `person_albums` — bảng thứ 2 cùng pattern

```sql
CREATE TABLE person_albums (
  person_id   UUID NOT NULL REFERENCES persons(id),
  album_id    UUID NOT NULL REFERENCES albums(id),
  created_by  UUID REFERENCES users(id),
  ...
  PRIMARY KEY (person_id, album_id)
);
```

Hoàn toàn tương tự — chỉ đổi tên:

```java
@Embeddable
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @EqualsAndHashCode
public class PersonAlbumId implements Serializable {
    @Column(name = "person_id") private UUID personId;
    @Column(name = "album_id")  private UUID albumId;
}

@Entity
@Table(name = "person_albums")
@Getter @Setter
public class PersonAlbum {

    @EmbeddedId
    private PersonAlbumId id;

    @MapsId("personId")
    @ManyToOne
    @JoinColumn(name = "person_id")
    private Person person;

    @MapsId("albumId")
    @ManyToOne
    @JoinColumn(name = "album_id")
    private Album album;

    @Column(name = "created_by")
    private UUID createdBy;
    // ... các field audit khác
}
```

## 9. Cách khác — `@IdClass` (biết để không nhầm, không khuyến nghị dùng)

JPA còn 1 cách nữa để làm composite key, dùng `@IdClass` thay vì
`@EmbeddedId`:

```java
@Entity
@IdClass(TreeMembershipId.class)     // class riêng, KHÔNG @Embeddable
public class TreeMembership {
    @Id private UUID treeId;         // field khoá nằm THẲNG trong entity
    @Id private UUID userId;         // không gom vào 1 object con
    // ...
}
```

**Khác biệt cốt lõi với `@EmbeddedId`:** `@IdClass` giữ các field khoá
**rải trực tiếp** trong entity (không gom vào 1 object `id` như
`@EmbeddedId`), class `TreeMembershipId` chỉ đóng vai trò "khuôn mẫu"
để JPA biết cấu trúc khoá, bản thân nó **không xuất hiện** như 1 field
thật trong entity.

**Khuyến nghị dùng `@EmbeddedId`** (như các bước trên) vì:
- Rõ ràng hơn — nhìn code thấy ngay `membership.getId().getTreeId()`,
  biết chắc đây là 1 phần của khoá chính
- Dễ tái sử dụng class khoá đó ở nơi khác (ví dụ truyền
  `TreeMembershipId` làm tham số method mà không cần cả `TreeMembership`)

`@IdClass` chỉ có lợi thế nhỏ: field khoá nằm phẳng, code gọi
`membership.getTreeId()` trực tiếp (không cần qua `.getId().getTreeId()`)
— nhưng đánh đổi sự rõ ràng, nên các dự án mới thường chọn `@EmbeddedId`.

## 10. Bẫy thường gặp — lỗi hay gặp khi mới học

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `IdentifierGenerationException` hoặc lỗi tương tự lúc save | Quên `@NoArgsConstructor` trên class `@Embeddable` | Thêm constructor rỗng |
| Dữ liệu load lên bị trùng/query trả sai kết quả khi dùng trong `Set`/`Map` | Quên `equals()`/`hashCode()` | Thêm `@EqualsAndHashCode` (Lombok) hoặc tự viết |
| Lỗi "Repeated column in mapping" khi tạo bảng | Khai cả `@Column(name="tree_id")` trong `TreeMembershipId` LẪN 1 cột riêng khác cũng map `tree_id` mà quên `@MapsId` | Dùng đúng `@MapsId("treeId")` trên quan hệ `@ManyToOne`, không khai `@JoinColumn` riêng không có `@MapsId` |
| Không tạo được `TreeMembership` mới bằng code | Set `id` (`TreeMembershipId`) nhưng quên set luôn `tree`/`user` (hoặc ngược lại) | Với `@MapsId`, chỉ cần set 1 trong 2 phía (thường set qua `tree`/`user`, Hibernate tự đồng bộ `id.treeId`) — nhưng an toàn nhất là set đủ cả 2 lúc code, tránh phụ thuộc hành vi ngầm |

---

## Tóm tắt bằng 1 câu

- **`@Embeddable`** — "class này chỉ là cụm field, không có bảng riêng"
- **`@EmbeddedId`** — "dùng 1 object Embeddable làm khoá chính, thay vì 1 field đơn"
- **`@MapsId("fieldName")`** — "quan hệ `@ManyToOne` này dùng LẠI đúng cột đã có trong id, không tạo cột mới"
- **`equals()`/`hashCode()`** — bắt buộc, vì Java mặc định so sánh theo địa chỉ bộ nhớ, không theo giá trị field

## Câu hỏi tự kiểm tra

1. Vì sao `tree_memberships` không có cột `id: UUID` riêng như các
   bảng khác?
2. `TreeMembershipId.treeId` và `TreeMembership.tree` — 2 field này có
   trùng lặp dữ liệu trong DB không? Vì sao?
3. Nếu quên override `equals()`/`hashCode()` trên `@Embeddable` class,
   hậu quả cụ thể là gì?
4. `@EmbeddedId` khác `@IdClass` ở điểm nào? Vì sao dự án này nên chọn
   `@EmbeddedId`?
