# Chapter 9 — Lists & Standard Functions

> **Bạn sẽ học được**:
> - Toàn bộ List API quan trọng — từ cơ bản đến nâng cao
> - Dict và Set — API đầy đủ và patterns thực tế
> - **Data processing pipelines** — kết hợp nhiều functions thành workflow
> - Cách chọn đúng function cho bài toán
> - Patterns phổ biến: groupBy, partition, zip, flatten
>
> **Yêu cầu trước**: [Chapter 8 — Pattern Matching](chapter_08_pattern_matching.md)
> **Thời gian đọc**: ~45 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Thành thạo List/Dict/Set API — xử lý bất kỳ bài toán data nào mà không cần vòng lặp.

---

Roc không có vòng lặp. Không có `for`, không có `while`, không có `loop`. Nghe giới hạn, nhưng thực ra đây là một thiết kế có chủ đích. Thay vì viết "lặp qua từng phần tử và làm gì đó", bạn nói với Roc: "biến đổi từng phần tử theo cách này" (`map`), "giữ những phần tử thỏa điều kiện này" (`keepIf`), "gộp tất cả thành một kết quả" (`walk`).

Ba functions đó — `map`, `keepIf`, `walk` — là ba trụ cột. Hầu hết mọi xử lý dữ liệu trong Roc đều quy về sự kết hợp của chúng qua pipeline.

## 9.1 — List API: Bản đồ toàn cảnh

Roc có khoảng 50 functions cho List. Đừng cố học hết — hãy hiểu **5 nhóm chính** trước:

| Nhóm | Functions | Ý nghĩa |
|------|-----------|---------|
| **Tạo** | `List.range`, `List.repeat`, `List.concat` | Tạo list mới |
| **Biến đổi** | `List.map`, `List.mapWithIndex` | 1 phần tử → 1 phần tử mới |
| **Lọc** | `List.keepIf`, `List.dropIf`, `List.keepOks`, `List.keepErrs` | Giữ/bỏ phần tử |
| **Gom** | `List.walk`, `List.walkBackwards` | Nhiều phần tử → 1 kết quả |
| **Tìm** | `List.first`, `List.last`, `List.get`, `List.findFirst` | Tìm phần tử cụ thể |

---

## 9.2 — Tạo Lists

Trước khi biến đổi dữ liệu, bạn cần có dữ liệu. Roc cung cấp nhiều cách tạo list — từ viết trực tiếp đến sinh ra từ range hoặc nối nhiều lists:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Tạo trực tiếp
    fruits = ["Táo", "Cam", "Xoài"]

    # List.range — dãy số
    oneToTen = List.range { start: At 1, end: At 10 }
    Stdout.line! "1→10: $(Inspect.toStr oneToTen)"

    # List.repeat — lặp lại giá trị
    fiveZeros = List.repeat 0 5
    Stdout.line! "5 zeros: $(Inspect.toStr fiveZeros)"
    # → [0, 0, 0, 0, 0]

    # List.concat — nối 2 lists
    ab = List.concat [1, 2, 3] [4, 5, 6]
    Stdout.line! "Nối: $(Inspect.toStr ab)"
    # → [1, 2, 3, 4, 5, 6]

    # List.append / List.prepend — thêm 1 phần tử
    withSix = List.append [1, 2, 3] 6
    withZero = List.prepend [1, 2, 3] 0
    Stdout.line! "Append 6: $(Inspect.toStr withSix)"     # [1,2,3,6]
    Stdout.line! "Prepend 0: $(Inspect.toStr withZero)"    # [0,1,2,3]

    # List.join — flatten list of lists
    nested = [[1, 2], [3, 4], [5]]
    flat = List.join nested
    Stdout.line! "Flatten: $(Inspect.toStr flat)"
    # → [1, 2, 3, 4, 5]
