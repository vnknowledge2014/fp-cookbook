# Chapter 4 — Getting Started

> **Bạn sẽ học được**:
> - Cài đặt Roc và chạy chương trình đầu tiên
> - Hiểu **platform model** — điều khiến Roc khác biệt mọi ngôn ngữ khác
> - Cấu trúc file `.roc` — `app`, `import`, `main`
> - Dùng REPL để thử nghiệm nhanh
> - Các lỗi thường gặp khi mới bắt đầu
>
> **Yêu cầu trước**: Biết dùng terminal (command line)
> **Thời gian đọc**: ~30 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Viết và chạy được chương trình Roc từ đầu, hiểu tại sao Roc cần "platform".

---

## 4.1 — Cài đặt Roc

Roc vẫn đang phát triển — cài đặt từ pre-built binary hoặc build từ source.

### macOS / Linux

```bash
# Tải và cài Roc
curl -fsSL https://roc-lang.org/install.sh | bash

# Kiểm tra
roc version
# Output: roc 0.x.x
```

### Windows

Roc hỗ trợ Windows qua WSL (Windows Subsystem for Linux). Cài WSL trước, rồi làm theo hướng dẫn Linux ở trên.

### Kiểm tra cài đặt

```bash
# Tạo file test nhanh
echo 'app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    Stdout.line! "Roc đã sẵn sàng! 🎉"' > test.roc

roc run test.roc
# Output: Roc đã sẵn sàng! 🎉
```

Nếu thấy output, bạn đã cài đặt thành công.

> **💡 Tip**: Roc tự tải platform lần chạy đầu tiên (có thể mất vài giây). Lần sau sẽ nhanh hơn vì đã cache.

---

## 4.2 — Platform Model: Điều khiến Roc đặc biệt

Platform model tách app code (pure logic) khỏi system code (IO). App viết business logic, platform cung cấp IO capabilities.

### Vấn đề của các ngôn ngữ khác

Trong Python, JavaScript, Rust — bạn gọi IO trực tiếp:

```python
# Python — gọi IO bất kỳ lúc nào
file = open("data.txt")     # đọc file
response = requests.get(url) # gọi HTTP
print("hello")              # in ra terminal
```

Bất kỳ dòng code nào cũng có thể **đọc file, gọi HTTP, xóa database**. Đây là nguồn gốc của bugs khó tìm — bạn không biết function nào có side effects.

### Cách Roc giải quyết: App + Platform

Roc tách chương trình thành **hai phần**:

```
┌─────────────────────────────────────┐
│            Your App (pure)          │  ← Bạn viết
│  Logic thuần túy, không IO          │
│  Records, tags, functions, pipelines│
├─────────────────────────────────────┤
│          Platform (handles IO)      │  ← Platform lo
│  Đọc file, HTTP, terminal, database│
│  basic-cli, basic-webserver, ...    │
└─────────────────────────────────────┘
```

**App** = logic của bạn. 100% pure — không đọc file, không gọi HTTP, không in ra terminal trực tiếp. Chỉ nhận data, xử lý, trả kết quả.

**Platform** = "người chạy bài" cho app. Platform cung cấp các khả năng IO (đọc file, HTTP, terminal). App nói "tôi muốn in cái này", platform thực hiện.

### Tại sao tách ra?

1. **Code bạn viết = pure** → dễ test, dễ đọc, dễ debug
2. **Side effects bị kiểm soát** → biết chính xác chỗ nào có IO
3. **Đổi platform = đổi môi trường** → cùng logic, chạy trên CLI hoặc web server
4. **Bảo mật** → app KHÔNG THỂ truy cập IO ngoài những gì platform cho phép

> **💡 Ẩn dụ**: App giống **đầu bếp** — chỉ biết nấu. Platform giống **nhà hàng** — lo bàn ghế, bếp, phục vụ. Đầu bếp nấu cùng một món, có thể làm ở nhà hàng 5 sao hay quán cơm bình dân — chỉ thay "platform".

### Các platform phổ biến

| Platform | Dùng cho | Cung cấp |
|----------|----------|----------|
| `basic-cli` | CLI apps | Terminal, file, HTTP client, env vars |
| `basic-webserver` | Web APIs | HTTP server, routing, request/response |
| `basic-ssg` | Static sites | File reading, HTML generation |

Trong cuốn sách này, chúng ta chủ yếu dùng **`basic-cli`**.

---

## 4.3 — Cấu trúc file Roc

Mỗi Roc app bắt đầu bằng `app [main]` declaration — chỉ định entry point và platform dependency.

### File đầy đủ

```roc
# filename: main.roc

# Dòng 1: Khai báo app — "đây là app, entry point là main, dùng platform basic-cli"
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

# Dòng 2: Import — lấy module từ platform để dùng
import cli.Stdout

# Dòng 3+: Code chính
main =
    Stdout.line! "Hello from Roc!"
```

Phân tích từng phần:

### `app [main] { cli: platform "..." }`

