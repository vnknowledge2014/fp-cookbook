# Chapter 10 — Opaque Types & Modules

> **Bạn sẽ học được**:
> - **Opaque types** — tạo domain types an toàn (`Email := Str`)
> - **Smart constructors** — validate data khi tạo, từ chối data sai
> - **Module system** — `exposes`, `imports`, tổ chức code cho dự án lớn
> - **Encapsulation** — ẩn implementation, chỉ expose API
> - Cách tổ chức dự án Roc nhiều files
>
> **Yêu cầu trước**: [Chapter 9 — Lists & Standard Functions](chapter_09_lists_and_standard_functions.md)
> **Thời gian đọc**: ~40 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Tổ chức code Roc thành modules, tạo domain types không thể sử dụng sai.

---

Bạn có bao giờ gọi sai thứ tự tham số trong một function nhận 4 chuỗi không? Compiler không báo lỗi — vì mọi tham số đều là `Str`. Bug chỉ xuất hiện khi code chạy trên production.

Opaque types giải quyết chuyện này. Thay vì `Str` cho mọi thứ, bạn tạo `Email`, `PhoneNumber`, `OrderId` — compiler phân biệt rõ ràng và bắt lỗi ngay lúc viết code. Module system cho phép bạn tách code thành nhiều file, mỗi file đóng gói một domain concept.

## 10.1 — Vấn đề: Str không đủ an toàn

Một bug kinh điển mà hầu hết developer đều từng gặp:

### Mọi thứ đều là Str?

```roc
# ❌ Thiết kế nguy hiểm — mọi thứ đều Str
sendEmail = \from, to, subject, body -> ...

main =
    # Compiler KHÔNG BẮT lỗi này!
    sendEmail "Chào bạn" "an@mail.com" "reply@me.com" "Nội dung"
    #          ↑ body      ↑ to          ↑ from?!       ↑ subject?!
    # Đổi nhầm thứ tự tham số → gửi mail sai → bug production
```

Compiler thấy 4 cái `Str` → hợp lệ. Nhưng logic **hoàn toàn sai**. Đây gọi là **"primitive obsession"** — dùng kiểu cơ bản cho mọi thứ.

### Giải pháp: Opaque types

```roc
# ✅ Mỗi domain concept có type riêng
Email := Str
Subject := Str
Body := Str

sendEmail : Email, Email, Subject, Body -> Task {} _
```

Bây giờ compiler **bắt lỗi** nếu bạn truyền `Body` vào chỗ cần `Email`. Dù bên trong đều là `Str`, **type khác nhau** → không thể nhầm.

---

## 10.2 — Tạo Opaque Types

Cú pháp `:=` tạo một type mới từ type có sẵn. Khác với type alias (chỉ là tên gọi tắt, Chapter 5), opaque type thật sự **khác biệt** — compiler coi `Email` và `Str` là hai kiểu hoàn toàn khác nhau.

### Cú pháp `:=`

```roc
# filename: Email.roc
module [Email, fromStr, toStr]

# Khai báo opaque type — Str bên trong, Email bên ngoài
Email := Str

# Smart constructor — validate trước khi tạo
fromStr : Str -> Result Email [InvalidEmail]
fromStr = \raw ->
    if Str.contains raw "@" && Str.contains raw "." then
        Ok (@Email raw)       # @Email = "bọc" Str thành Email
    else
        Err InvalidEmail

# Unwrap — lấy Str bên trong
toStr : Email -> Str
toStr = \@Email inner -> inner    # @Email pattern = "mở" Email ra
```

Giải thích:
- `Email := Str` — khai báo: Email **bọc** Str bên trong
- `@Email raw` — **tạo** Email từ Str (chỉ dùng được trong module này)
- `\@Email inner ->` — **mở** Email lấy Str bên trong

### Sử dụng

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import Email exposing [Email]

main =
    when Email.fromStr "an@mail.com" is
        Ok email ->
            Stdout.line! "Email hợp lệ: $(Email.toStr email)"
        Err InvalidEmail ->
            Stdout.line! "Email không hợp lệ!"

    when Email.fromStr "not-an-email" is
        Ok _ -> Stdout.line! "Hợp lệ"
        Err InvalidEmail -> Stdout.line! "❌ Từ chối: không có @"
    # Output:
    # Email hợp lệ: an@mail.com
    # ❌ Từ chối: không có @