```

---

## 9.3 — Biến đổi: `map` và bạn bè

`List.map` là function bạn sẽ dùng nhiều nhất. Nó nhận một list và một function, áp dụng function đó vào từng phần tử, trả về list mới cùng độ dài. Phần tử gốc không bị thay đổi.

### `List.map` — biến đổi từng phần tử

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    prices = [45000, 50000, 35000, 60000]

    # Tính thuế 10%
    withTax = List.map prices \p -> p + (p * 10 // 100)
    Stdout.line! "Sau thuế: $(Inspect.toStr withTax)"
    # → [49500, 55000, 38500, 66000]

    # Format thành chuỗi
    formatted = List.map prices \p ->
        "$(Num.toStr p)đ"
    Stdout.line! "Formatted: $(Inspect.toStr formatted)"
    # → ["45000đ", "50000đ", "35000đ", "60000đ"]

    # map với records
    students = [
        { name: "An", score: 85 },
        { name: "Bình", score: 92 },
        { name: "Cường", score: 78 },
    ]
    names = List.map students \s -> s.name
    Stdout.line! "Tên: $(Inspect.toStr names)"
    # → ["An", "Bình", "Cường"]
```

### `List.mapWithIndex` — biến đổi kèm index

```roc
main =
    items = ["Phở", "Bún bò", "Cơm tấm"]

    numbered = List.mapWithIndex items \item, index ->
        "$(Num.toStr (index + 1)). $(item)"
    # → ["1. Phở", "2. Bún bò", "3. Cơm tấm"]

    List.forEach numbered \line -> Stdout.line! line
```

---

## 9.4 — Lọc: `keepIf`, `dropIf`, `keepOks`

Nếu `map` biến đổi từng phần tử, thì `keepIf` chọn những phần tử bạn muốn giữ. List đầu ra có thể ngắn hơn list đầu vào — hoặc rỗng nếu không phần tử nào thỏa.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    # keepIf — giữ phần tử thỏa điều kiện
    evens = List.keepIf numbers \x -> x %% 2 == 0
    Stdout.line! "Chẵn: $(Inspect.toStr evens)"
    # → [2, 4, 6, 8, 10]

    # dropIf — bỏ phần tử thỏa điều kiện (ngược keepIf)
    noSmall = List.dropIf numbers \x -> x < 5
    Stdout.line! "≥ 5: $(Inspect.toStr noSmall)"
    # → [5, 6, 7, 8, 9, 10]

    # keepOks — lọc Results, chỉ giữ Ok values
    rawInputs = ["42", "hello", "7", "abc", "99"]
    validNumbers = List.keepOks rawInputs Str.toI64
    Stdout.line! "Parsed: $(Inspect.toStr validNumbers)"
    # → [42, 7, 99] (bỏ "hello" và "abc")

    # keepErrs — ngược lại, chỉ giữ Err values
    errors = List.keepErrs rawInputs \s ->
        when Str.toI64 s is
            Ok _ -> Err (InvalidInput s)
            Err _ -> Ok s
    # Giữ những chuỗi không parse được
```

### Lọc records

```roc
main =
    products = [
        { name: "iPhone", price: 25000000, inStock: Bool.true },
        { name: "Galaxy", price: 20000000, inStock: Bool.false },
        { name: "Pixel", price: 18000000, inStock: Bool.true },
        { name: "OnePlus", price: 15000000, inStock: Bool.true },
    ]

    # Chỉ giữ sản phẩm còn hàng VÀ giá < 20 triệu
    affordable = products
        |> List.keepIf \p -> p.inStock
        |> List.keepIf \p -> p.price < 20000000

    List.forEach affordable \p ->
        Stdout.line! "$(p.name): $(Num.toStr p.price)đ"
    # Output:
    # Pixel: 18000000đ
    # OnePlus: 15000000đ
