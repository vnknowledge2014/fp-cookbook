# Chapter 2 — Algorithmic Thinking & Complexity

> **Bạn sẽ học được**:
> - Big-O là gì — tại sao code này chạy nhanh hơn code kia
> - Recursion — function gọi chính nó (cách FP thay thế vòng lặp `for`)
> - Tail-call optimization — viết recursion mà không bị tràn bộ nhớ
> - `List.walk` — "vũ khí tối thượng" thay thế mọi vòng lặp
> - Binary search, divide & conquer — tư duy chia để trị
>
> **Yêu cầu trước**: [Chapter 0 — Roc in 10 Minutes](chapter_00_roc_in_10_minutes.md)
> **Thời gian đọc**: ~45 phút | **Level**: CS Foundations
> **Kết quả cuối cùng**: Nhìn vào bất kỳ đoạn code nào và ước lượng được "code này chạy nhanh hay chậm" bằng Big-O.

---

## 2.1 — Big-O: Đo tốc độ code

Big-O cho biết code chậm đi bao nhiêu khi data lớn lên. O(n) = tuyến tính, O(n²) = bình phương, O(log n) = cực nhanh.

### Câu chuyện mở đầu

Bạn có 1.000 học sinh. Bạn cần tìm bạn "An" trong danh sách.

**Cách 1**: Đọc lần lượt từ đầu đến cuối → tệ nhất phải đọc 1.000 tên.
**Cách 2**: Danh sách đã xếp theo ABC. Mở giữa → "M" → An ở nửa đầu → mở giữa nửa đầu → lặp lại → chỉ cần ~10 lần.

Cách 1 mất 1.000 bước. Cách 2 mất 10 bước. Nhưng làm sao nói điều này một cách "khoa học"?

Đó là **Big-O** — cách các lập trình viên đo "code này chậm hay nhanh khi data lớn lên".

### Big-O là gì?

Big-O trả lời câu hỏi: **"Khi input tăng gấp đôi, thời gian tăng bao nhiêu?"**

| Big-O | Tên gọi | Ý nghĩa | Ví dụ đời thường |
|-------|---------|---------|------------------|
| O(1) | Constant | Luôn nhanh, bất kể data lớn | Mở tủ lạnh lấy nước — có 1 chai hay 100 chai, bạn biết chỗ |
| O(log n) | Logarithmic | Rất nhanh, tăng chậm | Tìm từ trong từ điển — mở giữa, loại nửa |
| O(n) | Linear | Tăng tỷ lệ thuận | Đếm tiền trong ví — nhiều tờ hơn = lâu hơn |
| O(n log n) | Linearithmic | Khá nhanh | Sắp xếp bài thi — chia nhỏ ra rồi ghép |
| O(n²) | Quadratic | Chậm! | So sánh từng cặp học sinh — 100 người = 10.000 cặp |

> **💡 Mẹo**: O(1) và O(log n) → tuyệt vời. O(n) → chấp nhận được. O(n²) → cẩn thận khi data lớn.

### Big-O trong Roc

Hãy xem Big-O của các thao tác phổ biến trên List trong Roc:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    numbers = [10, 20, 30, 40, 50]

    # O(1) — lấy phần tử đầu tiên, luôn nhanh
    first = List.first numbers
    when first is
        Ok val -> Stdout.line! "First: $(Num.toStr val)"
        Err ListWasEmpty -> Stdout.line! "List rỗng!"

    # O(1) — lấy theo index
    third = List.get numbers 2
    when third is
        Ok val -> Stdout.line! "Index 2: $(Num.toStr val)"
        Err OutOfBounds -> Stdout.line! "Out of bounds!"

    # O(n) — tìm phần tử (phải duyệt lần lượt)
    has30 = List.contains numbers 30
    Stdout.line! "Có 30? $(if has30 then "Có" else "Không")"

    # O(n) — đếm tổng (phải duyệt hết)
    total = List.walk numbers 0 \state, x -> state + x
    Stdout.line! "Tổng: $(Num.toStr total)"
    # Output:
    # First: 10
    # Index 2: 30
    # Có 30? Có
    # Tổng: 150
