# Chapter 14 — Abilities

> **Bạn sẽ học được**:
> - **Abilities** — Roc's answer to traits/interfaces/type classes
> - Built-in abilities: `Eq`, `Hash`, `Inspect`
> - Tại sao Roc **không có** Functor, Monad, hay abstract type classes
> - Custom abilities — tạo interface cho opaque types
> - Deriving abilities tự động vs implement thủ công
> - Khi nào dùng abilities, khi nào dùng tag unions
>
> **Yêu cầu trước**: [Chapter 13 — Error Handling with Result](chapter_13_error_handling.md)
> **Thời gian đọc**: ~40 phút | **Level**: Intermediate
> **Kết quả cuối cùng**: Hiểu khi nào cần abilities và cách tạo API polymorphic — đơn giản, practical, không academic.

---

Haskell có hơn 200 type classes. Rust có hàng chục traits. Roc có bao nhiêu? **Ba built-in** (`Eq`, `Hash`, `Inspect`) và khả năng tạo custom abilities. Không có Functor, không có Monad, không có Applicative. Đây là thiết kế có chủ đích — Roc chọn đơn giản hơn là đầy đủ.

## 14.1 — Vấn đề: Polymorphism mà không cần Class Hierarchy

Bạn muốn viết một function `contains` kiểm tra phần tử có trong list không. Function này cần dùng `==` — nhưng `==` hoạt động với kiểu nào? Kiểu nào "biết" cách so sánh?

### Bạn muốn viết hàm generic

```roc
# Hàm này CẦN so sánh 2 giá trị
# Nhưng giá trị có thể là Num, Str, record, tag union...
# Làm sao viết 1 hàm cho TẤT CẢ?

contains = \list, target ->
    List.any list \item -> item == target
    # ← "==" hoạt động với MỌI kiểu???
```

Trong Java: bạn cần `Comparable` interface. Trong Rust: bạn cần `PartialEq` trait. Trong Haskell: bạn cần `Eq` type class.

Trong Roc: bạn cần **ability** `Eq`.

### Abilities = practical traits

Roc **cố tình** chỉ có vài abilities — không có abstract Functor/Monad/Applicative. Triết lý: **thực tế hơn lý thuyết**.

---

## 14.2 — Built-in Abilities

Roc có 3 built-in abilities mà hầu hết types tự động implement. Bạn hiếm khi cần nghĩ về chúng — trừ khi dùng opaque types.

### `Eq` — So sánh bằng

`Eq` cho phép dùng `==` và `!=`:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Numbers — tự động có Eq
    Stdout.line! "5 == 5? $(Inspect.toStr (5 == 5))"           # Bool.true

    # Strings — tự động có Eq
    Stdout.line! "abc == abc? $(Inspect.toStr ("abc" == "abc"))" # Bool.true

    # Lists — so sánh từng phần tử
    Stdout.line! "[1,2] == [1,2]? $(Inspect.toStr ([1, 2] == [1, 2]))"  # Bool.true
    Stdout.line! "[1,2] == [1,3]? $(Inspect.toStr ([1, 2] == [1, 3]))"  # Bool.false

    # Records — so sánh từng field
    a = { name: "An", age: 25 }
    b = { name: "An", age: 25 }
    c = { name: "Bình", age: 30 }
    Stdout.line! "a == b? $(Inspect.toStr (a == b))"   # Bool.true
    Stdout.line! "a == c? $(Inspect.toStr (a == c))"   # Bool.false

    # Tags — so sánh tag name + payload
    Stdout.line! "Red == Red? $(Inspect.toStr (Red == Red))"   # Bool.true
    Stdout.line! "Red == Blue? $(Inspect.toStr (Red == Blue))" # Bool.false
```

Hầu hết types trong Roc **tự động có `Eq`** — bạn không cần viết gì.

### `Hash` — Hashing

`Hash` cho phép type dùng trong `Dict` và `Set` (làm key):

```roc
# Dict dùng Hash cho keys
prices = Dict.fromList [("Phở", 45000)]   # Str implements Hash → OK

# Set dùng Hash cho elements
colors = Set.fromList [Red, Green, Blue]   # Tags implement Hash → OK
```

### `Inspect` — Debug / Display

`Inspect` cho phép `Inspect.toStr` — chuyển bất kỳ giá trị nào thành chuỗi debug:

```roc
main =
    value = { name: "An", scores: [85, 92, 78], status: Active }
    Stdout.line! (Inspect.toStr value)
    # → { name: "An", scores: [85, 92, 78], status: Active }