```

---

## 9.5 — Gom: `List.walk` nâng cao

`List.walk` là function **mạnh nhất** trong List API. Thực tế, bạn có thể viết `map`, `keepIf`, và mọi function khác bằng `walk` — nó là "ngôn ngữ assembly" của xử lý list. Ý tưởng: bắt đầu với một giá trị khởi tạo, đi qua từng phần tử, cập nhật giá trị đó, và trả về kết quả cuối cùng.

Chapter 2 đã giới thiệu `List.walk` cơ bản. Giờ hãy xem các patterns nâng cao mà bạn sẽ dùng trong dự án thực tế:

### Pattern: Group by

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    transactions = [
        { item: "Phở", amount: 45000, category: "Food" },
        { item: "Cà phê", amount: 25000, category: "Drink" },
        { item: "Bún bò", amount: 50000, category: "Food" },
        { item: "Trà đá", amount: 5000, category: "Drink" },
        { item: "Cơm tấm", amount: 40000, category: "Food" },
        { item: "Sinh tố", amount: 30000, category: "Drink" },
    ]

    # Group by category → Dict category (List transaction)
    grouped = List.walk transactions (Dict.empty {}) \dict, t ->
        existing = Dict.get dict t.category |> Result.withDefault []
        Dict.insert dict t.category (List.append existing t)

    # Tính tổng mỗi category
    Dict.walk grouped {} \_, category, items ->
        total = List.walk items 0 \s, t -> s + t.amount
        count = List.len items
        Stdout.line! "$(category): $(Num.toStr count) món, tổng $(Num.toStr total)đ"
    # Output:
    # Food: 3 món, tổng 135000đ
    # Drink: 3 món, tổng 60000đ
```

### Pattern: Partition (chia 2 nhóm)

```roc
main =
    numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    # Walk để chia thành 2 list
    partitioned = List.walk numbers { evens: [], odds: [] } \groups, x ->
        if x %% 2 == 0 then
            { groups & evens: List.append groups.evens x }
        else
            { groups & odds: List.append groups.odds x }

    Stdout.line! "Chẵn: $(Inspect.toStr partitioned.evens)"
    Stdout.line! "Lẻ: $(Inspect.toStr partitioned.odds)"
    # Chẵn: [2, 4, 6, 8, 10]
    # Lẻ: [1, 3, 5, 7, 9]
```

### Pattern: Running statistics

```roc
main =
    scores = [85, 92, 78, 95, 88, 73, 91]

    stats = List.walk scores { min: Num.maxI64, max: 0, sum: 0, count: 0 } \s, x ->
        {
            min: if x < s.min then x else s.min,
            max: if x > s.max then x else s.max,
            sum: s.sum + x,
            count: s.count + 1,
        }

    avg = stats.sum // stats.count
    Stdout.line! "Min: $(Num.toStr stats.min)"
    Stdout.line! "Max: $(Num.toStr stats.max)"
    Stdout.line! "Avg: $(Num.toStr avg)"
    Stdout.line! "Count: $(Num.toStr stats.count)"
    # Min: 73, Max: 95, Avg: 86, Count: 7
```

---

## 9.6 — Tìm kiếm & Sắp xếp

`List.findFirst`, `List.sortWith` — hai operations phổ biến khi xử lý data. Sort trong Roc nhận comparator function — bạn quyết định thứ tự.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    students = [
        { name: "An", score: 85 },
        { name: "Bình", score: 92 },
        { name: "Cường", score: 78 },
        { name: "Dũng", score: 95 },
        { name: "Em", score: 88 },
    ]

    # findFirst — tìm phần tử đầu tiên thỏa điều kiện
    topStudent = List.findFirst students \s -> s.score >= 90
    when topStudent is
        Ok s -> Stdout.line! "Top student: $(s.name) ($(Num.toStr s.score))"
        Err NotFound -> Stdout.line! "Không có ai ≥ 90"
    # → Top student: Bình (92)

    # sortWith — sắp xếp với comparator
    byScore = List.sortWith students \a, b ->
        Num.compare b.score a.score    # giảm dần
    List.forEach byScore \s ->
        Stdout.line! "  $(s.name): $(Num.toStr s.score)"
    # Dũng: 95, Bình: 92, Em: 88, An: 85, Cường: 78

    # any / all — kiểm tra điều kiện
    anyFail = List.any students \s -> s.score < 50
    allPass = List.all students \s -> s.score >= 50
    Stdout.line! "Có ai trượt? $(Inspect.toStr anyFail)"     # Bool.false
    Stdout.line! "Tất cả đậu? $(Inspect.toStr allPass)"     # Bool.true

    # len, isEmpty
    Stdout.line! "Sĩ số: $(Num.toStr (List.len students))"
    Stdout.line! "Rỗng? $(Inspect.toStr (List.isEmpty students))"