```

### Bảng Big-O cho Roc Lists

| Thao tác | Big-O | Giải thích |
|----------|-------|------------|
| `List.get list index` | O(1) | Nhảy thẳng tới vị trí |
| `List.first list` | O(1) | Lấy phần tử đầu |
| `List.last list` | O(1) | Lấy phần tử cuối |
| `List.len list` | O(1) | Roc lưu sẵn độ dài |
| `List.prepend list x` | O(n)* | Copy list rồi thêm đầu |
| `List.append list x` | O(1)** | Thêm cuối (thường nhanh nhờ unique ownership) |
| `List.map list f` | O(n) | Duyệt hết, tạo list mới |
| `List.walk list init f` | O(n) | Duyệt hết, gom kết quả |
| `List.contains list x` | O(n) | Duyệt cho đến khi tìm thấy |
| `List.sortWith list cmp` | O(n log n) | Sắp xếp |

> **\*Lưu ý về Roc**: Roc dùng **reference-counted unique ownership**. Khi list chỉ có 1 owner (thường xuyên), Roc **sửa trực tiếp** (in-place mutation) thay vì copy → nhanh hơn bạn tưởng. Chi tiết → Chapter 3.

---

## ✅ Checkpoint 2.1

> Đến đây bạn phải hiểu:
> 1. **Big-O** = đo tốc độ code khi data lớn lên
> 2. O(1) → tuyệt. O(n) → ổn. O(n²) → cẩn thận
> 3. `List.get` O(1), `List.map/walk` O(n), `List.sortWith` O(n log n)
>
> **Test nhanh**: Bạn có list 1.000.000 phần tử. `List.get list 999999` mất bao lâu so với `List.get list 0`?
> <details><summary>Đáp án</summary>Giống nhau! Cả hai đều O(1) — nhảy thẳng tới vị trí, không phụ thuộc vào index.</details>

---

## 2.2 — Recursion: Function gọi chính nó

Roc không có vòng lặp — recursion là cách duy nhất để lặp. Base case + recursive case = mọi vòng lặp bạn cần.

### Tại sao cần recursion?

Trong nhiều ngôn ngữ, bạn dùng vòng lặp `for`:

```python
# Python
total = 0
for x in [1, 2, 3, 4, 5]:
    total += x
# total = 15
```

Nhưng **Roc không có vòng lặp**. Không có `for`, không có `while`. Vậy làm sao lặp?

Hai cách:
1. **Recursion** — function gọi chính nó
2. **`List.walk`** — hàm có sẵn thay thế vòng lặp (phần 2.3)

### Recursion cơ bản

Bắt đầu với ví dụ đơn giản: tính giai thừa.

```
5! = 5 × 4 × 3 × 2 × 1 = 120
```

Recursion nghĩ khác: **"5! = 5 × (4!)"**

```
5! = 5 × 4!
4! = 4 × 3!
3! = 3 × 2!
2! = 2 × 1!
1! = 1            ← điểm dừng (base case)
```

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Giai thừa — recursion cơ bản
factorial = \n ->
    if n <= 1 then
        1                       # base case — điểm dừng
    else
        n * (factorial (n - 1)) # gọi chính mình với n nhỏ hơn

main =
    Stdout.line! "5! = $(Num.toStr (factorial 5))"
    Stdout.line! "10! = $(Num.toStr (factorial 10))"
    # Output:
    # 5! = 120
    # 10! = 3628800
```

Mọi recursion đều cần **2 phần**:
1. **Base case** — khi nào dừng? (`n <= 1 → 1`)
2. **Recursive case** — bước nhỏ hơn (`n * factorial (n - 1)`)

> **💡 Ẩn dụ**: Recursion giống **búp bê Nga (matryoshka)**. Mở búp bê to, bên trong có búp bê nhỏ hơn, mở tiếp... cho đến búp bê bé nhất (base case). Rồi ghép lại từ trong ra ngoài.

### Vấn đề: Stack Overflow

Mỗi lần function gọi chính nó, máy tính **dùng thêm bộ nhớ** (gọi là "stack frame"). Nếu gọi quá nhiều lần:

```
factorial 100000
= 100000 * factorial 99999
= 100000 * (99999 * factorial 99998)
= 100000 * (99999 * (99998 * ...))
→ 100.000 stack frames → 💥 Stack Overflow!
```

Giải pháp? **Tail-call optimization.**

### Tail-call Optimization (TCO)

**Tail call** = lời gọi đệ quy là **thao tác cuối cùng** của function. Khi đó, compiler có thể **tái sử dụng** stack frame thay vì tạo mới → không bao giờ overflow.

So sánh:

