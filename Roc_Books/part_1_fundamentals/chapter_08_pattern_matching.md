# Chapter 8 — Pattern Matching

> **Bạn sẽ học được**:
> - `when...is` — cú pháp pattern matching đầy đủ
> - **Exhaustive matching** — tại sao compiler bắt bạn xử lý mọi trường hợp
> - Destructuring records và tags trong patterns
> - **Guards** — thêm điều kiện `if` trong pattern
> - Nested patterns và wildcard `_`
> - Pattern matching trên Lists
>
> **Yêu cầu trước**: [Chapter 7 — Records & Tag Unions](chapter_07_records_and_tag_unions.md)
> **Thời gian đọc**: ~40 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Dùng pattern matching thay `if/else` — code rõ ràng, compiler giúp bạn không quên trường hợp nào.

---

Pattern matching là công cụ bạn sẽ dùng nhiều nhất trong Roc — nhiều hơn cả `if/else`. Với `if/else`, bạn kiểm tra điều kiện. Với pattern matching, bạn kiểm tra **cấu trúc dữ liệu**: giá trị này là `Ok` hay `Err`? Tag này là `Circle` hay `Rectangle`? List này rỗng hay có phần tử?

Điểm mấu chốt: compiler **bắt buộc** bạn xử lý mọi trường hợp. Quên một case? Không compile được. Thêm một tag mới vào union? Compiler chỉ đúng những chỗ cần cập nhật. Đây là lý do FP có ít bug hơn trong các hệ thống lớn.

## 8.1 — `when...is` cơ bản

`when...is` thay thế chuỗi `if/else if/else` dài — gọn hơn, an toàn hơn (compiler kiểm tra bạn đã xử lý hết mọi trường hợp).

### Thay thế if/else dài

```roc
# ❌ if/else dài dòng, dễ quên trường hợp
grade = \score ->
    if score >= 90 then "A"
    else if score >= 80 then "B"
    else if score >= 70 then "C"
    else if score >= 60 then "D"
    else "F"

# ✅ Pattern matching — rõ ràng hơn (nhưng ở đây if/else cũng OK)
```

Sức mạnh thực sự của pattern matching không phải thay `if/else` — mà là **match trên cấu trúc dữ liệu**:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

describe = \value ->
    when value is
        Ok result -> "✅ Thành công: $(result)"
        Err NotFound -> "❌ Không tìm thấy"
        Err Unauthorized -> "🔒 Không có quyền"
        Err (ServerError code) -> "💥 Lỗi server: $(Num.toStr code)"

main =
    Stdout.line! (describe (Ok "Dữ liệu người dùng"))
    Stdout.line! (describe (Err NotFound))
    Stdout.line! (describe (Err Unauthorized))
    Stdout.line! (describe (Err (ServerError 500)))
    # Output:
    # ✅ Thành công: Dữ liệu người dùng
    # ❌ Không tìm thấy
    # 🔒 Không có quyền
    # 💥 Lỗi server: 500
```

Mỗi `->` gọi là một **branch** (nhánh). `when...is` kiểm tra từ trên xuống, chạy branch đầu tiên khớp.

---

## 8.2 — Exhaustive Matching: Compiler canh gác

Exhaustive matching — không phải cú pháp gọn — mới là giá trị thật của pattern matching. Compiler **kiểm tra bạn đã xử lý hết mọi trường hợp chưa**. Trong dự án thực tế, đây là thứ cứu bạn khỏi những bug khó tìm nhất.

### Compiler bắt lỗi "quên trường hợp"

```roc
# ❌ Compiler error — quên xử lý Yellow!
describeLight = \light ->
    when light is
        Red -> "Dừng"
        Green -> "Đi"
        # Yellow → ???

# Roc compiler nói:
# "This `when` does not cover all the possibilities.
#  Missing: Yellow"
```

Compiler đảm bảo bạn **không bao giờ quên** một trường hợp. Thêm variant mới vào tag union? Compiler chỉ chính xác những chỗ cần cập nhật.

### Wildcard `_` — catch-all

Nếu **cố ý** muốn bỏ qua một số trường hợp:

```roc
isUrgent = \priority ->
    when priority is
        Critical -> Bool.true
        High -> Bool.true
        _ -> Bool.false        # Medium, Low, Info... đều false
