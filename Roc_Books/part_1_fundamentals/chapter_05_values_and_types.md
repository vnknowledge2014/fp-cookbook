# Chapter 5 — Values & Types

> **Bạn sẽ học được**:
> - Tất cả kiểu dữ liệu cơ bản trong Roc: số, chuỗi, boolean
> - **Type inference** — tại sao Roc gần như không bao giờ cần bạn ghi type
> - **Structural typing** — types được xác định bởi cấu trúc, không phải tên
> - Type annotations — khi nào nên viết, khi nào không cần
> - Cách Roc xử lý số khác biệt so với mọi ngôn ngữ khác
>
> **Yêu cầu trước**: [Chapter 4 — Getting Started](chapter_04_getting_started.md)
> **Thời gian đọc**: ~35 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Hiểu rõ hệ thống types của Roc — nền tảng cho mọi thứ từ đây trở đi.

---

Python cho phép `x = 42` rồi ngay sau đó `x = "hello"` — cùng một biến chứa hai kiểu khác nhau. JavaScript cũng vậy. Tiện, nhưng đổi lại bạn chỉ phát hiện lỗi kiểu khi chương trình đang chạy — thường là lúc 2 giờ sáng trên production.

Roc chọn con đường khác: **mọi giá trị đều có type cố định**, và compiler kiểm tra hết trước khi chạy. Điều thú vị là bạn hầu như không cần ghi type ra — Roc tự suy ra được.

## 5.1 — Mọi thứ đều có Type

Trong Roc, **mọi giá trị đều có type** — và compiler biết type đó dù bạn không ghi ra.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Compiler tự biết type của từng giá trị:
    age = 25              # Num — số
    name = "An"           # Str — chuỗi
    isActive = Bool.true  # Bool — đúng/sai
    scores = [9, 8, 10]   # List (Num *) — danh sách số

    Stdout.line! "$(name), $(Num.toStr age) tuổi, active: $(Inspect.toStr isActive)"
```

Bạn **không cần ghi type** — Roc tự suy ra. Đây gọi là **type inference**.

---

## 5.2 — Số (Numbers)

Python có `int` và `float`. JavaScript chỉ có `number`. Roc làm khác — nó có hơn 10 kiểu số khác nhau, từ `U8` (0–255) cho đến `Dec` (số thập phân chính xác cho tiền tệ). Nghe có vẻ phức tạp, nhưng thực tế bạn hầu như không cần quan tâm — viết `42` và compiler tự chọn kiểu phù hợp.

Khi nào bạn MỚI cần chỉ định? Khi xử lý tiền (dùng `Dec` để không bị lỗi làm tròn), khi gọi API yêu cầu đúng kích thước (port là `U16`), hoặc khi cần tiết kiệm bộ nhớ.

### Roc có hệ thống số đặc biệt

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Số nguyên — compiler tự chọn kiểu phù hợp
    count = 42              # Num * (generic)
    age : U8                # Bạn CÓ THỂ chỉ định nếu muốn
    age = 25

    # Số thực
    pi = 3.14159            # Frac * (generic)
    price : F64             # Hoặc chỉ định cụ thể
    price = 19.99

    # Phép tính — hoạt động tự nhiên
    total = price * 3.0
    Stdout.line! "3 × $(Num.toStr price) = $(Num.toStr total)"

    # Chia số nguyên vs số thực
    intDiv = 17 // 5        # → 3 (chia nguyên, bỏ phần dư)
    floatDiv = 17.0 / 5.0   # → 3.4 (chia thực)
    remainder = 17 %% 5     # → 2 (phần dư)

    Stdout.line! "17 // 5 = $(Num.toStr intDiv)"
    Stdout.line! "17.0 / 5.0 = $(Num.toStr floatDiv)"
    Stdout.line! "17 %% 5 = $(Num.toStr remainder)"
```

### Bảng kiểu số