```

---

## 9.7 — Dict & Set: Patterns nâng cao

List lưu data theo thứ tự. Nhưng khi cần tra cứu nhanh theo key ("tìm user theo email"), hoặc loại bỏ trùng lặp, bạn cần **Dict** và **Set**. Dict là key-value store, Set là tập hợp giá trị duy nhất.

### Dict: Frequency counter

Đếm tần suất là pattern phổ biến nhất với Dict — đếm số lần xuất hiện của mỗi giá trị:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Đếm tần suất — pattern phổ biến nhất
countFrequency = \list ->
    List.walk list (Dict.empty {}) \dict, item ->
        current = Dict.get dict item |> Result.withDefault 0
        Dict.insert dict item (current + 1)

main =
    votes = ["A", "B", "A", "C", "A", "B", "A", "C", "B", "A"]

    freq = countFrequency votes
    Dict.walk freq {} \_, candidate, count ->
        bar = Str.repeat "█" count
        Stdout.line! "$(candidate): $(bar) ($(Num.toStr count))"
    # Output:
    # A: █████ (5)
    # B: ███ (3)
    # C: ██ (2)
```

### Dict: Lookup table + default

```roc
main =
    # Dictionary làm lookup table
    emojiMap = Dict.fromList [
        ("sun", "☀️"),
        ("rain", "🌧️"),
        ("snow", "❄️"),
        ("cloud", "☁️"),
    ]

    weatherWords = ["sun", "rain", "wind", "snow"]

    emojis = List.map weatherWords \word ->
        Dict.get emojiMap word |> Result.withDefault "❓"
    # → ["☀️", "🌧️", "❓", "❄️"]

    Stdout.line! (Str.joinWith emojis " ")
```

### Set: Venn diagram operations

```roc
main =
    classA = Set.fromList ["An", "Bình", "Cường", "Dũng"]
    classB = Set.fromList ["Cường", "Dũng", "Em", "Phúc"]

    # Học cả 2 lớp
    both = Set.intersection classA classB
    # → {"Cường", "Dũng"}

    # Chỉ lớp A
    onlyA = Set.difference classA classB
    # → {"An", "Bình"}

    # Tất cả
    all = Set.union classA classB
    # → {"An", "Bình", "Cường", "Dũng", "Em", "Phúc"}

    Stdout.line! "Cả 2: $(Num.toStr (Set.len both))"
    Stdout.line! "Chỉ A: $(Num.toStr (Set.len onlyA))"
    Stdout.line! "Tất cả: $(Num.toStr (Set.len all))"
```

---

## 9.8 — Pipeline Master Class