```

`_` = "bất kỳ giá trị nào khác". Nhưng **hãy cẩn thận** — nếu sau này thêm tag `Emergency`, nó sẽ rơi vào `_` thay vì báo lỗi.

> **💡 Quy tắc**: Ưu tiên **liệt kê hết** thay vì dùng `_`. Chỉ dùng `_` khi danh sách tag quá dài hoặc bạn chắc chắn logic catch-all là đúng.

---

## 8.3 — Destructuring trong Patterns

Pattern matching không chỉ kiểm tra giá trị là gì — nó đồng thời **"mở hộp" lấy data bên trong ra**. Khi bạn viết `Circle radius ->`, Roc vừa kiểm tra đây có phải `Circle` không, vừa gán giá trị bên trong vào biến `radius`. Hai việc trong một bước.

### Destructure tags với data

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

area = \shape ->
    when shape is
        Circle radius -> 3.14159 * radius * radius
        Rectangle width height -> width * height
        Square side -> side * side

perimeter = \shape ->
    when shape is
        Circle radius -> 2.0 * 3.14159 * radius
        Rectangle width height -> 2.0 * (width + height)
        Square side -> 4.0 * side

main =
    shapes = [Circle 5.0, Rectangle 4.0 6.0, Square 3.0]

    List.forEach shapes \shape ->
        a = area shape
        p = perimeter shape
        label = when shape is
            Circle _ -> "Tròn"
            Rectangle _ _ -> "HCN"
            Square _ -> "Vuông"
        Stdout.line! "$(label): S=$(Num.toStr a), C=$(Num.toStr p)"
    # Output:
    # Tròn: S=78.53975, C=31.4159
    # HCN: S=24, C=20
    # Vuông: S=9, C=12
```

Chú ý `Circle _` — dấu `_` bỏ qua radius khi không cần giá trị đó.

### Destructure records trong tags

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

processEvent = \event ->
    when event is
        UserSignedUp { email, plan } ->
            "📧 Đăng ký mới: $(email) ($(plan))"
        OrderPlaced { orderId, total } ->
            "🛒 Đơn hàng #$(Num.toStr orderId): $(Num.toStr total)đ"
        PaymentReceived { orderId, method } ->
            "💰 Thanh toán #$(Num.toStr orderId) qua $(method)"
        UserDeleted { userId } ->
            "🗑️ Xóa user #$(Num.toStr userId)"

main =
    events = [
        UserSignedUp { email: "an@mail.com", plan: "Premium" },
        OrderPlaced { orderId: 42, total: 150000 },
        PaymentReceived { orderId: 42, method: "MoMo" },
        UserDeleted { userId: 99 },
    ]

    List.forEach events \event ->
        Stdout.line! (processEvent event)
    # Output:
    # 📧 Đăng ký mới: an@mail.com (Premium)
    # 🛒 Đơn hàng #42: 150000đ
    # 💰 Thanh toán #42 qua MoMo
    # 🗑️ Xóa user #99
```

Pattern matching + records = **event processing** rất tự nhiên. Mỗi event có data riêng, bạn destructure chính xác những gì cần.

---

## 8.4 — Guards: Thêm điều kiện

Pattern matching kiểm tra cấu trúc, nhưng đôi khi bạn cần kiểm tra thêm giá trị. Ví dụ: cùng là Gold member, nhưng đơn hàng trên 500k sẽ được giảm nhiều hơn. Guards (`if`) cho phép thêm điều kiện vào pattern:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

classifyAge = \age ->
    when age is
        a if a < 0 -> "❌ Tuổi không hợp lệ"
        a if a < 6 -> "👶 Mầm non"
        a if a < 12 -> "🧒 Tiểu học"
        a if a < 16 -> "📚 THCS"
        a if a < 19 -> "🎓 THPT"
        a if a < 23 -> "🏫 Đại học"
        _ -> "👨‍💼 Đi làm"

main =
    ages = [3, 8, 14, 17, 21, 35]
    List.forEach ages \age ->
        Stdout.line! "$(Num.toStr age) tuổi → $(classifyAge age)"
    # Output:
    # 3 tuổi → 👶 Mầm non
    # 8 tuổi → 🧒 Tiểu học
    # 14 tuổi → 📚 THCS
    # 17 tuổi → 🎓 THPT
    # 21 tuổi → 🏫 Đại học
    # 35 tuổi → 👨‍💼 Đi làm
```

### Guards kết hợp destructuring

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