| Kiểu | Ý nghĩa | Phạm vi | Khi nào dùng |
|------|---------|---------|-------------|
| `U8` | Số nguyên không dấu 8-bit | 0 → 255 | Tuổi, byte |
| `U16` | Không dấu 16-bit | 0 → 65,535 | Port number |
| `U32` | Không dấu 32-bit | 0 → ~4 tỷ | ID, count |
| `U64` | Không dấu 64-bit | 0 → rất lớn | Timestamps |
| `I32` | Có dấu 32-bit | -2 tỷ → 2 tỷ | Số có thể âm |
| `I64` | Có dấu 64-bit | Rất nhỏ → rất lớn | Mặc định cho số nguyên |
| `F32` | Số thực 32-bit | ±3.4 × 10³⁸ | Graphics, ML |
| `F64` | Số thực 64-bit | ±1.7 × 10³⁰⁸ | Mặc định cho số thực |
| `Dec` | Số thập phân chính xác | 28 chữ số | **Tiền** — không mất precision |

> **💡 Quy tắc chọn kiểu số**:
> - Không chắc? Để Roc tự chọn (viết `42`, không ghi type)
> - Tiền bạc? Dùng `Dec` — không bao giờ mất precision
> - Performance? Dùng kiểu nhỏ nhất đủ dùng

### `Dec` — Kiểu tiền tệ

Nếu bạn từng gặp lỗi `0.1 + 0.2 = 0.30000000000000004` trong JavaScript hay Python, bạn hiểu vấn đề. Số thực `F64` lưu trữ theo dạng nhị phân — một số giá trị thập phân không thể biểu diễn chính xác. `Dec` giải quyết chuyện này: nó lưu trữ 28 chữ số thập phân chính xác, phù hợp cho mọi phép tính tiền tệ.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # F64 — KHÔNG an toàn cho tiền!
    badTotal : F64
    badTotal = 0.1 + 0.2
    # Có thể = 0.30000000000000004 thay vì 0.3!

    # Dec — AN TOÀN cho tiền
    price : Dec
    price = 45000.50
    quantity : Dec
    quantity = 3.0
    total = price * quantity

    Stdout.line! "Tổng: $(Num.toStr total)đ"
    # Output: Tổng: 135001.5đ — chính xác!
```

### Chuyển đổi kiểu số

```roc
# Chuyển số → chuỗi
text = Num.toStr 42        # → "42"

# Chuyển giữa các kiểu số
big : I64
big = 1000

small : U8
small = Num.toU8 big       # ⚠️ Có thể mất data nếu giá trị > 255
```

---

## 5.3 — Chuỗi (Str)

Xử lý chuỗi chiếm phần lớn thời gian của bất kỳ lập trình viên nào — hiển thị thông báo, xử lý input, format output. Roc dùng UTF-8 theo mặc định, nghĩa là tiếng Việt, emoji, và mọi ngôn ngữ khác hoạt động luôn mà không cần cấu hình gì.

### Cơ bản

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Chuỗi đơn giản
    greeting = "Xin chào!"

    # String interpolation — chèn giá trị bằng $(...)
    name = "An"
    age = 25
    intro = "Tôi là $(name), $(Num.toStr age) tuổi"
    Stdout.line! intro

    # Chuỗi nhiều dòng — dùng """
    poem =
        """
        Roc là ngôn ngữ
        Pure functional
        Nhanh và an toàn
        """
    Stdout.line! poem
```

