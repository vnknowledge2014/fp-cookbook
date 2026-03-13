# Chapter 21 — Serialization

> **Bạn sẽ học được**:
> - `Encode` và `Decode` abilities — serialize/deserialize tự động
> - **JSON** — đọc/ghi JSON với Roc
> - Parse external data vào domain types an toàn
> - **Encoding formats** — JSON, CSV, custom formats
> - Kết hợp serialization với domain validation
>
> **Yêu cầu trước**: [Chapter 20 — Platform Separation](chapter_20_platform_separation.md)
> **Thời gian đọc**: ~35 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Đọc/ghi data từ JSON/CSV, chuyển đổi giữa external formats và domain types an toàn.

---

Dữ liệu trong Roc sống trong bộ nhớ dưới dạng records, tags, lists. Nhưng khi gửi qua network, lưu vào database, hay đọc từ file — bạn cần chuyển chúng thành JSON, CSV, hay binary. Serialization là cầu nối giữa thế giới types của Roc và thế giới bên ngoài.

## Serialization — Biên giới giữa domain và thế giới bên ngoài

Khi domain model (Tag Unions, Opaque Types) gặp thế giới bên ngoài (JSON, database rows), bạn cần **translation layer**. Chapter này dạy bạn encode/decode patterns trong Roc — giữ domain model pure, validate data ở biên giới.


## 21.1 — Tại sao cần Serialization?

Domain types sống trong bộ nhớ — `Email`, `Money`, `Order`. Nhưng database lưu strings và numbers, API trả JSON, files chứa CSV. Serialization chuyển đổi giữa hai thế giới.

### Boundaries = nơi data vào/ra

```
┌─────────────────────────────────┐
│         Your App (Roc)          │
│  ┌─────────────────────────┐   │
│  │      Domain Types       │   │  ← Type-safe, validated
│  │  Order, Money, Email    │   │
│  └────────┬────────────────┘   │
│           │                     │
│  ┌────────▼────────────────┐   │
│  │   Serialization Layer   │   │  ← Convert ↔ external format
│  └────────┬────────────────┘   │
└───────────┼─────────────────────┘
            │
    ┌───────▼───────┐
    │ External World │  ← JSON, CSV, HTTP, Files
    │ (untyped, messy)│
    └───────────────┘
```

Data từ thế giới bên ngoài (JSON, CSV, user input) = **untyped, messy**. Domain types = **typed, validated**. Serialization bridge giữa 2 thế giới.

---

## 21.2 — Encode & Decode Abilities

Roc có 2 abilities cho serialization: `Encode` (domain → bytes) và `Decode` (bytes → domain). Chúng hoạt động với bất kỳ format nào — JSON, CSV, binary — qua format modules.

### `Encode` — Domain → External format

```roc
# Roc types tự động implement Encode cho records, tags, primitives
# Encode biến Roc value → bytes (theo format)

user = { name: "An", age: 25, active: Bool.true }
# Encode.toBytes user jsonFormat → JSON bytes
```

### `Decode` — External format → Domain

```roc
# Decode biến bytes → Roc value (có thể thất bại!)
# Kết quả luôn là Result — vì data ngoài có thể sai

# Decode.fromBytes jsonBytes jsonFormat → Result { name: Str, age: U64 } _
```

---

## 21.3 — JSON trong Roc

JSON là format phổ biến nhất cho web APIs. Roc xử lý JSON qua `json` package — encode records thành JSON strings và decode JSON strings thành Roc values.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Roc dùng Encode/Decode với JSON format
# JSON encode/decode sẵn có trong basic-cli

main =
    # Record → JSON string (thủ công cho demo)
    user = { name: "An", age: 25, role: "admin" }
    jsonStr = toJson user
    Stdout.line! "JSON: $(jsonStr)"

    # Parse JSON string → record
    when fromJson "{\"name\":\"Bình\",\"age\":30,\"role\":\"user\"}" is
        Ok parsed -> Stdout.line! "Parsed: $(parsed.name), $(Num.toStr parsed.age)"
        Err msg -> Stdout.line! "❌ Parse error: $(msg)"

# ═══════ JSON HELPERS (simplified) ═══════

toJson : { name : Str, age : U64, role : Str } -> Str
toJson = \record ->
    "{\"name\":\"$(record.name)\",\"age\":$(Num.toStr record.age),\"role\":\"$(record.role)\"}"