```roc
# ❌ KHÔNG phải tail call — sau khi gọi factorial, còn phải nhân với n
factorial = \n ->
    if n <= 1 then 1
    else n * (factorial (n - 1))     # "n *" xảy ra SAU khi gọi đệ quy

# ✅ Tail call — lời gọi đệ quy là thao tác CUỐI CÙNG
factorialTail = \n, acc ->
    if n <= 1 then acc               # trả kết quả tích lũy
    else factorialTail (n - 1) (acc * n)  # gọi đệ quy = thao tác cuối
```

Mẹo: dùng **accumulator** (`acc`) để tích lũy kết quả dần dần, thay vì chờ tất cả lời gọi return rồi mới tính.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Tail-recursive factorial — an toàn với mọi n
factorialHelper = \n, acc ->
    if n <= 1 then
        acc
    else
        factorialHelper (n - 1) (acc * n)

# Wrapper cho dễ dùng — user chỉ cần truyền n
factorial = \n -> factorialHelper n 1

main =
    Stdout.line! "5! = $(Num.toStr (factorial 5))"
    Stdout.line! "20! = $(Num.toStr (factorial 20))"
    # Output:
    # 5! = 120
    # 20! = 2432902008176640000
```

> **💡 Quy tắc**: Trong Roc, **luôn viết tail-recursive** khi có thể. Hoặc tốt hơn — dùng `List.walk` (phần tiếp theo).

---

## ✅ Checkpoint 2.2

> Đến đây bạn phải hiểu:
> 1. **Recursion** = function gọi chính nó, cần base case + recursive case
> 2. Recursion thông thường **tốn stack** → có thể stack overflow
> 3. **Tail call** = gọi đệ quy là thao tác cuối → compiler tối ưu → an toàn
> 4. Dùng **accumulator** để biến recursion thường thành tail-recursive
>
> **Test nhanh**: Function nào là tail-recursive?
> ```roc
> # A
> sumA = \list ->
>     when list is
>         [] -> 0
>         [first, .. as rest] -> first + sumA rest
>
> # B
> sumB = \list, acc ->
>     when list is
>         [] -> acc
>         [first, .. as rest] -> sumB rest (acc + first)
> ```
> <details><summary>Đáp án</summary>B — vì `sumB rest (acc + first)` là thao tác cuối cùng. Còn A phải cộng `first +` SAU khi gọi xong `sumA rest`.</details>

---

## 2.3 — `List.walk` — Thay thế mọi vòng lặp

`List.walk` là higher-order function mạnh nhất — nó thay thế `for`, `while`, `reduce`, `fold` trong một API duy nhất.

### Tại sao `List.walk`?

Recursion trên list là pattern cực kỳ phổ biến. Thay vì viết đi viết lại, Roc cung cấp **`List.walk`** — function đã có sẵn, đã được tối ưu, và dễ đọc hơn.

`List.walk` giống hệt **fold** trong Haskell/Elixir, **reduce** trong Python/JavaScript.

### Cách hoạt động

```
List.walk [1, 2, 3] startValue combineFunction
```

Đọc: "Đi qua list [1, 2, 3], bắt đầu với `startValue`, và mỗi bước dùng `combineFunction` để gom kết quả."

Hình ảnh:

```
List:    [1,    2,    3]
          ↓     ↓     ↓
Start → [+1] → [+2] → [+3] → Kết quả
  0       1      3      6       = 6
```

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    numbers = [1, 2, 3, 4, 5]

    # Tính tổng — giống "for x in list: total += x"
    total = List.walk numbers 0 \state, x -> state + x
    Stdout.line! "Tổng: $(Num.toStr total)"

    # Tìm max — giống "for x in list: max = max(max, x)"
    maxVal = List.walk numbers 0 \state, x ->
        if x > state then x else state
    Stdout.line! "Max: $(Num.toStr maxVal)"

    # Đếm số chẵn — giống "for x in list: if x % 2 == 0: count += 1"
    evenCount = List.walk numbers 0 \state, x ->
        if x % 2 == 0 then state + 1 else state
    Stdout.line! "Số chẵn: $(Num.toStr evenCount)"

    # Đảo ngược list
    reversed = List.walk numbers [] \state, x -> List.prepend state x
    Stdout.line! "Đảo: $(listToStr reversed)"
    # Output:
    # Tổng: 15
    # Max: 5
    # Số chẵn: 2
    # Đảo: [5, 4, 3, 2, 1]

listToStr = \list ->
    inner = List.walk list "" \state, x ->
        if Str.isEmpty state then
            Num.toStr x
        else
            "$(state), $(Num.toStr x)"
    "[$(inner)]"
```

