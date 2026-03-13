# Chapter 12 — Tasks & Effects ⭐

> **Bạn sẽ học được**:
> - `Task ok err` — type đại diện cho side effects
> - Dấu `!` — sugar syntax cho chaining Tasks
> - `Task.await` và `Task.map` — dạng dài, hiểu cơ chế bên trong
> - Đọc/ghi file, đọc stdin, HTTP request
> - Xử lý lỗi trong Tasks
> - Orchestrate nhiều side effects thành workflow
>
> **Yêu cầu trước**: [Chapter 11 — Purity by Default](chapter_11_purity_by_default.md)
> **Thời gian đọc**: ~45 phút | **Level**: Intermediate
> **Kết quả cuối cùng**: Viết CLI app hoàn chỉnh — đọc input, xử lý, ghi output — đúng kiến trúc FP.

---

Pure functions không thể in ra màn hình, đọc file, hay gọi HTTP. Vậy làm sao viết chương trình thực tế? Câu trả lời: **Tasks**.

Task là một **mô tả** cho side effect — không phải side effect. Khi bạn viết `Stdout.line "hello"`, bạn không in được gì cả — bạn tạo một đơn đặt hàng nói "hãy in dòng này". Chỉ khi platform nhận đơn và thực hiện, text mới xuất hiện. Dấu `!` là cách bạn nói "gửi đơn ngay".

## 12.1 — Task là gì?

Task là khái niệm trung tâm cho IO trong Roc. Hiểu Task = hiểu cách Roc nói chuyện với thế giới bên ngoài.

### Khái niệm

`Task ok err` là một **mô tả** cho side effect. Nó nói: "Khi được chạy, tôi sẽ thực hiện IO và:
- Thành công → trả `ok`
- Thất bại → trả `err`"

```roc
# Task {} []    = "In ra terminal, không trả gì, không lỗi"
Stdout.line : Str -> Task {} []

# Task Str [ReadErr]  = "Đọc stdin, trả Str, có thể lỗi ReadErr"
Stdin.line : Task Str [ReadErr]

# Task (List U8) [FileReadErr ...]  = "Đọc file, trả bytes, có thể lỗi"
File.readBytes : Str -> Task (List U8) [FileReadErr Path IOErr]
```

> **💡 Ẩn dụ**: Task giống **đơn đặt hàng**. Khi bạn viết `Stdout.line "hello"`, bạn **tạo đơn** — nhưng chưa gửi. Platform nhận đơn và **thực hiện**. Dấu `!` = "gửi đơn ngay bây giờ".

### Task KHÔNG chạy cho đến khi được gọi

```roc
# Đây chỉ tạo Task — chưa chạy gì cả
printHello = Stdout.line "Hello"
# Lúc này CHƯA in gì

# Dấu ! = chạy Task ngay
main =
    Stdout.line! "Now it prints!"
    # Bây giờ mới in
```

---

## 12.2 — Dấu `!` — Simple Task Chaining

Dấu `!` là cách bạn sẽ dùng 99% thời gian. Nó nói: "chạy Task này và cho tôi kết quả, rồi tiếp tục". Nếu Task thất bại, dấu `!` tự động truyền lỗi lên cho caller.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Stdin

main =
    # Mỗi dòng có ! = "chạy Task này, rồi tiếp"
    Stdout.line! "Bạn tên gì?"

    name = Stdin.line!               # đọc input → gán vào name

    Stdout.line! "Bạn bao nhiêu tuổi?"

    ageStr = Stdin.line!

    greeting = buildGreeting name ageStr    # pure function!

    Stdout.line! greeting

# Pure core — không ! → dễ test
buildGreeting : Str, Str -> Str
buildGreeting = \name, ageStr ->
    when Str.toI64 ageStr is
        Ok age ->
            if age < 18 then
                "Chào em $(name)! 👋"
            else
                "Chào anh/chị $(name)! 🤝"
        Err _ ->
            "Chào $(name)! (tuổi không hợp lệ)"
```

### Quy tắc dùng `!`

```roc
# 1. Gán kết quả Task vào biến
value = SomeModule.function!      # chạy Task, gán Ok value

# 2. Chạy Task không cần kết quả
Stdout.line! "hello"              # chạy, bỏ qua ()

# 3. Dùng trong biểu thức
len = Str.countUtf8Bytes (Stdin.line!)    # đọc input rồi đếm bytes
```

---

## 12.3 — Bên trong: `Task.await` và `Task.map`

Dấu `!` thực ra là sugar syntax cho `Task.await`. Hiểu cơ chế giúp bạn xử lý trường hợp phức tạp.

### `Task.await` — Chạy Task, rồi dùng kết quả cho Task tiếp theo

```roc
# Dấu ! sugar:
name = Stdin.line!
Stdout.line! "Chào $(name)"