fromJson : Str -> Result { name : Str, age : U64, role : Str } Str
fromJson = \raw ->
    # Simplified parser cho demo
    nameResult = extractField raw "name"
    ageResult = extractField raw "age"
    roleResult = extractField raw "role"
    when (nameResult, ageResult, roleResult) is
        (Ok name, Ok ageStr, Ok role) ->
            when Str.toU64 ageStr is
                Ok age -> Ok { name, age, role }
                Err _ -> Err "Invalid age"
        _ -> Err "Missing fields"

extractField : Str, Str -> Result Str [FieldNotFound]
extractField = \json, field ->
    pattern = "\"$(field)\":"
    when Str.splitFirst json pattern is
        Ok { after, .. } ->
            # Lấy value sau ":"
            trimmed = Str.trim after
            if Str.startsWith trimmed "\"" then
                # String value
                rest = Str.replaceFirst trimmed "\"" ""
                when Str.splitFirst rest "\"" is
                    Ok { before, .. } -> Ok before
                    Err _ -> Err FieldNotFound
            else
                # Number value
                when Str.splitFirst trimmed "," is
                    Ok { before, .. } -> Ok (Str.trim before)
                    Err _ ->
                        when Str.splitFirst trimmed "}" is
                            Ok { before, .. } -> Ok (Str.trim before)
                            Err _ -> Err FieldNotFound
        Err _ -> Err FieldNotFound
```

---

## 21.4 — CSV Parsing

CSV đơn giản hơn JSON nhưng có nhiều edge cases: quoted fields, escaped commas, encoding. Parser dưới đây xử lý các trường hợp phổ biến nhất.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ CSV PARSER — PURE ═══════

parseCSV : Str -> List (List Str)
parseCSV = \content ->
    content
    |> Str.split "\n"
    |> List.keepIf \line -> !(Str.isEmpty (Str.trim line))
    |> List.map \line ->
        Str.split line ","
        |> List.map Str.trim

# Parse CSV with headers → List of Dict
parseCSVWithHeaders : Str -> Result (List (Dict Str Str)) [EmptyCSV, NoHeaders]
parseCSVWithHeaders = \content ->
    rows = parseCSV content
    when List.first rows is
        Err _ -> Err EmptyCSV
        Ok headers ->
            dataRows = List.dropFirst rows 1
            records = List.map dataRows \row ->
                List.map2 headers row \h, v -> (h, v)
                |> Dict.fromList
            Ok records

# ═══════ DOMAIN CONVERSION ═══════

# CSV record → Domain type (với validation)
csvToStudent : Dict Str Str -> Result { name : Str, score : I64, grade : Str } [MissingField Str, InvalidScore]
csvToStudent = \record ->
    name = Dict.get record "name" |> Result.mapErr? \_ -> MissingField "name"
    scoreStr = Dict.get record "score" |> Result.mapErr? \_ -> MissingField "score"
    score = Str.toI64 scoreStr |> Result.mapErr? \_ -> InvalidScore
    grade = classifyScore score
    Ok { name, score, grade }

classifyScore = \score ->
    if score >= 90 then "A"
    else if score >= 80 then "B"
    else if score >= 70 then "C"
    else if score >= 60 then "D"
    else "F"

main =
    csvData =
        """
        name, score
        An, 95
        Bình, 82
        Cường, 67
        Dũng, 45
        """

    when parseCSVWithHeaders csvData is
        Ok records ->
            Stdout.line! "=== Bảng điểm ==="
            List.forEach records \record ->
                when csvToStudent record is
                    Ok student ->
                        Stdout.line! "  $(student.name): $(Num.toStr student.score) → $(student.grade)"
                    Err (MissingField field) ->
                        Stdout.line! "  ❌ Thiếu field: $(field)"
                    Err InvalidScore ->
                        Stdout.line! "  ❌ Điểm không hợp lệ"
        Err EmptyCSV -> Stdout.line! "❌ CSV rỗng"
        Err NoHeaders -> Stdout.line! "❌ Không có headers"
    # Output:
    # === Bảng điểm ===
    #   An: 95 → A
    #   Bình: 82 → B
    #   Cường: 67 → D
    #   Dũng: 45 → F
```

---

## 21.5 — Anti-Corruption Layer: External → Domain

Dữ liệu từ bên ngoài hiếm khi khớp chính xác với domain types. Anti-corruption layer parse raw data thành domain types — validate, transform, reject dữ liệu sai. Boundary rõ ràng giữa "thế giới bên ngoài" và "thế giới domain".