applyDiscount = \order ->
    when order is
        { total, membership: Gold } if total >= 500000 ->
            discount = total * 20 // 100
            "🥇 Gold VIP: giảm $(Num.toStr discount)đ"
        { total, membership: Gold } ->
            discount = total * 10 // 100
            "🥇 Gold: giảm $(Num.toStr discount)đ"
        { total, membership: Silver } if total >= 300000 ->
            discount = total * 10 // 100
            "🥈 Silver combo: giảm $(Num.toStr discount)đ"
        { total, membership: Silver } ->
            discount = total * 5 // 100
            "🥈 Silver: giảm $(Num.toStr discount)đ"
        { membership: Bronze } ->
            "🥉 Bronze: giảm 2%"
        _ ->
            "Khách thường: không giảm"

main =
    orders = [
        { total: 600000, membership: Gold },
        { total: 200000, membership: Gold },
        { total: 400000, membership: Silver },
        { total: 100000, membership: Bronze },
    ]

    List.forEach orders \order ->
        Stdout.line! "$(Num.toStr order.total)đ → $(applyDiscount order)"
    # Output:
    # 600000đ → 🥇 Gold VIP: giảm 120000đ
    # 200000đ → 🥇 Gold: giảm 20000đ
    # 400000đ → 🥈 Silver combo: giảm 40000đ
    # 100000đ → 🥉 Bronze: giảm 2%
```

> **💡 Thứ tự quan trọng!** Guard cụ thể hơn phải đặt **TRƯỚC** guard chung hơn. `Gold if total >= 500000` trước `Gold` thường.

---

## 8.5 — Pattern Matching trên Lists

Roc cho phép pattern matching trực tiếp trên cấu trúc list — list rỗng, list 1 phần tử, list nhiều phần tử. Đây là nền tảng cho recursive processing — kỹ thuật xử lý dữ liệu cơ bản nhất trong FP.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

describeList = \list ->
    when list is
        [] -> "List rỗng"
        [only] -> "Chỉ có 1 phần tử: $(Num.toStr only)"
        [first, second] -> "Có 2: $(Num.toStr first) và $(Num.toStr second)"
        [first, .. as rest] -> "Bắt đầu bằng $(Num.toStr first), còn $(Num.toStr (List.len rest)) phần tử nữa"

main =
    Stdout.line! (describeList [])
    Stdout.line! (describeList [42])
    Stdout.line! (describeList [1, 2])
    Stdout.line! (describeList [10, 20, 30, 40, 50])
    # Output:
    # List rỗng
    # Chỉ có 1 phần tử: 42
    # Có 2: 1 và 2
    # Bắt đầu bằng 10, còn 4 phần tử nữa
```

Giải thích patterns:
- `[]` — list rỗng
- `[only]` — list đúng 1 phần tử, gán vào `only`
- `[first, second]` — list đúng 2 phần tử
- `[first, .. as rest]` — ít nhất 1 phần tử, `first` là đầu tiên, `rest` là phần còn lại

### Recursive processing với list patterns

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Tìm phần tử lớn nhất — recursion + pattern matching
findMax = \list ->
    when list is
        [] -> Err EmptyList
        [only] -> Ok only
        [first, .. as rest] ->
            when findMax rest is
                Ok restMax ->
                    if first >= restMax then Ok first else Ok restMax
                Err EmptyList -> Ok first

main =
    when findMax [3, 7, 2, 9, 4] is
        Ok max -> Stdout.line! "Max: $(Num.toStr max)"
        Err EmptyList -> Stdout.line! "List rỗng!"
    # Output: Max: 9
```

---

## 8.6 — Nested Patterns

Khi dữ liệu phức tạp — `Result` lồng trong `Result`, record chứa tag chứa record — bạn có thể match sâu nhiều tầng trong một lần. Không cần gán vào biến tạm rồi match lại:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

processResult = \result ->
    when result is
        # Nested: Result bên ngoài → Result bên trong
        Ok (Ok value) -> "✅✅ Thành công hoàn toàn: $(value)"
        Ok (Err innerErr) -> "✅❌ Thành công nhưng data lỗi: $(innerErr)"
        Err outerErr -> "❌ Thất bại: $(outerErr)"

# Nested tags + records
handleResponse = \response ->
    when response is
        Success { data: UserData { name, email } } ->
            "👤 $(name) ($(email))"
        Success { data: ProductList items } ->
            "📦 $(Num.toStr (List.len items)) sản phẩm"
        Failure { code: 404 } ->
            "❌ Không tìm thấy (404)"
        Failure { code, message } ->
            "❌ Lỗi $(Num.toStr code): $(message)"

main =
    Stdout.line! (processResult (Ok (Ok "data loaded")))
    Stdout.line! (processResult (Ok (Err "parse failed")))
    Stdout.line! (processResult (Err "network timeout"))

    Stdout.line! (handleResponse (Success { data: UserData { name: "An", email: "an@mail.com" } }))
    Stdout.line! (handleResponse (Failure { code: 404, message: "Not Found" }))
    Stdout.line! (handleResponse (Failure { code: 500, message: "Internal Error" }))
    # Output:
    # ✅✅ Thành công hoàn toàn: data loaded
    # ✅❌ Thành công nhưng data lỗi: parse failed
    # ❌ Thất bại: network timeout
    # 👤 An (an@mail.com)
    # ❌ Không tìm thấy (404)
    # ❌ Lỗi 500: Internal Error
```

