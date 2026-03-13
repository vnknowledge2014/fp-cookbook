# Chapter 3 — Functional Data Structures

> **Bạn sẽ học được**:
> - Tại sao Roc "functional nhưng nhanh" — bí mật của reference counting & unique ownership
> - Khi nào Roc copy data, khi nào sửa trực tiếp (in-place mutation)
> - `Dict` và `Set` — hai cấu trúc dữ liệu quan trọng nhất ngoài List
> - Cách biểu diễn Graph bằng `Dict` — ADT (Algebraic Data Type) trong thực tế
> - Big-O so sánh: List vs Dict vs Set
>
> **Yêu cầu trước**: [Chapter 2 — Algorithmic Thinking](chapter_02_algorithmic_thinking.md)
> **Thời gian đọc**: ~40 phút | **Level**: CS Foundations
> **Kết quả cuối cùng**: Chọn đúng data structure cho bất kỳ bài toán nào, hiểu trade-off giữa chúng.

---

## 3.1 — Bí mật của Roc: Functional nhưng nhanh

FP truyền thống copy data mỗi lần "thay đổi" — chậm. Roc dùng unique references và in-place mutation ẩn — nhanh như imperative code.

### Vấn đề của Functional Programming truyền thống

Trong FP thuần túy, data là **immutable** — không thay đổi được. Khi bạn "thêm phần tử vào list", thực ra bạn **tạo list mới**:

```
# List gốc
original = [1, 2, 3]

# "Thêm" 4 → tạo list MỚI, list cũ không đổi
withFour = List.append original 4
# original vẫn là [1, 2, 3]
# withFour là [1, 2, 3, 4]
```

Trong Haskell hay Erlang, việc này **copy toàn bộ list** → chậm, tốn bộ nhớ.

Roc giải quyết bằng cách thông minh: **nếu không ai dùng list cũ nữa → sửa trực tiếp, không copy**.

### Reference Counting & Unique Ownership

Roc theo dõi **bao nhiêu chỗ đang dùng** mỗi giá trị. Cơ chế này gọi là **reference counting** (đếm tham chiếu):

```
# Trường hợp 1: chỉ 1 owner → sửa trực tiếp (nhanh!)
numbers = [1, 2, 3]
result = List.append numbers 4
# "numbers" không còn ai dùng → Roc sửa trực tiếp
# → KHÔNG copy, nhanh như mutable code! O(1)
```

```
# Trường hợp 2: nhiều owner → phải copy (chậm hơn)
numbers = [1, 2, 3]
backup = numbers                # bây giờ 2 owner!
result = List.append numbers 4
# "numbers" vẫn được "backup" dùng → Roc phải copy
# → Copy rồi sửa, O(n)
```

Hình ảnh:

```
Trường hợp 1 (unique):        Trường hợp 2 (shared):
numbers → [1, 2, 3]           numbers → [1, 2, 3] ← backup
         ↓ append 4                    ↓ append 4 (phải copy!)
result → [1, 2, 3, 4]         result → [1, 2, 3, 4]  (bản mới)
(sửa trực tiếp!)              backup → [1, 2, 3]      (bản cũ vẫn còn)
```

> **💡 Nguyên tắc**: Viết code theo kiểu "dùng xong thì bỏ" → Roc tối ưu tự động. Không cần suy nghĩ thủ công — compiler làm hết.

### Tại sao điều này quan trọng?