Mọi thứ bạn đã học — map, keepIf, walk, Dict, Set — kết hợp lại thành **data processing pipeline**. Ví dụ sau mô phỏng một báo cáo doanh thu thực tế: từ dữ liệu thô, qua nhiều bước xử lý, ra kết quả có nghĩa.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Raw data — danh sách đơn hàng
    orders = [
        { customer: "An", items: ["Phở", "Cà phê"], total: 70000, status: Delivered },
        { customer: "An", items: ["Bún bò"], total: 50000, status: Delivered },
        { customer: "Bình", items: ["Cơm tấm", "Trà đá"], total: 45000, status: Preparing },
        { customer: "Cường", items: ["Phở", "Sinh tố"], total: 75000, status: Delivered },
        { customer: "An", items: ["Cơm tấm"], total: 40000, status: Cancelled },
        { customer: "Bình", items: ["Phở"], total: 45000, status: Delivered },
    ]

    # Pipeline: Báo cáo doanh thu theo khách hàng (chỉ tính đơn delivered)
    Stdout.line! "=== Báo cáo doanh thu ==="

    # Bước 1: Lọc đơn đã giao
    delivered = orders |> List.keepIf \o -> o.status == Delivered

    # Bước 2: Group by customer
    byCustomer = List.walk delivered (Dict.empty {}) \dict, o ->
        existing = Dict.get dict o.customer |> Result.withDefault []
        Dict.insert dict o.customer (List.append existing o)

    # Bước 3: Tính stats cho mỗi customer
    Dict.walk byCustomer {} \_, customer, customerOrders ->
        total = List.walk customerOrders 0 \s, o -> s + o.total
        count = List.len customerOrders
        avgOrder = total // (Num.toI64 count)
        allItems = customerOrders
            |> List.map \o -> o.items
            |> List.join
            |> List.len

        Stdout.line! "$(customer): $(Num.toStr count) đơn, $(Num.toStr allItems) món, tổng $(Num.toStr total)đ (TB: $(Num.toStr avgOrder)đ)"

    # Bước 4: Tổng kết
    grandTotal = List.walk delivered 0 \s, o -> s + o.total
    Stdout.line! "---"
    Stdout.line! "Tổng doanh thu: $(Num.toStr grandTotal)đ"
    # Output:
    # === Báo cáo doanh thu ===
    # An: 2 đơn, 3 món, tổng 120000đ (TB: 60000đ)
    # Cường: 1 đơn, 2 món, tổng 75000đ (TB: 75000đ)
    # Bình: 1 đơn, 1 món, tổng 45000đ (TB: 45000đ)
    # ---
    # Tổng doanh thu: 240000đ
```

---


## ✅ Checkpoint 9

> Đến đây bạn phải hiểu:
> 1. `List.map` = biến đổi mỗi phần tử, `List.keepIf` = lọc, `List.walk` = fold
> 2. Pipeline `|>` biến chuỗi operations thành dây chuyền dễ đọc
> 3. `List.walk` là general nhất — mọi operations khác đều có thể viết bằng `walk`
>
> **Test nhanh**: `List.map [1,2,3] (\x -> x * 2)` trả về gì?
> <details><summary>Đáp án</summary>`[2, 4, 6]` — map biến đổi từng phần tử, giữ nguyên cấu trúc list.</details>

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Chọn đúng function

Chọn function phù hợp cho mỗi bài toán:

```
a) Chuyển list giá VNĐ sang USD (1 USD = 25000)
b) Lọc email hợp lệ (chứa "@")
c) Tính tổng lương cả công ty
d) Tìm sản phẩm đầu tiên giá < 100k
e) Nối 2 list khách hàng
```

<details><summary>✅ Lời giải</summary>

```
a) List.map — biến đổi từng phần tử
b) List.keepIf — lọc theo điều kiện
c) List.walk — gom nhiều → 1
d) List.findFirst — tìm phần tử đầu tiên
e) List.concat — nối 2 lists
```

</details>

---

**Bài 2** (10 phút): Word analyzer

Viết pipeline phân tích đoạn văn:

```roc
text = "the quick brown fox jumps over the lazy dog the fox"

# Yêu cầu:
# 1. Tách thành từ
# 2. Đếm tổng số từ
# 3. Đếm số từ duy nhất
# 4. Tìm từ xuất hiện nhiều nhất
# 5. Sắp xếp từ theo alphabet
```

<details><summary>✅ Lời giải</summary>

```roc
main =
    text = "the quick brown fox jumps over the lazy dog the fox"
    words = Str.split text " "

    # Tổng
    Stdout.line! "Tổng từ: $(Num.toStr (List.len words))"

    # Duy nhất
    unique = Set.fromList words
    Stdout.line! "Từ duy nhất: $(Num.toStr (Set.len unique))"

    # Tần suất
    freq = List.walk words (Dict.empty {}) \d, w ->
        c = Dict.get d w |> Result.withDefault 0
        Dict.insert d w (c + 1)

    # Từ nhiều nhất
    mostCommon = Dict.walk freq { word: "", count: 0 } \best, word, count ->
        if count > best.count then { word, count } else best
    Stdout.line! "Nhiều nhất: \"$(mostCommon.word)\" ($(Num.toStr mostCommon.count)x)"

    # Sort alphabet
    sorted = List.sortWith (Set.toList unique) Str.compare
    Stdout.line! "Alphabet: $(Inspect.toStr sorted)"