### Các hàm xử lý Str

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    text = "Hello, Roc!"

    # Nối chuỗi
    full = Str.concat "Hello" " World"
    Stdout.line! full    # → "Hello World"

    # Kiểm tra rỗng
    empty = Str.isEmpty ""
    Stdout.line! "\"\" rỗng? $(Inspect.toStr empty)"    # → Bool.true

    # Đếm bytes UTF-8
    length = Str.countUtf8Bytes "Việt Nam"
    Stdout.line! "\"Việt Nam\" = $(Num.toStr length) bytes"

    # Tách chuỗi
    parts = Str.split "a,b,c" ","
    Stdout.line! "Split: $(Inspect.toStr parts)"    # → ["a", "b", "c"]

    # Ghép danh sách chuỗi
    joined = Str.joinWith ["Roc", "is", "fun"] " "
    Stdout.line! joined    # → "Roc is fun"

    # Kiểm tra chứa
    hasRoc = Str.contains "I love Roc" "Roc"
    Stdout.line! "Có 'Roc'? $(Inspect.toStr hasRoc)"    # → Bool.true

    # Bắt đầu/kết thúc bằng
    startsHello = Str.startsWith "Hello World" "Hello"
    Stdout.line! "Starts with Hello? $(Inspect.toStr startsHello)"

    # Trim khoảng trắng
    trimmed = Str.trim "  hello  "
    Stdout.line! "Trim: '$(trimmed)'"    # → "hello"
```

> **💡 Lưu ý**: Roc dùng UTF-8 — chuỗi tiếng Việt hoạt động hoàn hảo. `Str.countUtf8Bytes` đếm bytes (không phải ký tự).

---

## 5.4 — Boolean (Bool)

Boolean chỉ có hai giá trị: đúng hoặc sai. Đơn giản, nhưng có một điểm khác biệt quan trọng so với nhiều ngôn ngữ khác: trong Roc, `if/else` là **expression** (trả giá trị), không phải **statement** (thực thi lệnh). Nghĩa là bạn có thể gán kết quả của `if/else` vào một biến — và compiler bắt buộc mọi nhánh phải trả cùng kiểu.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    isRaining = Bool.true
    hasUmbrella = Bool.false

    # AND — cả hai đều đúng
    goOut = Bool.and (Bool.not isRaining) hasUmbrella
    # Không mưa VÀ có ô → đi chơi

    # OR — ít nhất một đúng
    stayHome = Bool.or isRaining (Bool.not hasUmbrella)
    # Đang mưa HOẶC không có ô → ở nhà

    # NOT — đảo ngược
    isSunny = Bool.not isRaining

    Stdout.line! "Trời mưa: $(Inspect.toStr isRaining)"
    Stdout.line! "Trời nắng: $(Inspect.toStr isSunny)"
    Stdout.line! "Ở nhà: $(Inspect.toStr stayHome)"

    # So sánh — trả về Bool
    a = 5
    b = 3
    Stdout.line! "5 > 3? $(Inspect.toStr (a > b))"       # → Bool.true
    Stdout.line! "5 == 3? $(Inspect.toStr (a == b))"      # → Bool.false
    Stdout.line! "5 != 3? $(Inspect.toStr (a != b))"      # → Bool.true
    Stdout.line! "5 >= 5? $(Inspect.toStr (a >= a))"      # → Bool.true
```

### `if/else` — luôn là expression

```roc
# if/else trong Roc TRẢ GIÁ TRỊ — giống ternary operator
status = if age >= 18 then "Người lớn" else "Trẻ em"

# Có thể nhiều nhánh
category =
    if score >= 90 then "Xuất sắc"
    else if score >= 70 then "Khá"
    else if score >= 50 then "Trung bình"
    else "Cần cố gắng"
```

> **💡 Khác biệt**: Trong Roc, `if/else` **luôn trả giá trị** — nó là expression, không phải statement. Mọi nhánh phải trả **cùng kiểu**.

---

## 5.5 — Type Inference: Compiler tự biết

Haskell, Elm, OCaml — các ngôn ngữ FP đều có type inference mạnh. Roc thừa hưởng trực tiếp từ Elm (và gián tiếp từ Hindley-Milner type system). Kết quả: bạn gần như không bao giờ cần ghi type — compiler suy ra được hết.

### Compiler thông minh hơn bạn tưởng

