# Chapter 24 — CLI Applications

> **Bạn sẽ học được**:
> - `basic-cli` platform — setup và cấu trúc project
> - **Argument parsing** — `Arg.list`, flags, subcommands
> - **File I/O** — đọc/ghi/copy/move files
> - **HTTP requests** — GET/POST từ CLI
> - **Interactive menus** — loop input, menu selection
> - Xây CLI app hoàn chỉnh: expense tracker
>
> **Yêu cầu trước**: [Chapter 23 — PBT Concepts](chapter_23_pbt_concepts.md)
> **Thời gian đọc**: ~45 phút | **Level**: Principal
> **Kết quả cuối cùng**: Xây được CLI tool thực tế — parse args, đọc/ghi file, tương tác user.

---

Lý thuyết đủ rồi — giờ xây ứng dụng thật. CLI application là cách nhanh nhất để biến kiến thức Roc thành sản phẩm: đọc arguments, xử lý files, output kết quả. Platform `basic-cli` cung cấp mọi thứ bạn cần.

## CLI Applications — Ứng dụng Roc đầu tiên

Đây là chapter thực hành — xây dựng CLI tool hoàn chỉnh. Bạn sẽ dùng `roc-cli` platform, parse arguments, đọc files, và output results. Mọi concept từ Part I-IV kết hợp lại.


## 24.1 — basic-cli Platform

Platform `basic-cli` là cách Roc nói chuyện với thế giới bên ngoài: đọc stdin, ghi stdout, xử lý files, gọi HTTP. Mọi IO trong CLI app đi qua platform này.

### Setup

```roc
# filename: main.roc
app [main] {
    cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br"
}

# Available modules:
import cli.Stdout      # in ra terminal
import cli.Stderr      # in lỗi
import cli.Stdin       # đọc input
import cli.File        # đọc/ghi file
import cli.Path        # path manipulation
import cli.Env         # environment variables
import cli.Arg         # command line arguments
import cli.Http        # HTTP requests
import cli.Dir         # directory operations
import cli.Sleep       # delay
```

### Build & Run

```bash
# Chạy trực tiếp
roc run main.roc

# Chạy với arguments
roc run main.roc -- arg1 arg2

# Build binary
roc build main.roc
./main arg1 arg2

# Run tests
roc test main.roc
```

---

## 24.2 — Argument Parsing

Chương trình CLI nhận input từ command line arguments: `myapp --output result.txt input.csv`. Parsing arguments theo cách FP — pattern matching trên list strings.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Arg

# ═══════ PURE: Parse arguments ═══════

parseArgs : List Str -> Result { command : Str, args : List Str } [NoCommand, ShowHelp]
parseArgs = \allArgs ->
    # args[0] = program name, args[1] = command, args[2..] = command args
    when List.get allArgs 1 is
        Err _ -> Err NoCommand
        Ok cmd ->
            if cmd == "--help" || cmd == "-h" then
                Err ShowHelp
            else
                Ok { command: cmd, args: List.dropFirst allArgs 2 }

helpText =
    """
    Usage: mytool <command> [args...]

    Commands:
      greet <name>     Chào ai đó
      count <n>        Đếm từ 1 đến n
      upper <text>     Chuyển thành chữ hoa
      help             Hiện trợ giúp
    """

# ═══════ SHELL ═══════

main =
    args = Arg.list!

    when parseArgs args is
        Err ShowHelp ->
            Stdout.line! helpText
        Err NoCommand ->
            Stdout.line! "❌ Thiếu command. Dùng --help để xem hướng dẫn."
        Ok { command, args: cmdArgs } ->
            when command is
                "greet" ->
                    name = List.get cmdArgs 0 |> Result.withDefault "World"
                    Stdout.line! "👋 Xin chào, $(name)!"
                "count" ->
                    nStr = List.get cmdArgs 0 |> Result.withDefault "10"
                    n = Str.toU64 nStr |> Result.withDefault 10
                    numbers = List.range { start: At 1, end: At n }
                    List.forEach numbers \i ->
                        Stdout.line! "$(Num.toStr i)"
                "upper" ->
                    text = Str.joinWith cmdArgs " "
                    if Str.isEmpty text then
                        Stdout.line! "❌ Cần text. Ví dụ: mytool upper hello world"
                    else
                        Stdout.line! (Str.toUppercase text)
                _ ->
                    Stdout.line! "❌ Command không tồn tại: $(command)"
```

```bash
roc run main.roc -- greet An
# 👋 Xin chào, An!

roc run main.roc -- count 5
# 1 2 3 4 5