```

### Bảng built-in abilities

| Ability | Cho phép | Tự động? |
|---------|---------|----------|
| `Eq` | `==`, `!=` | ✅ Hầu hết types |
| `Hash` | Dict key, Set element | ✅ Hầu hết types |
| `Inspect` | `Inspect.toStr` | ✅ Hầu hết types |

---

## 14.3 — Abilities cho Opaque Types

Opaque types (Chapter 10) mặc định **không có** ability nào. Muốn so sánh hai Email? Phải nói rõ `Email := Str implements [Eq]`. Đây là thiết kế có chủ đích — bạn chọn chính xác abilities nào cần.

### Vấn đề: Opaque types KHÔNG tự động có abilities

```roc
# filename: Email.roc
module [Email, fromStr, toStr]

Email := Str

fromStr = \raw -> if Str.contains raw "@" then Ok (@Email raw) else Err InvalidEmail
toStr = \@Email inner -> inner
```

```roc
# ❌ Lỗi! Email không có Eq
email1 = Email.fromStr "a@b.com" |> Result.withDefault ???
email2 = Email.fromStr "a@b.com" |> Result.withDefault ???
email1 == email2    # compile error: Email doesn't implement Eq!
```

### Giải pháp: `implements`

```roc
# filename: Email.roc
module [Email, fromStr, toStr]

# Đánh dấu: Email implements Eq, Hash, Inspect
Email := Str implements [Eq, Hash, Inspect]

fromStr = \raw -> if Str.contains raw "@" then Ok (@Email raw) else Err InvalidEmail
toStr = \@Email inner -> inner
```

Bây giờ:

```roc
email1 == email2         # ✅ OK — dùng Eq của Str bên trong
dict = Dict.single email1 "user"   # ✅ OK — dùng Hash
Stdout.line! (Inspect.toStr email1) # ✅ OK — hiển thị debug
```

`implements [Eq, Hash, Inspect]` nói: "derive abilities này từ type bên trong (Str)". Roc tự dùng `==` của Str cho `==` của Email.

---

## 14.4 — Custom Implementations

Đôi khi bạn muốn **so sánh khác** mặc định:

```roc
# filename: CaseInsensitiveStr.roc
module [CaseInsensitiveStr, fromStr, toStr]

# So sánh case-insensitive
CaseInsensitiveStr := Str implements [Eq { isEq: ciEqual }, Hash, Inspect]

ciEqual : CaseInsensitiveStr, CaseInsensitiveStr -> Bool
ciEqual = \@CaseInsensitiveStr a, @CaseInsensitiveStr b ->
    # So sánh sau khi lowercase cả 2
    toLower a == toLower b

toLower : Str -> Str
toLower = \s ->
    Str.toUtf8 s
    |> List.map \byte ->
        if byte >= 65 && byte <= 90 then byte + 32 else byte
    |> Str.fromUtf8
    |> Result.withDefault s

fromStr : Str -> CaseInsensitiveStr
fromStr = \s -> @CaseInsensitiveStr s

toStr : CaseInsensitiveStr -> Str
toStr = \@CaseInsensitiveStr inner -> inner
```

```roc
# Sử dụng
a = CaseInsensitiveStr.fromStr "Hello"
b = CaseInsensitiveStr.fromStr "HELLO"
a == b    # → Bool.true! (vì custom Eq)
```

---

## 14.5 — Custom Abilities

Ngoài 3 built-in, bạn có thể tạo abilities riêng — giống interface trong Java hay trait trong Rust. Dùng khi nhiều types cần cùng "hành vi" nhưng implementation khác nhau.

### Tạo ability của riêng bạn

```roc
# filename: Describable.roc
module [Describable, describe]

# Khai báo ability
Describable implements
    describe : val -> Str where val implements Describable
```

```roc
# filename: Product.roc
module [Product, create]

import Describable

Product := { name : Str, price : U64 } implements [
    Describable { describe: describeProduct },
    Eq,
    Inspect,
]

describeProduct : Product -> Str
describeProduct = \@Product p ->
    "📦 $(p.name) — $(Num.toStr p.price)đ"

create : Str, U64 -> Product
create = \name, price -> @Product { name, price }
```

```roc
# filename: User.roc
module [User, create]

import Describable

