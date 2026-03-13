# Chapter 23 — PBT Concepts

> **Bạn sẽ học được**:
> - **Property-Based Testing** (PBT) — tư duy "properties" thay vì "examples"
> - 7 loại properties: round-trip, idempotency, invariant, metamorphic, oracle, symmetry, hard-to-prove/easy-to-verify
> - Viết PBT tests trong Roc bằng `expect` + helper generators
> - Khi nào dùng example tests, khi nào dùng PBT
> - Domain modeling + PBT = code tự chứng minh đúng
>
> **Yêu cầu trước**: [Chapter 22 — Testing](chapter_22_testing.md)
> **Thời gian đọc**: ~40 phút | **Level**: Principal
> **Kết quả cuối cùng**: Viết 1 property test thay 50 example tests — bắt bugs mà examples bỏ sót.

---

Unit test kiểm tra 5-10 ví dụ cụ thể. Property-Based Testing (PBT) kiểm tra **hàng nghìn** ví dụ ngẫu nhiên — và tìm ra edge cases mà bạn không bao giờ nghĩ tới. Kết hợp với pure functions, PBT trở thành vũ khí mạnh nhất để đảm bảo code đúng.

## 23.1 — Example Tests vs Property Tests

Example test: `expect add 2 3 == 5`. Property test: `expect add a b == add b a` **cho mọi a, b**. Framework tự sinh hàng nghìn inputs — tìm edge cases bạn không nghĩ tới.

### Example tests: "cho input X, kỳ vọng output Y"

```roc
expect List.reverse [1, 2, 3] == [3, 2, 1]
expect List.reverse [4, 5] == [5, 4]
expect List.reverse [] == []
```

3 cases. Nếu bug xảy ra ở list 1000 phần tử? Bạn không test được hết.

### Property tests: "với MỌI input, TÍNH CHẤT này luôn đúng"

```roc
# Property: reverse reverse = identity
# Đúng cho MỌI list, không chỉ [1,2,3]
expect
    list = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    List.reverse (List.reverse list) == list

expect
    list = [42]
    List.reverse (List.reverse list) == list

expect
    list = []
    List.reverse (List.reverse list) == list
```

> **💡 Ẩn dụ**: Example tests = **flashlight** — sáng đúng chỗ bạn chiếu. Property tests = **đèn sân** — sáng khắp nơi.

---

## 23.2 — Property 1: Round-Trip (Encode → Decode → Original)

Property là invariant — điều luôn đúng bất kể input. "Sort rồi unsort cho list gốc" → sai. "Sort hai lần cho cùng kết quả" → đúng. Chọn property đúng là kỹ năng quan trọng nhất.

### Khái niệm

Nếu bạn **encode** rồi **decode**, phải về lại giá trị ban đầu:

```
value → encode → encoded → decode → value' → value' == value ✅
```

```roc
# ═══════ DOMAIN ═══════

encode = \n -> Num.toStr n
decode = \s -> Str.toI64 s

# ═══════ ROUND-TRIP PROPERTY ═══════

# ∀ n : decode(encode(n)) == Ok n
expect decode (encode 0) == Ok 0
expect decode (encode 42) == Ok 42
expect decode (encode (-100)) == Ok (-100)
expect decode (encode 999999) == Ok 999999

# Test với nhiều values
expect
    testValues = [0, 1, -1, 42, -42, 100, -100, 999999, -999999]
    List.all testValues \n ->
        decode (encode n) == Ok n
```

### Ứng dụng thực tế

```roc
# CSV round-trip
toCSVRow = \{ name, age, score } ->
    "$(name),$(Num.toStr age),$(Num.toStr score)"

fromCSVRow = \row ->
    parts = Str.split row ","
    when (List.get parts 0, List.get parts 1, List.get parts 2) is
        (Ok name, Ok ageStr, Ok scoreStr) ->
            when (Str.toU8 ageStr, Str.toI64 scoreStr) is
                (Ok age, Ok score) -> Ok { name, age, score }
                _ -> Err ParseError
        _ -> Err ParseError

# Round-trip property
expect
    original = { name: "An", age: 25, score: 95 }
    fromCSVRow (toCSVRow original) == Ok original

expect
    records = [
        { name: "An", age: 25, score: 95 },
        { name: "Binh", age: 30, score: 82 },
        { name: "Cuong", age: 20, score: 67 },
    ]
    List.all records \r ->
        fromCSVRow (toCSVRow r) == Ok r
```

---

## 23.3 — Property 2: Idempotency (Làm lại = giữ nguyên)