```

> **💡 Điểm mấu chốt**: Bên ngoài module **KHÔNG THỂ** tạo `Email` trực tiếp. Phải qua `Email.fromStr` → validation **luôn chạy**. Không bao giờ có Email sai format trong hệ thống.

---

## 10.3 — Smart Constructors: Validate mọi thứ

Một opaque type tốt không chỉ phân biệt kiểu — nó còn **đảm bảo data hợp lệ**. Triết lý này gọi là "Parse, don't validate": thay vì kiểm tra data nhiều chỗ, bạn validate **một lần** lúc tạo. Sau đó, bất kỳ function nào nhận `Email` đều biết chắc nó hợp lệ — không cần kiểm tra lại.

### Pattern: Parse, don't validate

Thay vì kiểm tra data nhiều chỗ, **parse ngay lúc tạo** — sau đó tin tưởng type:

```roc
# filename: Money.roc
module [Money, fromCents, toDong, add, multiply]

# Tiền lưu bằng đồng (cent), không bao giờ âm
Money := U64

fromCents : U64 -> Money
fromCents = \amount -> @Money amount

# Smart constructor từ số thập phân
fromDong : I64 -> Result Money [NegativeAmount]
fromDong = \dong ->
    if dong < 0 then
        Err NegativeAmount
    else
        Ok (@Money (Num.toU64 dong))

toDong : Money -> U64
toDong = \@Money cents -> cents

# Phép cộng — luôn an toàn (U64 + U64 không bao giờ âm)
add : Money, Money -> Money
add = \@Money a, @Money b -> @Money (a + b)

# Nhân với số lượng
multiply : Money, U64 -> Money
multiply = \@Money amount, qty -> @Money (amount * qty)
```

```roc
# filename: main.roc
import Money exposing [Money]

main =
    price = Money.fromCents 45000
    quantity = 3
    total = Money.multiply price quantity

    Stdout.line! "Tổng: $(Num.toStr (Money.toDong total))đ"
    # → Tổng: 135000đ

    # ❌ Không thể tạo tiền âm
    when Money.fromDong (-5000) is
        Ok _ -> Stdout.line! "OK"
        Err NegativeAmount -> Stdout.line! "❌ Tiền không thể âm!"
```

### Thêm ví dụ: Username, Age, OrderId

```roc
# filename: Username.roc
module [Username, fromStr, toStr]

Username := Str

fromStr : Str -> Result Username [TooShort, TooLong, InvalidChars]
fromStr = \raw ->
    trimmed = Str.trim raw
    len = Str.countUtf8Bytes trimmed
    if len < 3 then
        Err TooShort
    else if len > 20 then
        Err TooLong
    else if Str.contains trimmed " " then
        Err InvalidChars
    else
        Ok (@Username trimmed)

toStr : Username -> Str
toStr = \@Username inner -> inner
```

```roc
# filename: Age.roc
module [Age, fromU8, toU8]

Age := U8

fromU8 : U8 -> Result Age [TooYoung, TooOld]
fromU8 = \years ->
    if years < 1 then Err TooYoung
    else if years > 150 then Err TooOld
    else Ok (@Age years)

toU8 : Age -> U8
toU8 = \@Age inner -> inner
```

> **💡 Pattern**: Smart constructor **luôn trả `Result`**. Nếu data hợp lệ → `Ok value`. Sai → `Err reason` với tag mô tả lý do cụ thể. Compiler bắt buộc caller xử lý lỗi.

---

## ✅ Checkpoint 10.1–10.3

> 1. **Opaque type** `Name := Str` — bọc type cơ bản, tạo type mới
> 2. **`@Name`** — chỉ dùng **trong module** để tạo/mở opaque type
> 3. **Smart constructor** — validate rồi mới tạo, trả `Result`
> 4. **"Parse, don't validate"** — data đã qua constructor = luôn hợp lệ

---

## 10.4 — Module System

Khi dự án lớn hơn 200 dòng, bạn cần tách code. Roc dùng module system đơn giản: mỗi file `.roc` là một module, khai báo rõ những gì **expose** ra ngoài. Mọi thứ không trong danh sách expose — tự động private.

### Tạo module

```roc
# filename: Greeting.roc

