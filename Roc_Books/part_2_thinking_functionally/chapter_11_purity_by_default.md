# Chapter 11 — Purity by Default

> **Bạn sẽ học được**:
> - **Pure function** là gì — và tại sao đây là mặc định trong Roc
> - **Side effects** — đọc file, HTTP, print — và tại sao chúng nguy hiểm
> - Roc kiểm soát side effects bằng **Tasks** (giới thiệu, chi tiết ở Ch 12)
> - **Pure core / Effectful shell** — kiến trúc chia đôi: logic thuần túy + IO ở rìa
> - Tại sao purity giúp code dễ test, dễ debug, ít bugs hơn
>
> **Yêu cầu trước**: [Chapter 10 — Opaque Types & Modules](../part_1_fundamentals/chapter_10_opaque_types_and_modules.md)
> **Thời gian đọc**: ~35 phút | **Level**: Intermediate
> **Kết quả cuối cùng**: Tư duy "tách pure logic ra khỏi IO" — kỹ năng quan trọng nhất của FP.

---

Năm 2012, công ty Knight Capital mất **440 triệu USD trong 45 phút** vì một function thay đổi global state mà không ai ngờ. Một biến toàn cục bị cập nhật sai cách, không ai phát hiện vì function đó được gọi ở nhiều nơi khác nhau.

Pure functions ngăn chặn loại bug này từ gốc: không global state, không side effects ẩn. Cùng input luôn cho cùng output — hôm nay, ngày mai, trên máy nào cũng vậy. Roc biến điều này thành **mặc định** — mọi function đều pure trừ khi bạn đánh dấu rõ bằng dấu `!`.

## 11.1 — Pure Function là gì?

Hai quy tắc đơn giản:

Một function là **pure** (thuần) khi:

1. **Cùng input → luôn cùng output** (deterministic)
2. **Không có side effects** (không đọc/ghi file, không gọi HTTP, không in ra terminal, không thay đổi global state)

```roc
# ✅ PURE — cùng input → luôn cùng output
add = \a, b -> a + b

# add 3 5 → luôn = 8, hôm nay, ngày mai, trên máy nào cũng vậy

# ✅ PURE
greet = \name -> "Xin chào $(name)!"

# greet "An" → luôn = "Xin chào An!"

# ✅ PURE
isAdult = \age -> age >= 18

# isAdult 20 → luôn = Bool.true
```

### So sánh với impure (không thuần)

```python
# Python — ❌ impure function
import datetime
import random

def greet(name):
    hour = datetime.datetime.now().hour  # phụ thuộc THỜI GIAN
    if hour < 12:
        return f"Good morning {name}"
    else:
        return f"Good afternoon {name}"
# greet("An") → 7AM = "Good morning", 2PM = "Good afternoon"
# Cùng input, KHÁC output! → impure

def roll_dice():
    return random.randint(1, 6)  # phụ thuộc RANDOM STATE
# roll_dice() → lần 1 = 3, lần 2 = 5 → impure

total = 0
def add_to_total(x):
    global total
    total += x     # THAY ĐỔI global state → side effect!
    return total
# add_to_total(5) → lần 1 = 5, lần 2 = 10 → impure
```

> **💡 Ẩn dụ**: Pure function giống **máy tính bỏ túi**. Bấm `3 + 5 =` lúc nào cũng ra 8. Impure function giống **người bán hàng** — hỏi "bao nhiêu?" lúc nào cũng khác tùy hàng còn hay hết, tùy khuyến mãi, tùy tâm trạng.

---

## 11.2 — Tại sao Purity quan trọng?

Biết pure function là gì thì dễ. Câu hỏi thực sự là: **tại sao phải quan tâm?** Bốn lý do thực tế:

### 1. Dễ test

```roc
# Pure → test cực kỳ đơn giản
expect add 3 5 == 8
expect add 0 0 == 0
expect add (-1) 1 == 0

# Không cần setup database, mock HTTP, fake file system
# Chỉ input → output → xong!
```

### 2. Dễ debug

```roc
# Pure function: nếu output sai, BUG CHẮC CHẮN trong function này
processOrder = \order ->
    items = order.items
    subtotal = List.walk items 0 \s, i -> s + i.price
    tax = subtotal * 10 // 100
    { order & total: subtotal + tax }

# processOrder trả sai total?
# → Bug 100% trong 4 dòng trên. Không phải do database, network, hay global state
```

### 3. Dễ hiểu (referential transparency)

