# Appendix E — Debugging & Tooling

> Hướng dẫn debug, tooling, editor setup, và đọc compiler errors cho Roc developers.

---

## E.1 — Đọc Compiler Errors

Roc compiler là **bạn đồng hành** — lỗi luôn rõ ràng, chỉ đúng dòng, gợi ý sửa.

### Missing field

```
── TYPE MISMATCH ──────────────────────── main.roc ─

This record is missing the `email` field:

6│  sendEmail user
                ^^^^

I was expecting `user` to have type:
    { email : Str, name : Str }a

But it actually has type:
    { name : Str }
```

**Đọc**: Function `sendEmail` cần record có `email`, nhưng `user` không có → thêm field `email`.

### Non-exhaustive pattern match

```
── UNSAFE PATTERN ──────────────────────── main.roc ─

This `when` does not cover all possibilities:

10│      when status is
11│          Placed -> "Đặt"
12│          Confirmed -> "Xác nhận"

Missing: Preparing, Ready, Completed, Cancelled

Tip: Add a branch for each missing pattern, or use `_` as a catch-all.
```

**Đọc**: `when` thiếu branches → thêm các tags còn thiếu hoặc dùng `_ ->`.

### Type mismatch in branches

```
── TYPE MISMATCH ──────────────────────── main.roc ─

The branches of this `when` produce different types:

15│      when x is
16│          Ok val -> val        # Str
17│          Err _ -> 0           # Num *

FIX: All branches must produce the same type.
```

**Đọc**: Mọi branch trong `when` phải trả cùng type → sửa cho nhất quán.

### Unused variable

```
── UNUSED VARIABLE ──────────────────── main.roc ─

Variable `temp` is defined but never used:

5│  temp = calculatePrice item

Tip: Prefix with _ if intentional: _temp
```

**Đọc**: Biến không dùng → xóa hoặc thêm `_` prefix.

---

## E.2 — Debugging Tools

### `Inspect.toStr` — Debug print

```roc
# In debug info cho bất kỳ value nào
import cli.Stdout

main =
    order = { id: 1, items: ["Phở", "Cà phê"], status: Placed }

    # Debug print — xem structure
    Stdout.line! "Debug: $(Inspect.toStr order)"
    # Output: Debug: { id: 1, items: ["Phở", "Cà phê"], status: Placed }
```

> **💡 Tip**: `Inspect.toStr` hoạt động cho MỌI type implement `Inspect` — records, tags, lists, dicts, numbers.

### `Dbg.log` — Debug không làm gián đoạn flow

```roc
# Dbg.log in debug info rồi trả lại giá trị — tiện cho pipelines
processOrder = \order ->
    order
    |> Dbg.log "Before validation"   # in debug, trả order
    |> validateOrder
    |> Dbg.log "After validation"    # in debug, trả validated order
    |> calculateTotal
```

### `roc check` — Type-check nhanh

```bash
# Chỉ kiểm tra types — KHÔNG compile — rất nhanh
roc check main.roc

# Kết quả:
# 0 errors and 0 warnings found in 42ms.
# HOẶC:
# 1 error found. (chi tiết theo format ở E.1)
```

### `roc test` — Chạy expect tests

```bash
# Chạy tất cả expect tests trong file
roc test main.roc

# Output thành công:
# 0 failed and 12 passed in 156ms.

# Output có lỗi:
# 1 failed and 11 passed in 160ms.
#
# ── EXPECT FAILED ──────────────────── main.roc ─
# This expectation failed:
# 25│  expect add 2 3 == 6
#
# When it evaluated, the result was:
#   5 == 6
#   which is: Bool.false
```

---

## E.3 — Editor Setup

### VS Code

1. Cài extension **"Roc"** từ Marketplace
2. Features:
   - Syntax highlighting
   - Type-on-hover
   - Go to definition
   - Error diagnostics real-time

### Vim/Neovim

```vim
" Dùng roc.vim plugin
" Hoặc tree-sitter grammar cho Neovim
Plug 'faldor20/tree-sitter-roc'
```