Generator tạo random values cho mỗi lần test. `Gen.int`, `Gen.str`, `Gen.list` — tùy chỉnh generator để tạo data phù hợp domain: positive numbers, valid emails, non-empty lists.

### Khái niệm

Áp dụng function **2 lần** cho cùng kết quả như **1 lần**:

```
f(f(x)) == f(x)
```

```roc
# ═══════ IDEMPOTENT FUNCTIONS ═══════

normalize = \s -> Str.trim s |> \t -> Str.replaceEach t "  " " "
sortList = \list -> List.sortAsc list
deduplicate = \list -> List.walk list [] \acc, item ->
    if List.contains acc item then acc else List.append acc item

# ═══════ IDEMPOTENCY PROPRYẾT ═══════

# Trim + normalize: apply 2 lần = apply 1 lần
expect
    input = "  hello   world  "
    once = normalize input
    twice = normalize once
    once == twice

# Sort: already sorted → sort again = same
expect
    list = [5, 3, 1, 4, 2]
    once = sortList list
    twice = sortList once
    once == twice

# Deduplicate: remove dupes twice = same as once
expect
    list = [1, 2, 2, 3, 3, 3, 1]
    once = deduplicate list
    twice = deduplicate once
    once == twice

# Batch test
expect
    inputs = ["  hi  ", "already clean", " multiple   spaces  here "]
    List.all inputs \s ->
        normalize (normalize s) == normalize s
```

---

## 23.4 — Property 3: Invariants (Luôn đúng, mọi input)

Shrinking tìm **input nhỏ nhất** gây lỗi. Thay vì báo lỗi "test failed with list of 1000 elements", PBT tìm "test failed with [0, -1]" — dễ debug hơn nhiều.

### Khái niệm

Một tính chất **luôn đúng** bất kể input:

```roc
# ═══════ INVARIANT: length sau sort không đổi ═══════

expect
    lists = [[3, 1, 2], [5, 4, 3, 2, 1], [], [42], [1, 1, 1]]
    List.all lists \list ->
        List.len (sortList list) == List.len list

# ═══════ INVARIANT: filter chỉ giảm, không tăng ═══════

expect
    lists = [[1, 2, 3, 4, 5], [], [10, 20, 30]]
    List.all lists \list ->
        filtered = List.keepIf list \x -> x > 2
        List.len filtered <= List.len list

# ═══════ INVARIANT: split + join = original ═══════

expect
    strings = ["hello world", "a,b,c", "one", ""]
    List.all strings \s ->
        Str.split s " " |> Str.joinWith " " == s

# ═══════ DOMAIN INVARIANT: total >= 0 ═══════

expect
    carts = [
        { items: [{ price: 100, qty: 2 }], discount: 0 },
        { items: [{ price: 50, qty: 1 }], discount: 50 },
        { items: [], discount: 0 },
    ]
    List.all carts \cart ->
        total = List.walk cart.items 0 \s, i -> s + i.price * i.qty
        discounted = total - (total * cart.discount // 100)
        discounted >= 0
```

---

## 23.5 — Property 4: Metamorphic (Thay đổi input → dự đoán output)

Round-trip property: `decode(encode(x)) == x`. Nếu serialize rồi deserialize cho ra giá trị gốc, serialization logic đúng. Pattern cực mạnh cho testing serialization.

### Khái niệm

Nếu thay đổi input theo cách **biết trước**, output phải thay đổi **tương ứng**:

```roc
# ═══════ METAMORPHIC: sort giữ nguyên khi thêm phần tử đã tồn tại ═══════

expect
    list = [3, 1, 4, 1, 5]
    sorted = sortList list
    # Thêm phần tử đã có → sort kết quả khác nhưng deduplicate giống
    withDupe = List.append list 3
    deduplicate (sortList withDupe) == deduplicate sorted

# ═══════ METAMORPHIC: multiply cả input → output nhân theo ═══════

expect
    items = [{ price: 100, qty: 2 }, { price: 50, qty: 3 }]
    total1 = List.walk items 0 \s, i -> s + i.price * i.qty
    # Nhân giá ×2 → total cũng ×2
    doubled = List.map items \i -> { i & price: i.price * 2 }
    total2 = List.walk doubled 0 \s, i -> s + i.price * i.qty
    total2 == total1 * 2

# ═══════ METAMORPHIC: reverse preserves length ═══════

expect
    list = [1, 2, 3, 4, 5]
    List.len (List.reverse list) == List.len list
```

---

## 23.6 — Property 5: Symmetry (A → B, B → A)