```roc
# Bạn có thể THAY THẾ function call bằng kết quả mà không đổi ý nghĩa

# Đọc code:
result = double (add 3 5)

# Bạn BIẾT CHẮC có thể thay thế bằng:
result = double 8
# Rồi:
result = 16

# Không cần lo "add có đổi global state không?" hay "double có crash bất ngờ không?"
```

### 4. An toàn cho concurrency

```roc
# Pure functions không chia sẻ state → chạy song song an toàn
# Không race conditions, không deadlocks, không data corruption
results = List.map hugeList \item -> expensiveComputation item
# Roc CÓ THỂ chạy song song — vì mỗi lời gọi độc lập
```

---

## 11.3 — Side Effects trong Roc: Chỉ qua Tasks

Câu hỏi tự nhiên: nếu mọi function đều pure, làm sao in ra terminal, đọc file, gọi API? Roc giải quyết bằng Tasks — mô tả side effect mà không thực thi ngay.

### Vấn đề

Nếu mọi function đều pure, thì làm sao in ra màn hình, đọc file, gọi API? Rõ ràng một chương trình không làm gì ngoài tính toán thì vô dụng.
- In ra terminal? (side effect!)
- Đọc file? (side effect!)
- Gọi HTTP? (side effect!)
- Lấy thời gian hiện tại? (side effect!)

### Cách Roc giải quyết: Tasks

Roc **không cấm** side effects — nhưng **đánh dấu** chúng. Side effects chỉ tồn tại trong `Task`:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Utc

# ✅ Pure function — không có dấu !
calculateDiscount : U64, U64 -> U64
calculateDiscount = \price, percent ->
    price * percent // 100

# ✅ Pure function — xử lý data
formatPrice : U64 -> Str
formatPrice = \amount ->
    "$(Num.toStr amount)đ"

# Dấu ! = side effect (Task)
main =
    originalPrice = 100000
    discount = calculateDiscount originalPrice 20   # pure — tính toán
    finalPrice = originalPrice - discount            # pure — tính toán
    message = formatPrice finalPrice                 # pure — format

    Stdout.line! "Giá gốc: $(formatPrice originalPrice)"        # side effect!
    Stdout.line! "Giảm giá: $(formatPrice discount)"            # side effect!
    Stdout.line! "Giá cuối: $(message)"                          # side effect!
    # Output:
    # Giá gốc: 100000đ
    # Giảm giá: 20000đ
    # Giá cuối: 80000đ
```

Chú ý:
- `calculateDiscount` và `formatPrice` — **pure**, không có `!`
- `Stdout.line!` — **impure** (IO), có dấu `!`
- Compiler **biết** function nào pure, function nào impure

> **💡 Quy tắc ở Roc**: Nếu không có dấu `!`, function là pure. Dấu `!` = "function này có side effect". Đơn giản vậy thôi.

---

## 11.4 — Pure Core / Effectful Shell

Đây là kiến trúc quan trọng nhất bạn sẽ học trong cuốn sách này. Ý tưởng: đặt tất cả business logic vào pure functions (core), đẩy IO ra rìa (shell). Core dễ test, dễ hiểu, dễ tái sử dụng. Shell chỉ kết nối core với thế giới bên ngoài.

### Kiến trúc quan trọng nhất của FP

```
┌──────────────────────────────────────────┐
│              Effectful Shell             │  ← IO ở đây
│  main, đọc file, HTTP, print            │
│  ┌──────────────────────────────────┐   │
│  │          Pure Core               │   │  ← Logic ở đây
│  │  tính toán, validate, transform  │   │
│  │  domain logic, business rules    │   │
│  │  KHÔNG có IO, KHÔNG có !         │   │
│  └──────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

**Pure Core** = tất cả business logic. Không biết terminal, file, hay HTTP là gì.
**Effectful Shell** = `main` và các Task. Đọc input, gọi pure core, ghi output.

### Ví dụ: Hệ thống đánh giá sản phẩm

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════════════════════════════════════════
# PURE CORE — không có ! → test dễ dàng
# ═══════════════════════════════════════════

validateRating : I64 -> Result U8 [RatingTooLow, RatingTooHigh]
validateRating = \raw ->
    if raw < 1 then Err RatingTooLow
    else if raw > 5 then Err RatingTooHigh
    else Ok (Num.toU8 raw)

averageRating : List U8 -> F64
averageRating = \ratings ->
    if List.isEmpty ratings then 0.0
    else
        total = List.walk ratings 0 \s, r -> s + Num.toU64 r
        Num.toF64 total / Num.toF64 (List.len ratings)

starBar : U8 -> Str
starBar = \rating ->
    filled = Str.repeat "★" (Num.toU64 rating)
    empty = Str.repeat "☆" (5 - Num.toU64 rating)
    "$(filled)$(empty)"