Nó cho phép bạn **viết code functional thuần túy** (dễ đọc, dễ test, không bugs) mà **chạy nhanh như code mutable** (C, Rust). Best of both worlds.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Pipeline này TRÔNG như tạo nhiều list trung gian
    # Nhưng Roc tối ưu: sửa trực tiếp ở mỗi bước (seamless mutation)
    result =
        [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        |> List.map \x -> x * x             # bình phương
        |> List.keepIf \x -> x > 10         # giữ > 10
        |> List.sortWith Num.compare        # sắp xếp
        |> List.walk 0 \s, x -> s + x       # tổng

    Stdout.line! "Tổng bình phương > 10: $(Num.toStr result)"
    # Output: Tổng bình phương > 10: 371
    # (16 + 25 + 36 + 49 + 64 + 81 + 100 = 371)
```

**Bạn viết code đẹp, Roc lo chuyện tốc độ.** Đây là triết lý cốt lõi của ngôn ngữ.

### So sánh với các ngôn ngữ khác

| Approach | Ngôn ngữ | Ưu | Nhược |
|----------|----------|-----|-------|
| Mutable (trực tiếp) | C, Go | Nhanh | Bugs do shared state |
| Immutable (copy hết) | Haskell, Erlang | An toàn | Chậm, tốn bộ nhớ |
| **Unique ownership** | **Roc**, Rust* | **Nhanh VÀ an toàn** | Cần compiler thông minh |

> *Rust dùng ownership cho memory safety, Roc dùng reference counting cho seamless mutation — mục tiêu khác nhau, triết lý tương tự.

---

## ✅ Checkpoint 3.1

> Đến đây bạn phải hiểu:
> 1. Roc dùng **reference counting** để theo dõi ai đang dùng data
> 2. **1 owner** → sửa trực tiếp (nhanh). **Nhiều owner** → copy rồi sửa (chậm hơn)
> 3. Pipeline `|>` thường chỉ có 1 owner ở mỗi bước → tối ưu tự động
> 4. Bạn viết code functional thuần túy, Roc lo tốc độ
>
> **Test nhanh**: Code nào nhanh hơn?
> ```roc
> # A — list chỉ có 1 owner
> result = List.append [1, 2, 3] 4
>
> # B — list có 2 owners
> original = [1, 2, 3]
> backup = original
> result = List.append original 4
> ```
> <details><summary>Đáp án</summary>A nhanh hơn — chỉ 1 owner nên Roc sửa trực tiếp O(1). B phải copy trước vì `backup` vẫn dùng data → O(n).</details>

---

## 3.2 — Dict: Tra cứu siêu nhanh

Dict (dictionary/hash map) tra cứu O(1) — cho key, nhận value ngay lập tức. Dùng khi cần lookup nhanh.

### Khi nào cần Dict?

List tìm kiếm bằng `List.contains` → O(n) — phải duyệt lần lượt. Khi list lớn, điều này chậm.

**Dict** (Dictionary) giải quyết bằng cách lưu dữ liệu theo **key-value pairs** → tra cứu O(log n).

Ẩn dụ: List giống **danh sách cuộn** — muốn tìm phải đọc từ đầu. Dict giống **từ điển** — nhảy thẳng tới chữ cái cần tìm.

### Sử dụng Dict

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Tạo Dict từ list các cặp (key, value)
    prices = Dict.fromList [
        ("Phở", 45000),
        ("Bún bò", 50000),
        ("Cơm tấm", 40000),
    ]

    # Tra cứu — O(log n), nhanh!
    when Dict.get prices "Phở" is
        Ok price -> Stdout.line! "Phở: $(Num.toStr price)đ"
        Err KeyNotFound -> Stdout.line! "Không có món này"

    # Thêm món mới — trả Dict mới (immutable!)
    updatedPrices = Dict.insert prices "Bún riêu" 35000

    # Đếm số món
    Stdout.line! "Tổng món: $(Num.toStr (Dict.len updatedPrices))"

    # Liệt kê tất cả
    Dict.walk updatedPrices {} \_, key, value ->
        Stdout.line! "  $(key): $(Num.toStr value)đ"

    # Output:
    # Phở: 45000đ
    # Tổng món: 4
    #   Phở: 45000đ
    #   Bún bò: 50000đ
    #   Cơm tấm: 40000đ
    #   Bún riêu: 35000đ
```

### Các thao tác phổ biến trên Dict

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    scores = Dict.fromList [
        ("An", 95),
        ("Bình", 87),
        ("Cường", 92),
    ]

    # Kiểm tra key tồn tại
    hasAn = Dict.contains scores "An"
    Stdout.line! "Có An? $(if hasAn then "Có" else "Không")"

    # Xóa key
    withoutBinh = Dict.remove scores "Bình"
    Stdout.line! "Sau xóa Bình: $(Num.toStr (Dict.len withoutBinh)) người"

    # Lấy danh sách keys
    names = Dict.keys scores
    Stdout.line! "Học sinh: $(Num.toStr (List.len names)) người"

    # Lấy danh sách values
    allScores = Dict.values scores

    # Tính điểm trung bình bằng List.walk
    total = List.walk allScores 0 \state, x -> state + x
    count = List.len allScores
    avg = total // (Num.toI64 count)
    Stdout.line! "Điểm TB: $(Num.toStr avg)"

    # Output:
    # Có An? Có
    # Sau xóa Bình: 2 người
    # Học sinh: 3 người
    # Điểm TB: 91
```

### Bảng Big-O cho Dict

| Thao tác | Big-O | So sánh với List |
|----------|-------|-----------------|
| `Dict.get dict key` | O(log n) | List.contains: O(n) |
| `Dict.insert dict key value` | O(log n) | — |
| `Dict.remove dict key` | O(log n) | — |
| `Dict.contains dict key` | O(log n) | List.contains: O(n) |
| `Dict.walk dict init f` | O(n) | Giống List.walk |
| `Dict.len dict` | O(1) | Giống List.len |

> **💡 Khi nào dùng Dict thay List?** Khi bạn cần **tra cứu theo key** nhanh. Ví dụ: tìm điểm theo tên học sinh, tìm giá theo tên sản phẩm, đếm tần suất từ.

---

## 3.3 — Set: Tập hợp không trùng lặp

Set = collection không có phần tử trùng. Kiểm tra membership O(1). Dùng cho dedup, intersection, union.

### Set là gì?

Set = **tập hợp các giá trị duy nhất** (không có phần tử trùng). Giống tấm vé vào cổng — mỗi người chỉ có 1 vé, quẹt lần thứ 2 thì không tính.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Tạo Set — tự động loại trùng lặp
    uniqueColors = Set.fromList ["Red", "Blue", "Red", "Green", "Blue"]
    Stdout.line! "Số màu: $(Num.toStr (Set.len uniqueColors))"
    # Output: Số màu: 3 (Red, Blue, Green — không có trùng)

    # Kiểm tra phần tử có trong set — O(log n)
    hasRed = Set.contains uniqueColors "Red"
    Stdout.line! "Có Red? $(if hasRed then "Có" else "Không")"

    # Thêm phần tử
    withYellow = Set.insert uniqueColors "Yellow"
    Stdout.line! "Sau thêm Yellow: $(Num.toStr (Set.len withYellow))"

    # Output:
    # Số màu: 3
    # Có Red? Có
    # Sau thêm Yellow: 4
```

### Phép toán tập hợp

Set hỗ trợ các phép toán từ toán học: hợp (union), giao (intersection), hiệu (difference):

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    frontend = Set.fromList ["An", "Bình", "Cường"]
    backend = Set.fromList ["Bình", "Dũng", "Em"]

    # Union — hợp: tất cả thành viên (cả 2 team)
    allDevs = Set.union frontend backend
    Stdout.line! "Tất cả: $(Num.toStr (Set.len allDevs)) người"
    # → 5 (An, Bình, Cường, Dũng, Em)

    # Intersection — giao: làm cả 2 team
    bothTeams = Set.intersection frontend backend
    Stdout.line! "Cả 2 team: $(Num.toStr (Set.len bothTeams)) người"
    # → 1 (Bình)

    # Difference — hiệu: chỉ frontend, không backend
    frontendOnly = Set.difference frontend backend
    Stdout.line! "Chỉ FE: $(Num.toStr (Set.len frontendOnly)) người"
    # → 2 (An, Cường)

    # Output:
    # Tất cả: 5 người
    # Cả 2 team: 1 người
    # Chỉ FE: 2 người
```

### Ứng dụng thực tế: Đếm từ duy nhất

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    words = ["roc", "is", "fast", "roc", "is", "fun", "fast"]

    # Đếm từ duy nhất bằng Set
    uniqueWords = Set.fromList words
    Stdout.line! "Tổng từ: $(Num.toStr (List.len words))"
    Stdout.line! "Từ duy nhất: $(Num.toStr (Set.len uniqueWords))"

    # Đếm tần suất mỗi từ bằng Dict
    frequency = List.walk words (Dict.empty {}) \dict, word ->
        when Dict.get dict word is
            Ok count -> Dict.insert dict word (count + 1)
            Err KeyNotFound -> Dict.insert dict word 1

    Dict.walk frequency {} \_, word, count ->
        Stdout.line! "  \"$(word)\" × $(Num.toStr count)"

    # Output:
    # Tổng từ: 7
    # Từ duy nhất: 4
    #   "roc" × 2
    #   "is" × 2
    #   "fast" × 2
    #   "fun" × 1
```

---

## ✅ Checkpoint 3.2–3.3

> Đến đây bạn phải hiểu:
> 1. **Dict** = key-value pairs, tra cứu O(log n) — dùng khi cần tìm theo key
> 2. **Set** = tập hợp không trùng — dùng khi cần loại duplicate hoặc phép toán tập hợp
> 3. Cả Dict và Set đều **immutable** — thao tác trả ra bản mới, bản cũ không đổi
>
> **Test nhanh**: Bạn cần kiểm tra "email này đã đăng ký chưa?" nhanh nhất. Dùng List hay Set?
> <details><summary>Đáp án</summary>Set — vì `Set.contains` là O(log n), còn `List.contains` là O(n). Với 1 triệu emails, Set chỉ cần ~20 bước, List cần tối đa 1 triệu bước.</details>

---

## 3.4 — Graph: Biểu diễn quan hệ phức tạp

Graph mô hình hóa quan hệ: social network, dependencies, routes. Nodes + edges — biểu diễn bằng Dict trong Roc.

### Tại sao cần Graph?

Không phải mọi data đều là "danh sách" hay "từ điển". Đôi khi data có **quan hệ nhiều-nhiều**:

- Bạn bè trên mạng xã hội (An là bạn của Bình, Bình là bạn của Cường…)
- Đường đi giữa các thành phố
- Dependencies giữa các modules trong code

Đây là **Graph** — tập hợp các **node** (đỉnh) và **edge** (cạnh nối).

### Biểu diễn Graph bằng Dict

Roc không có kiểu Graph riêng, nhưng bạn dùng `Dict` rất tự nhiên:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Graph = Dict node (List neighbor)
    # Mạng bạn bè: An ↔ Bình, An ↔ Cường, Bình ↔ Cường, Bình ↔ Dũng
    friendGraph = Dict.fromList [
        ("An", ["Bình", "Cường"]),
        ("Bình", ["An", "Cường", "Dũng"]),
        ("Cường", ["An", "Bình"]),
        ("Dũng", ["Bình"]),
    ]

    # Tìm bạn bè của An
    when Dict.get friendGraph "An" is
        Ok friends ->
            Stdout.line! "Bạn bè của An: $(Num.toStr (List.len friends)) người"
        Err KeyNotFound ->
            Stdout.line! "Không tìm thấy An"

    # Đếm tổng connections
    totalEdges = Dict.walk friendGraph 0 \count, _, friends ->
        count + List.len friends
    # Mỗi edge đếm 2 lần (A→B và B→A), nên chia 2
    Stdout.line! "Tổng kết nối: $(Num.toStr (totalEdges // 2))"

    # Ai có nhiều bạn nhất?
    mostPopular = Dict.walk friendGraph { name: "", count: 0 } \best, name, friends ->
        friendCount = List.len friends
        if friendCount > best.count then
            { name, count: friendCount }
        else
            best
    Stdout.line! "Nhiều bạn nhất: $(mostPopular.name) ($(Num.toStr mostPopular.count) bạn)"

    # Output:
    # Bạn bè của An: 2 người
    # Tổng kết nối: 4
    # Nhiều bạn nhất: Bình (3 bạn)
```

### Directed Graph — Đơn hàng phụ thuộc

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Directed Graph: module A phụ thuộc vào B, C...
    # Edge A → B nghĩa là "A cần B"
    dependencies = Dict.fromList [
        ("App", ["Auth", "Database", "Logger"]),
        ("Auth", ["Database", "Logger"]),
        ("Database", ["Logger"]),
        ("Logger", []),
    ]

    # Module nào không phụ thuộc gì? (leaf nodes)
    leafModules = Dict.walk dependencies [] \leaves, name, deps ->
        if List.isEmpty deps then
            List.append leaves name
        else
            leaves
    Stdout.line! "Leaf modules (không phụ thuộc):"
    List.forEach leafModules \name ->
        Stdout.line! "  $(name)"

    # Module nào được phụ thuộc nhiều nhất?
    # Đếm số lần mỗi module xuất hiện trong danh sách deps
    depCount = Dict.walk dependencies (Dict.empty {}) \counts, _, deps ->
        List.walk deps counts \c, dep ->
            current = when Dict.get c dep is
                Ok n -> n
                Err KeyNotFound -> 0
            Dict.insert c dep (current + 1)

    Dict.walk depCount {} \_, name, count ->
        Stdout.line! "  $(name) được $(Num.toStr count) module phụ thuộc"

    # Output:
    # Leaf modules (không phụ thuộc):
    #   Logger
    #   Logger được 3 module phụ thuộc
    #   Database được 2 module phụ thuộc
    #   Auth được 1 module phụ thuộc
```

Chú ý: bạn dùng **Dict + List + List.walk** — ba công cụ từ chapters trước — để xây dựng cấu trúc phức tạp. Đó là sức mạnh của FP: **các thành phần nhỏ kết hợp thành hệ thống lớn**.

---

## 3.5 — Chọn đúng Data Structure

Chọn sai data structure = code chậm gấp 100x. Bảng dưới giúp chọn đúng cho từng use case.

### Bảng so sánh tổng hợp

| Cần gì? | Data structure | Ví dụ |
|---------|---------------|-------|
| Danh sách có thứ tự | **List** | Lịch sử đơn hàng, dãy số |
| Tra cứu theo key | **Dict** | Tìm giá sản phẩm, tìm điểm theo tên |
| Loại trùng lặp | **Set** | Email đã đăng ký, tags duy nhất |
| Quan hệ nhiều-nhiều | **Dict Str (List Str)** | Bạn bè, dependencies |
| Đếm tần suất | **Dict key U64** | Đếm từ, thống kê |

### Big-O so sánh

| Thao tác | List | Dict | Set |
|----------|------|------|-----|
| Truy cập theo index | O(1) | — | — |
| Tìm kiếm | O(n) | O(log n) | O(log n) |
| Thêm cuối | O(1)* | O(log n) | O(log n) |
| Xóa | O(n) | O(log n) | O(log n) |
| Duyệt hết | O(n) | O(n) | O(n) |

> *O(1) amortized khi unique ownership

### Decision flowchart

```
Bạn cần gì?
├── Thứ tự quan trọng? → List
├── Tra cứu theo key? → Dict
├── Chỉ cần biết "có hay không"? → Set
├── Quan hệ giữa nhiều thứ? → Dict key (List value) (Graph)
└── Đếm số lần xuất hiện? → Dict key U64
```

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Chọn data structure

Chọn data structure phù hợp cho mỗi bài toán:

```
a) Lưu danh sách 100 bài hát yêu thích, có thứ tự
b) Kiểm tra username có trùng không khi đăng ký
c) Tìm số điện thoại theo tên liên lạc
d) Tìm đường đi ngắn nhất giữa 2 thành phố
e) Đếm số lần mỗi chữ cái xuất hiện trong chuỗi
```

<details><summary>✅ Lời giải Bài 1</summary>

```
a) List — cần thứ tự
b) Set — chỉ cần "có hay không", nhanh O(log n)
c) Dict Str Str — key = tên, value = số điện thoại
d) Dict Str (List Str) — Graph, dùng BFS/DFS
e) Dict Str U64 — key = chữ cái, value = số lần
```

</details>

---

**Bài 2** (10 phút): Từ điển Việt-Anh

Xây dựng từ điển Việt-Anh dùng `Dict`. Yêu cầu:
- Thêm 5 từ vào từ điển
- Tra cứu 1 từ có và 1 từ không có
- Đếm tổng số từ

<details><summary>✅ Lời giải Bài 2</summary>

```roc
main =
    viDict = Dict.fromList [
        ("xin chào", "hello"),
        ("cảm ơn", "thank you"),
        ("tạm biệt", "goodbye"),
        ("nước", "water"),
        ("cà phê", "coffee"),
    ]

    # Tra cứu
    when Dict.get viDict "cà phê" is
        Ok eng -> Stdout.line! "cà phê = $(eng)"
        Err KeyNotFound -> Stdout.line! "Không tìm thấy"

    when Dict.get viDict "bia" is
        Ok eng -> Stdout.line! "bia = $(eng)"
        Err KeyNotFound -> Stdout.line! "\"bia\" chưa có trong từ điển"

    Stdout.line! "Tổng: $(Num.toStr (Dict.len viDict)) từ"
```

</details>

---

**Bài 3** (15 phút): Phân tích mạng xã hội

Cho graph sau, viết functions:

```roc
socialGraph = Dict.fromList [
    ("An", ["Bình", "Cường", "Dũng"]),
    ("Bình", ["An", "Cường"]),
    ("Cường", ["An", "Bình", "Dũng", "Em"]),
    ("Dũng", ["An", "Cường"]),
    ("Em", ["Cường"]),
]

# a) Đếm tổng số kết nối (edges)
# b) Tìm người có ít bạn nhất
# c) Kiểm tra An và Em có phải bạn trực tiếp không
```

<details><summary>💡 Gợi ý</summary>

- a) Walk qua graph, tổng List.len, chia 2
- b) Walk qua graph, so sánh List.len, giữ min
- c) Dict.get An → kiểm tra List.contains friends "Em"