```roc
# Bạn viết:
add = \a, b -> a + b

# Compiler tự suy ra:
# add : Num a, Num a -> Num a
# ("Cho 2 số cùng kiểu, trả về số cùng kiểu đó")

# Bạn viết:
greet = \name -> "Hello, $(name)!"

# Compiler suy ra:
# greet : Str -> Str
# ("Cho 1 chuỗi, trả về chuỗi")
```

Roc suy ra type bằng cách nhìn **cách bạn dùng** giá trị:
- Dùng `+` → phải là `Num`
- Dùng `$()` → phải là `Str`
- Dùng `Bool.and` → phải là `Bool`

### Khi nào NÊN viết type annotation?

```roc
# ✅ NÊN — khi function là public API (dễ đọc cho người khác)
processOrder : { name : Str, quantity : U32 } -> Str
processOrder = \order ->
    "$(order.name) × $(Num.toStr order.quantity)"

# ✅ NÊN — khi chỉ định kiểu số cụ thể
port : U16
port = 8080

# ❌ KHÔNG CẦN — biến local rõ ràng
greeting = "hello"     # rõ ràng là Str, không cần ghi
total = 1 + 2          # rõ ràng là Num, không cần ghi
```

> **💡 Quy tắc thực tế**: Viết type annotation cho **function definitions** (đặc biệt khi expose ra module). Không cần cho **biến local**.

---

## 5.6 — Structural Typing: Type = cấu trúc

Structural typing là điểm khác biệt lớn nhất giữa Roc và các ngôn ngữ như Rust hay Java.

### Ý tưởng cốt lõi

Trong Rust, `struct Student { name: String }` và `struct Teacher { name: String }` là hai kiểu hoàn toàn khác nhau — dù cấu trúc giống hệt. Bạn không thể truyền `Student` vào function nhận `Teacher`.

Roc làm ngược lại: **cấu trúc giống = cùng kiểu**. Một function nhận record có field `name : Str` sẽ chấp nhận bất kỳ record nào có field đó, bất kể record còn chứa những gì khác.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Function nhận record có field "name" kiểu Str
greet = \person -> "Xin chào $(person.name)!"

main =
    # Cả 3 record đều hoạt động — vì đều có field "name : Str"
    student = { name: "An", grade: 10 }
    teacher = { name: "Minh", subject: "Toán" }
    visitor = { name: "Bob" }

    Stdout.line! (greet student)    # ✅ Có field name
    Stdout.line! (greet teacher)    # ✅ Có field name
    Stdout.line! (greet visitor)    # ✅ Có field name
    # Output:
    # Xin chào An!
    # Xin chào Minh!
    # Xin chào Bob!
```

`greet` không cần biết record có field `grade` hay `subject` — nó chỉ cần `name`. Đây là **structural typing** — "nếu nó có field tôi cần, tôi chấp nhận".

> **💡 Ẩn dụ**: Structural typing giống **ổ cắm USB**. Bạn không cần hỏi "đây là USB của Samsung hay Apple?" — chỉ cần hỏi "nó có cổng USB-C không?" Có → cắm được.

### So sánh với Nominal Typing (Rust, Java)

```
# Rust (nominal typing) — TÊN phải khớp
struct Student { name: String, grade: u8 }
struct Teacher { name: String, subject: String }
# Student ≠ Teacher — dù cùng có field "name"

