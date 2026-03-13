# Chapter 22 — Testing

> **Bạn sẽ học được**:
> - `expect` keyword — cách duy nhất viết test trong Roc
> - **TDD cycle** — Red → Green → Refactor với Roc
> - Test functions thuần, domain logic, và workflows
> - Tổ chức tests — inline vs module tests
> - **Edge cases** và boundary conditions
> - Tại sao pure functions làm testing cực kỳ đơn giản
>
> **Yêu cầu trước**: [Chapter 21 — Serialization](../part_4_ddd/chapter_21_serialization.md)
> **Thời gian đọc**: ~40 phút | **Level**: Principal
> **Kết quả cuối cùng**: Viết tests tự tin — cover đủ cases, chạy nhanh, dễ đọc.

---

Pure functions là giấc mơ của testing: không mock, không setup, không teardown. Truyền input, kiểm tra output — xong. Roc tích hợp testing vào ngôn ngữ với keyword `expect` — không cần framework, không cần import thêm gì.

## Testing trong Roc — Pure functions = Easy tests

Roc's purity-by-default là gift cho testing: pure functions không cần mock, không cần setup, không cần teardown. Input → output, assert. Chapter này dạy `roc test`, expect syntax, và cách test code có Tasks.


Roc có lợi thế testing đặc biệt: **purity by default**. Pure function = test không cần setup, teardown, mock. Input → output, `expect`. `roc test` built-in, chạy tất cả `expect` expressions, không cần framework.

Code có Tasks (IO) tách riêng: pure logic test trực tiếp (fast), Task-heavy code test qua integration (slower). Pattern FC/IS từ Ch11 maximize phần pure.

## 22.1 — Testing trong Roc: Đơn giản tột bậc

Testing trong Roc bắt đầu bằng một keyword: `expect`. Không framework, không import, không config. Viết assertion ngay trong file, chạy `roc test` — xong.

### Không framework, không assertion library

Trong hầu hết ngôn ngữ:

```python
# Python — cần test framework
import pytest

class TestCalculator:
    def test_add(self):
        assert add(2, 3) == 5

    def test_divide_by_zero(self):
        with pytest.raises(ZeroDivisionError):
            divide(10, 0)
```

Trong Roc — chỉ cần `expect`:

```roc
# Không import gì cả. Không class. Không decorator.
# Chỉ: expect <expression that should be Bool.true>

add = \a, b -> a + b

expect add 2 3 == 5
expect add 0 0 == 0
expect add (-1) 1 == 0
```

Chạy: `roc test main.roc` → Done.

---

## 22.2 — Cú pháp `expect`

Unit test kiểm tra một function cụ thể. Pure function → unit test cực đơn giản: truyền input, kiểm tra output. Không setup, không teardown, không side effects.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ FUNCTIONS ═══════

absolute = \n -> if n < 0 then -n else n
clamp = \n, low, high -> Num.max low (Num.min high n)
isEven = \n -> n % 2 == 0

# ═══════ TESTS ═══════

# Basic: expect <boolean expression>
expect absolute 5 == 5
expect absolute (-3) == 3
expect absolute 0 == 0

# Clamp
expect clamp 5 0 10 == 5     # trong range
expect clamp (-2) 0 10 == 0   # dưới min → min
expect clamp 15 0 10 == 10    # trên max → max
expect clamp 0 0 10 == 0      # edge: exactly min
expect clamp 10 0 10 == 10    # edge: exactly max

# Boolean
expect isEven 4 == Bool.true
expect isEven 7 == Bool.false
expect isEven 0 == Bool.true

# Multi-line expect — dùng let bindings
expect
    result = clamp 50 0 100
    result == 50

expect
    a = absolute (-42)
    b = absolute 42
    a == b    # absoluteValue is symmetric

main =
    Stdout.line! "Run: roc test main.roc"
```

### Khi test fail

```
── EXPECT FAILED ─────────────────────
This expectation was not true:

    18│  expect clamp 15 0 10 == 15

