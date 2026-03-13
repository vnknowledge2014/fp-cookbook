# Chapter 13 — Error Handling with Result

> **Bạn sẽ học được**:
> - `Result ok err` — cách duy nhất xử lý lỗi trong Roc (không exceptions!)
> - **Railway-oriented programming** — chuỗi xử lý tự dừng khi gặp lỗi
> - `Result.try` — chaining nhiều operations có thể lỗi
> - `Result.map`, `Result.mapErr` — biến đổi giá trị và lỗi
> - **Error hierarchy** — thiết kế tag unions cho lỗi thực tế
> - `Result.withDefault`, `Result.isOk` — tiện ích phổ biến
>
> **Yêu cầu trước**: [Chapter 12 — Tasks & Effects](chapter_12_tasks_and_effects.md)
> **Thời gian đọc**: ~40 phút | **Level**: Intermediate
> **Kết quả cuối cùng**: Xử lý lỗi gracefully — không crash, không bỏ sót, không spaghetti try/catch.

---

Bạn đã bao giờ nhận alert lúc 3 giờ sáng vì production crash? Nguyên nhân thường gặp: một exception không ai bắt, ở một chỗ không ai ngờ. Python, JavaScript, Java đều cho phép function throw exception bất kỳ lúc nào mà không báo trước trong type.

Roc không có exceptions. Mọi lỗi đều nằm trong type: `Result ok err`. Nhìn type là biết function có thể thất bại theo cách nào. Compiler bắt buộc bạn xử lý hết.

## 13.1 — Tại sao không có Exceptions?

Exceptions có một vấn đề cơ bản: chúng **ẩn**. Nhìn vào một function Python, bạn không biết nó có thể throw exception nào:

### Vấn đề của Exceptions

```python
# Python — lỗi ẩn khắp nơi
def process_order(data):
    order = json.loads(data)          # 💥 JSONDecodeError?
    user = find_user(order["userId"]) # 💥 KeyError? UserNotFound?
    charge(user, order["total"])      # 💥 PaymentFailed? InsufficientFunds?
    send_email(user.email)            # 💥 SMTPError?
    return "OK"

# Bạn KHÔNG BIẾT function nào có thể crash
# Quên try/catch ở đâu → production crash lúc 3 giờ sáng
```

### Cách Roc: Result bắt buộc xử lý

```roc
# Roc — mọi lỗi đều hiển thị trong type
processOrder : Str -> Result Str [InvalidJson, UserNotFound, PaymentFailed Str, EmailFailed]
```

Nhìn type là biết **tất cả cách function có thể thất bại**. Compiler buộc bạn xử lý hết.

---

## 13.2 — Result cơ bản

Result có hai trạng thái: `Ok value` (thành công) và `Err reason` (thất bại). Pattern matching trên Result buộc bạn xử lý cả hai — không có cách nào "quên" xử lý lỗi.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