# Dòng đầu tiên: module + expose list
module [greet, farewell]

# Chỉ 2 functions này visible bên ngoài
greet : Str -> Str
greet = \name -> "Xin chào $(name)!"

farewell : Str -> Str
farewell = \name -> "Tạm biệt $(name)!"

# Function nội bộ — KHÔNG expose → private
formatWithEmoji = \emoji, msg -> "$(emoji) $(msg)"
```

### Import module

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import Greeting                        # import module

main =
    Stdout.line! (Greeting.greet "An")      # Qualified: Module.function
    Stdout.line! (Greeting.farewell "An")

    # Greeting.formatWithEmoji → ❌ error! private function
```

### Import với `exposing`

```roc
import Greeting exposing [greet]       # lấy thẳng tên greet

main =
    Stdout.line! (greet "An")              # Không cần prefix Greeting.
    Stdout.line! (Greeting.farewell "An")  # Vẫn cần prefix cho các hàm khác
```

---

## 10.5 — Tổ chức dự án thực tế

Lý thuyết đủ rồi. Giờ hãy xem một dự án nhỏ hoàn chỉnh — hệ thống quán cà phê với nhiều modules tương tác với nhau. Mỗi module có một trách nhiệm duy nhất.

### Cấu trúc thư mục

```
my-cafe-app/
├── main.roc                 # Entry point
├── Menu.roc                 # Module: menu items
├── Order.roc                # Module: order management
├── Payment.roc              # Module: payment processing
└── Validation.roc           # Module: shared validation
```

### Ví dụ hoàn chỉnh: Mini Café System

**Menu.roc:**

```roc
# filename: Menu.roc
module [MenuItem, DrinkSize, create, price, describe]

DrinkSize : [S, M, L]

MenuItem := { name : Str, basePrice : U64, size : DrinkSize }

create : Str, U64, DrinkSize -> MenuItem
create = \name, basePrice, size ->
    @MenuItem { name, basePrice, size }

price : MenuItem -> U64
price = \@MenuItem item ->
    multiplier = when item.size is
        S -> 100
        M -> 130
        L -> 160
    item.basePrice * multiplier // 100

describe : MenuItem -> Str
describe = \@MenuItem item ->
    sizeLabel = when item.size is
        S -> "nhỏ"
        M -> "vừa"
        L -> "lớn"
    "$(item.name) ($(sizeLabel))"
```

**Order.roc:**

```roc
# filename: Order.roc
module [Order, OrderId, create, addItem, total, itemCount, describe]

import Menu exposing [MenuItem]

OrderId := U32

Order := { id : OrderId, items : List MenuItem, status : [New, Preparing, Ready, Delivered] }

create : U32 -> Order
create = \rawId ->
    @Order { id: @OrderId rawId, items: [], status: New }

addItem : Order, MenuItem -> Order
addItem = \@Order order, item ->
    @Order { order & items: List.append order.items item }

total : Order -> U64
total = \@Order order ->
    List.walk order.items 0 \sum, item -> sum + Menu.price item

itemCount : Order -> U64
itemCount = \@Order order ->
    Num.toU64 (List.len order.items)

describe : Order -> Str
describe = \@Order order ->
    idNum = when order.id is
        @OrderId n -> n
    statusStr = when order.status is
        New -> "📝 Mới"
        Preparing -> "☕ Đang pha"
        Ready -> "✅ Sẵn sàng"
        Delivered -> "🎉 Đã giao"
    "Đơn #$(Num.toStr idNum) | $(Num.toStr (itemCount (@Order order))) món | $(Num.toStr (total (@Order order)))đ | $(statusStr)"
```

**main.roc:**

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import Menu
import Order