---

## 8.7 — Pattern Matching thực tế: Command Parser

Ví dụ cuối kết hợp mọi thứ bạn đã học: một "mini language" cho commands. Mỗi command là một tag, một số mang data, một số cần guards. Đây là pattern rất phổ biến trong thực tế — xử lý lệnh, routing, event handling:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Một "mini language" cho commands
executeCommand = \cmd ->
    when cmd is
        Help ->
            "📖 Các lệnh: help, greet <name>, calc <a> <op> <b>, quit"
        Greet { name } ->
            "👋 Xin chào $(name)!"
        Calc { a, op: Add, b } ->
            "🔢 $(Num.toStr a) + $(Num.toStr b) = $(Num.toStr (a + b))"
        Calc { a, op: Sub, b } ->
            "🔢 $(Num.toStr a) - $(Num.toStr b) = $(Num.toStr (a - b))"
        Calc { a, op: Mul, b } ->
            "🔢 $(Num.toStr a) × $(Num.toStr b) = $(Num.toStr (a * b))"
        Calc { a, op: Div, b } if b != 0 ->
            "🔢 $(Num.toStr a) ÷ $(Num.toStr b) = $(Num.toStr (a // b))"
        Calc { op: Div, b: 0, .. } ->
            "❌ Không thể chia cho 0!"
        Quit ->
            "👋 Tạm biệt!"
        Unknown input ->
            "❓ Không hiểu lệnh: $(input)"

main =
    commands = [
        Help,
        Greet { name: "An" },
        Calc { a: 10, op: Add, b: 5 },
        Calc { a: 20, op: Div, b: 4 },
        Calc { a: 7, op: Div, b: 0 },
        Unknown "xyzzy",
        Quit,
    ]

    List.forEach commands \cmd ->
        Stdout.line! (executeCommand cmd)
    # Output:
    # 📖 Các lệnh: help, greet <name>, calc <a> <op> <b>, quit
    # 👋 Xin chào An!
    # 🔢 10 + 5 = 15
    # 🔢 20 ÷ 4 = 5
    # ❌ Không thể chia cho 0!
    # ❓ Không hiểu lệnh: xyzzy
    # 👋 Tạm biệt!
```

Mọi thứ kết hợp: tags (`Help`, `Greet`, `Calc`), records (`{ a, op, b }`), nested tags trong record (`op: Add`), guards (`if b != 0`), wildcard (`..`).

---


## ✅ Checkpoint 8

> Đến đây bạn phải hiểu:
> 1. `when...is` = exhaustive pattern matching — **compiler bắt buộc** cover mọi case
> 2. Destructuring = "mở hộp" lấy data bên trong ra
> 3. Guards (`if condition`) cho phép thêm điều kiện vào pattern
>
> **Test nhanh**: `when x is` mà quên 1 variant — chuyện gì xảy ra?
> <details><summary>Đáp án</summary>Compiler báo lỗi! Exhaustive matching bắt buộc cover mọi trường hợp — đây là safety net quan trọng nhất.</details>

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Phân loại tam giác

Viết function nhận 3 cạnh, trả loại tam giác:

```roc
# classifyTriangle 3 3 3 → "Đều"
# classifyTriangle 3 3 5 → "Cân"
# classifyTriangle 3 4 5 → "Thường"
```

<details><summary>✅ Lời giải</summary>

```roc
classifyTriangle = \a, b, c ->
    when (a == b, b == c, a == c) is
        (Bool.true, Bool.true, _) -> "Đều"
        (Bool.true, _, _) -> "Cân"
        (_, Bool.true, _) -> "Cân"
        (_, _, Bool.true) -> "Cân"
        _ -> "Thường"
