# Chapter 7 — Records & Tag Unions ⭐

> **Bạn sẽ học được**:
> - **Records** — nhóm data liên quan, structural typing, record update syntax
> - **Tag Unions** — "hoặc A hoặc B" với data đi kèm
> - **Open vs Closed types** — tại sao Roc linh hoạt hơn enum/struct truyền thống
> - **Record update syntax** — tạo bản mới từ bản cũ
> - **Kết hợp Records + Tags** — thiết kế domain models thực tế
>
> **Yêu cầu trước**: [Chapter 6 — Functions & Pipelines](chapter_06_functions_and_pipelines.md)
> **Thời gian đọc**: ~45 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Thiết kế domain model cho bất kỳ bài toán nào bằng records + tag unions.

---

Chapter này có dấu ⭐ vì một lý do: records và tag unions là **hai công cụ thiết kế quan trọng nhất** trong Roc. Nếu functions (Chapter 6) là cách bạn biến đổi dữ liệu, thì records và tags là cách bạn **tổ chức** dữ liệu.

Record cho phép bạn góp nhiều giá trị lại thành một đơn vị có nghĩa — thay vì truyền riêng từng `name`, `age`, `email`, bạn gói chúng thành một `user`. Tag union cho phép bạn nói "giá trị này là A hoặc B hoặc C" — và compiler đảm bảo bạn xử lý mọi trường hợp.

## 7.1 — Records: Nhóm data liên quan

Thay vì 5 biến rời rạc `name`, `age`, `email`, `phone`, `address` — nhóm chúng vào một record. Rõ ràng hơn, khó sai hơn.

### Tạo record

Record là cách đơn giản nhất để góp nhiều giá trị liên quan lại. Giống `struct` trong Rust/Go, `object` trong JS — nhưng khác ở chỗ: **không cần khai báo trước**. Bạn cứ viết ra cấu trúc và compiler tự hiểu.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Tạo record — cứ viết, compiler tự hiểu
    student = {
        name: "An",
        age: 16,
        grade: 10,
        school: "THPT Nguyễn Huệ",
    }

    # Truy cập field bằng dấu chấm
    Stdout.line! "$(student.name), lớp $(Num.toStr student.grade)"
    Stdout.line! "Trường: $(student.school)"
    # Output:
    # An, lớp 10
    # Trường: THPT Nguyễn Huệ
```

### Records là immutable

Nếu bạn quen với JavaScript, bạn có thể viết `student.grade = 11` để sửa giá trị. Trong Roc, điều này không tồn tại. Records sau khi tạo không thể thay đổi — bạn chỉ có thể **tạo bản mới** từ bản cũ. Nghe bất tiện, nhưng thực tế đây là lý do code FP ít bug hơn: không ai có thể sửa dữ liệu của bạn từ một chỗ khác mà bạn không biết.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    original = { name: "An", age: 16, score: 85 }

    # Record update syntax — tạo bản mới, chỉ đổi field cần thiết
    updated = { original & score: 95 }

    # original KHÔNG BỊ THAY ĐỔI
    Stdout.line! "Cũ: $(original.name), $(Num.toStr original.score) điểm"
    Stdout.line! "Mới: $(updated.name), $(Num.toStr updated.score) điểm"
    # Output:
    # Cũ: An, 85 điểm
    # Mới: An, 95 điểm

    # Sửa nhiều fields cùng lúc
    graduated = { original & age: 18, score: 92 }
    Stdout.line! "Tốt nghiệp: $(Num.toStr graduated.age) tuổi, $(Num.toStr graduated.score) điểm"
```

`{ original & score: 95 }` đọc là: "tạo record giống `original`, nhưng `score` = 95".

> **💡 Hiệu suất**: Nhờ reference counting (Chapter 3), nếu `original` không ai dùng nữa, Roc **sửa trực tiếp** thay vì copy → nhanh!

### Destructuring records

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Destructuring trong tham số function
introduce = \{ name, age } ->
    "$(name), $(Num.toStr age) tuổi"

# Hoặc truy cập trực tiếp qua parameter
greet = \person ->
    "Chào $(person.name)!"

main =
    user = { name: "An", age: 25, role: "Developer" }

    Stdout.line! (introduce user)   # "An, 25 tuổi"
    Stdout.line! (greet user)       # "Chào An!"