classifyProduct : F64 -> Str
classifyProduct = \avg ->
    if avg >= 4.5 then "🏆 Xuất sắc"
    else if avg >= 3.5 then "👍 Tốt"
    else if avg >= 2.5 then "😐 Trung bình"
    else "👎 Kém"

formatReview : { user : Str, rating : U8, comment : Str } -> Str
formatReview = \review ->
    "  $(starBar review.rating) $(review.user): $(review.comment)"

# ═══════════════════════════════════════════
# EFFECTFUL SHELL — có ! → IO ở đây
# ═══════════════════════════════════════════

main =
    # Data (trong thực tế đọc từ database/API)
    reviews = [
        { user: "An", rating: 5, comment: "Tuyệt vời!" },
        { user: "Bình", rating: 4, comment: "Khá tốt" },
        { user: "Cường", rating: 3, comment: "Bình thường" },
        { user: "Dũng", rating: 5, comment: "Rất thích" },
    ]

    # Pure computations — không IO nào ở đây
    ratings = List.map reviews \r -> r.rating
    avg = averageRating ratings
    classification = classifyProduct avg

    # IO — chỉ ở đây
    Stdout.line! "=== Đánh giá sản phẩm ==="
    List.forEach reviews \r ->
        Stdout.line! (formatReview r)
    Stdout.line! "---"
    Stdout.line! "Trung bình: $(Num.toStr avg) $(classification)"
    # Output:
    # === Đánh giá sản phẩm ===
    #   ★★★★★ An: Tuyệt vời!
    #   ★★★★☆ Bình: Khá tốt
    #   ★★★☆☆ Cường: Bình thường
    #   ★★★★★ Dũng: Rất thích
    # ---
    # Trung bình: 4.25 👍 Tốt
```

**Quan sát**: `validateRating`, `averageRating`, `starBar`, `classifyProduct`, `formatReview` — tất cả **pure**. Bạn test chúng chỉ bằng `expect`:

```roc
# Test pure functions — KHÔNG CẦN mock IO
expect validateRating 3 == Ok 3
expect validateRating 0 == Err RatingTooLow
expect validateRating 6 == Err RatingTooHigh
expect averageRating [4, 5, 3] == 4.0
expect starBar 3 == "★★★☆☆"
expect classifyProduct 4.5 == "🏆 Xuất sắc"
```

---

## 11.5 — So sánh: Roc vs Các ngôn ngữ khác

Mỗi ngôn ngữ xử lý side effects khác nhau. Python thoải mái — bất kỳ function nào cũng có thể làm bất kỳ thứ gì. Haskell cực đoan — IO monad bắt buộc đánh dấu mọi effect. Roc ở giữa — thiết thực hơn Haskell, an toàn hơn Python.

| | Python / JS / Go | Rust | Haskell | **Roc** |
|---|---|---|---|---|
| Mặc định | Impure | Impure | Pure | **Pure** |
| Side effects | Bất kỳ đâu | Bất kỳ đâu | Chỉ trong `IO` monad | **Chỉ qua `Task`** |
| Compiler biết pure/impure? | ❌ | ❌ | ✅ | ✅ |
| Cấm mutation? | ❌ | ❌ (`mut`) | ✅ | ✅ |
| Cấm exceptions? | ❌ | ❌ (`panic!`) | ❌ | ✅ (`Result` only) |

Roc giống Haskell ở purity nhưng **đơn giản hơn nhiều** — không cần hiểu Monads. Chỉ cần: dấu `!` = IO, không `!` = pure.

---

## 11.6 — Không có Mutation, Không có Exceptions

Roc loại bỏ hai nguồn bug lớn nhất: mutation (thay đổi biến đã có) và exceptions (crash bất ngờ). Không mutation → không race conditions. Không exceptions → mọi lỗi explicit trong type.

### Không mutation

```roc
# ❌ Không có "thay đổi biến" trong Roc
# Không có: x = 5; x = x + 1

# ✅ Tạo giá trị mới
x = 5
y = x + 1    # y = 6, x vẫn = 5

# ✅ Record update → tạo bản mới
user = { name: "An", age: 25 }
olderUser = { user & age: 26 }
# user vẫn = { name: "An", age: 25 }
```

### Không exceptions

```roc
# Python: divide(10, 0) → 💥 ZeroDivisionError (crash!)
# JavaScript: JSON.parse("invalid") → 💥 SyntaxError (crash!)

# Roc: KHÔNG BAO GIỜ crash bất ngờ
safeDivide = \a, b ->
    if b == 0 then Err DivisionByZero
    else Ok (a / b)