roc run main.roc -- upper hello world
# HELLO WORLD
```

---

## 24.3 — File Operations

Đọc file, ghi file, kiểm tra file tồn tại — những operations cơ bản nhất. Tất cả đều trả `Task` với error types cụ thể — không bao giờ crash bất ngờ.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.File
import cli.Path

# ═══════ FILE UTILITIES ═══════

# Đọc file an toàn
readFileSafe : Str -> Task (Result Str Str) _
readFileSafe = \filename ->
    result = File.readUtf8 (Path.fromStr filename) |> Task.attempt!
    when result is
        Ok content -> Task.ok (Ok content)
        Err _ -> Task.ok (Err "Không đọc được file: $(filename)")

# Đếm thông tin file
countFileStats = \content ->
    lines = Str.split content "\n"
    lineCount = List.len lines
    wordCount = Str.split content " "
        |> List.keepIf \w -> !(Str.isEmpty (Str.trim w))
        |> List.len
    byteCount = Str.countUtf8Bytes content
    { lines: lineCount, words: wordCount, bytes: byteCount }

# Tìm và thay thế
findAndReplace = \content, find, replace ->
    Str.replaceEach content find replace

main =
    # Ghi file
    File.writeUtf8! (Path.fromStr "sample.txt")
        """
        Xin chào từ Roc!
        Đây là dòng thứ hai.
        Và dòng thứ ba.
        """
    Stdout.line! "✅ Đã ghi sample.txt"

    # Đọc và phân tích
    content = File.readUtf8! (Path.fromStr "sample.txt")
    stats = countFileStats content
    Stdout.line! "📊 $(Num.toStr stats.lines) dòng, $(Num.toStr stats.words) từ, $(Num.toStr stats.bytes) bytes"

    # Tìm và thay thế → ghi file mới
    replaced = findAndReplace content "Roc" "🚀 Roc"
    File.writeUtf8! (Path.fromStr "sample_edited.txt") replaced
    Stdout.line! "✅ Đã tạo sample_edited.txt"

    # Append
    File.writeUtf8! (Path.fromStr "log.txt") ""
    appendToFile! "log.txt" "[INFO] App started"
    appendToFile! "log.txt" "[INFO] Processing..."
    appendToFile! "log.txt" "[INFO] Done"
    Stdout.line! "✅ Đã ghi log.txt"

appendToFile! = \filename, line ->
    result = File.readUtf8 (Path.fromStr filename) |> Task.attempt!
    existing = when result is
        Ok content -> content
        Err _ -> ""
    newContent = if Str.isEmpty existing then line else "$(existing)\n$(line)"
    File.writeUtf8! (Path.fromStr filename) newContent
```

---

## 24.4 — HTTP Requests

CLI app cũng có thể gọi API bên ngoài. `Http.get` và `Http.send` trả Task — kết hợp với `?` operator để xử lý lỗi network tự nhiên.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Http

main =
    # GET request
    request = {
        method: Get,
        headers: [],
        url: "https://httpbin.org/get",
        mimeType: "",
        body: [],
        timeout: TimeoutMilliseconds 5000,
    }

    result = Http.send request |> Task.attempt!

    when result is
        Ok response ->
            Stdout.line! "Status: $(Num.toStr response.statusCode)"
            bodyStr = response.body |> Str.fromUtf8 |> Result.withDefault "(binary)"
            # In 200 bytes đầu
            preview =
                bytes = Str.toUtf8 bodyStr
                if List.len bytes > 200 then
                    truncated = List.sublist bytes { start: 0, len: 200 }
                        |> Str.fromUtf8
                        |> Result.withDefault "(truncated)"
                    "$(truncated)..."
                else
                    bodyStr
            Stdout.line! "Body preview: $(preview)"
        Err _ ->
            Stdout.line! "❌ HTTP request failed"