# Roc (structural typing) — CẤU TRÚC phải khớp  
# { name: Str, grade: U8 } tương thích với bất kỳ { name: Str, ... }
# Không cần khai báo struct, không cần tên
```

| | Nominal (Rust/Java) | Structural (Roc) |
|---|---|---|
| Type = | Tên | Cấu trúc |
| Khai báo trước? | Bắt buộc | Không cần |
| Tương thích? | Cùng tên mới OK | Cùng cấu trúc là OK |
| Ưu điểm | Rõ ràng, nghiêm ngặt | Linh hoạt, ít boilerplate |

### Structural typing cho Tags

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Function chỉ cần tag Red, Green, hoặc Blue
colorToHex = \color ->
    when color is
        Red -> "#FF0000"
        Green -> "#00FF00"
        Blue -> "#0000FF"

main =
    # Bạn không khai báo "enum Color" ở đâu cả
    # Cứ viết tag trực tiếp — compiler tự hiểu
    Stdout.line! "Red = $(colorToHex Red)"
    Stdout.line! "Green = $(colorToHex Green)"
    Stdout.line! "Blue = $(colorToHex Blue)"
    # Output:
    # Red = #FF0000
    # Green = #00FF00
    # Blue = #0000FF
```

Compiler suy ra type của `colorToHex`:

```
colorToHex : [Red, Green, Blue] -> Str
```

Nghĩa là: "Nhận một trong 3 tags `Red`, `Green`, `Blue` — trả Str."

---

## 5.7 — Type Aliases (Đặt tên cho type)

Khi type dài và xuất hiện nhiều lần, viết lại `{ x : F64, y : F64 }` mỗi chỗ vừa mệt vừa dễ sai. Type alias cho phép bạn đặt tên ngắn gọn:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Type alias — đặt tên cho type dài
Point : { x : F64, y : F64 }
Color : [Red, Green, Blue, Custom Str]

distance : Point, Point -> F64
distance = \p1, p2 ->
    dx = p1.x - p2.x
    dy = p1.y - p2.y
    Num.sqrt (dx * dx + dy * dy)

colorName : Color -> Str
colorName = \c ->
    when c is
        Red -> "Đỏ"
        Green -> "Xanh lá"
        Blue -> "Xanh dương"
        Custom name -> name

main =
    origin = { x: 0.0, y: 0.0 }
    target = { x: 3.0, y: 4.0 }

    dist = distance origin target
    Stdout.line! "Khoảng cách: $(Num.toStr dist)"   # → 5.0

    Stdout.line! (colorName Red)           # → Đỏ
    Stdout.line! (colorName (Custom "Tím")) # → Tím
```

Một điều cần nhớ: type alias chỉ là **tên gọi tắt**, không tạo type mới thật sự. `Point` vẫn là `{ x : F64, y : F64 }` — bất kỳ record nào có cùng cấu trúc vẫn tương thích. Muốn type thật sự khác biệt, không lẫn lộn được? Chờ Chapter 10 (Opaque Types).

---


## ✅ Checkpoint 5

> Đến đây bạn phải hiểu:
> 1. Roc có **structural typing** — types khớp theo cấu trúc, không theo tên
> 2. Type inference tự suy ra types — bạn hiếm khi cần ghi annotation
> 3. Number types: `I64`, `U64`, `F64`, `Dec` — chọn theo nhu cầu
>
> **Test nhanh**: `x = 42` — Roc suy ra type gì cho `x`?
> <details><summary>Đáp án</summary>`Num *` — generic number type. Khi được dùng trong context cụ thể sẽ trở thành `I64`, `U64`, etc.</details>

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Kiểu dữ liệu

Không chạy code — đoán type của mỗi giá trị:

```roc
a = 42
b = "hello"
c = Bool.true
d = 3.14
e = [1, 2, 3]
f = { name: "An", age: 25 }
g = Ok "success"
h = \x -> x * 2
```

<details><summary>✅ Lời giải</summary>

```
a : Num *           — số (generic)
b : Str             — chuỗi
c : Bool            — boolean
d : Frac *          — số thực (generic)
e : List (Num *)    — danh sách số
f : { name : Str, age : Num * }  — record
g : Result Str *    — Result thành công với Str
h : Num a -> Num a  — function nhận số, trả số
```

</details>

---

**Bài 2** (10 phút): String processor

Viết các function xử lý chuỗi:

```roc
# a) Kiểm tra chuỗi có phải palindrome (đọc xuôi ngược giống nhau)
# isPalindrome "aba" → Bool.true
# isPalindrome "abc" → Bool.false

