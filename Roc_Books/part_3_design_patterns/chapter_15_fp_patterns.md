# Chapter 15 — FP Patterns

> **Bạn sẽ học được**:
> - **Strategy Pattern** — functions as first-class values thay class hierarchy
> - **Command Pattern** — tag unions thay command objects
> - **Middleware Pattern** — function composition `Task -> Task`
> - **Builder Pattern** — record update + pipeline
> - **Interpreter Pattern** — tách mô tả khỏi thực thi
> - Tại sao FP **không cần** GoF design patterns — vì đã có built-in
>
> **Yêu cầu trước**: [Chapter 14 — Abilities](../part_2_thinking_functionally/chapter_14_abilities.md)
> **Thời gian đọc**: ~45 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Nhận diện và áp dụng FP patterns — code gọn hơn OOP 10 lần.

---

Trong OOP, bạn học 23 Gang-of-Four patterns — Strategy, Observer, Factory, Decorator... Trong FP, hầu hết những patterns đó biến mất. Không phải vì FP thiếu giải pháp, mà vì functions bậc cao + closures đã giải quyết cùng bài toán một cách tự nhiên hơn.

## 15.1 — Tại sao FP không cần Design Patterns?

Strategy pattern? Trong FP, truyền function làm tham số. Observer pattern? Higher-order function nhận callback. Factory? Function trả function. Phần lớn OOP patterns là workaround cho việc functions không phải first-class citizens.

### Trong OOP

GoF (Gang of Four) có 23 design patterns. Nhiều pattern tồn tại vì **OOP thiếu tính năng**:

- **Strategy** → vì OOP không truyền function dễ dàng
- **Observer** → vì OOP cần manual pub/sub
- **Factory** → vì OOP constructors cứng nhắc
- **Decorator** → vì OOP không compose functions đơn giản

### Trong FP

Nhiều patterns **biến mất** vì ngôn ngữ đã hỗ trợ sẵn:

| OOP Pattern | FP Equivalent | Roc |
|-------------|--------------|-----|
| Strategy | **Truyền function** | `\strategy -> ...` |
| Command | **Tag union** | `[Save, Load, Delete]` |
| Factory | **Smart constructor** | `Email.fromStr` |
| Decorator | **Function composition** | `f \|> g \|> h` |
| Observer | **Event list + walk** | `List.walk events` |
| Builder | **Record update** | `{ default & field: val }` |
| Iterator | **List.map/walk** | — sẵn có |
| Singleton | **Module constant** | `port = 8080` |

> **💡 Peter Norvig** (Google AI Director): *"16 trong 23 GoF patterns hoặc biến mất hoặc đơn giản hơn đáng kể trong ngôn ngữ functional."*

---

## 15.2 — Strategy Pattern: Truyền function

Strategy pattern trong OOP cần interface, class implement, dependency injection. Trong FP, bạn chỉ cần truyền function. Đơn giản hơn, linh hoạt hơn.

### OOP: Cần interface + classes

```python
# Python OOP — Strategy Pattern
class SortStrategy:
    def sort(self, data): pass

class BubbleSort(SortStrategy):
    def sort(self, data): ...

class QuickSort(SortStrategy):
    def sort(self, data): ...

def process(data, strategy: SortStrategy):
    return strategy.sort(data)
```

### FP Roc: Chỉ truyền function

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# "Strategy" = function bình thường
processData = \data, sortStrategy, filterStrategy, formatStrategy ->
    data
    |> sortStrategy
    |> filterStrategy
    |> formatStrategy

main =
    scores = [85, 42, 97, 63, 78, 91, 55]

    # Strategy 1: Sort tăng, giữ >= 70, format dạng "score điểm"
    result1 = processData scores
        (\items -> List.sortWith items Num.compare)
        (\items -> List.keepIf items \x -> x >= 70)
        (\items -> List.map items \x -> "$(Num.toStr x) điểm")

    Stdout.line! "Top scores: $(Inspect.toStr result1)"
    # → ["78 điểm", "85 điểm", "91 điểm", "97 điểm"]

    # Strategy 2: Sort giảm, giữ top 3, format dạng emoji
    result2 = processData scores
        (\items -> List.sortWith items \a, b -> Num.compare b a)
        (\items -> List.takeFirst items 3)
        (\items -> List.mapWithIndex items \x, i ->
            medal = when i is
                0 -> "🥇"
                1 -> "🥈"
                _ -> "🥉"
            "$(medal) $(Num.toStr x)")

    Stdout.line! "Podium: $(Inspect.toStr result2)"
    # → ["🥇 97", "🥈 91", "🥉 85"]