### Helix

Roc có tree-sitter grammar built-in — chỉ cần cài Roc LSP:

```bash
# Roc language server (built-in từ roc binary)
roc server
```

---

## E.4 — Roc Format

```bash
# Auto-format code — chuẩn hóa style
roc format main.roc

# Format cả thư mục
roc format .

# Kiểm tra format mà không sửa (CI)
roc format --check main.roc
```

Roc format rules:
- **4 spaces** indent (không tab)
- Trailing comma trong lists/records
- Consistent spacing quanh operators

---

## E.5 — Performance

### Build sizes

```bash
# Debug build (nhanh, lớn)
roc build main.roc
ls -lh main        # ~5-10 MB

# Optimized build (chậm hơn, nhỏ hơn, nhanh hơn)
roc build --optimize main.roc
ls -lh main        # ~1-3 MB
```

### Profiling tips

```roc
# ═══════ Đo thời gian thực thi ═══════
import cli.Utc

main =
    startTime = Utc.now!

    # ... work ...
    result = heavyComputation data

    endTime = Utc.now!
    elapsed = Utc.deltaAsMillis startTime endTime

    Stdout.line! "Hoàn thành trong $(Num.toStr elapsed)ms"
```

### Common performance patterns

| Pattern | Nhanh ✅ | Chậm ❌ |
|---------|---------|---------|
| List prepend | `List.prepend list x` O(1) | `List.append list x` O(n) |
| String building | `Str.joinWith parts ""` | Nối lặp: `a ++ b ++ c` |
| Dict lookup | `Dict.get dict key` O(1) | `List.findFirst list pred` O(n) |
| Unique ownership | Roc auto in-place update | Copy khi shared |

---

## E.6 — Debug Strategies

### Strategy 1: Shrink the problem

```roc
# Thay vì debug 100 items, test với 1-2 items trước
expect
    # Small test case
    result = processItems [item1]
    result == expectedOutput
```

### Strategy 2: Type-driven debugging

```roc
# Thêm type annotation → compiler cho biết chính xác đâu sai
processOrder : { items : List Item, status : OrderStatus } -> Result Order [EmptyOrder, InvalidItem Str]
processOrder = \order -> ...
# Nếu type annotation không match code → compiler chỉ chính xác dòng sai
```

### Strategy 3: Pipeline breakpoints

```roc
# Tách pipeline dài thành biến trung gian
result =
    rawInput
    |> parse            # Kiểm tra: parse đúng chưa?
    |> validate         # Kiểm tra: validate đúng chưa?
    |> transform        # Kiểm tra: transform đúng chưa?
    |> save

# Debug: tách ra
parsed = parse rawInput
_  = Dbg.log "parsed" parsed
validated = validate parsed
_  = Dbg.log "validated" validated
# ...
```

### Strategy 4: `expect` as debug tool

```roc
# Dùng expect để verify intermediate state
expect
    input = { items: [{ name: "Phở", price: 45000 }], discount: 10 }
    step1 = calculateSubtotal input
    step1 == 45000    # Verify step 1 correct

expect
    input = { items: [{ name: "Phở", price: 45000 }], discount: 10 }
    step2 = applyDiscount (calculateSubtotal input) input.discount
    step2 == 40500    # Verify step 2 correct
```

---

## E.7 — Troubleshooting Quick Reference

| Vấn đề | Có thể do | Debug cách |
|--------|----------|-----------|
| Compiler error khó hiểu | Type mismatch sâu | Thêm type annotations |
| Test fail nhưng code "đúng" | Logic edge case | Thêm `Dbg.log` ở giữa pipeline |
| App chạy chậm | List.append trong loop | Đổi sang List.prepend + reverse |
| File IO crash | File không tồn tại | Dùng `Task.attempt!` |
| String interpolation lỗi | Giá trị không phải Str | Dùng `Num.toStr`, `Inspect.toStr` |
| Platform download chậm | Lần đầu chạy / mạng yếu | Kiểm tra URL, internet. Cache sau lần đầu |