User := { name : Str, role : [Admin, Member] } implements [
    Describable { describe: describeUser },
    Eq,
    Inspect,
]

describeUser : User -> Str
describeUser = \@User u ->
    roleStr = when u.role is
        Admin -> "👑 Admin"
        Member -> "👤 Member"
    "$(roleStr): $(u.name)"

create : Str, [Admin, Member] -> User
create = \name, role -> @User { name, role }
```

```roc
# Sử dụng — polymorphic!
main =
    # items : List Str (vì Describable.describe trả Str)
    items = [
        Describable.describe (Product.create "iPhone" 25000000),
        Describable.describe (User.create "An" Admin),
    ]
    List.forEach items \desc -> Stdout.line! desc
    # 📦 iPhone — 25000000đ
    # 👑 Admin: An
```

---

## 14.6 — Abilities vs Tag Unions: Khi nào dùng gì?

Câu hỏi thường gặp: khi nào dùng tag union (Chapter 7), khi nào dùng ability? Câu trả lời ngắn: tag union khi bạn biết hết các variants, ability khi bạn muốn cho phép mở rộng.

### Tag unions — "closed set" (tập đóng)

```roc
# Bạn biết TẤT CẢ variants từ đầu
Shape : [Circle F64, Rectangle F64 F64, Triangle F64 F64]

area : Shape -> F64
area = \shape ->
    when shape is
        Circle r -> 3.14 * r * r
        Rectangle w h -> w * h
        Triangle b h -> 0.5 * b * h
```

### Abilities — "open set" (tập mở)

```roc
# Bất kỳ ai cũng có thể THÊM type mới implements Describable
# Không cần sửa code cũ
```

### Bảng so sánh

| | Tag Unions | Abilities |
|---|---|---|
| Khi nào | Biết hết variants | Cho phép mở rộng |
| Thêm variant mới | Sửa code cũ (thêm branch) | Không sửa code cũ |
| Thêm function mới | Viết function mới, ko sửa gì | Có thể phải sửa mỗi implementation |
| Ví dụ | `Shape`, `Payment`, `Status` | `Describable`, `Serializable` |
| Phổ biến hơn trong Roc? | **Rất phổ biến** | Ít dùng hơn |

> **💡 Quy tắc**: Trong Roc, **ưu tiên tag unions**. Chỉ dùng abilities khi bạn **thật sự cần** extensibility — cho phép code khác thêm types mới mà không sửa code bạn.

---

## 14.7 — Roc KHÔNG có Functor/Monad

Nếu bạn đến từ Haskell hay Scala, bạn sẽ ngạc nhiên: Roc không có Functor, Monad, hay Applicative. Thay vì một `map` generic cho mọi container, Roc có `List.map`, `Result.map`, `Task.map` riêng biệt. Đơn giản hơn, explicit hơn.

### Tại sao?

Roc **cố tình** không có:
- `Functor` (generic map)
- `Monad` (generic bind/flatMap)
- `Applicative`
- Abstract type classes

**Lý do**: Chúng thêm complexity mà **hầu hết lập trình viên không cần**.

```roc
# Haskell: generic map cho mọi Functor
fmap :: Functor f => (a -> b) -> f a -> f b

# Roc: cứ viết cụ thể cho từng type
List.map : List a, (a -> b) -> List b
Result.map : Result a err, (a -> b) -> Result b err
Task.map : Task a err, (a -> b) -> Task b err
```

Ba functions `map` này **đều có tên `map`** nhưng ở 3 modules khác nhau. Bạn không cần hiểu Functor theory — chỉ cần biết `List.map`, `Result.map`, `Task.map`.

> **💡 Triết lý Roc**: "Simple made easy." Đủ tốt > hoàn hảo về mặt lý thuyết.

---


## ✅ Checkpoint 14

> Đến đây bạn phải hiểu:
> 1. Abilities = built-in typeclasses: `Eq`, `Hash`, `Inspect`, `Encode`, `Decode`
> 2. Không có user-defined abilities — Roc chọn simplicity over flexibility
> 3. Opaque types opt-in abilities: `Email := Str implements [Eq, Hash]`
>
> **Test nhanh**: `Email := Str` — mặc định có `Eq` ability không?
> <details><summary>Đáp án</summary>Không! Opaque types mặc định không có ability nào. Phải ghi rõ `implements [Eq]`.</details>

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Thiếu ability nào?

Cho code sau, đoán lỗi compile:

```roc
Money := U64