```

Với destructuring `\{ name, age }`, bạn "mở" record ra — lấy chính xác fields cần dùng.

---

## 7.2 — Open Records: Linh hoạt tuyệt đối

Roc có một khái niệm mà nhiều ngôn ngữ khác không có: **open records**. Một function có thể nói "tôi cần record có field `name`, nhưng nó có thêm bao nhiêu field khác cũng được". Đây là sự khác biệt giữa **closed** (chính xác các fields này) và **open** (có ít nhất các fields này).

### Open vs Closed

```roc
# Closed record — chính xác các fields này, không hơn không kém
exactPerson : { name : Str, age : U32 }

# Open record — CÓ ÍT NHẤT các fields này (có thể có thêm)
# Compiler suy ra khi bạn không ghi type
greet = \person -> "Chào $(person.name)!"
# Type: { name : Str }a -> Str
# Chữ 'a' = "có thể có thêm fields khác"
```

Open record là lý do `greet` hoạt động với **bất kỳ record nào có field `name`**:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Function chỉ cần field "name" — open record
greet = \person -> "Chào $(person.name)!"

# Function cần "name" VÀ "email" — vẫn open
sendWelcome = \user ->
    "Gửi email tới $(user.email): Chào mừng $(user.name)!"

main =
    student = { name: "An", age: 16, grade: 10 }
    employee = { name: "Bình", email: "binh@company.com", department: "IT" }
    visitor = { name: "Cường" }

    # greet nhận bất kỳ record nào có "name"
    Stdout.line! (greet student)     # ✅
    Stdout.line! (greet employee)    # ✅
    Stdout.line! (greet visitor)     # ✅

    # sendWelcome cần cả "name" VÀ "email"
    Stdout.line! (sendWelcome employee)   # ✅

    # sendWelcome visitor → ❌ Compiler error: "visitor" không có field "email"
```

> **💡 Ẩn dụ**: Open record giống **đơn xin việc** — "cần có CMND và bằng lái". Bạn có thêm bằng đại học, hộ chiếu? Vẫn OK, miễn có 2 thứ kia.

---

## ✅ Checkpoint 7.1–7.2

> 1. **Record** = nhóm data theo tên field. Không cần khai báo trước.
> 2. **Record update** = `{ old & field: newValue }` — tạo bản mới, không sửa bản cũ.
> 3. **Destructuring** = `\{ name, age } -> ...` — mở record lấy fields.
> 4. **Open record** = "có ít nhất fields này" — linh hoạt, tái sử dụng cao.

---

## 7.3 — Tag Unions: Chọn một trong nhiều

Nếu record là "type AND" (có name VÀ age VÀ email), thì tag union là "type OR" (là Coffee HOẶC Tea HOẶC Smoothie). Trong Rust gọi là `enum`, trong TypeScript là discriminated union. Roc gọi đơn giản là tag union — và không cần khai báo trước.

### Tag cơ bản

Một tag là giá trị có tên, viết HOA chữ cái đầu. Bạn không khai báo ở đâu cả — cứ viết và compiler tự hiểu:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Tag union — không cần khai báo! Cứ viết.
describeLight = \light ->
    when light is
        Red -> "🔴 Dừng lại!"
        Yellow -> "🟡 Chuẩn bị..."
        Green -> "🟢 Đi!"

main =
    Stdout.line! (describeLight Red)
    Stdout.line! (describeLight Yellow)
    Stdout.line! (describeLight Green)
    # Output:
    # 🔴 Dừng lại!
    # 🟡 Chuẩn bị...
    # 🟢 Đi!
```

Compiler suy ra kiểu: `describeLight : [Red, Yellow, Green] -> Str`. Bạn không cần khai báo `enum TrafficLight` ở đâu cả — compiler thấy bạn xử lý 3 tags và tự hiểu type.

### Tags mang data (Payload)

Một tag không chỉ là nhãn — nó có thể **chứa data bên trong**. Giống như `enum` trong Rust có thể chứa giá trị (`Some(42)`, `Err("fail")`), mỗi tag trong Roc có thể mang data khác nhau:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

describeShape = \shape ->
    when shape is
        Circle radius ->
            area = 3.14159 * radius * radius
            "⭕ Hình tròn r=$(Num.toStr radius), S=$(Num.toStr area)"
        Rectangle width height ->
            area = width * height
            "▬ HCN $(Num.toStr width)×$(Num.toStr height), S=$(Num.toStr area)"
        Triangle base height ->
            area = base * height / 2.0
            "△ Tam giác đáy=$(Num.toStr base), cao=$(Num.toStr height), S=$(Num.toStr area)"

main =
    shapes = [
        Circle 5.0,
        Rectangle 4.0 6.0,
        Triangle 3.0 8.0,
    ]

    List.forEach shapes \shape ->
        Stdout.line! (describeShape shape)
    # Output:
    # ⭕ Hình tròn r=5, S=78.53975
    # ▬ HCN 4×6, S=24
    # △ Tam giác đáy=3, cao=8, S=12
```