safeDivide : I64, I64 -> Result I64 [DivisionByZero]
safeDivide = \a, b ->
    if b == 0 then Err DivisionByZero
    else Ok (a // b)

parseAge : Str -> Result U8 [NotANumber, OutOfRange]
parseAge = \input ->
    when Str.toI64 input is
        Err _ -> Err NotANumber
        Ok n ->
            if n < 0 || n > 150 then Err OutOfRange
            else Ok (Num.toU8 n)

main =
    # when...is — cách rõ ràng nhất
    when safeDivide 10 3 is
        Ok result -> Stdout.line! "10 / 3 = $(Num.toStr result)"
        Err DivisionByZero -> Stdout.line! "❌ Chia cho 0!"

    when safeDivide 10 0 is
        Ok result -> Stdout.line! "10 / 0 = $(Num.toStr result)"
        Err DivisionByZero -> Stdout.line! "❌ Chia cho 0!"

    when parseAge "25" is
        Ok age -> Stdout.line! "Tuổi hợp lệ: $(Num.toStr age)"
        Err NotANumber -> Stdout.line! "❌ Không phải số"
        Err OutOfRange -> Stdout.line! "❌ Tuổi ngoài phạm vi"

    # Output:
    # 10 / 3 = 3
    # ❌ Chia cho 0!
    # Tuổi hợp lệ: 25
```

---

## 13.3 — Tiện ích Result phổ biến

Không phải lúc nào cũng cần pattern matching. Roc cung cấp helper functions cho các tình huống phổ biến: dùng giá trị mặc định, kiểm tra thành/bại, biến đổi kết quả.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Result.withDefault — dùng giá trị mặc định nếu lỗi
    age = Str.toI64 "abc" |> Result.withDefault 0
    Stdout.line! "Age: $(Num.toStr age)"     # → 0

    # Result.isOk / Result.isErr — kiểm tra
    valid = Result.isOk (Str.toI64 "42")
    Stdout.line! "42 ok? $(Inspect.toStr valid)"   # → Bool.true

    # Result.map — biến đổi Ok value, giữ nguyên Err
    doubled = Str.toI64 "21" |> Result.map \n -> n * 2
    Stdout.line! "21 * 2 = $(Inspect.toStr doubled)"   # → Ok 42

    tripled = Str.toI64 "abc" |> Result.map \n -> n * 3
    Stdout.line! "abc * 3 = $(Inspect.toStr tripled)"   # → Err ...

    # Result.mapErr — biến đổi Err, giữ nguyên Ok
    result = Str.toI64 "abc" |> Result.mapErr \_ -> InvalidInput "abc"
    Stdout.line! "Mapped err: $(Inspect.toStr result)"
```

### Bảng tham chiếu

| Function | Ý nghĩa | Khi nào dùng |
|----------|---------|-------------|
| `Result.withDefault` | `Ok val → val`, `Err → default` | Cần giá trị mặc định |
| `Result.isOk` / `isErr` | Kiểm tra thành/bại | Khi chỉ cần biết OK hay không |
| `Result.map` | Biến đổi Ok value | Muốn transform kết quả |
| `Result.mapErr` | Biến đổi Err value | Muốn đổi kiểu lỗi |
| `Result.try` | Chain operations | Pipeline nhiều bước (xem 13.4) |

---

## 13.4 — Railway-Oriented Programming

Khi một workflow có nhiều bước và mỗi bước có thể thất bại, bạn không muốn viết `if err then return err` ở mỗi dòng. Railway-oriented programming giải quyết chuyện này: dữ liệu chạy trên 2 đường ray — Ok và Err. Khi một bước thất bại, mọi bước sau tự động bị bỏ qua.

### Ý tưởng

Hình dung data chạy trên **2 đường ray song song**:

```
Ok track:  ─── step1 ─── step2 ─── step3 ─── ✅ Success
                │           │         │
Err track: ────┘───────────┘─────────┘──── ❌ First Error
```

Khi một bước **thất bại**, data nhảy sang Err track và **bỏ qua tất cả bước sau**. Không cần `if err then return` ở mỗi bước.

### `Result.try` — Chaining

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Mỗi bước có thể thất bại
validateName : Str -> Result Str [EmptyName]
validateName = \name ->
    if Str.isEmpty (Str.trim name) then Err EmptyName
    else Ok (Str.trim name)

validateEmail : Str -> Result Str [InvalidEmail]
validateEmail = \email ->
    if Str.contains email "@" then Ok email
    else Err InvalidEmail

validateAge : Str -> Result U8 [InvalidAge, AgeOutOfRange]
validateAge = \ageStr ->
    when Str.toI64 ageStr is
        Err _ -> Err InvalidAge
        Ok n ->
            if n < 1 || n > 150 then Err AgeOutOfRange
            else Ok (Num.toU8 n)

# Railway: nếu bước nào lỗi → dừng ngay, trả Err đó
registerUser : Str, Str, Str -> Result { name : Str, email : Str, age : U8 } _
registerUser = \rawName, rawEmail, rawAge ->
    name = validateName? rawName           # ? = Result.try shorthand
    email = validateEmail? rawEmail
    age = validateAge? rawAge
    Ok { name, email, age }

main =
    # Thành công
    when registerUser "An" "an@mail.com" "25" is
        Ok user -> Stdout.line! "✅ $(user.name) ($(user.email), $(Num.toStr user.age))"
        Err err -> Stdout.line! "❌ $(Inspect.toStr err)"

    # Lỗi ở bước 1 — bỏ qua bước 2, 3
    when registerUser "" "an@mail.com" "25" is
        Ok _ -> Stdout.line! "OK"
        Err EmptyName -> Stdout.line! "❌ Tên rỗng!"
        Err _ -> Stdout.line! "❌ Lỗi khác"

    # Lỗi ở bước 2
    when registerUser "An" "no-at-sign" "25" is
        Ok _ -> Stdout.line! "OK"
        Err InvalidEmail -> Stdout.line! "❌ Email sai format!"
        Err _ -> Stdout.line! "❌ Lỗi khác"

    # Output:
    # ✅ An (an@mail.com, 25)
    # ❌ Tên rỗng!
    # ❌ Email sai format!
```

### `?` là gì?

Dấu `?` sau biểu thức Result = "nếu `Ok`, lấy giá trị ra; nếu `Err`, **dừng function và trả Err**":

```roc
# Dấu ? sugar:
name = validateName? rawName

# Tương đương:
name = when validateName rawName is
    Ok val -> val
    Err err -> return Err err   # dừng function!
```

Giống `?` operator trong Rust— ngắn gọn, rõ ý.

---

## 13.5 — Pipeline với Result

Railway pattern áp dụng vào thực tế: mỗi bước trong pipeline là một function trả Result, dấu `?` nối chúng lại. CSV parse → validate → classify → format — lỗi ở bước nào tự động dừng pipeline.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Pipeline: parse CSV → validate → transform → format
processCSVLine : Str -> Result Str [InvalidFormat, InvalidScore, ScoreTooLow]
processCSVLine = \line ->
    # Bước 1: Parse
    { name, scoreStr } = parseCSV? line

    # Bước 2: Validate
    score = parseScore? scoreStr

    # Bước 3: Classify (pure, không thể lỗi → dùng trực tiếp)
    grade = classify score

    # Bước 4: Format
    Ok "$(name): $(Num.toStr score) điểm → $(grade)"

parseCSV : Str -> Result { name : Str, scoreStr : Str } [InvalidFormat]
parseCSV = \line ->
    when Str.splitFirst line "," is
        Ok { before, after } -> Ok { name: Str.trim before, scoreStr: Str.trim after }
        Err _ -> Err InvalidFormat

parseScore : Str -> Result I64 [InvalidScore, ScoreTooLow]
parseScore = \s ->
    when Str.toI64 s is
        Err _ -> Err InvalidScore
        Ok n ->
            if n < 0 then Err ScoreTooLow
            else Ok n

classify : I64 -> Str
classify = \score ->
    if score >= 90 then "A"
    else if score >= 80 then "B"
    else if score >= 70 then "C"
    else "D"

main =
    lines = ["An, 95", "Bình, 82", "bad-format", "Cường, abc", "Dũng, -5"]

    List.forEach lines \line ->
        when processCSVLine line is
            Ok result -> Stdout.line! result
            Err InvalidFormat -> Stdout.line! "❌ \"$(line)\": sai format (cần name,score)"
            Err InvalidScore -> Stdout.line! "❌ \"$(line)\": điểm không phải số"
            Err ScoreTooLow -> Stdout.line! "❌ \"$(line)\": điểm âm!"
    # Output:
    # An: 95 điểm → A
    # Bình: 82 điểm → B
    # ❌ "bad-format": sai format (cần name,score)
    # ❌ "Cường, abc": điểm không phải số
    # ❌ "Dũng, -5": điểm âm!
```

---

## 13.6 — Error Hierarchy: Thiết kế tag unions cho lỗi

Trong dự án thực tế, lỗi không chỉ là một cái tag đơn giản. Bạn cần phân loại: lỗi validation, lỗi database, lỗi authentication — mỗi loại cần xử lý khác nhau. Tag unions là công cụ hoàn hảo cho việc này.

### Tag unions = hoàn hảo cho error types

```roc
# Lỗi cụ thể theo domain
ValidationError : [
    EmptyField Str,              # field nào rỗng
    TooShort { field : Str, min : U64 },
    TooLong { field : Str, max : U64 },
    InvalidFormat { field : Str, expected : Str },
]

DatabaseError : [
    ConnectionFailed Str,
    NotFound { table : Str, id : U64 },
    DuplicateKey { table : Str, key : Str },
]

# Kết hợp → Application Error
AppError : [
    Validation ValidationError,
    Database DatabaseError,
    Unauthorized,
    InternalError Str,
]
```

### Ví dụ thực tế: User registration

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Validate từng field — lỗi cụ thể
validateUsername : Str -> Result Str [TooShort, TooLong, HasSpaces]
validateUsername = \raw ->
    trimmed = Str.trim raw
    len = Str.countUtf8Bytes trimmed
    if len < 3 then Err TooShort
    else if len > 20 then Err TooLong
    else if Str.contains trimmed " " then Err HasSpaces
    else Ok trimmed

validatePassword : Str -> Result Str [PasswordTooShort, NoDigit]
validatePassword = \raw ->
    if Str.countUtf8Bytes raw < 8 then Err PasswordTooShort
    else
        bytes = Str.toUtf8 raw
        hasDigit = List.any bytes \b -> b >= 48 && b <= 57
        if hasDigit then Ok raw
        else Err NoDigit

# Pipeline dùng ?
register : Str, Str -> Result { username : Str, password : Str } _
register = \rawUser, rawPass ->
    username = validateUsername? rawUser
    password = validatePassword? rawPass
    Ok { username, password }

# Error → user-friendly message
errorMessage : _ -> Str
errorMessage = \err ->
    when err is
        TooShort -> "Username quá ngắn (tối thiểu 3 ký tự)"
        TooLong -> "Username quá dài (tối đa 20 ký tự)"
        HasSpaces -> "Username không được có khoảng trắng"
        PasswordTooShort -> "Mật khẩu quá ngắn (tối thiểu 8 ký tự)"
        NoDigit -> "Mật khẩu phải có ít nhất 1 chữ số"

main =
    testCases = [
        ("An", "short"),
        ("Good_User", "password1"),
        ("a", "validpass1"),
        ("Has Space", "validpass1"),
        ("Good_User", "nodigits"),
    ]

    List.forEach testCases \(user, pass) ->
        when register user pass is
            Ok account ->
                Stdout.line! "✅ $(account.username) đăng ký thành công!"
            Err err ->
                Stdout.line! "❌ $(user): $(errorMessage err)"
    # Output:
    # ❌ An: Mật khẩu quá ngắn (tối thiểu 8 ký tự)
    # ✅ Good_User đăng ký thành công!
    # ❌ a: Username quá ngắn (tối thiểu 3 ký tự)
    # ❌ Has Space: Username không được có khoảng trắng
    # ❌ Good_User: Mật khẩu phải có ít nhất 1 chữ số
```

---

## 13.7 — Collecting Errors: Báo tất cả lỗi cùng lúc

Đôi khi bạn muốn báo **tất cả** lỗi, không chỉ cái đầu tiên:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Validate tất cả fields, thu thập errors
validateForm : { name : Str, email : Str, age : Str } -> Result { name : Str, email : Str, age : U8 } (List Str)
validateForm = \form ->
    errors = []

    nameResult = if Str.isEmpty (Str.trim form.name) then Err "Tên không được rỗng" else Ok (Str.trim form.name)
    emailResult = if Str.contains form.email "@" then Ok form.email else Err "Email cần có @"
    ageResult = when Str.toI64 form.age is
        Ok n if n >= 1 && n <= 150 -> Ok (Num.toU8 n)
        Ok _ -> Err "Tuổi phải từ 1-150"
        Err _ -> Err "Tuổi phải là số"

    # Thu thập lỗi
    allErrors =
        []
        |> \es -> when nameResult is Ok _ -> es; Err e -> List.append es e
        |> \es -> when emailResult is Ok _ -> es; Err e -> List.append es e
        |> \es -> when ageResult is Ok _ -> es; Err e -> List.append es e

    if List.isEmpty allErrors then
        # Mọi field hợp lệ
        when (nameResult, emailResult, ageResult) is
            (Ok name, Ok email, Ok age) -> Ok { name, email, age }
            _ -> Err ["Unexpected error"]
    else
        Err allErrors

main =
    when validateForm { name: "", email: "bad", age: "abc" } is
        Ok _ -> Stdout.line! "OK"
        Err errors ->
            Stdout.line! "❌ Lỗi:"
            List.forEach errors \e ->
                Stdout.line! "  • $(e)"
    # Output:
    # ❌ Lỗi:
    #   • Tên không được rỗng
    #   • Email cần có @
    #   • Tuổi phải là số
```

---


## ✅ Checkpoint 13

> Đến đây bạn phải hiểu:
> 1. `Result ok err` = thành công hoặc thất bại — **compiler bắt xử lý cả hai**
> 2. `Result.try` = chain kết quả, dừng ở error đầu tiên (Railway pattern)
> 3. Roc không có exceptions — mọi error đều explicit trong type
>
> **Test nhanh**: Roc có `try/catch` không?
> <details><summary>Đáp án</summary>Không! Roc dùng `Result ok err` — errors explicit trong type system.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Safe math pipeline

Viết pipeline tính: `parse → multiply → divide → format`:

```roc
# safeMath "10" "3" "2"
# → parse 10, 3, 2 → 10 * 3 = 30 → 30 / 2 = 15 → "Kết quả: 15"
# safeMath "abc" "3" "2" → Err ParseError
# safeMath "10" "3" "0" → Err DivisionByZero
```

<details><summary>✅ Lời giải</summary>

```roc
safeMath : Str, Str, Str -> Result Str [ParseError, DivisionByZero]
safeMath = \aStr, bStr, cStr ->
    a = Str.toI64 aStr |> Result.mapErr \_ -> ParseError
    |> Result.try \aVal ->
        b = Str.toI64 bStr |> Result.mapErr \_ -> ParseError
        |> Result.try \bVal ->
            c = Str.toI64 cStr |> Result.mapErr \_ -> ParseError
            |> Result.try \cVal ->
                product = aVal * bVal
                if cVal == 0 then Err DivisionByZero
                else Ok "Kết quả: $(Num.toStr (product // cVal))"

# Hoặc dùng ?
safeMath2 : Str, Str, Str -> Result Str [ParseError, DivisionByZero]
safeMath2 = \aStr, bStr, cStr ->
    a = Str.toI64 aStr |> Result.mapErr? \_ -> ParseError
    b = Str.toI64 bStr |> Result.mapErr? \_ -> ParseError
    c = Str.toI64 cStr |> Result.mapErr? \_ -> ParseError
    if c == 0 then Err DivisionByZero
    else Ok "Kết quả: $(Num.toStr (a * b // c))"
```

</details>

---

**Bài 2** (15 phút): Order validation system

Validate đơn hàng với nhiều bước:

```roc
# validateOrder:
# 1. items không rỗng → Err EmptyOrder
# 2. mỗi item.quantity > 0 → Err InvalidQuantity item.name
# 3. total > 0 → Err ZeroTotal
# 4. Trả Ok { items, total, itemCount }
```

<details><summary>✅ Lời giải</summary>

```roc
validateOrder = \order ->
    if List.isEmpty order.items then
        Err EmptyOrder
    else
        invalidItem = List.findFirst order.items \i -> i.quantity <= 0
        when invalidItem is
            Ok item -> Err (InvalidQuantity item.name)
            Err NotFound ->
                total = List.walk order.items 0 \s, i -> s + (i.price * i.quantity)
                if total <= 0 then Err ZeroTotal
                else Ok { items: order.items, total, itemCount: List.len order.items }
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `?` không hoạt động | Function không trả `Result` | Function phải return `Result ok err` |
| Error type mismatch | Các bước trả `Err` types khác nhau | Dùng `Result.mapErr` để thống nhất error type |
| `This when doesn't cover all patterns` | Quên xử lý một Err variant | Thêm branch cho error bị thiếu |
| Railway dừng quá sớm | `?` đúng hoạt động — dừng tại lỗi đầu tiên | Nếu cần tất cả lỗi → dùng collect pattern (13.7) |

---

## Tóm tắt

- ✅ **`Result ok err`** = cách duy nhất xử lý lỗi. Không exceptions, không crash bất ngờ.
- ✅ **`?` operator** = "nếu Ok lấy ra, nếu Err dừng function". Ngắn gọn, rõ ý.
- ✅ **Railway-oriented programming** = chuỗi operations trên 2 track (Ok/Err). Lỗi → tự động bỏ qua bước sau.
- ✅ **`Result.map`** = biến đổi Ok. **`Result.mapErr`** = biến đổi Err. **`Result.withDefault`** = fallback.
- ✅ **Error hierarchy** = dùng tag unions — `[TooShort, TooLong, HasSpaces]`. Lỗi cụ thể → message rõ ràng.
- ✅ **Collect pattern** = thu thập tất cả lỗi thay vì dừng ở lỗi đầu tiên.

## Tiếp theo

→ Chapter 14: **Abilities** — Roc's answer to type classes/traits/interfaces. `Eq`, `Hash`, `Inspect`, custom abilities, và cách chúng cho phép polymorphism mà không cần class hierarchy.