# Tương đương Task.await:
Task.await Stdin.line \name ->
    Stdout.line "Chào $(name)"
```

`Task.await` nói: "Chạy Task đầu tiên. Khi xong, lấy kết quả, truyền vào function để tạo Task tiếp theo."

### `Task.map` — Biến đổi kết quả Task (không tạo Task mới)

```roc
# Task.map = biến đổi giá trị bên trong Task
# Giống List.map nhưng cho Task

# Đọc input, rồi TRIM — không cần Task mới
cleanInput = Stdin.line |> Task.map Str.trim
# cleanInput : Task Str [ReadErr]
```

### So sánh

```
Task.await : Task a e, (a -> Task b e) -> Task b e
# "Chạy Task A, lấy a, tạo Task B → chạy Task B"

Task.map : Task a e, (a -> b) -> Task b e
# "Chạy Task A, lấy a, biến đổi thành b (PURE, không IO mới)"
```

---

## 12.4 — Đọc & Ghi File

File operations là side effects phổ biến nhất. `File.readUtf8` đọc file thành `Str`, `File.writeUtf8` ghi `Str` vào file. Cả hai trả `Task` với error types cụ thể — file không tồn tại, không có quyền, disk đầy.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.File
import cli.Path

main =
    # Ghi file
    File.writeUtf8! (Path.fromStr "hello.txt") "Xin chào từ Roc! 🎉"
    Stdout.line! "✅ Đã ghi file"

    # Đọc file
    content = File.readUtf8! (Path.fromStr "hello.txt")
    Stdout.line! "📄 Nội dung: $(content)"

    # Đọc file + xử lý (pure core)
    processFile! "hello.txt"

# Pure function — xử lý nội dung file
processContent : Str -> { lines : U64, words : U64, chars : U64 }
processContent = \content ->
    lines = content |> Str.split "\n" |> List.len
    words = content |> Str.split " " |> List.len
    chars = Str.countUtf8Bytes content
    { lines: Num.toU64 lines, words: Num.toU64 words, chars }

# Shell function — đọc file rồi gọi pure logic
processFile! : Str -> Task {} _
processFile! = \filename ->
    content = File.readUtf8! (Path.fromStr filename)
    stats = processContent content    # pure!
    Stdout.line! "📊 $(filename): $(Num.toStr stats.lines) dòng, $(Num.toStr stats.words) từ, $(Num.toStr stats.chars) bytes"
```

---

## 12.5 — Xử lý lỗi trong Tasks

Tasks có thể thất bại — file không tồn tại, network timeout, permission denied. `Task.attempt` bắt lỗi và cho bạn quyết định: retry, log, hay trả default value.

### `Task.attempt` — Bắt lỗi, tiếp tục chạy

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.File
import cli.Path

main =
    # Task.attempt = chạy Task, trả Result thay vì crash
    result = File.readUtf8 (Path.fromStr "maybe_exists.txt") |> Task.attempt!

    when result is
        Ok content ->
            Stdout.line! "📄 Đọc được: $(content)"
        Err _ ->
            Stdout.line! "❌ File không tồn tại — dùng default"
            Stdout.line! "📝 Tạo file mới..."
            File.writeUtf8! (Path.fromStr "maybe_exists.txt") "Default content"
            Stdout.line! "✅ Đã tạo file mới"
```

### Đọc nhiều files an toàn

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.File
import cli.Path

# Đọc file an toàn — trả default nếu lỗi
readFileOrDefault : Str, Str -> Task Str _
readFileOrDefault = \filename, default ->
    result = File.readUtf8 (Path.fromStr filename) |> Task.attempt!
    when result is
        Ok content -> Task.ok content
        Err _ -> Task.ok default

main =
    config = readFileOrDefault! "config.txt" "port=8080\nhost=localhost"
    Stdout.line! "Config:\n$(config)"
```

---

## 12.6 — Stdin: Interactive CLI

Đọc input từ user qua `Stdin.line!`. Kết hợp với `when...is` để xử lý commands — nền tảng cho CLI applications ở Chapter 24.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Stdin

main =
    Stdout.line! "=== Mini Calculator ==="

    Stdout.line! "Nhập số thứ nhất:"
    aStr = Stdin.line!

    Stdout.line! "Nhập phép tính (+, -, *, /):"
    opStr = Stdin.line!

    Stdout.line! "Nhập số thứ hai:"
    bStr = Stdin.line!

    # Parse + tính (pure)
    result = calculate aStr opStr bStr
    Stdout.line! result