```

</details>

---

**Bài 3** (15 phút): Student report card

```roc
students = [
    { name: "An", subjects: [{ name: "Toán", score: 85 }, { name: "Văn", score: 72 }, { name: "Anh", score: 90 }] },
    { name: "Bình", subjects: [{ name: "Toán", score: 95 }, { name: "Văn", score: 88 }, { name: "Anh", score: 76 }] },
    { name: "Cường", subjects: [{ name: "Toán", score: 60 }, { name: "Văn", score: 55 }, { name: "Anh", score: 68 }] },
]

# Yêu cầu:
# 1. Tính điểm trung bình mỗi học sinh
# 2. Xếp loại: ≥85 Giỏi, ≥70 Khá, ≥50 TB, <50 Yếu
# 3. Sắp xếp theo TB giảm dần
# 4. Tìm môn có điểm cao nhất toàn trường
```

<details><summary>✅ Lời giải</summary>

```roc
main =
    students = [ ... ] # như trên

    # 1 + 2: TB và xếp loại
    results = List.map students \s ->
        total = List.walk s.subjects 0 \sum, sub -> sum + sub.score
        avg = total // (Num.toI64 (List.len s.subjects))
        grade = if avg >= 85 then "Giỏi"
                else if avg >= 70 then "Khá"
                else if avg >= 50 then "TB"
                else "Yếu"
        { name: s.name, avg, grade, subjects: s.subjects }

    # 3: Sort giảm dần
    sorted = List.sortWith results \a, b -> Num.compare b.avg a.avg
    List.forEach sorted \r ->
        Stdout.line! "$(r.name): $(Num.toStr r.avg) ($(r.grade))"

    # 4: Điểm cao nhất toàn trường
    allScores = students
        |> List.map \s -> s.subjects
        |> List.join
    best = List.walk allScores { name: "", score: 0 } \b, sub ->
        if sub.score > b.score then sub else b
    Stdout.line! "Cao nhất: $(best.name) = $(Num.toStr best.score)"
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `List.walk` cho sai kết quả | Giá trị khởi tạo sai | Tổng→0, tích→1, list→[], record→{...} |
| `List.keepOks` loại hết | Function truyền vào trả `Err` cho mọi phần tử | Kiểm tra function — debug với 1 phần tử trước |
| `List.sortWith` không sort | Comparator trả sai thứ tự | `Num.compare a b` = tăng, `Num.compare b a` = giảm |
| Dict.walk thứ tự lạ | Dict **không đảm bảo** thứ tự | Sort keys trước nếu cần thứ tự |
| Type mismatch trong pipeline | Output bước trước ≠ input bước sau | Kiểm tra type từng bước — comment dần từ dưới lên |

---

## Tóm tắt

- ✅ **5 nhóm List API**: Tạo (range, repeat, concat) → Biến đổi (map) → Lọc (keepIf, keepOks) → Gom (walk) → Tìm (findFirst, any, all)
- ✅ **List.walk patterns**: groupBy (→Dict), partition (→2 lists), running stats (→record), frequency counter
- ✅ **Dict patterns**: Frequency counter, lookup table + default, group by
- ✅ **Set patterns**: Union/intersection/difference, deduplication
- ✅ **Pipeline** = kết hợp map → keepIf → walk thành data processing workflow

## Tiếp theo

→ Chapter 10: **Opaque Types & Modules** — tạo domain types an toàn (`Email := Str`), smart constructors, module system (`exposes`, `imports`), và cách tổ chức code Roc cho dự án lớn.