```

---

## 24.5 — Interactive CLI: Menu Loop

CLI không chỉ chạy một lần rồi thoát. Menu loop cho phép user chọn action, xử lý, rồi quay lại. Pattern: recursive function + pattern matching trên user input.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Stdin

# ═══════ PURE CORE ═══════

formatContact = \contact ->
    "$(contact.name) — 📞 $(contact.phone) — 📧 $(contact.email)"

searchContacts = \contacts, query ->
    q = Str.toUppercase query
    List.keepIf contacts \c ->
        Str.contains (Str.toUppercase c.name) q
        || Str.contains c.phone query
        || Str.contains (Str.toUppercase c.email) q

# ═══════ EFFECTFUL SHELL ═══════

showMenu =
    """

    ╔══════════════════════════╗
    ║    📇 Contact Manager    ║
    ╠══════════════════════════╣
    ║  1. Xem danh sách        ║
    ║  2. Thêm liên hệ        ║
    ║  3. Tìm kiếm             ║
    ║  4. Thoát                ║
    ╚══════════════════════════╝
    Chọn (1-4):
    """

main =
    Stdout.line! "Chào mừng đến Contact Manager!"
    mainLoop! [
        { name: "An", phone: "0901234567", email: "an@mail.com" },
        { name: "Bình", phone: "0912345678", email: "binh@mail.com" },
    ]

mainLoop! = \contacts ->
    Stdout.line! showMenu
    choice = Stdin.line!

    when choice is
        "1" ->
            # Xem danh sách
            Stdout.line! "\n=== Danh sách ($(Num.toStr (List.len contacts))) ==="
            List.forEachWithIndex contacts \contact, i ->
                Stdout.line! "  $(Num.toStr (i + 1)). $(formatContact contact)"
            mainLoop! contacts

        "2" ->
            # Thêm liên hệ
            Stdout.line! "Tên:"
            name = Stdin.line!
            Stdout.line! "SĐT:"
            phone = Stdin.line!
            Stdout.line! "Email:"
            email = Stdin.line!
            newContacts = List.append contacts { name, phone, email }
            Stdout.line! "✅ Đã thêm $(name)"
            mainLoop! newContacts

        "3" ->
            # Tìm kiếm
            Stdout.line! "Nhập từ khóa:"
            query = Stdin.line!
            results = searchContacts contacts query
            if List.isEmpty results then
                Stdout.line! "🔍 Không tìm thấy"
            else
                Stdout.line! "🔍 Tìm thấy $(Num.toStr (List.len results)):"
                List.forEach results \c ->
                    Stdout.line! "  → $(formatContact c)"
            mainLoop! contacts

        "4" ->
            Stdout.line! "👋 Tạm biệt!"

        _ ->
            Stdout.line! "❓ Chọn 1-4"
            mainLoop! contacts
```

---

## 24.6 — Ví dụ hoàn chỉnh: Expense Tracker

Kết hợp mọi thứ: argument parsing, file IO, interactive menu, domain logic — thành một ứng dụng quản lý chi tiêu hoàn chỉnh.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Stdin
import cli.File
import cli.Path
import cli.Arg

dataFile = "expenses.csv"

# ═══════ PURE CORE ═══════

parseExpenses : Str -> List { category : Str, amount : U64, note : Str }
parseExpenses = \content ->
    Str.split content "\n"
    |> List.dropFirst 1  # skip header
    |> List.keepOks \line ->
        parts = Str.split line "," |> List.map Str.trim
        when (List.get parts 0, List.get parts 1, List.get parts 2) is
            (Ok cat, Ok amtStr, Ok note) ->
                when Str.toU64 amtStr is
                    Ok amt -> Ok { category: cat, amount: amt, note }
                    Err _ -> Err InvalidAmount
            _ -> Err InvalidRow

toCSV : List { category : Str, amount : U64, note : Str } -> Str
toCSV = \expenses ->
    header = "category,amount,note"
    rows = List.map expenses \e ->
        "$(e.category),$(Num.toStr e.amount),$(e.note)"
    [header] |> List.concat rows |> Str.joinWith "\n"

totalByCategory : List { category : Str, amount : U64, note : Str } -> Dict Str U64
totalByCategory = \expenses ->
    List.walk expenses (Dict.empty {}) \dict, e ->
        current = Dict.get dict e.category |> Result.withDefault 0
        Dict.insert dict e.category (current + e.amount)

grandTotal : List { category : Str, amount : U64, note : Str } -> U64
grandTotal = \expenses ->
    List.walk expenses 0 \s, e -> s + e.amount

formatMoney : U64 -> Str
formatMoney = \amount ->
    "$(Num.toStr amount)đ"

# ═══════ EFFECTFUL SHELL ═══════

loadExpenses! : Task (List { category : Str, amount : U64, note : Str }) _
loadExpenses! =
    result = File.readUtf8 (Path.fromStr dataFile) |> Task.attempt!
    when result is
        Ok content -> Task.ok (parseExpenses content)
        Err _ -> Task.ok []

saveExpenses! = \expenses ->
    File.writeUtf8! (Path.fromStr dataFile) (toCSV expenses)