```

</details>

---

**Bài 2** (10 phút): HTTP response handler

Viết `handleResponse` cho các trường hợp:

```roc
# Ok { status: 200, body } → "✅ $(body)"
# Ok { status: 201, body } → "🆕 Created: $(body)"
# Ok { status: 204, .. }   → "✅ No content"
# Ok { status, .. } if status >= 400 && status < 500 → "❌ Client error $(status)"
# Ok { status, .. } if status >= 500 → "💥 Server error $(status)"
# Err Timeout → "⏰ Timeout"
# Err NetworkError msg → "🌐 Network: $(msg)"
```

<details><summary>✅ Lời giải</summary>

```roc
handleResponse = \response ->
    when response is
        Ok { status: 200, body } -> "✅ $(body)"
        Ok { status: 201, body } -> "🆕 Created: $(body)"
        Ok { status: 204, .. } -> "✅ No content"
        Ok { status, .. } if status >= 400 && status < 500 ->
            "❌ Client error $(Num.toStr status)"
        Ok { status, .. } if status >= 500 ->
            "💥 Server error $(Num.toStr status)"
        Ok { status, .. } ->
            "ℹ️ Status $(Num.toStr status)"
        Err Timeout -> "⏰ Timeout"
        Err (NetworkError msg) -> "🌐 Network: $(msg)"
```

</details>

---

**Bài 3** (15 phút): Expression evaluator

Viết `evaluate` cho biểu thức toán học:

```roc
# Lit 5               → Ok 5
# Add (Lit 3) (Lit 4)  → Ok 7
# Mul (Lit 2) (Add (Lit 3) (Lit 4))  → Ok 14
# Div (Lit 10) (Lit 0) → Err DivisionByZero
```

<details><summary>💡 Gợi ý</summary>

Dùng recursion. Mỗi branch gọi `evaluate` cho sub-expressions, rồi dùng `Result.try` hoặc `when...is` để xử lý errors.

</details>

<details><summary>✅ Lời giải</summary>

```roc
evaluate = \expr ->
    when expr is
        Lit n -> Ok n
        Add left right ->
            when (evaluate left, evaluate right) is
                (Ok l, Ok r) -> Ok (l + r)
                (Err e, _) -> Err e
                (_, Err e) -> Err e
        Mul left right ->
            when (evaluate left, evaluate right) is
                (Ok l, Ok r) -> Ok (l * r)
                (Err e, _) -> Err e
                (_, Err e) -> Err e
        Div left right ->
            when (evaluate left, evaluate right) is
                (Ok _, Ok 0) -> Err DivisionByZero
                (Ok l, Ok r) -> Ok (l // r)
                (Err e, _) -> Err e
                (_, Err e) -> Err e
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `This when does not cover all possibilities` | Quên branch | Thêm branch bị thiếu — compiler chỉ rõ tag nào |
| `The branches of this when produce different types` | Branches trả khác type | Mọi branch phải trả CÙNG type |
| Guard không match dù điều kiện đúng | Thứ tự branches sai | Branch cụ thể hơn phải ĐẶT TRƯỚC |
| `This pattern is redundant` | Branch trước đã cover hết | Xóa branch thừa hoặc sắp xếp lại |
| Destructure sai field name | Typo trong destructuring | Kiểm tra tên field chính xác |

---

## Tóm tắt

- ✅ **`when...is`** = pattern matching — kiểm tra cấu trúc dữ liệu, chạy branch đầu tiên khớp.
- ✅ **Exhaustive matching** = compiler bắt buộc xử lý TẤT CẢ trường hợp. Thêm variant → compiler chỉ chỗ cần sửa.
- ✅ **Destructuring** = "mở" data ra. `Circle radius`, `{ name, age }`, `Ok value`.
- ✅ **Guards** = `if condition` — thêm điều kiện logic, branch cụ thể trước branch chung.
- ✅ **List patterns** — `[]`, `[only]`, `[first, .. as rest]`.
- ✅ **Nested patterns** — match sâu nhiều tầng: `Ok (Ok value)`, `Success { data: UserData { name } }`.
- ✅ **Wildcard `_`** = bỏ qua giá trị. `..` = bỏ qua các fields còn lại.

## Tiếp theo

→ Chapter 9: **Lists & Standard Functions** — deep dive vào List API: `map`, `keepIf`, `walk`, `sortWith`, `groupBy`, và cách chúng kết hợp thành data processing pipelines mạnh mẽ.