```

Không cần interface, không cần classes — **function IS the strategy**.

### Ứng dụng: Configurable pricing

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Pricing strategies — mỗi cái là function
noDiscount = \price -> price
percentOff = \percent -> \price -> price - (price * percent // 100)
fixedOff = \amount -> \price -> Num.max 0 (price - amount)
buyOneGetOne = \price -> price // 2

applyPricing = \items, strategy ->
    List.map items \item ->
        { item & finalPrice: strategy item.price }

main =
    items = [
        { name: "Áo", price: 200000, finalPrice: 0 },
        { name: "Quần", price: 350000, finalPrice: 0 },
        { name: "Giày", price: 500000, finalPrice: 0 },
    ]

    # Đổi strategy chỉ cần đổi 1 argument
    regular = applyPricing items noDiscount
    sale30 = applyPricing items (percentOff 30)
    bogo = applyPricing items buyOneGetOne

    Stdout.line! "=== Giá gốc ==="
    List.forEach regular \i -> Stdout.line! "  $(i.name): $(Num.toStr i.finalPrice)đ"

    Stdout.line! "=== Giảm 30% ==="
    List.forEach sale30 \i -> Stdout.line! "  $(i.name): $(Num.toStr i.finalPrice)đ"

    Stdout.line! "=== Mua 1 tặng 1 ==="
    List.forEach bogo \i -> Stdout.line! "  $(i.name): $(Num.toStr i.finalPrice)đ"
```

---

## 15.3 — Command Pattern: Tag Unions

Builder pattern xây dựng object phức tạp từng bước. Trong FP, record update `{ config & timeout: 5000 }` kết hợp với pipeline làm được điều tương tự — không cần class Builder.

### Commands = data, không phải objects

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Commands là TAG UNIONS — data thuần túy
# Có thể lưu, serialize, replay, undo!

executeCommand = \state, command ->
    when command is
        AddItem item ->
            { state & items: List.append state.items item }
        RemoveItem name ->
            { state & items: List.keepIf state.items \i -> i.name != name }
        SetDiscount percent ->
            { state & discount: percent }
        ClearCart ->
            { state & items: [] }

# Execute nhiều commands bằng List.walk
executeAll = \initialState, commands ->
    List.walk commands initialState executeCommand

main =
    emptyCart = { items: [], discount: 0 }

    commands = [
        AddItem { name: "Phở", price: 45000 },
        AddItem { name: "Cà phê", price: 25000 },
        AddItem { name: "Bánh mì", price: 15000 },
        SetDiscount 10,
        RemoveItem "Bánh mì",
    ]

    finalCart = executeAll emptyCart commands

    Stdout.line! "=== Giỏ hàng ==="
    List.forEach finalCart.items \i ->
        Stdout.line! "  $(i.name): $(Num.toStr i.price)đ"
    Stdout.line! "Giảm giá: $(Num.toStr finalCart.discount)%"

    # Undo = bỏ command cuối → replay
    undone = executeAll emptyCart (List.dropLast commands 1)
    Stdout.line! "\n=== Sau Undo ==="
    List.forEach undone.items \i ->
        Stdout.line! "  $(i.name): $(Num.toStr i.price)đ"
```

**Ưu điểm của Command as Data:**
1. **Undo/Redo** — replay commands thiếu cái cuối
2. **History** — lưu log tất cả commands
3. **Serializable** — gửi qua network, lưu database
4. **Testable** — tạo list commands, check final state

---

## 15.4 — Middleware Pattern: Function Composition

Interpreter pattern biến data structure thành hành vi. Đây là pattern tự nhiên nhất trong FP — tag union mô tả "làm gì", function xử lý "làm thế nào".

### Middleware = `fn -> fn`

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Mỗi middleware nhận request → trả modified request
# Middleware = function nhận function, trả function

# Log middleware
withLogging = \handler -> \request ->
    # Before
    logLine = "[LOG] $(request.method) $(request.path)"
    { response: handler request, log: logLine }

# Auth middleware
withAuth = \handler -> \request ->
    if request.token == "valid-token" then
        handler request
    else
        { status: 401, body: "Unauthorized" }

# Timing middleware (pure — tính "giả")
withTiming = \handler -> \request ->
    response = handler request
    { response & headers: Dict.insert (Result.withDefault response.headers (Dict.empty {})) "X-Process" "fast" }

# Handler thực tế
helloHandler = \request ->
    { status: 200, body: "Hello $(request.path)!", headers: Dict.empty {} }

main =
    # Compose middlewares
    request = { method: "GET", path: "/api/users", token: "valid-token" }

    # Apply handler trực tiếp
    response = helloHandler request
    Stdout.line! "Direct: $(Num.toStr response.status) — $(response.body)"

    # Apply với auth middleware
    authedHandler = withAuth helloHandler
    response2 = authedHandler request
    Stdout.line! "With auth: $(Num.toStr response2.status) — $(response2.body)"

    # Unauthorized
    badRequest = { method: "GET", path: "/secret", token: "bad" }
    response3 = authedHandler badRequest
    Stdout.line! "Bad token: $(Num.toStr response3.status) — $(response3.body)"
```