### `List.walk` vs Recursion vs Vòng lặp

| | Vòng lặp `for` | Recursion tay | `List.walk` |
|---|---|---|---|
| Có trong Roc? | ❌ Không | ✅ Có | ✅ Có |
| Dễ đọc? | Dễ | Trung bình | Dễ |
| An toàn? | — | Cần tail-call | ✅ Đã tối ưu |
| Khi nào dùng? | — | Logic phức tạp (tree, graph) | **Hầu hết trường hợp** |

> **💡 Quy tắc thực tế**: Dùng `List.walk` cho 90% trường hợp. Chỉ viết recursion tay khi xử lý cấu trúc phức tạp (tree, graph, nested data). Luôn ưu tiên `List.map` và `List.keepIf` nếu đủ.

### Họ hàng của `List.walk`

Roc có nhiều functions dựng sẵn thay thế các pattern phổ biến:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    numbers = [1, 2, 3, 4, 5]

    # List.map — biến đổi mỗi phần tử
    doubled = List.map numbers \x -> x * 2
    # → [2, 4, 6, 8, 10]

    # List.keepIf — lọc theo điều kiện
    evens = List.keepIf numbers \x -> x % 2 == 0
    # → [2, 4]

    # List.any — có phần tử nào thỏa điều kiện?
    hasEven = List.any numbers \x -> x % 2 == 0
    # → Bool.true

    # List.all — TẤT CẢ phần tử thỏa điều kiện?
    allPositive = List.all numbers \x -> x > 0
    # → Bool.true

    # Pipeline — kết hợp nhiều bước bằng |>
    result =
        numbers
        |> List.map \x -> x * 2        # nhân đôi
        |> List.keepIf \x -> x > 4     # giữ > 4
        |> List.walk 0 \s, x -> s + x  # tính tổng
    # [1,2,3,4,5] → [2,4,6,8,10] → [6,8,10] → 24

    Stdout.line! "Pipeline result: $(Num.toStr result)"
    # Output: Pipeline result: 24
```

> **💡 Pipeline `|>`** đọc từ trên xuống dưới, giống dây chuyền sản xuất. Data chảy qua từng bước. Rất dễ đọc, dễ debug (xóa 1 bước, đọc kết quả trung gian).

---

## ✅ Checkpoint 2.3

> Đến đây bạn phải hiểu:
> 1. **`List.walk`** = duyệt hết list, gom kết quả — thay thế vòng lặp `for`
> 2. `List.walk list startValue \state, x -> ...` — 3 tham số: list, giá trị bắt đầu, hàm gom
> 3. `List.map` = biến đổi, `List.keepIf` = lọc, `List.any`/`List.all` = kiểm tra
> 4. **Pipeline `|>`** kết hợp nhiều bước xử lý data
>
> **Test nhanh**: Viết `List.walk` tính tích (product) của `[2, 3, 5]`?
> <details><summary>Đáp án</summary>`List.walk [2, 3, 5] 1 \state, x -> state * x` → 30. Bắt đầu từ 1 (đơn vị nhân), nhân dần.</details>

---

## 2.4 — Binary Search & Divide and Conquer

Chia đôi, tìm nửa đúng, lặp lại — O(log n) thay vì O(n). Từ 1 triệu phần tử chỉ cần 20 bước.

### Binary Search — Tìm kiếm siêu nhanh

Quay lại câu chuyện 1.000 học sinh. **Binary search** = mở giữa, loại nửa, lặp lại.

```
Tìm "An" trong ["An", "Bình", "Cường", "Dũng", "Em"]

Bước 1: Giữa = "Cường" → "An" < "Cường" → tìm nửa trái
Bước 2: Giữa = "Bình"  → "An" < "Bình"  → tìm nửa trái
Bước 3: Giữa = "An"    → Tìm thấy! ✅