main =
    # Tạo menu items
    coffee = Menu.create "Cà phê" 25000 M
    tea = Menu.create "Trà sen" 20000 L
    smoothie = Menu.create "Sinh tố bơ" 35000 M

    # Tạo đơn hàng và thêm items
    order =
        Order.create 1
        |> Order.addItem coffee
        |> Order.addItem tea
        |> Order.addItem smoothie

    # In thông tin
    Stdout.line! "=== Đơn hàng ==="
    Stdout.line! (Order.describe order)
    Stdout.line! ""
    Stdout.line! "Chi tiết:"
    Stdout.line! "  $(Menu.describe coffee): $(Num.toStr (Menu.price coffee))đ"
    Stdout.line! "  $(Menu.describe tea): $(Num.toStr (Menu.price tea))đ"
    Stdout.line! "  $(Menu.describe smoothie): $(Num.toStr (Menu.price smoothie))đ"
    Stdout.line! "---"
    Stdout.line! "Tổng: $(Num.toStr (Order.total order))đ"
    # Output:
    # === Đơn hàng ===
    # Đơn #1 | 3 món | 87500đ | 📝 Mới
    #
    # Chi tiết:
    #   Cà phê (vừa): 32500đ
    #   Trà sen (lớn): 32000đ
    #   Sinh tố bơ (vừa): 45500đ  (35000 × 130 / 100)
```

Nhìn lại kiến trúc:
- **`Menu`** expose `MenuItem` (opaque) + `create`, `price`, `describe`
- **`Order`** import `Menu`, expose `Order` (opaque) + API
- **`main.roc`** chỉ dùng public API — không thể tạo `MenuItem` hay `Order` sai

---

## 10.6 — Best Practices

Sau đây là 4 quy tắc mà bạn nên tuân theo khi thiết kế opaque types và modules. Chúng không bắt buộc — nhưng dự án nào tuân theo sẽ dễ đọc, dễ sửa, và ít bug hơn.

### 1. Expose types, ẩn implementation

```roc
# ✅ Expose type name + API functions
module [Email, fromStr, toStr, domain]

# ❌ KHÔNG expose := operator → user không tự tạo được
```

### 2. Mỗi domain concept = 1 opaque type

```roc
# ❌ Nguy hiểm — dễ nhầm
processOrder : Str, Str, U64, Str -> ...
#              orderId  email  amount  address

# ✅ An toàn — compiler bắt lỗi
processOrder : OrderId, Email, Money, Address -> ...
```

### 3. Module = 1 file, 1 trách nhiệm

```
# ✅ Tốt — mỗi file 1 concept
Email.roc       → Email type + validation
Money.roc       → Money type + arithmetic
Order.roc       → Order type + business logic
Validation.roc  → Shared validation helpers

# ❌ Tệ — 1 file làm mọi thứ
Utils.roc       → Email, Money, Order, Validation... 🤯
```

### 4. API surface nhỏ nhất có thể

```roc
# ✅ Chỉ expose những gì user cần
module [Email, fromStr, toStr]

# ❌ Expose quá nhiều → coupling cao
module [Email, fromStr, toStr, isValid, rawBytes, internalParse, ...]
```

---

## 🏋️ Bài tập

**Bài 1** (10 phút): PhoneNumber opaque type

Tạo module `PhoneNumber.roc`:
- Chỉ chấp nhận số bắt đầu bằng "0", dài 10 ký tự
- Expose: `PhoneNumber`, `fromStr`, `toStr`, `format` (hiển thị dạng `0xxx-xxx-xxx`)

<details><summary>✅ Lời giải</summary>

```roc
# filename: PhoneNumber.roc
module [PhoneNumber, fromStr, toStr, format]

PhoneNumber := Str

fromStr : Str -> Result PhoneNumber [InvalidLength, MustStartWithZero]
fromStr = \raw ->
    trimmed = Str.trim raw
    if Str.countUtf8Bytes trimmed != 10 then
        Err InvalidLength
    else if !(Str.startsWith trimmed "0") then
        Err MustStartWithZero
    else
        Ok (@PhoneNumber trimmed)

toStr : PhoneNumber -> Str
toStr = \@PhoneNumber inner -> inner

format : PhoneNumber -> Str
format = \@PhoneNumber raw ->
    bytes = Str.toUtf8 raw
    part1 = List.takeFirst bytes 4 |> Str.fromUtf8 |> Result.withDefault ""
    part2 = List.sublist bytes { start: 4, len: 3 } |> Str.fromUtf8 |> Result.withDefault ""
    part3 = List.takeLast bytes 3 |> Str.fromUtf8 |> Result.withDefault ""
    "$(part1)-$(part2)-$(part3)"