---

## 15.5 — Builder Pattern: Record Update Pipeline

Middleware là pipeline — mỗi bước nhận request, xử lý, truyền cho bước tiếp theo. Logging, authentication, rate limiting — tất cả là functions nối nhau.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# "Builder" = record mặc định + pipeline of updates
defaultQuery = {
    table: "",
    fields: ["*"],
    filters: [],
    orderBy: NoOrder,
    limit: NoLimit,
}

from = \table -> { defaultQuery & table }
select = \query, fields -> { query & fields }
where = \query, condition -> { query & filters: List.append query.filters condition }
orderAsc = \query, field -> { query & orderBy: Asc field }
orderDesc = \query, field -> { query & orderBy: Desc field }
limitTo = \query, n -> { query & limit: Limit n }

toSQL = \query ->
    fieldsStr = Str.joinWith query.fields ", "
    base = "SELECT $(fieldsStr) FROM $(query.table)"

    withFilters =
        if List.isEmpty query.filters then base
        else "$(base) WHERE $(Str.joinWith query.filters " AND ")"

    withOrder = when query.orderBy is
        NoOrder -> withFilters
        Asc field -> "$(withFilters) ORDER BY $(field) ASC"
        Desc field -> "$(withFilters) ORDER BY $(field) DESC"

    when query.limit is
        NoLimit -> withOrder
        Limit n -> "$(withOrder) LIMIT $(Num.toStr n)"

main =
    # Builder pipeline — đọc như tiếng Anh!
    query =
        from "users"
        |> select ["name", "email", "age"]
        |> where "age >= 18"
        |> where "active = true"
        |> orderDesc "created_at"
        |> limitTo 10

    Stdout.line! (toSQL query)
    # → SELECT name, email, age FROM users WHERE age >= 18 AND active = true ORDER BY created_at DESC LIMIT 10
```

---

## 15.6 — Interpreter Pattern: Tách mô tả khỏi thực thi

State machine trong FP dùng tag unions cho trạng thái và pattern matching cho transitions. Compiler đảm bảo bạn xử lý mọi transition — không thể quên edge case.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Step 1: MÔ TẢ workflow bằng data (tag unions)
# KHÔNG thực thi — chỉ mô tả

buildReport = \config ->
    [
        FetchData { source: config.dataSource },
        ValidateData { rules: config.rules },
        Transform { operation: config.transform },
        FormatOutput { template: config.outputFormat },
        SaveResult { destination: config.saveTo },
    ]

# Step 2: THỰC THI (interpreter) — ĐỌC mô tả, CHẠY từng bước
executeStep = \step ->
    when step is
        FetchData { source } ->
            "📡 Fetching from $(source)..."
        ValidateData { rules } ->
            "✅ Validating $(Num.toStr (List.len rules)) rules..."
        Transform { operation } ->
            "🔄 Transforming: $(operation)..."
        FormatOutput { template } ->
            "📊 Formatting as $(template)..."
        SaveResult { destination } ->
            "💾 Saving to $(destination)..."

main =
    config = {
        dataSource: "database://orders",
        rules: ["non-null", "positive-amount", "valid-date"],
        transform: "aggregate-by-month",
        outputFormat: "HTML table",
        saveTo: "reports/monthly.html",
    }

    steps = buildReport config

    Stdout.line! "=== Report Pipeline ==="
    List.forEach steps \step ->
        Stdout.line! (executeStep step)

    Stdout.line! "---"
    Stdout.line! "Tổng: $(Num.toStr (List.len steps)) bước"
    # Output:
    # === Report Pipeline ===
    # 📡 Fetching from database://orders...
    # ✅ Validating 3 rules...
    # 🔄 Transforming: aggregate-by-month...
    # 📊 Formatting as HTML table...
    # 💾 Saving to reports/monthly.html...
    # ---
    # Tổng: 5 bước
```