Mỗi tag có thể mang **data khác nhau**:
- `Circle` mang 1 số (bán kính)
- `Rectangle` mang 2 số (rộng, cao)
- `Triangle` mang 2 số (đáy, cao)

### Tags mang Records

Khi tag cần nhiều fields có tên, dùng record:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

describePayment = \payment ->
    when payment is
        Cash amount ->
            "💵 Tiền mặt: $(Num.toStr amount)đ"
        Card { number, bank } ->
            last4 = Str.toUtf8 number |> List.takeLast 4 |> Str.fromUtf8 |> Result.withDefault "????"
            "💳 Thẻ $(bank): ****$(last4)"
        EWallet { provider, phone } ->
            "📱 $(provider): $(phone)"

main =
    payments = [
        Cash 50000,
        Card { number: "4111111111111234", bank: "VCB" },
        EWallet { provider: "MoMo", phone: "0901234567" },
    ]

    List.forEach payments \p ->
        Stdout.line! (describePayment p)
    # Output:
    # 💵 Tiền mặt: 50000đ
    # 💳 Thẻ VCB: ****1234
    # 📱 MoMo: 0901234567
```

---

## 7.4 — Open Tag Unions

Giống open records, tag unions cũng có thể **mở** — nhận nhiều tags hơn bạn xử lý:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Closed — xử lý chính xác 3 tags
describeColor : [Red, Green, Blue] -> Str
describeColor = \color ->
    when color is
        Red -> "Đỏ"
        Green -> "Xanh lá"
        Blue -> "Xanh dương"

# Open — xử lý ít nhất Red và Green, có thể có thêm
isWarm : [Red, Yellow]a -> Bool
isWarm = \color ->
    when color is
        Red -> Bool.true
        Yellow -> Bool.true
        _ -> Bool.false        # catch-all cho các tags khác

main =
    Stdout.line! (describeColor Red)              # ✅
    Stdout.line! (Inspect.toStr (isWarm Red))     # ✅ Bool.true
    Stdout.line! (Inspect.toStr (isWarm Blue))    # ✅ Bool.false (catch-all)
```

- **Closed** `[Red, Green, Blue]` — compiler kiểm tra bạn xử lý **tất cả** tags
- **Open** `[Red, Yellow]a` — có thể nhận thêm tags, cần `_` catch-all

> **💡 Quy tắc**: Hầu hết thời gian, bạn **không viết type annotation** cho tags — compiler tự chọn. Chỉ ghi khi cần.

---

## 7.5 — Records + Tags = Domain Modeling

Đây là nơi mọi thứ kết hợp lại. Trong thực tế, bạn không dùng records riêng, tags riêng — bạn kết hợp chúng để **mô hình hóa bài toán**. Record nói "một đơn hàng có đồ uống VÀ size VÀ topping". Tag union nói "đồ uống là Coffee HOẶC Tea".

### Ví dụ: Hệ thống quán café

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Tag unions cho từng thuộc tính
# DrinkType = Coffee HOẶC Tea HOẶC Smoothie
# Size = S HOẶC M HOẶC L
# Topping = NoTopping HOẶC BubblePearl HOẶC Jelly

# Record kết hợp tất cả
# Order = { drink, size, topping, status }

# Status flow: New → Preparing → Ready → PickedUp

describeOrder = \order ->
    drink = when order.drink is
        Coffee -> "Cà phê"
        Tea -> "Trà"
        Smoothie -> "Sinh tố"

    size = when order.size is
        S -> "nhỏ"
        M -> "vừa"
        L -> "lớn"

    topping = when order.topping is
        NoTopping -> ""
        BubblePearl -> " + trân châu"
        Jelly -> " + thạch"

    icon = when order.status is
        New -> "📝"
        Preparing -> "☕"
        Ready -> "✅"
        PickedUp -> "🎉"

    "$(icon) $(drink) size $(size)$(topping)"

advanceOrder = \order ->
    newStatus = when order.status is
        New -> Preparing
        Preparing -> Ready
        Ready -> PickedUp
        PickedUp -> PickedUp
    { order & status: newStatus }