</details>

<details><summary>✅ Lời giải Bài 3</summary>

```roc
# a) Tổng edges
totalEdges = Dict.walk socialGraph 0 \count, _, friends ->
    count + List.len friends
edges = totalEdges // 2     # chia 2 vì mỗi edge đếm 2 lần
# → 6

# b) Ít bạn nhất
leastPopular = Dict.walk socialGraph { name: "", count: 999 } \best, name, friends ->
    c = List.len friends
    if c < best.count then { name, count: c } else best
# → Em (1 bạn)

# c) An và Em có bạn trực tiếp?
areFriends = when Dict.get socialGraph "An" is
    Ok friends -> List.contains friends "Em"
    Err KeyNotFound -> Bool.false
# → Bool.false (An không có Em trong danh sách bạn trực tiếp)
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| `Dict.get` trả `Err KeyNotFound` | Key không tồn tại hoặc sai chính tả | Kiểm tra key — Roc phân biệt hoa/thường |
| Set.fromList cho ít phần tử hơn | Có phần tử trùng — Set tự loại trùng | Đây là behavior đúng, không phải bug |
| Performance chậm khi dùng List để tìm kiếm | List.contains là O(n) | Chuyển sang Dict hoặc Set → O(log n) |
| Dict.walk thứ tự không đúng | Dict **không đảm bảo thứ tự** | Dùng List nếu cần thứ tự, hoặc sort keys trước |

---

## Tóm tắt

- ✅ **Roc Lists** nhanh nhờ **reference counting + unique ownership** — sửa trực tiếp khi chỉ có 1 owner, copy khi có nhiều owners.
- ✅ **Dict** = key-value pairs, tra cứu O(log n). Dùng khi cần tìm theo key.
- ✅ **Set** = tập hợp không trùng lặp. Hỗ trợ union, intersection, difference.
- ✅ **Graph** = `Dict node (List neighbor)` — Roc không có kiểu Graph riêng, dùng Dict + List.
- ✅ **Chọn đúng data structure** = kỹ năng quan trọng nhất. List cho thứ tự, Dict cho tra cứu, Set cho uniqueness.

## Tiếp theo

→ **Part I bắt đầu!** Chapter 4: **Getting Started** — cài Roc, hiểu platform model, viết chương trình đầu tiên. Từ đây bạn sẽ học kỹ từng feature của ngôn ngữ.