**Ưu điểm**: Mô tả (data) tách biệt thực thi (interpreter). Có thể:
- **Dry run** — chỉ in mô tả, không chạy thật
- **Đổi interpreter** — cùng mô tả, chạy test vs production
- **Serialize** — lưu workflow vào file/database

---


## ✅ Checkpoint 15

> Đến đây bạn phải hiểu:
> 1. **Strategy** = truyền function làm tham số (thay vì OOP interface)
> 2. **Command** = tag union variant chứa data (enum as command)
> 3. **Middleware** = `Task -> Task` composition — chuỗi xử lý
>
> **Test nhanh**: OOP dùng Strategy interface. FP dùng gì thay thế?
> <details><summary>Đáp án</summary>Higher-order function! Truyền function trực tiếp làm tham số.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Validation strategies

Viết hệ thống validation với interchangeable strategies:

```roc
# validateField nhận: value và list strategies
# Mỗi strategy: Str -> Result {} Str
# Trả: Ok {} hoặc Err (List Str) tất cả lỗi

notEmpty = \val -> if Str.isEmpty (Str.trim val) then Err "Không được rỗng" else Ok {}
minLength = \min -> \val -> if Str.countUtf8Bytes val < min then Err "Tối thiểu $(Num.toStr min) ký tự" else Ok {}
```

<details><summary>✅ Lời giải</summary>

```roc
validateField = \value, strategies ->
    errors = List.walk strategies [] \errs, strategy ->
        when strategy value is
            Ok _ -> errs
            Err msg -> List.append errs msg
    if List.isEmpty errors then Ok {} else Err errors

# Sử dụng
usernameRules = [notEmpty, minLength 3, \v -> if Str.contains v " " then Err "Không có khoảng trắng" else Ok {}]

when validateField "ab" usernameRules is
    Ok _ -> ...
    Err errors -> List.forEach errors \e -> Stdout.line! "  ❌ $(e)"
# → ❌ Tối thiểu 3 ký tự
```

</details>

---

**Bài 2** (15 phút): Text editor commands

Xây dựng text editor bằng Command pattern:

```roc
# Commands: Insert Str | DeleteLast | ToUpper | ToLower | Undo
# State: { text: Str, history: List Str }
# Undo = khôi phục text từ history
```

<details><summary>✅ Lời giải</summary>

```roc
execute = \state, cmd ->
    when cmd is
        Insert text ->
            { text: Str.concat state.text text, history: List.append state.history state.text }
        DeleteLast ->
            bytes = Str.toUtf8 state.text
            newText = List.dropLast bytes 1 |> Str.fromUtf8 |> Result.withDefault ""
            { text: newText, history: List.append state.history state.text }
        ToUpper ->
            { text: Str.toUppercase state.text, history: List.append state.history state.text }
        Undo ->
            when List.last state.history is
                Ok prev -> { text: prev, history: List.dropLast state.history 1 }
                Err _ -> state

initialState = { text: "", history: [] }
commands = [Insert "hello", Insert " world", ToUpper, Undo]
final = List.walk commands initialState execute
# final.text → "hello world" (undo reversed ToUpper)
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Strategy function type mismatch | Functions có type khác nhau | Đảm bảo cùng signature: `a -> b` |
| Command walk trả type sai | State type thay đổi giữa các commands | Tất cả commands phải trả cùng state type |
| Middleware compose sai thứ tự | Outer middleware chạy trước | `withAuth (withLog handler)`: auth chạy trước log |

---

## Tóm tắt

- ✅ **Strategy** = truyền function. Không cần interface/class. Function IS the strategy.
- ✅ **Command** = tag union. Serializable, testable, undo/redo. `List.walk commands state execute`.
- ✅ **Middleware** = `fn -> fn`. Compose bằng nesting: `withAuth (withLog handler)`.
- ✅ **Builder** = default record + pipeline `|>`. Đọc như tiếng Anh tự nhiên.
- ✅ **Interpreter** = tách mô tả (data) khỏi thực thi (interpreter). Dry run, swap interpreter.
- ✅ FP **không cần** GoF patterns — functions + tag unions + records đã đủ.

## Tiếp theo

→ Chapter 16: **Event Sourcing** — lưu sự kiện thay vì trạng thái hiện tại. Events as tag unions, `List.walk` rebuild state, projections, snapshots — pattern hoàn hảo cho FP.