# b) Đếm số từ trong câu
# wordCount "Roc is fun" → 3

# c) Viết hoa chữ cái đầu
# capitalize "hello" → "Hello"
```

<details><summary>💡 Gợi ý</summary>

- Palindrome: chuyển Str → List U8 (`Str.toUtf8`), đảo ngược, so sánh
- WordCount: `Str.split` theo " ", rồi `List.len`
- Capitalize: lấy byte đầu, trừ 32 (ASCII), ghép lại

</details>

<details><summary>✅ Lời giải</summary>

```roc
# a) Palindrome
isPalindrome = \s ->
    bytes = Str.toUtf8 s
    reversed = List.reverse bytes
    bytes == reversed

# b) Word count
wordCount = \s ->
    words = Str.split s " "
    List.len words

# c) Capitalize (ASCII, đơn giản)
capitalize = \s ->
    bytes = Str.toUtf8 s
    when List.first bytes is
        Ok firstByte ->
            if firstByte >= 97 && firstByte <= 122 then
                upper = firstByte - 32
                rest = List.dropFirst bytes 1
                when Str.fromUtf8 (List.prepend rest upper) is
                    Ok result -> result
                    Err _ -> s
            else
                s
        Err _ -> s
```

</details>

---

**Bài 3** (10 phút): Structural typing

Viết function `describe` nhận **bất kỳ record nào có field `name : Str` và `age : Num *`**, trả mô tả:

```roc
# describe { name: "An", age: 25 } → "An (25 tuổi)"
# describe { name: "Roc", age: 3, language: "FP" } → "Roc (3 tuổi)"
# Record thứ 2 có field thêm "language" — vẫn phải hoạt động!
```

<details><summary>✅ Lời giải</summary>

```roc
describe = \thing -> "$(thing.name) ($(Num.toStr thing.age) tuổi)"

# Compiler suy ra type:
# describe : { name : Str, age : Num a }b -> Str
# b = "có thể có thêm fields khác" (open record)
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `This value is a Num, but I need a Str` | Quên `Num.toStr` khi chèn số vào chuỗi | Dùng `$(Num.toStr x)` |
| `These types don't match` trong phép tính | Trộn số nguyên và số thực | `17 / 5` sai → dùng `17 // 5` (nguyên) hoặc `17.0 / 5.0` (thực) |
| `I can't find field .name on this record` | Record không có field bạn truy cập | Kiểm tra chính tả — Roc phân biệt hoa/thường |
| `This if does not have an else` | `if` trong Roc **bắt buộc** có `else` | Thêm nhánh `else` — mọi nhánh cùng kiểu |

---

## Tóm tắt

- ✅ **Numbers**: `U8`–`U64` (không dấu), `I32`–`I64` (có dấu), `F32`/`F64` (thực), `Dec` (tiền). Thường để compiler tự chọn.
- ✅ **Str**: UTF-8, interpolation `$(...)`, multiline `"""..."""`. Các hàm: `Str.concat`, `Str.split`, `Str.trim`, `Str.contains`.
- ✅ **Bool**: `Bool.true`/`Bool.false`. `Bool.and`, `Bool.or`, `Bool.not`. `if/else` là expression.
- ✅ **Type inference**: Compiler suy ra type từ cách bạn dùng. Viết annotation cho function API, bỏ qua cho biến local.
- ✅ **Structural typing**: Type = cấu trúc, không phải tên. Record có đúng fields → tương thích. Tag cùng tên → tương thích.

## Tiếp theo

→ Chapter 6: **Functions & Pipelines** — auto-currying, pipeline `|>`, backpassing `<-`, và cách Roc biến chuỗi function thành dây chuyền xử lý data.