# Caller BẮT BUỘC xử lý cả 2 trường hợp
when safeDivide 10 0 is
    Ok result -> ...
    Err DivisionByZero -> ...

# Không có "quên try/catch rồi crash production"
```

---


## ✅ Checkpoint 11

> Đến đây bạn phải hiểu:
> 1. Mọi function Roc đều **pure** — cùng input luôn cho cùng output
> 2. Side effects chỉ qua **Tasks** — app code pure, platform xử lý IO
> 3. Pure = dễ test, dễ reason about, dễ compose
>
> **Test nhanh**: Function pure có thể đọc file không?
> <details><summary>Đáp án</summary>Không! Đọc file là side effect — phải dùng Task thông qua platform.</details>

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Pure hay Impure?

Đánh dấu mỗi function là Pure (P) hay Impure (I):

```
a) add = \a, b -> a + b
b) Stdout.line! "hello"
c) isEven = \x -> x %% 2 == 0
d) Utc.now!
e) List.map [1,2,3] \x -> x * 2
f) File.readUtf8! "data.txt"
g) formatPrice = \n -> "$(Num.toStr n)đ"
```

<details><summary>✅ Lời giải</summary>

```
a) P — cùng input → cùng output
b) I — in ra terminal (IO)
c) P — chỉ tính toán
d) I — phụ thuộc thời gian hiện tại
e) P — biến đổi list, không IO
f) I — đọc file (IO)
g) P — chỉ format chuỗi
```

</details>

---

**Bài 2** (15 phút): Refactor thành Pure Core

Tách business logic ra khỏi IO:

```roc
# ❌ Hiện tại: logic lẫn IO
main =
    items = [
        { name: "Phở", price: 45000 },
        { name: "Cà phê", price: 25000 },
    ]

    # Logic + IO trộn lẫn
    total = List.walk items 0 \s, i -> s + i.price
    Stdout.line! "Tổng: $(Num.toStr total)đ"
    tax = total * 10 // 100
    Stdout.line! "Thuế: $(Num.toStr tax)đ"
    final = total + tax
    Stdout.line! "Cuối: $(Num.toStr final)đ"

# ✅ Hãy tách thành: pure functions + main chỉ IO
```

<details><summary>✅ Lời giải</summary>

```roc
# Pure core
calculateTotal = \items ->
    List.walk items 0 \s, i -> s + i.price

calculateTax = \subtotal, percent ->
    subtotal * percent // 100

generateReceipt = \items, taxPercent ->
    subtotal = calculateTotal items
    tax = calculateTax subtotal taxPercent
    final = subtotal + tax
    { subtotal, tax, final }

# Effectful shell
main =
    items = [
        { name: "Phở", price: 45000 },
        { name: "Cà phê", price: 25000 },
    ]

    receipt = generateReceipt items 10

    Stdout.line! "Tổng: $(Num.toStr receipt.subtotal)đ"
    Stdout.line! "Thuế: $(Num.toStr receipt.tax)đ"
    Stdout.line! "Cuối: $(Num.toStr receipt.final)đ"
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| "Tôi cần random numbers" | `Random` là side effect | Dùng `Task` hoặc truyền seed vào pure function |
| "Tôi cần current time" | `Utc.now` là side effect | Gọi `Utc.now!` ở shell, truyền giá trị vào pure core |
| "Code quá nhiều `!`" | Logic lẫn IO | Refactor: tách tính toán ra pure functions |
| Pure function cần IO data | Thiết kế sai | Đọc IO ở shell → truyền data vào pure function |

---

## Tóm tắt

- ✅ **Pure function** = cùng input → cùng output, không side effects. Đây là **mặc định** trong Roc.
- ✅ **Side effects** (IO) chỉ tồn tại trong **Tasks** — đánh dấu bằng `!`.
- ✅ **Pure core / Effectful shell** — logic ở giữa (pure), IO ở rìa (shell). Kiến trúc FP cốt lõi.
- ✅ **Không mutation** — tạo giá trị mới thay vì sửa. `{ record & field: newValue }`.
- ✅ **Không exceptions** — dùng `Result` thay throw/catch. Compiler bắt buộc xử lý lỗi.
- ✅ **Lợi ích**: dễ test (chỉ input→output), dễ debug (lỗi cô lập), dễ hiểu (referential transparency), an toàn concurrency.

## Tiếp theo

→ Chapter 12: **Tasks & Effects** ⭐ — deep dive vào `Task ok err`, `Task.await`, `Task.map`, cách đọc file, gọi HTTP, và orchestrate side effects.