Idempotent property: `f(f(x)) == f(x)`. Sort đã sorted list → không đổi. Format đã formatted string → không đổi. Phát hiện bug khi function thay đổi state bất ngờ.

```roc
# ═══════ SYMMETRY: add và subtract ═══════

expect
    pairs = [(100, 30), (50, 50), (0, 0), (200, 1)]
    List.all pairs \(a, b) ->
        result = a + b - b
        result == a

# ═══════ SYMMETRY: encode/decode (= round-trip!) ═══════

expect
    values = [VND, USD, EUR]
    List.all values \currency ->
        encoded = when currency is
            VND -> "VND"
            USD -> "USD"
            EUR -> "EUR"
        decoded = when encoded is
            "VND" -> Ok VND
            "USD" -> Ok USD
            "EUR" -> Ok EUR
            _ -> Err Unknown
        decoded == Ok currency
```

---

## 23.7 — Property 6: Hard to Prove, Easy to Verify

Commutativity, associativity — các tính chất toán học áp dụng cho code. `merge(a, b) == merge(b, a)` cho conflict-free operations. Math properties → robust code.

### Khái niệm

Khó tìm đáp án, nhưng DỄ kiểm tra đáp án đúng:

```roc
# ═══════ SORT: khó sort, dễ verify sorted ═══════

isSorted = \list ->
    List.walk (List.dropFirst list 1) { prev: List.first list, sorted: Bool.true } \state, current ->
        when state.prev is
            Ok p -> { prev: Ok current, sorted: state.sorted && p <= current }
            Err _ -> { prev: Ok current, sorted: Bool.true }
    |> .sorted

expect
    lists = [[5, 3, 1, 4, 2], [1], [], [3, 3, 3], [10, 9, 8, 7]]
    List.all lists \list ->
        isSorted (sortList list)

# ═══════ FILTER: easy to verify all elements match predicate ═══════

expect
    list = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    filtered = List.keepIf list \x -> x > 5
    List.all filtered \x -> x > 5

# ═══════ DOMAIN: valid order → all items have qty > 0 ═══════

expect
    items = [
        { name: "A", qty: 2, price: 100 },
        { name: "B", qty: 1, price: 200 },
        { name: "C", qty: 3, price: 50 },
    ]
    # Sau validate, mọi qty phải > 0
    List.all items \item -> item.qty > 0
```

---

## 23.8 — PBT cho Domain Models

Oracle testing: so sánh implementation mới với implementation cũ (hoặc brute-force). Nếu cả hai cho cùng kết quả trên mọi input → implementation mới đúng.

### Event Sourcing: state rebuilding property

```roc
# PROPERTY: Rebuild state từ events[0..n] rồi apply event[n+1]
#         == Rebuild state từ events[0..n+1]

applyEvent = \state, event ->
    when event is
        Deposited { amount } -> { state & balance: state.balance + amount }
        Withdrawn { amount } -> { state & balance: state.balance - amount }

expect
    events = [Deposited { amount: 100 }, Withdrawn { amount: 30 }, Deposited { amount: 50 }]
    initialState = { balance: 0 }

    # Rebuild tất cả cùng lúc
    fullRebuild = List.walk events initialState applyEvent

    # Rebuild từng bước
    step1 = List.walk (List.takeFirst events 1) initialState applyEvent
    step2 = List.walk (List.takeFirst events 2) initialState applyEvent
    step3 = List.walk (List.takeFirst events 3) initialState applyEvent

    # Property: incremental == full
    step3 == fullRebuild
    && applyEvent step2 (Deposited { amount: 50 }) == step3
```

### Shopping cart properties

```roc
# PROPERTY: addItem then removeItem = original
expect
    cart = { items: [{ name: "A", qty: 1 }] }
    cart
    |> addItem "B" 100 1
    |> removeItem "B"
    == cart

# PROPERTY: total = sum of (price × qty) for all items
expect
    items = [
        { name: "A", price: 100, qty: 2 },
        { name: "B", price: 50, qty: 3 },
    ]
    cart = { items, discount: 0 }
    expectedTotal = 100 * 2 + 50 * 3
    subtotal cart == expectedTotal

# PROPERTY: empty cart total = 0
expect subtotal { items: [], discount: 0 } == 0
```

---

## 23.9 — Khi nào dùng gì?

PBT trong thực tế: không thay thế unit tests, mà bổ sung. Unit tests cho readability, PBT cho confidence. Dùng PBT cho core domain logic, parsing, serialization.