```
app [main] { cli: platform "url" }
 │    │        │        │      │
 │    │        │        │      └── URL tải platform
 │    │        │        └── đây là platform (không phải package bình thường)
 │    │        └── tên ngắn cho platform (dùng trong import)
 │    └── danh sách entry points (ở đây chỉ có main)
 └── đây là application (không phải module)
```

### `import cli.Stdout`

```
import cli.Stdout
        │    │
        │    └── Module Stdout (cung cấp hàm in ra terminal)
        └── Tên platform (đã khai báo ở dòng app)
```

Sau khi import, bạn gọi `Stdout.line!` — hàm in 1 dòng ra terminal.

### `main =`

Entry point — nơi chương trình bắt đầu chạy. Platform sẽ gọi `main` khi khởi động.

---

## 4.4 — Chương trình đầu tiên: Từng bước

Hello World trong Roc — từ tạo file đến chạy chương trình. Mỗi dòng code được giải thích.

### Bước 1: Tạo file

```bash
mkdir my-first-roc-app
```

Tạo file `main.roc`:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    Stdout.line! "=== Máy tính đơn giản ==="

    a = 15
    b = 4

    Stdout.line! "$(Num.toStr a) + $(Num.toStr b) = $(Num.toStr (a + b))"
    Stdout.line! "$(Num.toStr a) - $(Num.toStr b) = $(Num.toStr (a - b))"
    Stdout.line! "$(Num.toStr a) × $(Num.toStr b) = $(Num.toStr (a * b))"
    Stdout.line! "$(Num.toStr a) ÷ $(Num.toStr b) = $(Num.toStr (a // b))"
```

### Bước 2: Chạy

```bash
roc run main.roc
# Output:
# === Máy tính đơn giản ===
# 15 + 4 = 19
# 15 - 4 = 11
# 15 × 4 = 60
# 15 ÷ 4 = 3
```

### Bước 3: Thêm logic

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Helper function — thuần túy, không IO
greet = \name, role ->
    when role is
        Student -> "📚 Chào bạn $(name)! Chúc học tốt."
        Teacher -> "🎓 Chào thầy/cô $(name)!"
        Guest -> "👋 Xin chào $(name)!"

main =
    Stdout.line! (greet "An" Student)
    Stdout.line! (greet "Minh" Teacher)
    Stdout.line! (greet "visitor" Guest)
    # Output:
    # 📚 Chào bạn An! Chúc học tốt.
    # 🎓 Chào thầy/cô Minh!
    # 👋 Xin chào visitor!
```

Chú ý: `greet` là **pure function** — không cần `import`, không cần platform. Nó chỉ nhận data, trả data. `Stdout.line!` — dấu `!` nói rằng đây là **side effect** (Task).

---

## 4.5 — REPL: Thử nghiệm nhanh

Roc có REPL (Read-Eval-Print Loop) để thử code nhanh mà không cần tạo file:

```bash
roc repl
```

```
» 1 + 2
3 : Num *

» "Hello" |> Str.concat " World"
"Hello World" : Str

» List.map [1, 2, 3] \x -> x * 10
[10, 20, 30] : List (Num *)

» addOne = \x -> x + 1
<function> : Num a -> Num a

» addOne 41
42 : Num *
```

REPL rất hữu ích khi bạn muốn:
- Kiểm tra nhanh một biểu thức
- Xem type của một giá trị
- Thử nghiệm function trước khi đưa vào file

> **💡 Tip**: REPL hiển thị **type** sau dấu `:`. Ví dụ `3 : Num *` nghĩa là "3 là một Num (số)".

---

## 4.6 — Các lệnh Roc hữu ích

`roc run`, `roc test`, `roc check`, `roc format` — 4 lệnh bạn sẽ dùng mỗi ngày.

```bash
# Chạy chương trình
roc run main.roc

# Chỉ kiểm tra type — không chạy (nhanh hơn)
roc check main.roc

# Build ra binary
roc build main.roc
# → tạo file ./main (có thể chạy trực tiếp: ./main)

# Build tối ưu hiệu năng
roc build --optimize main.roc

# REPL
roc repl

# Xem help
roc --help
```

| Lệnh | Dùng khi | Tốc độ |
|-------|----------|--------|
| `roc run` | Phát triển, test nhanh | Nhanh (không optimize) |
| `roc check` | Kiểm tra lỗi type | Rất nhanh (không compile) |
| `roc build` | Build binary để deploy | Chậm hơn |
| `roc build --optimize` | Production | Chậm nhất, binary nhanh nhất |

---

## 4.7 — Dấu `!` trong Roc: Task notation

Bạn đã thấy `Stdout.line!` — dấu `!` ở cuối. Đây là **Task notation** — cú pháp ngắn gọn cho side effects.

```roc
# Dấu ! = "thực hiện side effect này, rồi tiếp tục"
main =
    Stdout.line! "Bước 1"    # in → tiếp tục
    Stdout.line! "Bước 2"    # in → tiếp tục
    Stdout.line! "Bước 3"    # in → kết thúc
```