Chỉ 3 bước thay vì 5 bước (linear search)
```

**Yêu cầu**: List phải **đã sắp xếp**.
**Big-O**: O(log n) — 1.000.000 phần tử chỉ cần ~20 bước!

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Binary search — tìm phần tử trong list đã sắp xếp
binarySearch = \sortedList, target ->
    len = List.len sortedList
    if len == 0 then
        Err NotFound
    else
        binarySearchHelper sortedList target 0 (len - 1)

binarySearchHelper = \list, target, low, high ->
    if low > high then
        Err NotFound
    else
        mid = (low + high) // 2
        when List.get list mid is
            Err OutOfBounds -> Err NotFound
            Ok midVal ->
                if midVal == target then
                    Ok mid                                  # tìm thấy!
                else if midVal < target then
                    binarySearchHelper list target (mid + 1) high   # tìm nửa phải
                else
                    binarySearchHelper list target low (mid - 1)   # tìm nửa trái

main =
    numbers = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91]

    when binarySearch numbers 23 is
        Ok index -> Stdout.line! "Tìm thấy 23 ở index $(Num.toStr index)"
        Err NotFound -> Stdout.line! "Không tìm thấy!"

    when binarySearch numbers 42 is
        Ok index -> Stdout.line! "Tìm thấy 42 ở index $(Num.toStr index)"
        Err NotFound -> Stdout.line! "42 không có trong list"
    # Output:
    # Tìm thấy 23 ở index 5
    # 42 không có trong list
```

Chú ý: function trả `Result` — `Ok index` hoặc `Err NotFound`. Compiler bắt buộc xử lý cả hai. Đây là Curry-Howard từ Chapter 1 đang hoạt động!

### Divide and Conquer — Chia để trị

Binary search là ví dụ của **divide and conquer** — pattern giải quyết bài toán bằng cách:

1. **Chia** bài toán thành bài toán nhỏ hơn
2. **Trị** (giải) từng bài toán nhỏ
3. **Ghép** kết quả lại

Ví dụ kinh điển: **Merge Sort** — sắp xếp bằng chia để trị.

```
Sắp xếp [38, 27, 43, 3, 9, 82, 10]

Chia:   [38, 27, 43, 3]    [9, 82, 10]
Chia:   [38, 27] [43, 3]   [9, 82] [10]
Chia:   [38][27] [43][3]   [9][82] [10]

Ghép:   [27, 38] [3, 43]   [9, 82] [10]
Ghép:   [3, 27, 38, 43]    [9, 10, 82]
Ghép:   [3, 9, 10, 27, 38, 43, 82] ✅
```

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Merge sort — chia để trị, O(n log n)
mergeSort = \list ->
    len = List.len list
    if len <= 1 then
        list                    # base case — list 0-1 phần tử đã sorted
    else
        mid = len // 2
        left = List.takeFirst list mid
        right = List.dropFirst list mid
        # Chia → trị → ghép
        merge (mergeSort left) (mergeSort right)

# Ghép 2 list đã sorted thành 1 list sorted
merge = \listA, listB ->
    mergeHelper listA listB []

mergeHelper = \listA, listB, result ->
    when (List.first listA, List.first listB) is
        (Ok a, Ok b) ->
            if a <= b then
                mergeHelper (List.dropFirst listA 1) listB (List.append result a)
            else
                mergeHelper listA (List.dropFirst listB 1) (List.append result b)
        (Ok _, Err _) -> List.concat result listA
        (Err _, Ok _) -> List.concat result listB
        (Err _, Err _) -> result

main =
    unsorted = [38, 27, 43, 3, 9, 82, 10]
    sorted = mergeSort unsorted

    Stdout.line! "Trước: $(listToStr unsorted)"
    Stdout.line! "Sau:   $(listToStr sorted)"
    # Output:
    # Trước: [38, 27, 43, 3, 9, 82, 10]
    # Sau:   [3, 9, 10, 27, 38, 43, 82]

listToStr = \list ->
    inner = List.walk list "" \state, x ->
        if Str.isEmpty state then
            Num.toStr x
        else
            "$(state), $(Num.toStr x)"
    "[$(inner)]"
```

**Tại sao merge sort quan trọng trong FP?**
- Nó dùng **recursion thuần túy** — không thay đổi data gốc
- O(n log n) — nhanh nhất cho comparison sort
- **Stable** — phần tử bằng nhau giữ thứ tự gốc

> **💡 Thực tế**: Trong Roc hàng ngày, bạn dùng `List.sortWith list compare` thay vì tự viết merge sort. Nhưng hiểu cách nó hoạt động giúp bạn chọn thuật toán đúng.

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Big-O

Xác định Big-O cho mỗi đoạn code:

```roc
# a) Lấy phần tử đầu tiên
first = List.first numbers

# b) Tìm phần tử lớn nhất
max = List.walk numbers 0 \state, x -> if x > state then x else state

# c) Kiểm tra tất cả dương
allPos = List.all numbers \x -> x > 0