### Pattern: Parse, don't validate

```roc
# ❌ Sai: để external data chạy khắp nơi
processRaw = \jsonStr ->
    dict = parseJson jsonStr   # Dict Str Str — untyped!
    # ... dùng Dict.get khắp nơi, quên check = crash

# ✅ Đúng: Parse external → Domain type NGAY tại boundary
processTyped = \jsonStr ->
    rawData = parseJson? jsonStr           # Step 1: parse format
    domainObj = toDomainType? rawData       # Step 2: validate + convert
    # Từ đây trở đi: chỉ dùng domain types, type-safe
    businessLogic domainObj
```

### Ví dụ: API response → Domain

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# External data (untyped, messy)
# Giả sử đã parse JSON thành Dict
rawApiResponse = Dict.fromList [
    ("product_name", "iPhone 15"),
    ("price_cents", "25000000"),
    ("currency", "VND"),
    ("in_stock", "true"),
    ("category", "electronics"),
]

# ═══════ ANTI-CORRUPTION LAYER ═══════

# Parse external → Domain type
toProduct : Dict Str Str -> Result Product [MissingField Str, InvalidPrice, InvalidCurrency]
toProduct = \raw ->
    name = requireField? raw "product_name"
    priceStr = requireField? raw "price_cents"
    currencyStr = requireField? raw "currency"
    stockStr = Dict.get raw "in_stock" |> Result.withDefault "false"

    price = Str.toU64 priceStr |> Result.mapErr? \_ -> InvalidPrice
    currency = parseCurrency? currencyStr
    inStock = stockStr == "true"

    Ok { name, price, currency, inStock }

requireField : Dict Str Str, Str -> Result Str [MissingField Str]
requireField = \dict, field ->
    Dict.get dict field |> Result.mapErr \_ -> MissingField field

parseCurrency : Str -> Result [VND, USD, EUR] [InvalidCurrency]
parseCurrency = \s ->
    when s is
        "VND" -> Ok VND
        "USD" -> Ok USD
        "EUR" -> Ok EUR
        _ -> Err InvalidCurrency

# Domain type
Product : { name : Str, price : U64, currency : [VND, USD, EUR], inStock : Bool }

# ═══════ DOMAIN LOGIC (pure, type-safe) ═══════

formatPrice : Product -> Str
formatPrice = \product ->
    symbol = when product.currency is
        VND -> "đ"
        USD -> "$"
        EUR -> "€"
    "$(Num.toStr product.price)$(symbol)"

describeProduct : Product -> Str
describeProduct = \product ->
    stockStatus = if product.inStock then "✅ Còn hàng" else "❌ Hết hàng"
    "$(product.name) — $(formatPrice product) — $(stockStatus)"

main =
    when toProduct rawApiResponse is
        Ok product ->
            Stdout.line! (describeProduct product)
            # → iPhone 15 — 25000000đ — ✅ Còn hàng
        Err (MissingField field) ->
            Stdout.line! "❌ API thiếu field: $(field)"
        Err InvalidPrice ->
            Stdout.line! "❌ Giá không hợp lệ"
        Err InvalidCurrency ->
            Stdout.line! "❌ Loại tiền không hỗ trợ"
```

---

## 21.6 — Domain → External: Serialization Output

Chiều ngược lại: từ domain types ra JSON/CSV cho API response hoặc file export. Pattern: map domain values sang DTO (Data Transfer Object) rồi encode.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ DOMAIN → JSON ═══════

orderToJson : { id : U64, items : List { name : Str, qty : U64, price : U64 }, total : U64 } -> Str
orderToJson = \order ->
    itemsJson = List.map order.items \item ->
        "{\"name\":\"$(item.name)\",\"qty\":$(Num.toStr item.qty),\"price\":$(Num.toStr item.price)}"
    |> Str.joinWith ","

    "{\"id\":$(Num.toStr order.id),\"items\":[$(itemsJson)],\"total\":$(Num.toStr order.total)}"

# ═══════ DOMAIN → CSV ═══════

orderToCSV : { id : U64, items : List { name : Str, qty : U64, price : U64 }, total : U64 } -> Str
orderToCSV = \order ->
    header = "item,qty,price"
    rows = List.map order.items \item ->
        "$(item.name),$(Num.toStr item.qty),$(Num.toStr item.price)"
    footer = "TOTAL,,$(Num.toStr order.total)"

    [header]
    |> List.concat rows
    |> List.append footer
    |> Str.joinWith "\n"

main =
    order = {
        id: 42,
        items: [
            { name: "Phở", qty: 2, price: 45000 },
            { name: "Cà phê", qty: 1, price: 25000 },
        ],
        total: 115000,
    }

    Stdout.line! "=== JSON ==="
    Stdout.line! (orderToJson order)

    Stdout.line! "\n=== CSV ==="
    Stdout.line! (orderToCSV order)
    # Output:
    # === JSON ===
    # {"id":42,"items":[{"name":"Phở","qty":2,"price":45000},{"name":"Cà phê","qty":1,"price":25000}],"total":115000}
    #
    # === CSV ===
    # item,qty,price
    # Phở,2,45000
    # Cà phê,1,25000
    # TOTAL,,115000
```