| Tiêu chí | Example Tests | Property Tests |
|----------|--------------|----------------|
| Khi nào | Behavior cụ thể, edge cases | Tính chất chung |
| Ưu điểm | Dễ đọc, dễ debug | Cover nhiều cases hơn |
| Nhược điểm | Chỉ test cases bạn nghĩ ra | Khó tìm property đúng |
| Ví dụ | `add 2 3 == 5` | `∀ a,b: add a b == add b a` |

**Thực tế**: Dùng **cả hai**. Examples cho happy path + edge cases. Properties cho invariants + round-trips.

---


## ✅ Checkpoint 23

> Đến đây bạn phải hiểu:
> 1. **Property** = "luôn đúng" cho mọi input: `reverse (reverse xs) == xs`
> 2. Thay vì 10 examples → 1 property kiểm tra 1000+ cases tự động
> 3. Categories: round-trip, idempotent, metamorphic, oracle
>
> **Test nhanh**: Round-trip property cho `encode/decode` là gì?
> <details><summary>Đáp án</summary>`decode (encode x) == Ok x` — encode rồi decode phải trả về giá trị ban đầu.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Tìm properties

Cho mỗi function, liệt kê ít nhất 2 properties:

```
a) List.map
b) List.len
c) Str.toUpper
d) Dict.insert + Dict.get
```

<details><summary>✅ Lời giải</summary>

```
a) List.map:
   1. Preserves length: List.len (List.map list f) == List.len list
   2. Identity: List.map list \x -> x == list
   3. Composition: List.map (List.map list f) g == List.map list \x -> g (f x)

b) List.len:
   1. Non-negative: List.len list >= 0
   2. Append increases: List.len (List.append list x) == List.len list + 1
   3. Concat adds: List.len (List.concat a b) == List.len a + List.len b

c) Str.toUpper:
   1. Idempotent: toUpper (toUpper s) == toUpper s
   2. Length preserved: Str.countUtf8Bytes (toUpper s) == Str.countUtf8Bytes s  (for ASCII)

d) Dict.insert + Dict.get:
   1. Round-trip: Dict.get (Dict.insert dict k v) k == Ok v
   2. Other keys unchanged: Dict.get (Dict.insert dict k v) otherKey == Dict.get dict otherKey
```

</details>

---

**Bài 2** (15 phút): Property tests cho Money

Viết property tests cho Money domain:

```roc
# Properties to test:
# 1. Round-trip: toStr → fromStr → original
# 2. Commutativity: add a b == add b a
# 3. Identity: add money (fromVND 0) == money
# 4. Invariant: amount after add >= each operand
```

<details><summary>✅ Lời giải</summary>

```roc
# Commutativity
expect
    pairs = [(100, 200), (0, 0), (500, 300)]
    List.all pairs \(a, b) ->
        add (fromVND a) (fromVND b) == add (fromVND b) (fromVND a)

# Identity
expect
    amounts = [0, 100, 999999]
    List.all amounts \n ->
        add (fromVND n) (fromVND 0) == Ok (fromVND n)

# Invariant: result >= each operand
expect
    pairs = [(100, 200), (0, 500), (300, 0)]
    List.all pairs \(a, b) ->
        when add (fromVND a) (fromVND b) is
            Ok result -> getAmount result >= a && getAmount result >= b
            Err _ -> Bool.false
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| "Không tìm được property" | Nghĩ quá phức tạp | Bắt đầu với round-trip và invariant — dễ nhất |
| Property quá yếu | Luôn đúng, không bắt bugs | Thêm ràng buộc cụ thể hơn |
| Property quá mạnh | Fail vì quá strict | Nới lỏng — dùng `>=` thay `==`, cho tolerance |
| Khó generate data | Roc chưa có PBT framework | Dùng danh sách test values thủ công |

---

## Tóm tắt

- ✅ **PBT** = test tính chất thay vì examples. 1 property > 50 examples.
- ✅ **Round-trip**: `decode(encode(x)) == x` — serialize/deserialize, parse/format.
- ✅ **Idempotency**: `f(f(x)) == f(x)` — normalize, sort, deduplicate.
- ✅ **Invariant**: tính chất luôn đúng — length preserved, total ≥ 0, sorted after sort.
- ✅ **Metamorphic**: thay đổi input → dự đoán output — double price → double total.
- ✅ **Symmetry**: A → B → A — add/subtract, insert/remove.
- ✅ **Easy to verify**: isSorted, all match predicate, valid state.
- ✅ **Dùng cả hai**: Examples cho specific cases, Properties cho general invariants.

## Tiếp theo

→ Chapter 24: **CLI Applications** — xây CLI apps hoàn chỉnh với `basic-cli`: argument parsing, file I/O, HTTP requests, interactive menus, và progress output.