main =
    order = {
        drink: Coffee,
        size: M,
        topping: BubblePearl,
        # Hoặc: topping: NoTopping, nếu không muốn topping
        status: New,
    }

    Stdout.line! (describeOrder order)

    order2 = advanceOrder order
    Stdout.line! (describeOrder order2)

    order3 = advanceOrder order2
    Stdout.line! (describeOrder order3)

    order4 = advanceOrder order3
    Stdout.line! (describeOrder order4)
    # Output:
    # 📝 Cà phê size vừa + trân châu
    # ☕ Cà phê size vừa + trân châu
    # ✅ Cà phê size vừa + trân châu
    # 🎉 Cà phê size vừa + trân châu
```

Các con số đáng chú ý: 3 loại đồ uống × 3 size × 3 topping × 4 trạng thái = **108 tổ hợp**. Tất cả đều hợp lệ — không có trạng thái " impossible" nào lọt qua compiler. Đây chính là ý tưởng "make illegal states unrepresentable" mà bạn đã gặp ở Chapter 1.

### Ví dụ: User authentication

Tag union cũng rất tự nhiên để mô hình hóa **state machine** — các trạng thái của một đối tượng, mỗi trạng thái mang data khác nhau:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# State machine hoàn chỉnh bằng tag union
# Mỗi trạng thái mang data khác nhau!
greetUser = \user ->
    when user is
        Anonymous ->
            "👤 Xin chào khách!"
        LoggedIn { name, role } ->
            prefix = when role is
                Admin -> "👑 Admin"
                Member -> "😊 Member"
            "$(prefix) $(name)"
        Banned { name, reason } ->
            "🚫 $(name) — bị cấm vì: $(reason)"

main =
    users = [
        Anonymous,
        LoggedIn { name: "An", role: Admin },
        LoggedIn { name: "Bình", role: Member },
        Banned { name: "Troll", reason: "Spam" },
    ]

    List.forEach users \user ->
        Stdout.line! (greetUser user)
    # Output:
    # 👤 Xin chào khách!
    # 👑 Admin An
    # 😊 Member Bình
    # 🚫 Troll — bị cấm vì: Spam
```

**Không thể tạo trạng thái vô nghĩa:**
- Không thể **Anonymous** mà có `name`
- Không thể **LoggedIn** mà có `reason` (lý do bị cấm)
- Không thể **Banned** mà có `role` (admin/member)

Đây chính là **"make illegal states unrepresentable"** từ Chapter 1!

---

## 7.6 — Record Patterns nâng cao

Roc có nhiều shorthand cho records — tiết kiệm code mà vẫn đọc rõ. Pattern phổ biến trong codebases lớn.

### Shorthand khi field = variable

```roc
# Khi tên biến trùng tên field, viết tắt:
name = "An"
age = 25

# Dài:
person = { name: name, age: age }

# Ngắn (shorthand):
person = { name, age }
```

### Nested destructuring

```roc
processOrder = \{ customer: { name }, items, total } ->
    count = List.len items
    "$(name): $(Num.toStr count) món, tổng $(Num.toStr total)đ"

main =
    order = {
        customer: { name: "An", phone: "0901234567" },
        items: ["Phở", "Cà phê"],
        total: 70000,
    }
    Stdout.line! (processOrder order)
    # → "An: 2 món, tổng 70000đ"
```

### Optional fields với record update pattern

```roc
# "Config mặc định" pattern — rất phổ biến
defaultConfig = {
    host: "localhost",
    port: 8080,
    debug: Bool.false,
    maxConnections: 100,
}

# User chỉ override những gì cần
devConfig = { defaultConfig & debug: Bool.true }
prodConfig = { defaultConfig & host: "api.example.com", port: 443 }
```

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Record cơ bản

Tạo record cho sách (`title`, `author`, `pages`, `isRead`) và viết function `bookSummary` trả mô tả.

<details><summary>✅ Lời giải</summary>

```roc
bookSummary = \book ->
    status = if book.isRead then "đã đọc ✅" else "chưa đọc 📖"
    "\"$(book.title)\" — $(book.author) ($(Num.toStr book.pages) trang, $(status))"

main =
    book = { title: "Clean Code", author: "Uncle Bob", pages: 431, isRead: Bool.true }
    Stdout.line! (bookSummary book)
    # → "Clean Code" — Uncle Bob (431 trang, đã đọc ✅)
```

</details>

---

**Bài 2** (10 phút): State machine

Thiết kế hệ thống đặt hàng delivery với tag union:

```
Trạng thái: Placed → Confirmed → Picked → Delivered → Rated { stars : U8 }
```

Viết `describeStatus` và `advance` functions.

<details><summary>💡 Gợi ý</summary>

