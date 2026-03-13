# Chapter 0 — Roc in 10 Minutes

> **Mục tiêu**: Biết đủ syntax Roc để **đọc hiểu** code examples trong Part 0 (CS Foundations).
> **Đây KHÔNG phải chapter dạy Roc** — Part I sẽ dạy lại mọi thứ sâu hơn.
>
> **Thời gian đọc**: ~10 phút | **Level**: Quick Primer
> **Sau chapter này**: Bạn có thể đọc hiểu mọi code example trong Chapter 1–3.

---

## Cài đặt & chạy thử

Cài Roc từ [roc-lang.org](https://www.roc-lang.org/install). Kiểm tra:

```bash
roc version
# Output: roc 0.x.x
```

Tạo file `main.roc`:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    Stdout.line! "Xin chào! 🎉"
```

Chạy:

```bash
roc run main.roc
# Output: Xin chào! 🎉
```

> **Giải thích nhanh dòng `app [main] { cli: platform "..." }`**:
> Roc tách chương trình thành **App** (logic của bạn) và **Platform** (xử lý IO). Dòng này nói: "Đây là app, entry point là `main`, dùng platform `basic-cli` để in ra terminal." Chi tiết → Chapter 4.

---

## Giá trị & kiểu cơ bản

Roc là statically typed — mọi giá trị có type cố định, compiler kiểm tra trước khi chạy.

```roc
# Số
age = 25          # I64 (số nguyên)
price = 19.99     # F64 (số thực)

# Chuỗi — dùng dấu ngoặc kép
name = "An"

# Boolean
isActive = Bool.true

# String interpolation — dùng $(...)
greeting = "Chào $(name), bạn $(Num.toStr age) tuổi"
```

Roc **không có `let`** — bạn viết thẳng `tên = giá trị`. Mọi giá trị đều **immutable** (không thay đổi được).

---

## Functions

Functions trong Roc là lambdas — viết bằng `\args -> body`. Không có keyword `def` hay `func`.

```roc
# Function cơ bản — dùng \ (lambda)
addOne = \x -> x + 1

# Gọi function — không cần dấu ngoặc!
result = addOne 5     # → 6

# Function nhiều tham số
add = \a, b -> a + b
sum = add 3 5         # → 8
```

Chỉ cần nhớ: `\tham_số -> kết_quả`. Dấu `\` = "đây là function".

---

## Records (giống struct/object)

Records nhóm data liên quan — giống struct trong Rust, object literal trong JavaScript.

```roc
# Record = nhóm các giá trị liên quan
user = { name: "An", age: 25, isActive: Bool.true }

# Truy cập field bằng dấu chấm
user.name      # → "An"
user.age       # → 25
```

**Không cần khai báo type trước** — cứ viết, compiler tự suy ra. Đây là **structural typing**.

---

## Tags (giống enum variants)

Tags đại diện cho các trường hợp khác nhau — giống enum variants trong Rust, union types trong TypeScript.

```roc
# Tags = giá trị có tên, bắt đầu bằng chữ HOA
status = Confirmed
color = Red

# Tags có thể chứa data
payment = Cash 50000
error = Err NotFound
success = Ok 42
```

**Cũng không cần khai báo trước** — cứ viết `Confirmed`, compiler hiểu.

---

## Pattern Matching — `when...is`

Đây là cách Roc xử lý "nếu/thì". Compiler **bắt buộc** xử lý mọi trường hợp:

```roc
describeLight = \light ->
    when light is
        Red -> "Dừng lại"
        Yellow -> "Chuẩn bị"
        Green -> "Đi"

# Gọi:
describeLight Red     # → "Dừng lại"
describeLight Green   # → "Đi"
```

Nếu bạn quên xử lý một trường hợp (ví dụ bỏ `Yellow`), compiler sẽ báo lỗi.

---

## Lists & xử lý data

Lists là collection chính trong Roc. Xử lý bằng `map`, `keepIf`, `walk` — không có vòng lặp.

```roc
# List — danh sách giá trị cùng kiểu
numbers = [1, 2, 3, 4, 5]

# List.map — áp dụng function cho mỗi phần tử
doubled = List.map numbers \x -> x * 2
# → [2, 4, 6, 8, 10]

# List.keepIf — lọc
evens = List.keepIf numbers \x -> x % 2 == 0
# → [2, 4]

# List.walk — gom lại (giống fold/reduce)
total = List.walk numbers 0 \state, x -> state + x
# → 15
```

---

## In ra terminal

IO trong Roc dùng Tasks. Dấu `!` = thực thi side effect ngay.

```roc
# Dùng Stdout.line! để in
main =
    Stdout.line! "Số 5 cộng 1 = $(Num.toStr (addOne 5))"

# Num.toStr chuyển số thành chuỗi
# $(...) chèn giá trị vào chuỗi
```

---

## Result — xử lý lỗi

Không exceptions — mọi lỗi explicit qua `Result ok err`. Compiler bắt xử lý.

```roc
# Ok = thành công, Err = thất bại
safeDivide = \a, b ->
    if b == 0 then
        Err DivisionByZero
    else
        Ok (a / b)

# Compiler BẮT BUỘC xử lý cả 2:
main =
    when safeDivide 10 3 is
        Ok result -> Stdout.line! "Kết quả: $(Num.toStr result)"
        Err DivisionByZero -> Stdout.line! "Lỗi: chia cho 0!"
```

---

## Inline Testing — `expect`

Roc có tính năng **test ngay trong code** — không cần file test riêng:

```roc
# Viết function
add = \a, b -> a + b

# Test ngay cạnh code!
expect add 2 3 == 5
expect add 0 0 == 0
expect add (-1) 1 == 0
```

Chạy test: `roc test main.roc`. Nếu mọi `expect` đều đúng → pass. Sai → Roc chỉ rõ dòng nào fail.

> **💡 Pro**: Trong Chapter 1-3, bạn sẽ thấy `expect` ở cuối mỗi ví dụ. Đây là cách Roc developers viết test — inline, gọn, chạy bằng `roc test`.

---

## Cheat Sheet — Roc vs ngôn ngữ khác

Bảng so sánh nhanh giúp bạn map kiến thức đã có sang Roc.

| Khái niệm | Roc | Rust | Python |
|---|---|---|---|
| Biến | `x = 5` | `let x = 5;` | `x = 5` |
| Function | `\x -> x + 1` | `\|x\| x + 1` | `lambda x: x + 1` |
| Record | `{ name: "An" }` | `struct` (cần khai báo) | `dict` |
| Tag | `Ok 42` | `enum` (cần khai báo) | — |
| Pattern match | `when x is` | `match x` | `match x:` (3.10+) |
| Print | `Stdout.line!` | `println!` | `print()` |
| String interpolation | `"$(expr)"` | `format!("{}", x)` | `f"{x}"` |
| Inline test | `expect a == b` | `#[test]` (file riêng) | `assert` |

---

## Đủ rồi! Bạn đã sẵn sàng cho Chapter 1

Bạn bây giờ có thể **đọc hiểu** mọi code example trong Part 0 (Chapter 1–3). Khi gặp syntax lạ, quay lại chapter này tra cứu.

**Part I (Chapter 4–10)** sẽ dạy lại mọi thứ chi tiết hơn — functions, records, tags, pattern matching, modules — nên đừng lo nếu chưa nhớ hết.

→ **Tiếp theo**: Chapter 1 — Math Foundations for FP