```

</details>

---

**Bài 2** (15 phút): Inventory module

Tạo hệ thống quản lý kho với 2 modules:

```
Product.roc — ProductId, Product, create, describe
Inventory.roc — Inventory, empty, addStock, removeStock, checkStock
```

Yêu cầu:
- `removeStock` trả `Err OutOfStock` nếu không đủ
- `checkStock` trả số lượng của 1 sản phẩm

<details><summary>✅ Lời giải</summary>

```roc
# filename: Product.roc
module [ProductId, Product, create, describe, id]

ProductId := U32

Product := { id : ProductId, name : Str, price : U64 }

create : U32, Str, U64 -> Product
create = \rawId, name, price ->
    @Product { id: @ProductId rawId, name, price }

id : Product -> ProductId
id = \@Product p -> p.id

describe : Product -> Str
describe = \@Product p -> "$(p.name) ($(Num.toStr p.price)đ)"

# filename: Inventory.roc
module [Inventory, empty, addStock, removeStock, checkStock]

import Product exposing [ProductId]

Inventory := Dict U32 U64

empty : Inventory
empty = @Inventory (Dict.empty {})

addStock : Inventory, ProductId, U64 -> Inventory
addStock = \@Inventory inv, @ProductId pid, qty ->
    current = Dict.get inv pid |> Result.withDefault 0
    @Inventory (Dict.insert inv pid (current + qty))

removeStock : Inventory, ProductId, U64 -> Result Inventory [OutOfStock]
removeStock = \@Inventory inv, @ProductId pid, qty ->
    current = Dict.get inv pid |> Result.withDefault 0
    if current < qty then
        Err OutOfStock
    else
        Ok (@Inventory (Dict.insert inv pid (current - qty)))

checkStock : Inventory, ProductId -> U64
checkStock = \@Inventory inv, @ProductId pid ->
    Dict.get inv pid |> Result.withDefault 0
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `@Email is not available here` | Dùng `@Email` ngoài module định nghĩa | Chỉ dùng `@` trong module có `:=`. Dùng `Email.fromStr` bên ngoài |
| `Module not found` | File name không khớp module name | File `Email.roc` phải khai báo `module [...]` |
| `This value is not exposed` | Function/type không trong expose list | Thêm vào `module [...]` |
| `These types are not the same` | Email ≠ Str dù bên trong là Str | Dùng `Email.toStr` để lấy Str. Đây là tính năng, không phải bug! |

---

## Tóm tắt

- ✅ **Opaque types** `Name := Str` — bọc type cơ bản, tạo domain type an toàn. `@Name` chỉ dùng trong module.
- ✅ **Smart constructors** — validate khi tạo, trả `Result`. Data qua constructor = luôn hợp lệ.
- ✅ **"Parse, don't validate"** — validate 1 lần ở boundary, tin tưởng types bên trong.
- ✅ **Module** = 1 file, `module [exposed items]`. Import bằng `import ModuleName`.
- ✅ **Encapsulation** — ẩn `:=`, chỉ expose public API. User không tạo/sửa trực tiếp opaque type.
- ✅ **Best practices**: mỗi concept 1 opaque type, API surface nhỏ, 1 module 1 trách nhiệm.

## 🎉 Kết thúc Part I — Roc Fundamentals!

Bạn đã nắm vững mọi feature cơ bản của Roc:

| Chapter | Kiến thức |
|---------|-----------|
| 4 | Setup, platform model, `roc run` |
| 5 | Values, types, structural typing |
| 6 | Functions, currying, pipeline `\|>` |
| 7 | Records, tag unions, domain modeling |
| 8 | Pattern matching, exhaustive, guards |
| 9 | List/Dict/Set API, data processing |
| **10** | **Opaque types, modules, encapsulation** |

## Tiếp theo

→ **Part II: Thinking Functionally** — Chapter 11: **Purity by Default** — tại sao mọi function Roc đều pure, side effects chỉ qua Tasks, và cách tư duy "pure core, effectful shell".