# Pure core — dễ test!
calculate : Str, Str, Str -> Str
calculate = \aStr, opStr, bStr ->
    aResult = Str.toF64 aStr
    bResult = Str.toF64 bStr

    when (aResult, bResult) is
        (Ok a, Ok b) ->
            answer = when opStr is
                "+" -> Ok (a + b)
                "-" -> Ok (a - b)
                "*" -> Ok (a * b)
                "/" ->
                    if b == 0.0 then Err DivByZero
                    else Ok (a / b)
                _ -> Err UnknownOp

            when answer is
                Ok val -> "$(aStr) $(opStr) $(bStr) = $(Num.toStr val)"
                Err DivByZero -> "❌ Không thể chia cho 0!"
                Err UnknownOp -> "❌ Phép tính không hợp lệ: $(opStr)"
        _ -> "❌ Nhập số không hợp lệ"
```

---

## 12.7 — Environment Variables & Arguments

Đọc environment variables (`Env.var`) và command line arguments (`Arg.list`) — cách CLI app nhận configuration mà không hardcode.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Env
import cli.Arg

main =
    # Đọc environment variable
    homeResult = Env.var "HOME" |> Task.attempt!
    home = when homeResult is
        Ok val -> val
        Err _ -> "(không tìm thấy)"
    Stdout.line! "HOME = $(home)"

    # Đọc command line arguments
    args = Arg.list!
    Stdout.line! "Arguments: $(Inspect.toStr args)"

    when List.get args 1 is
        Ok name -> Stdout.line! "Chào $(name)!"
        Err _ -> Stdout.line! "Usage: roc run main.roc -- <name>"
```

---

## 12.8 — Ví dụ tổng hợp: Todo CLI App

Kết hợp mọi thứ: đọc stdin, ghi file, xử lý errors — thành ứng dụng todo hoàn chỉnh. Chú ý cách pure core (domain logic) và effectful shell (IO) tách biệt rõ ràng.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Stdin
import cli.File
import cli.Path

todoFile = "todos.txt"

# ═══════ PURE CORE ═══════

parseTodos : Str -> List Str
parseTodos = \content ->
    content
    |> Str.split "\n"
    |> List.keepIf \line -> !(Str.isEmpty (Str.trim line))

formatTodos : List Str -> Str
formatTodos = \todos ->
    if List.isEmpty todos then
        "📭 Không có task nào!"
    else
        List.mapWithIndex todos \todo, index ->
            "  $(Num.toStr (index + 1)). $(todo)"
        |> Str.joinWith "\n"

todosToFile : List Str -> Str
todosToFile = \todos ->
    Str.joinWith todos "\n"

addTodo : List Str, Str -> List Str
addTodo = \todos, newItem ->
    List.append todos (Str.trim newItem)

removeTodo : List Str, U64 -> Result (List Str) [InvalidIndex]
removeTodo = \todos, index ->
    if index >= 1 && index <= Num.toU64 (List.len todos) then
        Ok (List.dropAt todos (Num.toU64 (index - 1)))
    else
        Err InvalidIndex

# ═══════ EFFECTFUL SHELL ═══════

loadTodos! : Task (List Str) _
loadTodos! =
    result = File.readUtf8 (Path.fromStr todoFile) |> Task.attempt!
    when result is
        Ok content -> Task.ok (parseTodos content)
        Err _ -> Task.ok []

saveTodos! : List Str -> Task {} _
saveTodos! = \todos ->
    File.writeUtf8! (Path.fromStr todoFile) (todosToFile todos)

main =
    Stdout.line! "=== 📝 Todo App ==="

    todos = loadTodos!

    Stdout.line! (formatTodos todos)
    Stdout.line! ""
    Stdout.line! "Lệnh: (a)dd / (r)emove / (q)uit"

    command = Stdin.line!

    when command is
        "a" ->
            Stdout.line! "Nhập task mới:"
            newItem = Stdin.line!
            updatedTodos = addTodo todos newItem
            saveTodos! updatedTodos
            Stdout.line! "✅ Đã thêm: $(Str.trim newItem)"
        "r" ->
            Stdout.line! "Xóa task số mấy?"
            indexStr = Stdin.line!
            when Str.toU64 indexStr is
                Ok index ->
                    when removeTodo todos index is
                        Ok updatedTodos ->
                            saveTodos! updatedTodos
                            Stdout.line! "🗑️ Đã xóa task #$(indexStr)"
                        Err InvalidIndex ->
                            Stdout.line! "❌ Số không hợp lệ"
                Err _ ->
                    Stdout.line! "❌ Nhập số!"
        "q" ->
            Stdout.line! "Tạm biệt! 👋"
        _ ->
            Stdout.line! "❓ Lệnh không hợp lệ"