main =
    a = @Money 100
    b = @Money 100
    
    Stdout.line! (Inspect.toStr (a == b))    # ???
```

<details><summary>✅ Lời giải</summary>

Lỗi: `Money` không implement `Eq` (và `Inspect`). Sửa:

```roc
Money := U64 implements [Eq, Inspect]
```

</details>

---

**Bài 2** (10 phút): Opaque types với abilities

Tạo `Temperature` opaque type:
- Lưu bằng Celsius (F64) bên trong
- Implement `Eq`, `Inspect`
- Smart constructor `fromCelsius`, `fromFahrenheit`
- Function `toCelsius`, `toFahrenheit`

<details><summary>✅ Lời giải</summary>

```roc
# filename: Temperature.roc
module [Temperature, fromCelsius, fromFahrenheit, toCelsius, toFahrenheit]

Temperature := F64 implements [Eq, Inspect]

fromCelsius : F64 -> Temperature
fromCelsius = \c -> @Temperature c

fromFahrenheit : F64 -> Temperature
fromFahrenheit = \f -> @Temperature ((f - 32.0) * 5.0 / 9.0)

toCelsius : Temperature -> F64
toCelsius = \@Temperature c -> c

toFahrenheit : Temperature -> F64
toFahrenheit = \@Temperature c -> c * 9.0 / 5.0 + 32.0
```

```roc
# Test
boiling = Temperature.fromCelsius 100.0
alsoBoiling = Temperature.fromFahrenheit 212.0
# boiling == alsoBoiling   # Bool.true (cả 2 = 100°C bên trong)
```

</details>

---

**Bài 3** (15 phút): Khi nào Tag Union, khi nào Ability?

Cho các bài toán, chọn approach:

```
a) Hệ thống thanh toán: Cash, Card, EWallet
b) Plugin system: cho phép user thêm plugin mới
c) HTTP status codes: 200, 404, 500...
d) Serialization: nhiều types cần serialize thành JSON
e) State machine: New → Processing → Done
```

<details><summary>✅ Lời giải</summary>

```
a) Tag Union — biết hết variants, thêm payment method = sửa code
b) Ability — user thêm plugin mà không sửa code core
c) Tag Union — tập cố định, exhaustive matching
d) Ability — nhiều types implement Serializable
e) Tag Union — trạng thái rõ ràng, exhaustive
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `Type doesn't implement Eq` | Opaque type chưa derive Eq | Thêm `implements [Eq]` |
| `Can't use as Dict key` | Type chưa implement Hash | Thêm `Hash` vào implements list |
| `Inspect.toStr` không hoạt động | Type chưa implement Inspect | Thêm `Inspect` |
| Custom Eq logic sai | Implement thủ công không symmetrical | Đảm bảo `a == b` ↔ `b == a` |

---

## Tóm tắt

- ✅ **Abilities** = Roc's traits/interfaces. Cho phép polymorphism — 1 function làm việc với nhiều types.
- ✅ **Built-in**: `Eq` (==), `Hash` (Dict/Set key), `Inspect` (debug display). Tự động cho hầu hết types.
- ✅ **Opaque types** cần `implements [Eq, Hash, Inspect]` — không tự động derive.
- ✅ **Custom implementation** — `Eq { isEq: myCustomEq }` cho logic so sánh riêng.
- ✅ **Custom abilities** — tạo interface mới, các types implement riêng.
- ✅ **Tag unions vs Abilities**: Tag unions = closed set (phổ biến hơn). Abilities = open set (khi cần extensibility).
- ✅ **Không Functor/Monad** — Roc chọn đơn giản: `List.map`, `Result.map`, `Task.map` riêng biệt.

## 🎉 Kết thúc Part II — Thinking Functionally!

| Chapter | Kiến thức |
|---------|-----------|
| 11 | Purity, side effects, pure core / effectful shell |
| 12 | Tasks, `!`, file IO, stdin, CLI app |
| 13 | Result, `?`, railway programming, error hierarchy |
| **14** | **Abilities, Eq/Hash/Inspect, custom abilities, tag unions vs abilities** |

## Tiếp theo

→ **Part III: Design Patterns** — Chapter 15: **FP Patterns** — Strategy, Builder, Interpreter qua functions. Không cần classes, không cần design pattern books — FP patterns tự nhiên hơn bạn tưởng.