- `advance` cho `Delivered` cần thêm thông tin stars → trả `Rated { stars: 5 }` mặc định
- `advance` cho `Rated` → giữ nguyên (đã hoàn tất)

</details>

<details><summary>✅ Lời giải</summary>

```roc
describeStatus = \status ->
    when status is
        Placed -> "📦 Đã đặt"
        Confirmed -> "✅ Đã xác nhận"
        Picked -> "🏍️ Đang giao"
        Delivered -> "📬 Đã giao"
        Rated { stars } -> "⭐ Đánh giá: $(Num.toStr stars)/5"

advance = \status ->
    when status is
        Placed -> Confirmed
        Confirmed -> Picked
        Picked -> Delivered
        Delivered -> Rated { stars: 5 }
        Rated _ -> status   # giữ nguyên

main =
    flow =
        Placed
        |> advance          # Confirmed
        |> advance          # Picked
        |> advance          # Delivered
        |> advance          # Rated { stars: 5 }

    Stdout.line! (describeStatus flow)
    # → ⭐ Đánh giá: 5/5
```

</details>

---

**Bài 3** (15 phút): E-commerce product catalog

Thiết kế model cho sản phẩm e-commerce:

```
Product:
  - name : Str
  - price : Dec
  - category : Electronics | Clothing { size : [S, M, L, XL] } | Book { isbn : Str }
  - availability : InStock U32 | OutOfStock | PreOrder { releaseDate : Str }
```

Viết `formatProduct` và `isAvailable` functions.

<details><summary>✅ Lời giải</summary>

```roc
formatProduct = \product ->
    cat = when product.category is
        Electronics -> "🔌 Điện tử"
        Clothing { size } ->
            sizeStr = when size is
                S -> "S"
                M -> "M"
                L -> "L"
                XL -> "XL"
            "👕 Quần áo ($(sizeStr))"
        Book { isbn } -> "📚 Sách ($(isbn))"

    stock = when product.availability is
        InStock qty -> "Còn $(Num.toStr qty)"
        OutOfStock -> "Hết hàng"
        PreOrder { releaseDate } -> "Pre-order $(releaseDate)"

    "$(product.name) | $(cat) | $(Num.toStr product.price)đ | $(stock)"

isAvailable = \product ->
    when product.availability is
        InStock qty -> qty > 0
        OutOfStock -> Bool.false
        PreOrder _ -> Bool.true

main =
    products = [
        { name: "iPhone 15", price: 25000000, category: Electronics, availability: InStock 50 },
        { name: "Áo thun", price: 150000, category: Clothing { size: M }, availability: OutOfStock },
        { name: "Clean Code", price: 350000, category: Book { isbn: "978-0132350884" }, availability: PreOrder { releaseDate: "2024-06-01" } },
    ]

    List.forEach products \p ->
        available = if isAvailable p then " ✅" else " ❌"
        Stdout.line! "$(formatProduct p)$(available)"
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `I can't find field .x on this record` | Record không có field `x` | Kiểm tra chính tả, hoa thường |
| `This record update doesn't change any fields` | Dùng `{ r & }` mà không đổi gì | Thêm ít nhất 1 field cần đổi |
| `These tags don't match` | Function trả tags khác type ở `when` branches | Mọi branch phải trả cùng type |
| `This doesn't cover all patterns` | Quên 1 tag trong `when..is` | Thêm branch bị thiếu hoặc dùng `_` catch-all |
| Tag tên giống nhau nhưng data khác | Structural typing — tag `Ok 1` ≠ `Ok "hi"` | Kiểm tra data type bên trong tag |

---

## Tóm tắt

- ✅ **Records** = Product type (AND) — nhóm data theo tên field. Không cần khai báo trước.
- ✅ **Record update** = `{ old & field: newValue }` — tạo bản mới, bản cũ không đổi.
- ✅ **Destructuring** = `\{ name, age } -> ...` — mở record lấy fields cần dùng.
- ✅ **Tag unions** = Sum type (OR) — chọn 1 trong nhiều variants. Mỗi tag có thể chứa data khác nhau.
- ✅ **Open types** — records: "có ít nhất fields này". Tags: "có ít nhất tags này".
- ✅ **Records + Tags** = công cụ thiết kế domain model. Kết hợp để **make illegal states unrepresentable**.
- ✅ **Structural typing** — không cần khai báo struct/enum. Cứ viết — compiler hiểu.

## Tiếp theo

→ Chapter 8: **Pattern Matching** — deep dive vào `when...is`, exhaustive matching, nested patterns, guards, và tại sao compiler bắt bạn xử lý mọi trường hợp.