---

## 21.7 — Tổng hợp: Read file → Parse → Domain → Process → Output

Pipeline hoàn chỉnh kết hợp mọi thứ: đọc file CSV, parse thành domain types, xử lý business logic, và output kết quả. Mỗi bước là một function, lỗi tự động propagate qua `?`.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Full pipeline: CSV input → Domain → Report output

# Pure: parse
parseStudentCSV = \content ->
    rows = Str.split content "\n"
        |> List.dropFirst 1    # bỏ header
        |> List.keepIf \l -> !(Str.isEmpty (Str.trim l))
    List.keepOks rows \line ->
        parts = Str.split line "," |> List.map Str.trim
        when (List.get parts 0, List.get parts 1, List.get parts 2) is
            (Ok name, Ok mathStr, Ok litStr) ->
                math = Str.toI64 mathStr |> Result.withDefault 0
                lit = Str.toI64 litStr |> Result.withDefault 0
                Ok { name, math, lit, avg: (math + lit) // 2 }
            _ -> Err InvalidRow

# Pure: analyze
analyzeClass = \students ->
    count = List.len students |> Num.toI64
    if count == 0 then
        { count: 0, avgScore: 0, topStudent: "N/A", passRate: 0 }
    else
        totalAvg = List.walk students 0 \s, st -> s + st.avg
        classAvg = totalAvg // count
        top = List.sortWith students \a, b -> Num.compare b.avg a.avg
            |> List.first |> Result.map .name |> Result.withDefault "N/A"
        passed = List.keepIf students \st -> st.avg >= 50 |> List.len |> Num.toI64
        passRate = passed * 100 // count
        { count, avgScore: classAvg, topStudent: top, passRate }

# Pure: format report
formatReport = \stats ->
    lines = [
        "╔════════════════════════════╗",
        "║     BÁO CÁO LỚP HỌC      ║",
        "╠════════════════════════════╣",
        "║ Sĩ số: $(Num.toStr stats.count)",
        "║ Điểm TB: $(Num.toStr stats.avgScore)",
        "║ Thủ khoa: $(stats.topStudent)",
        "║ Tỷ lệ đạt: $(Num.toStr stats.passRate)%",
        "╚════════════════════════════╝",
    ]
    Str.joinWith lines "\n"

main =
    # Giả sử đọc từ file (hardcoded cho demo)
    csvContent =
        """
        name, math, lit
        An, 95, 88
        Bình, 72, 65
        Cường, 45, 38
        Dũng, 88, 92
        Em, 55, 60
        """

    students = parseStudentCSV csvContent
    stats = analyzeClass students
    report = formatReport stats

    Stdout.line! report

    # Chi tiết từng học sinh
    Stdout.line! "\n=== Chi tiết ==="
    List.forEach students \st ->
        grade = if st.avg >= 80 then "Giỏi" else if st.avg >= 65 then "Khá" else if st.avg >= 50 then "TB" else "Yếu"
        Stdout.line! "  $(st.name): Toán=$(Num.toStr st.math) Văn=$(Num.toStr st.lit) TB=$(Num.toStr st.avg) → $(grade)"
```

---


## ✅ Checkpoint 21

> Đến đây bạn phải hiểu:
> 1. `Encode`/`Decode` abilities = serialize/deserialize tự động
> 2. Domain types ≠ DTO types — tách biệt internal model vs external format
> 3. JSON via platform — app định nghĩa types, platform handle format
>
> **Test nhanh**: `Encode` ability cho phép làm gì?
> <details><summary>Đáp án</summary>Tự động convert Roc values thành bytes (JSON, MessagePack, etc.).</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Config file parser

Viết parser cho file config dạng INI:

```ini
[server]
host=localhost
port=8080

[database]
url=postgres://localhost/mydb
pool_size=10
```

<details><summary>✅ Lời giải</summary>

```roc
parseINI = \content ->
    Str.split content "\n"
    |> List.walk { currentSection: "", sections: Dict.empty {} } \ctx, line ->
        trimmed = Str.trim line
        if Str.startsWith trimmed "[" && Str.endsWith trimmed "]" then
            section = trimmed |> Str.replaceFirst "[" "" |> Str.replaceFirst "]" ""
            { ctx & currentSection: section }
        else if Str.contains trimmed "=" then
            when Str.splitFirst trimmed "=" is
                Ok { before, after } ->
                    existing = Dict.get ctx.sections ctx.currentSection |> Result.withDefault (Dict.empty {})
                    updated = Dict.insert existing (Str.trim before) (Str.trim after)
                    { ctx & sections: Dict.insert ctx.sections ctx.currentSection updated }
                Err _ -> ctx
        else
            ctx
    |> .sections
```

</details>

---

**Bài 2** (15 phút): Domain ↔ JSON round-trip

Tạo `encode` và `decode` cho type Product:

```roc
# Product { name: "iPhone", price: 25000000, category: Electronics }
# → JSON: {"name":"iPhone","price":25000000,"category":"electronics"}
# → Product { name: "iPhone", price: 25000000, category: Electronics }
# Round-trip: decode (encode product) == Ok product
```

<details><summary>✅ Lời giải</summary>

```roc
encodeProduct = \p ->
    cat = when p.category is
        Electronics -> "electronics"
        Clothing -> "clothing"
        Food -> "food"
    "{\"name\":\"$(p.name)\",\"price\":$(Num.toStr p.price),\"category\":\"$(cat)\"}"

decodeProduct = \json ->
    name = extractField? json "name"
    priceStr = extractField? json "price"
    catStr = extractField? json "category"
    price = Str.toU64 priceStr |> Result.mapErr? \_ -> InvalidPrice
    category = when catStr is
        "electronics" -> Ok Electronics
        "clothing" -> Ok Clothing
        "food" -> Ok Food
        _ -> Err InvalidCategory
    cat = category?
    Ok { name, price, category: cat }

# Test round-trip
expect
    original = { name: "iPhone", price: 25000000, category: Electronics }
    encoded = encodeProduct original
    decoded = decodeProduct encoded
    decoded == Ok original
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| JSON parse bất ngờ fail | Field name sai case | Kiểm tra exact field names từ API |
| CSV split sai | Data chứa dấu phẩy | Dùng quoted CSV hoặc escape |
| Round-trip mất data | Encode bỏ sót fields | Test: `decode (encode x) == Ok x` |
| Domain type quá strict | External data có optional fields | Dùng `Result.withDefault` cho optional |

---

## Tóm tắt

- ✅ **Encode/Decode** = serialize (Domain→bytes) / deserialize (bytes→Domain). Always `Result` for decode.
- ✅ **JSON/CSV** = format phổ biến. Parse thành `Dict` hoặc `List` rồi convert sang domain types.
- ✅ **Anti-Corruption Layer** = parse external → domain type NGAY tại boundary. "Parse, don't validate."
- ✅ **Domain → Output** = `toJson`, `toCSV` — pure functions format domain data.
- ✅ **Full pipeline** = Read file → Parse → Domain → Process → Format → Output.
- ✅ **Round-trip test** = `decode (encode x) == Ok x` — đảm bảo không mất data.

## 🎉 Kết thúc Part IV — Domain-Driven Design!

| Chapter | Chủ đề |
|---------|--------|
| 17 | DDD Introduction — Ubiquitous Language, Value Objects, Bounded Contexts |
| 18 | Domain Modeling — State Machines, Event-Sourced Models |
| 19 | Workflows as Pipelines — `?` chains, composable steps |
| 20 | Platform Separation — Compiler-enforced Onion Architecture |
| **21** | **Serialization — JSON, CSV, Anti-Corruption Layer** |

## Tiếp theo

→ **Part V: Testing & Apps** — Chapter 22: **Testing** — `expect` keyword, TDD cycle, test organization, property-based testing concepts.