# d) Sắp xếp
sorted = List.sortWith numbers Num.compare

# e) Tìm trong list đã sắp xếp bằng binary search
found = binarySearch sortedNumbers target
```

<details><summary>✅ Lời giải Bài 1</summary>

```
a) O(1) — lấy phần tử đầu
b) O(n) — duyệt hết list
c) O(n) — duyệt hết list (tệ nhất)
d) O(n log n) — sắp xếp so sánh
e) O(log n) — mỗi bước loại nửa
```

</details>

---

**Bài 2** (10 phút): Viết bằng `List.walk`

Dùng `List.walk` để viết các function sau:

```roc
# a) Tính tích (product) của list số
# product [2, 3, 5] = 30

# b) Đếm số phần tử lớn hơn 10
# countAbove10 [5, 15, 3, 20, 8] = 2

# c) Tìm chuỗi dài nhất
# longestStr ["hi", "hello", "hey"] = "hello"
```

<details><summary>💡 Gợi ý</summary>

- Product: bắt đầu từ 1, nhân dần
- CountAbove10: bắt đầu từ 0, cộng 1 mỗi khi x > 10
- LongestStr: bắt đầu từ "", so sánh `Str.countUtf8Bytes`

</details>

<details><summary>✅ Lời giải Bài 2</summary>

```roc
# a) Tích
product = \list -> List.walk list 1 \state, x -> state * x

# b) Đếm > 10
countAbove10 = \list ->
    List.walk list 0 \state, x ->
        if x > 10 then state + 1 else state

# c) Chuỗi dài nhất
longestStr = \list ->
    List.walk list "" \state, x ->
        if Str.countUtf8Bytes x > Str.countUtf8Bytes state then x
        else state
```

</details>

---

**Bài 3** (15 phút): Tail-recursive Fibonacci

Viết function tính số Fibonacci thứ n (fib 0 = 0, fib 1 = 1, fib n = fib(n-1) + fib(n-2)):

```roc
# Yêu cầu: tail-recursive, dùng 2 accumulators
# fibonacci 10 = 55
# fibonacci 20 = 6765
```

<details><summary>💡 Gợi ý</summary>

Dùng 2 accumulators: `a` (fib trước) và `b` (fib hiện tại). Mỗi bước: `a ← b`, `b ← a + b`, `n ← n - 1`.

</details>

<details><summary>✅ Lời giải Bài 3</summary>

```roc
fibHelper = \n, a, b ->
    if n == 0 then
        a
    else
        fibHelper (n - 1) b (a + b)

fibonacci = \n -> fibHelper n 0 1

# fibonacci 10 → fibHelper 10 0 1
#                fibHelper 9  1 1
#                fibHelper 8  1 2
#                fibHelper 7  2 3
#                ...
#                fibHelper 0  55 89 → 55 ✅
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Recursion chạy mãi không dừng | Quên base case hoặc base case sai | Kiểm tra: recursive case có tiến gần base case không? |
| Stack overflow | Recursion không phải tail-call | Chuyển sang tail-recursive (dùng accumulator) hoặc `List.walk` |
| `List.walk` cho kết quả sai | Giá trị bắt đầu sai | Tổng → bắt đầu 0. Tích → bắt đầu 1. List → bắt đầu `[]` |
| Binary search không tìm thấy | List chưa sắp xếp | Binary search **yêu cầu** sorted list |

---

## Tóm tắt

- ✅ **Big-O** đo tốc độ code khi data lớn. O(1) > O(log n) > O(n) > O(n log n) > O(n²).
- ✅ **Recursion** = function gọi chính nó. Cần **base case** (điểm dừng) + **recursive case** (bước nhỏ hơn).
- ✅ **Tail-call optimization** = gọi đệ quy cuối cùng → compiler tối ưu. Dùng **accumulator**.
- ✅ **`List.walk`** thay thế vòng lặp `for`. 90% trường hợp dùng `List.walk`, `List.map`, `List.keepIf`.
- ✅ **Binary search** O(log n) — chia đôi mỗi bước. Yêu cầu list đã sorted.
- ✅ **Divide & conquer** — chia bài toán nhỏ, giải riêng, ghép lại. Merge sort O(n log n).

## Tiếp theo

→ Chapter 3: **Functional Data Structures** — Roc Lists hoạt động thế nào bên trong (reference counting, unique ownership, in-place mutation), `Dict`, `Set`, và tại sao Roc nhanh dù "functional".