When it was simplified, I got:

    10 == 15

Which is Bool.false
```

Roc cho bạn **giá trị thực tế** — dễ debug.

---

## 22.3 — Test Result types

Test doubles thay thế dependencies trong test: fake database, stub API response. Trong FP, bạn truyền function thay vì inject object — đơn giản hơn mock frameworks.

```roc
# ═══════ FUNCTION ═══════

safeDivide = \a, b ->
    if b == 0 then Err DivisionByZero
    else Ok (a // b)

parseAge = \input ->
    when Str.toI64 input is
        Err _ -> Err NotANumber
        Ok n ->
            if n < 0 then Err Negative
            else if n > 150 then Err TooOld
            else Ok (Num.toU8 n)

# ═══════ TESTS ═══════

# Test Ok case
expect safeDivide 10 3 == Ok 3
expect safeDivide 0 5 == Ok 0

# Test Err case
expect safeDivide 10 0 == Err DivisionByZero

# Dùng Result.isOk / Result.isErr
expect Result.isOk (safeDivide 10 2)
expect Result.isErr (safeDivide 10 0)

# Test parseAge
expect parseAge "25" == Ok 25
expect parseAge "abc" == Err NotANumber
expect parseAge "-5" == Err Negative
expect parseAge "200" == Err TooOld

# Boundary values
expect parseAge "0" == Ok 0
expect parseAge "150" == Ok 150
expect parseAge "151" == Err TooOld
```

---

## 22.4 — Test Domain Logic

Integration test kiểm tra nhiều modules phối hợp. Gọi pipeline từ đầu đến cuối, kiểm tra output cuối cùng. Phát hiện lỗi ở boundary giữa các modules.

```roc
# ═══════ DOMAIN: Shopping Cart ═══════

emptyCart = { items: [], discount: 0 }

addItem = \cart, name, price, qty ->
    { cart & items: List.append cart.items { name, price, qty } }

removeItem = \cart, name ->
    { cart & items: List.keepIf cart.items \i -> i.name != name }

subtotal = \cart ->
    List.walk cart.items 0 \s, item -> s + item.price * item.qty

applyDiscount = \cart, percent ->
    { cart & discount: percent }

totalWithDiscount = \cart ->
    sub = subtotal cart
    sub - (sub * cart.discount // 100)

itemCount = \cart ->
    List.walk cart.items 0 \s, item -> s + item.qty

# ═══════ TESTS ═══════

# Empty cart
expect subtotal emptyCart == 0
expect itemCount emptyCart == 0
expect totalWithDiscount emptyCart == 0

# Add items
expect
    cart = emptyCart |> addItem "Phở" 45000 2
    subtotal cart == 90000

expect
    cart =
        emptyCart
        |> addItem "Phở" 45000 2
        |> addItem "Cà phê" 25000 1
    subtotal cart == 115000

# Remove items
expect
    cart =
        emptyCart
        |> addItem "Phở" 45000 2
        |> addItem "Cà phê" 25000 1
        |> removeItem "Phở"
    subtotal cart == 25000

# Remove non-existent item — no crash
expect
    cart = emptyCart |> addItem "Phở" 45000 1 |> removeItem "Bánh mì"
    subtotal cart == 45000

# Discount
expect
    cart =
        emptyCart
        |> addItem "Phở" 100000 1
        |> applyDiscount 10
    totalWithDiscount cart == 90000

# 0% discount = no change
expect
    cart = emptyCart |> addItem "Phở" 100000 1 |> applyDiscount 0
    totalWithDiscount cart == 100000

# 100% discount = free
expect
    cart = emptyCart |> addItem "Phở" 100000 1 |> applyDiscount 100
    totalWithDiscount cart == 0

# Item count
expect
    cart =
        emptyCart
        |> addItem "Phở" 45000 2
        |> addItem "Cà phê" 25000 3
    itemCount cart == 5
```

---

## 22.5 — TDD Cycle: Red → Green → Refactor

Test organization: nhóm tests theo module, đặt tên rõ ràng, viết test cho edge cases. Quy tắc: mọi bug tìm được → viết test trước khi fix.

### Bước 1: RED — Viết test trước, chưa có code

```roc
# Viết test TRƯỚC
expect classifyBMI 18.5 == "Normal"
expect classifyBMI 15.0 == "Underweight"
expect classifyBMI 27.0 == "Overweight"
expect classifyBMI 32.0 == "Obese"
```

### Bước 2: GREEN — Viết code tối thiểu để pass

```roc
classifyBMI = \bmi ->
    if bmi < 18.5 then "Underweight"
    else if bmi < 25.0 then "Normal"
    else if bmi < 30.0 then "Overweight"
    else "Obese"
```

### Bước 3: REFACTOR — Cải thiện mà không sửa behavior

```roc
# Thêm edge case tests
expect classifyBMI 18.5 == "Normal"      # boundary: exactly 18.5
expect classifyBMI 24.9 == "Normal"      # boundary: just under 25
expect classifyBMI 25.0 == "Overweight"  # boundary: exactly 25
expect classifyBMI 29.9 == "Overweight"  # boundary: just under 30
expect classifyBMI 30.0 == "Obese"       # boundary: exactly 30
```

Tests vẫn pass → refactoring an toàn.

---

## 22.6 — Test Validation Pipelines

Coverage không phải mục tiêu — understanding mới là mục tiêu. 100% coverage không có nghĩa 100% correct. Tập trung test business logic quan trọng nhất.

```roc
# ═══════ DOMAIN ═══════

validateUsername = \raw ->
    trimmed = Str.trim raw
    len = Str.countUtf8Bytes trimmed
    if Str.isEmpty trimmed then Err EmptyUsername
    else if len < 3 then Err (TooShort len)
    else if len > 20 then Err (TooLong len)
    else if Str.contains trimmed " " then Err HasSpaces
    else Ok trimmed

registerUser = \rawName, rawEmail ->
    name = validateUsername? rawName
    email = if Str.contains rawEmail "@" then Ok rawEmail else Err InvalidEmail
    e = email?
    Ok { name, email: e, createdAt: "2024-01-01" }

# ═══════ TESTS: Individual validators ═══════

expect validateUsername "An123" == Ok "An123"
expect validateUsername "  An123  " == Ok "An123"      # trims whitespace
expect validateUsername "" == Err EmptyUsername
expect validateUsername "  " == Err EmptyUsername        # only spaces
expect validateUsername "ab" == Err (TooShort 2)
expect Str.repeat "a" 21 |> validateUsername == Err (TooLong 21)
expect validateUsername "has space" == Err HasSpaces

# ═══════ TESTS: Full pipeline ═══════

# Happy path
expect
    result = registerUser "An" "an@mail.com"
    when result is
        Ok user -> user.name == "An" && user.email == "an@mail.com"
        Err _ -> Bool.false

# First validation fails → stops
expect registerUser "" "an@mail.com" == Err EmptyUsername

# Second validation fails
expect registerUser "An" "bad-email" == Err InvalidEmail

# First validation fails first (even if second is also bad)
expect registerUser "" "bad" == Err EmptyUsername
```

---

## 22.7 — Test State Machines

Golden tests so sánh output hiện tại với snapshot đã lưu. Input → function → kết quả → so sánh với file .golden. Phát hiện regression tự động.

```roc
# ═══════ DOMAIN ═══════

confirmOrder = \order ->
    when order.status is
        Draft -> Ok { order & status: Confirmed }
        _ -> Err (InvalidTransition "Must be Draft")

shipOrder = \order ->
    when order.status is
        Confirmed -> Ok { order & status: Shipped }
        _ -> Err (InvalidTransition "Must be Confirmed")

cancelOrder = \order ->
    when order.status is
        Draft -> Ok { order & status: Cancelled }
        Confirmed -> Ok { order & status: Cancelled }
        _ -> Err (InvalidTransition "Cannot cancel")

# ═══════ TESTS ═══════

# Valid transitions
expect
    order = { id: 1, status: Draft }
    confirmOrder order == Ok { id: 1, status: Confirmed }

expect
    order = { id: 1, status: Confirmed }
    shipOrder order == Ok { id: 1, status: Shipped }

# Invalid transitions
expect
    order = { id: 1, status: Shipped }
    Result.isErr (confirmOrder order)

expect
    order = { id: 1, status: Draft }
    Result.isErr (shipOrder order)   # can't skip Confirmed

# Cancel from multiple states
expect
    Result.isOk (cancelOrder { id: 1, status: Draft })

expect
    Result.isOk (cancelOrder { id: 1, status: Confirmed })

expect
    Result.isErr (cancelOrder { id: 1, status: Shipped })

# Full workflow
expect
    order = { id: 1, status: Draft }
    result =
        confirmed = confirmOrder? order
        shipped = shipOrder? confirmed
        Ok shipped
    result == Ok { id: 1, status: Shipped }
```

---

## 22.8 — Tổ chức Tests

Test strategy cho dự án thực: unit tests cho domain logic (nhiều nhất), integration tests cho workflows (vừa phải), end-to-end tests cho critical paths (ít nhất). Hình kim tự tháp.

### Inline tests — cùng file với code

```roc
# filename: Money.roc
module [Money, fromVND, add, toStr]

Money := { amount : U64, currency : [VND, USD] } implements [Eq, Inspect]

fromVND = \amount -> @Money { amount, currency: VND }

add = \@Money a, @Money b ->
    if a.currency != b.currency then Err CurrencyMismatch
    else Ok (@Money { amount: a.amount + b.amount, currency: a.currency })

toStr = \@Money m ->
    "$(Num.toStr m.amount)đ"

# Tests ngay trong module — gần code nhất
expect add (fromVND 100) (fromVND 200) == Ok (fromVND 300)
expect add (fromVND 0) (fromVND 0) == Ok (fromVND 0)
expect toStr (fromVND 45000) == "45000đ"
```

### Khi nào inline, khi nào tách file?

| Inline (cùng file) | Tách file |
|---|---|
| Unit tests cho 1 function | Integration tests nhiều modules |
| Ít tests (< 20) | Nhiều tests (> 20) |
| Tests đọc cùng code | Tests cần setup phức tạp |

---

## 22.9 — Edge Cases Checklist

Khi viết tests, luôn kiểm tra:

| Category | Test cases |
|----------|-----------|
| **Empty** | `""`, `[]`, `Dict.empty {}`, `0` |
| **One** | 1 phần tử, 1 ký tự, giá trị 1 |
| **Boundary** | min, max, min-1, max+1, exactly at boundary |
| **Negative** | Số âm, input invalid, field thiếu |
| **Large** | List rất dài, string rất dài, số rất lớn |
| **Duplicate** | Giá trị trùng, add cùng item 2 lần |
| **Order** | Sorted vs unsorted input, reversed |
| **Unicode** | Emoji, Vietnamese characters, special chars |

```roc
# Ví dụ edge cases cho List.reverse
expect List.reverse [] == []                           # empty
expect List.reverse [1] == [1]                         # one element
expect List.reverse [1, 2, 3] == [3, 2, 1]           # normal
expect List.reverse [1, 1, 1] == [1, 1, 1]           # all same
expect List.reverse (List.reverse [1, 2, 3]) == [1, 2, 3]  # round-trip!
```

---


## ✅ Checkpoint 22

> Đến đây bạn phải hiểu:
> 1. `expect` = built-in test keyword, viết ngay cạnh function
> 2. `roc test` chạy tất cả `expect` statements
> 3. Pure functions = dễ test nhất — cùng input luôn cùng output
>
> **Test nhanh**: `expect add 2 3 == 5` — file này là test file riêng?
> <details><summary>Đáp án</summary>Không! `expect` viết ngay trong cùng file — inline testing.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): TDD — Password validator

Viết tests TRƯỚC, sau đó code:

```roc
# Requirements:
# - Tối thiểu 8 ký tự
# - Ít nhất 1 chữ hoa
# - Ít nhất 1 chữ số
# - Không chứa khoảng trắng
```

<details><summary>✅ Lời giải</summary>

```roc
# Tests first!
expect validatePassword "Secure1pass" == Ok "Secure1pass"
expect validatePassword "short" == Err TooShort
expect validatePassword "nouppercase1" == Err NoUppercase
expect validatePassword "NODIGITHERE" == Err NoDigit
expect validatePassword "Has Space1" == Err HasSpaces
expect validatePassword "12345678" == Err NoUppercase
expect validatePassword "ABCDEFGH" == Err NoDigit
expect validatePassword "Ab1" == Err TooShort

# Code
validatePassword = \raw ->
    if Str.countUtf8Bytes raw < 8 then Err TooShort
    else if Str.contains raw " " then Err HasSpaces
    else
        bytes = Str.toUtf8 raw
        hasUpper = List.any bytes \b -> b >= 65 && b <= 90
        hasDigit = List.any bytes \b -> b >= 48 && b <= 57
        if !hasUpper then Err NoUppercase
        else if !hasDigit then Err NoDigit
        else Ok raw
```

</details>

---

**Bài 2** (15 phút): Test event-sourced model

Viết tests cho bank account event sourcing:

```roc
# Test: applyEvent, rebuildState, validateCommand
# Cover: deposit, withdraw, insufficient funds, account inactive
```

<details><summary>✅ Lời giải</summary>

```roc
expect
    state = rebuildState [AccountOpened { owner: "An", balance: 0 }]
    state.owner == "An" && state.balance == 0

expect
    state = rebuildState [
        AccountOpened { owner: "An", balance: 0 },
        Deposited { amount: 1000 },
        Withdrawn { amount: 300 },
    ]
    state.balance == 700

expect
    state = { owner: "An", balance: 500, isActive: Bool.true }
    validateCommand state (Withdraw { amount: 300 }) == Ok (Withdrawn { amount: 300 })

expect
    state = { owner: "An", balance: 500, isActive: Bool.true }
    validateCommand state (Withdraw { amount: 900 }) == Err InsufficientFunds

expect
    state = { owner: "An", balance: 500, isActive: Bool.false }
    validateCommand state (Deposit { amount: 100 }) == Err AccountInactive
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| `expect` không chạy | Dùng `roc run` thay vì `roc test` | Chạy `roc test filename.roc` |
| Test pass nhưng logic sai | Test quá lỏng (chỉ check `Result.isOk`) | Test giá trị cụ thể: `== Ok expectedValue` |
| Quá nhiều tests, khó maintain | Tests lặp lại pattern | Tạo helper functions cho common test setup |
| Khó test IO functions | IO functions cần platform | Tách logic thành pure functions → test pure |

---

## Tóm tắt

- ✅ **`expect`** = cách duy nhất test. Không framework, không assertions — chỉ boolean expressions.
- ✅ **`roc test`** = chạy tất cả `expect` trong file. Fail → hiển thị giá trị thực tế.
- ✅ **TDD** = viết test trước → code tối thiểu → refactor. Natural fit cho pure functions.
- ✅ **Test Result** = `== Ok value`, `== Err ErrorTag`, hoặc `Result.isOk`/`Result.isErr`.
- ✅ **Test domain logic** = không mock, không setup — gọi function, check kết quả.
- ✅ **Edge cases** = empty, one, boundary, negative, large, duplicate, unicode.
- ✅ **Pure functions = trivially testable**. Đây là lợi thế lớn nhất của Roc.

## Tiếp theo

→ Chapter 23: **PBT Concepts** — Property-Based Testing: round-trip, idempotency, metamorphic properties. Thay vì viết 10 examples, viết 1 property che phủ 1000 cases.