Không có `!`, bạn phải viết dài hơn (Task.await — sẽ học kỹ ở Chapter 12):

```roc
# Không dùng ! — dài hơn nhưng cùng ý nghĩa
main =
    Task.await (Stdout.line "Bước 1") \_ ->
    Task.await (Stdout.line "Bước 2") \_ ->
    Stdout.line "Bước 3"
```

Bây giờ chỉ cần nhớ: **`!` = thực hiện IO, rồi tiếp tục**. Chi tiết → Chapter 12.

---


## ✅ Checkpoint 4

> Đến đây bạn phải hiểu:
> 1. `roc run` = biên dịch + chạy, `roc test` = chạy `expect` tests
> 2. Mọi Roc app cần khai báo platform (`basic-cli`, `basic-webserver`)
> 3. `!` suffix = effectful call (đọc file, in ra màn hình)
>
> **Test nhanh**: `Stdout.line! "hello"` — dấu `!` nghĩa là gì?
> <details><summary>Đáp án</summary>Đây là effectful call — thực hiện side effect (in ra terminal). Không có `!` = pure function.</details>

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Hello World mở rộng

Viết chương trình in ra 3 dòng:
- Tên bạn
- Năm sinh
- Ngôn ngữ lập trình yêu thích

<details><summary>✅ Lời giải</summary>

```roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    name = "An"
    birthYear = 2000
    favLang = "Roc"

    Stdout.line! "Tên: $(name)"
    Stdout.line! "Năm sinh: $(Num.toStr birthYear)"
    Stdout.line! "Ngôn ngữ yêu thích: $(favLang)"
```

</details>

---

**Bài 2** (10 phút): Bảng cửu chương

Viết chương trình in bảng cửu chương cho số 7:

```
7 × 1 = 7
7 × 2 = 14
...
7 × 10 = 70
```

<details><summary>💡 Gợi ý</summary>

Dùng `List.forEach` với list `[1, 2, 3, ..., 10]`. Tạo list bằng `List.range { start: At 1, end: At 10 }`.

</details>

<details><summary>✅ Lời giải</summary>

```roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    n = 7
    Stdout.line! "=== Bảng cửu chương $(Num.toStr n) ==="

    List.forEach (List.range { start: At 1, end: At 10 }) \i ->
        result = n * i
        Stdout.line! "$(Num.toStr n) × $(Num.toStr i) = $(Num.toStr result)"
```

</details>

---

**Bài 3** (15 phút): Mini calculator

Viết chương trình tính:
- Diện tích hình chữ nhật (dài × rộng)
- Diện tích hình tròn (π × r²)
- Chu vi hình tròn (2 × π × r)

Dùng function riêng cho mỗi phép tính.

<details><summary>✅ Lời giải</summary>

```roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

pi = 3.14159

rectangleArea = \length, width -> length * width
circleArea = \radius -> pi * radius * radius
circleCircumference = \radius -> 2.0 * pi * radius

main =
    Stdout.line! "=== Máy tính hình học ==="

    rArea = rectangleArea 5.0 3.0
    Stdout.line! "HCN 5×3: diện tích = $(Num.toStr rArea)"

    cArea = circleArea 4.0
    Stdout.line! "Tròn r=4: diện tích = $(Num.toStr cArea)"

    cCirc = circleCircumference 4.0
    Stdout.line! "Tròn r=4: chu vi = $(Num.toStr cCirc)"
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `roc: command not found` | Chưa cài Roc hoặc chưa thêm vào PATH | Cài lại theo hướng dẫn ở 4.1 |
| `Could not find platform` | URL platform sai hoặc không có internet | Kiểm tra URL, kiểm tra kết nối mạng |
| `Indentation error` | Thụt lề không nhất quán | Roc dùng **4 spaces** — không dùng tab |
| `I was expecting...` | Thiếu dòng `import` | Thêm `import cli.Stdout` sau dòng `app` |
| Chương trình không in gì | Quên dấu `!` sau `Stdout.line` | Phải viết `Stdout.line!` (có dấu chấm than) |

---

## Tóm tắt

- ✅ **Cài Roc**: `curl -fsSL https://roc-lang.org/install.sh | bash`
- ✅ **Platform model**: App (pure logic) + Platform (handles IO) — tách biệt hoàn toàn
- ✅ **Cấu trúc file**: `app [main] { cli: platform "..." }` → `import cli.Stdout` → `main = ...`
- ✅ **`roc run`** để chạy, **`roc check`** để kiểm tra type nhanh, **`roc repl`** để thử nghiệm
- ✅ **Dấu `!`** = thực hiện side effect (Task). `Stdout.line!` in ra terminal

## Tiếp theo

→ Chapter 5: **Values & Types** — tìm hiểu sâu về hệ thống types: numbers, strings, booleans, structural typing, type inference. Bạn sẽ hiểu tại sao Roc gần như không bao giờ cần bạn ghi type annotation.