main =
    args = Arg.list!

    when List.get args 1 is
        Ok "add" ->
            # expense add <category> <amount> <note>
            cat = List.get args 2 |> Result.withDefault "other"
            amtStr = List.get args 3 |> Result.withDefault "0"
            note = List.dropFirst args 4 |> Str.joinWith " "
            when Str.toU64 amtStr is
                Ok amount ->
                    expenses = loadExpenses!!
                    newExpenses = List.append expenses { category: cat, amount, note }
                    saveExpenses! newExpenses
                    Stdout.line! "✅ +$(formatMoney amount) [$(cat)] $(note)"
                Err _ ->
                    Stdout.line! "❌ Số tiền không hợp lệ: $(amtStr)"

        Ok "list" ->
            expenses = loadExpenses!!
            if List.isEmpty expenses then
                Stdout.line! "📭 Chưa có chi tiêu nào"
            else
                Stdout.line! "=== Chi tiêu ==="
                List.forEachWithIndex expenses \e, i ->
                    Stdout.line! "  $(Num.toStr (i + 1)). [$(e.category)] $(formatMoney e.amount) — $(e.note)"
                Stdout.line! "───────────────"
                Stdout.line! "Tổng: $(formatMoney (grandTotal expenses))"

        Ok "summary" ->
            expenses = loadExpenses!!
            byCategory = totalByCategory expenses
            Stdout.line! "=== Tổng kết ==="
            Dict.walk byCategory {} \_, cat, total ->
                Stdout.line! "  $(cat): $(formatMoney total)"
            Stdout.line! "───────────────"
            Stdout.line! "TỔNG: $(formatMoney (grandTotal expenses))"

        Ok "clear" ->
            saveExpenses! []
            Stdout.line! "🗑️ Đã xóa tất cả"

        _ ->
            Stdout.line!
                """
                💰 Expense Tracker — Usage:
                  expense add <category> <amount> <note>
                  expense list
                  expense summary
                  expense clear
                
                Example:
                  expense add food 45000 Phở sáng
                  expense add transport 30000 Grab
                  expense summary
                """
```

```bash
roc run main.roc -- add food 45000 Phở sáng
# ✅ +45000đ [food] Phở sáng

roc run main.roc -- add transport 30000 Grab đi làm
# ✅ +30000đ [transport] Grab đi làm

roc run main.roc -- summary
# === Tổng kết ===
#   food: 45000đ
#   transport: 30000đ
# ───────────────
# TỔNG: 75000đ
```

---


## ✅ Checkpoint 24

> Đến đây bạn phải hiểu:
> 1. `basic-cli` platform: `File`, `Stdin`, `Stdout`, `Http`, `Arg`
> 2. Domain logic thuần → gọi platform API khi cần IO
> 3. `!` suffix cho mọi effectful operations
>
> **Test nhanh**: `Arg.list!` trả về kiểu gì?
> <details><summary>Đáp án</summary>`List Str` — danh sách arguments dòng lệnh.</details>

---

## 🏋️ Bài tập

**Bài 1** (15 phút): Note-taking CLI

Viết CLI app quản lý ghi chú:

```bash
note add "Họp team lúc 3h"
note list
note search "họp"
note delete 2
```

<details><summary>💡 Gợi ý</summary>

- Lưu notes vào file `notes.txt`, mỗi dòng 1 note
- `add` = append dòng mới
- `list` = đọc file, in ra với số thứ tự
- `search` = filter lines chứa query
- `delete` = đọc tất cả, bỏ dòng thứ n, ghi lại

</details>

---

**Bài 2** (20 phút): File stats tool

Viết tool phân tích directory:

```bash
fstats ./src
# Files: 12
# Total size: 45.2 KB
# Extensions: .roc (8), .md (3), .txt (1)
# Largest: main.roc (8.5 KB)
```

<details><summary>💡 Gợi ý</summary>

- Dùng `Dir.list` để liệt kê files
- `File.readBytes` để đọc kích thước
- Pure functions: `groupByExtension`, `findLargest`, `formatSize`

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| `Arg.list` trả list rỗng | Chạy `roc run` không có `--` | Dùng `roc run main.roc -- args` |
| File read crash | File chưa tồn tại | Dùng `Task.attempt!` bắt lỗi |
| Interactive loop stack overflow | Recursion quá sâu | Roc có tail-call optimization — thường không vấn đề |
| HTTP request timeout | Server chậm | Tăng timeout: `TimeoutMilliseconds 10000` |

---

## Tóm tắt

- ✅ **basic-cli** = platform cho CLI apps. Stdout, Stdin, File, Http, Arg, Env.
- ✅ **Argument parsing** = `Arg.list!` → parse pure function → dispatch commands.
- ✅ **File I/O** = `File.readUtf8!`/`writeUtf8!` + `Task.attempt!` cho safety.
- ✅ **HTTP** = `Http.send` + request record.
- ✅ **Interactive loop** = recursive `mainLoop!` function + menu.
- ✅ **Full app pattern** = pure core (parse, format, calculate) + effectful shell (IO, file, args).

## Tiếp theo

→ Chapter 25: **Web Services** — `basic-webserver` platform: routing, JSON API, request/response handling, middleware, và xây REST API hoàn chỉnh.