```

Nhìn lại kiến trúc:
- **Pure core**: `parseTodos`, `formatTodos`, `addTodo`, `removeTodo` — test bằng `expect`
- **Effectful shell**: `loadTodos!`, `saveTodos!`, `main` — IO ở đây

---


## ✅ Checkpoint 12

> Đến đây bạn phải hiểu:
> 1. `Task ok err` = mô tả side effect chưa thực hiện ("recipe, not the cake")
> 2. `Task.await` = chain tasks, `!` = shortcut cho await
> 3. Platform là engine chạy Tasks — app chỉ mô tả "làm gì", platform thực hiện
>
> **Test nhanh**: `Task.await` và `!` khác nhau thế nào?
> <details><summary>Đáp án</summary>Cùng ý nghĩa! `!` là syntax sugar cho `Task.await`.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): File word counter

Viết CLI app đọc file, đếm số dòng, từ, và ký tự (giống lệnh `wc`):

```bash
roc run wc.roc -- myfile.txt
# Output: myfile.txt: 42 dòng, 256 từ, 1534 bytes
```

<details><summary>💡 Gợi ý</summary>

- Lấy filename từ `Arg.list!` index 1
- Đọc file bằng `File.readUtf8!`
- Tách logic đếm thành pure function `countStats`

</details>

<details><summary>✅ Lời giải</summary>

```roc
import cli.{Stdout, File, Path, Arg}

countStats = \content ->
    lines = content |> Str.split "\n" |> List.len
    words = content |> Str.split " " |> List.keepIf \w -> !(Str.isEmpty (Str.trim w)) |> List.len
    bytes = Str.countUtf8Bytes content
    { lines, words, bytes }

main =
    args = Arg.list!
    when List.get args 1 is
        Ok filename ->
            content = File.readUtf8! (Path.fromStr filename)
            stats = countStats content
            Stdout.line! "$(filename): $(Num.toStr stats.lines) dòng, $(Num.toStr stats.words) từ, $(Num.toStr stats.bytes) bytes"
        Err _ ->
            Stdout.line! "Usage: roc run wc.roc -- <filename>"
```

</details>

---

**Bài 2** (15 phút): Config loader

Viết app đọc file config dạng `key=value`, parse thành Dict, in ra:

```
# config.txt
host=localhost
port=8080
debug=true
```

<details><summary>✅ Lời giải</summary>

```roc
# Pure
parseConfig = \content ->
    content
    |> Str.split "\n"
    |> List.keepIf \l -> !(Str.isEmpty (Str.trim l)) && !(Str.startsWith l "#")
    |> List.walk (Dict.empty {}) \dict, line ->
        parts = Str.splitFirst line "="
        when parts is
            Ok { before, after } -> Dict.insert dict (Str.trim before) (Str.trim after)
            Err _ -> dict

# Shell
main =
    result = File.readUtf8 (Path.fromStr "config.txt") |> Task.attempt!
    config = when result is
        Ok content -> parseConfig content
        Err _ -> Dict.empty {}

    Stdout.line! "=== Config ==="
    Dict.walk config {} \_, key, value ->
        Stdout.line! "  $(key) = $(value)"
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `This Task doesn't have a !` | Quên dấu `!` — tạo Task nhưng không chạy | Thêm `!` sau function name |
| `Expected Task but got Str` | Dùng `!` trên pure function | Bỏ `!` — chỉ dùng cho Task/IO functions |
| File read crash | File không tồn tại | Dùng `Task.attempt!` để bắt lỗi |
| Stdin.line trả chuỗi rỗng | User nhấn Enter không gõ gì | Kiểm tra `Str.isEmpty` trước khi xử lý |

---

## Tóm tắt

- ✅ **`Task ok err`** = mô tả side effect. `ok` = type khi thành công, `err` = type khi thất bại.
- ✅ **Dấu `!`** = chạy Task ngay, lấy kết quả. Sugar syntax cho `Task.await`.
- ✅ **`Task.await`** = chạy Task A → lấy kết quả → tạo Task B. **`Task.map`** = biến đổi kết quả (pure).
- ✅ **`Task.attempt`** = chạy Task → trả `Result` thay vì crash. An toàn cho file/HTTP.
- ✅ **IO**: `Stdout.line!`, `Stdin.line!`, `File.readUtf8!`, `File.writeUtf8!`, `Env.var`, `Arg.list!`.
- ✅ **Pure core / Effectful shell** — tách logic (pure, testable) khỏi IO (shell).

## Tiếp theo

→ Chapter 13: **Error Handling with `Result`** — `Result.try` (chaining), `Result.map`, `Result.mapErr`, railway-oriented programming, và cách xây dựng error hierarchy cho real-world apps.
