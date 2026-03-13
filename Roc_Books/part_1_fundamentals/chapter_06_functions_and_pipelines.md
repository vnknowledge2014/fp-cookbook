# Chapter 6 — Functions & Pipelines

> **Bạn sẽ học được**:
> - Named functions vs anonymous functions (lambdas)
> - Currying — tại sao mọi function trong Roc thực ra chỉ nhận 1 tham số
> - Pipeline `|>` — biến code thành dây chuyền sản xuất
> - Backpassing `<-` — cú pháp ngắn gọn cho callbacks
> - Function composition — ghép functions thành functions mới
>
> **Yêu cầu trước**: [Chapter 5 — Values & Types](chapter_05_values_and_types.md)
> **Thời gian đọc**: ~35 phút | **Level**: Beginner
> **Kết quả cuối cùng**: Viết code theo phong cách FP — data flows through a pipeline of small functions.

---

Ở các chapter trước, bạn đã làm quen với values, types, và cách Roc nhìn nhận dữ liệu. Nhưng dữ liệu nằm yên thì chưa có ích — bạn cần **biến đổi** nó. Cộng hai số, lọc một danh sách, format một chuỗi hiển thị lên màn hình. Tất cả những hành động đó, trong Roc, đều quy về một thứ duy nhất: **functions**.

Chapter này sẽ đi từ cách viết function cơ bản nhất, đến một kỹ thuật mà nhiều người mới thấy lạ lẫm — currying — rồi đến pipeline, thứ sẽ thay đổi hoàn toàn cách bạn đọc và viết code. Nếu bạn từng viết code kiểu `result = trim(uppercase(concat(a, b)))` và thấy đau đầu khi phải đọc từ trong ra ngoài, pipeline sẽ giải quyết chuyện đó.

## 6.1 — Hai cách viết Function

Trong hầu hết các ngôn ngữ lập trình, function có cú pháp riêng — `def` trong Python, `function` trong JavaScript, `fn` trong Rust. Roc thì khác. Roc không có từ khóa riêng cho function. Thay vào đó, bạn viết một **lambda** (hàm vô danh) rồi gán nó cho một cái tên. Nghe có vẻ lạ, nhưng thực ra đơn giản hơn bạn tưởng.

### Named functions (top-level)

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Named function — khai báo ở top-level
add = \a, b -> a + b

double = \x -> x * 2

isEven = \x -> x % 2 == 0

formatGreeting = \name, greeting ->
    "$(greeting), $(name)!"

main =
    Stdout.line! (Num.toStr (add 3 5))                # → 8
    Stdout.line! (Num.toStr (double 21))               # → 42
    Stdout.line! (Inspect.toStr (isEven 4))            # → Bool.true
    Stdout.line! (formatGreeting "An" "Xin chào")      # → Xin chào, An!
```

Nhìn kỹ cú pháp: `add = \a, b -> a + b`. Phần bên phải dấu `=` là một lambda — dấu `\` bắt đầu, liệt kê tham số, rồi `->` chỉ đến phần thân function. Bên trái dấu `=` chỉ đơn giản là bạn đặt tên cho nó.

Điều này nghĩa là trong Roc, **mọi function đều là lambda**. Không có syntax riêng biệt cho "named function" — `add = \a, b -> a + b` hoàn toàn tương đương `fn add(a, b)` hay `def add(a, b)` trong các ngôn ngữ khác. Bạn chỉ gán lambda cho một cái tên, thế thôi.

Tại sao thiết kế như vậy? Vì Roc muốn functions là "first-class citizens" — chúng có thể được truyền đi, gán lại, nhét vào list, đưa làm tham số cho function khác, hoàn toàn giống như bạn làm với một con số hay một chuỗi ký tự. Khi function không có gì đặc biệt hơn một giá trị bình thường, code trở nên linh hoạt hơn rất nhiều.

### Anonymous functions (inline lambdas)

Nhưng không phải lúc nào bạn cũng cần đặt tên. Khi logic đủ ngắn và chỉ dùng đúng một chỗ, viết lambda trực tiếp tại nơi cần dùng sẽ gọn hơn:

```roc
main =
    numbers = [1, 2, 3, 4, 5]

    # Lambda inline — không cần tên
    doubled = List.map numbers \x -> x * 2
    # → [2, 4, 6, 8, 10]

    # Lambda phức tạp hơn
    descriptions = List.map numbers \x ->
        if isEven x then "$(Num.toStr x) chẵn"
        else "$(Num.toStr x) lẻ"
    # → ["1 lẻ", "2 chẵn", "3 lẻ", "4 chẵn", "5 lẻ"]
```

Ở ví dụ đầu, `\x -> x * 2` quá ngắn để cần một cái tên riêng. Nhưng ở ví dụ thứ hai, lambda đã có logic `if/else` bên trong — nếu dài hơn nữa, bạn nên tách ra thành named function cho dễ đọc.

### Khi nào dùng named vs anonymous?

| Named | Anonymous |
|-------|-----------|
| Dùng lại nhiều lần | Dùng 1 lần |
| Logic phức tạp, cần tên rõ ràng | Logic đơn giản (<1 dòng) |
| `add = \a, b -> a + b` | `\x -> x * 2` |

Ranh giới ở đây không cứng nhắc — nhưng nếu bạn thấy mình phải đọc lambda hai lần để hiểu nó làm gì, đó là lúc nên tách ra.

---

## 6.2 — Type Annotations cho Functions

Ở Chapter 5, bạn đã thấy Roc tự suy ra type mà không cần ghi. Type inference tiện, nhưng có giới hạn — khi function phức tạp hoặc khi người khác đọc code của bạn (hoặc chính bạn hai tháng sau), annotation giúp hiểu ngay function nhận gì, trả gì mà không cần đọc phần thân.

Cú pháp annotation trong Roc viết riêng một dòng phía trên function:

```roc
# Viết annotation TRƯỚC function
add : Num a, Num a -> Num a
add = \a, b -> a + b

# Đọc: "add nhận 2 Num cùng kiểu, trả Num cùng kiểu"

isPositive : Num a -> Bool
isPositive = \x -> x > 0

greet : Str, Str -> Str
greet = \name, title -> "$(title) $(name)"

# Function nhận function (higher-order)
applyTwice : (a -> a), a -> a
applyTwice = \f, x -> f (f x)
```

Đọc annotation từ trái sang phải: những thứ trước mũi tên `->` là input, thứ cuối cùng là output. `Num a, Num a -> Num a` nghĩa là "cho tôi hai số cùng kiểu, tôi trả lại một số cùng kiểu."

Một quy ước cần nhớ: **chữ thường** (`a`) là generic type — "bất kỳ kiểu nào cũng được, miễn là nhất quán". **Chữ HOA** (`Str`, `Bool`, `Num`) là kiểu cụ thể. Giống như trong đại số, `a` là biến — nó đại diện cho một kiểu, nhưng bạn chưa biết kiểu nào cho đến khi gọi function.

Dòng thú vị nhất ở trên là `applyTwice : (a -> a), a -> a`. Tham số đầu tiên `(a -> a)` chính là **một function khác** — function nhận kiểu `a` và trả kiểu `a`. Đây là cách Roc biểu đạt higher-order functions qua type: "Bạn đưa tôi một function, tôi sẽ dùng nó."

---

## 6.3 — Currying: Mỗi function chỉ nhận 1 tham số

Đây có lẽ là khái niệm khiến nhiều người lần đầu gặp functional programming phải dừng lại suy nghĩ. Nhưng đừng lo — ý tưởng đằng sau nó thực ra rất quen thuộc.

### Khái niệm

Giả sử bạn đến quán phở và gọi: "Cho tôi phở gà, thêm hành." Người bán ghi nhận hai thông tin — loại thịt và topping — rồi bưng ra một tô phở. Nhưng nếu quán đông, bạn có thể gọi trước: "Cho tôi phở gà" — người bán ghi nhận loại thịt, rồi hỏi tiếp: "Thêm gì không?" Hai bước thay vì một, nhưng kết quả giống hệt.

Currying hoạt động y như vậy. Khi bạn viết `add = \a, b -> a + b`, **Roc thực ra tạo 2 functions lồng nhau**:

```
add = \a -> (\b -> a + b)

# Gọi add 3 5 thực chất là:
# Bước 1: add 3    → trả ra function \b -> 3 + b
# Bước 2: (\b -> 3 + b) 5  → trả ra 8
```

Điều này gọi là **currying** — đặt theo tên nhà toán học Haskell Curry. Mỗi function trông như nhận nhiều tham số, nhưng thực ra compiler biến nó thành chuỗi functions, mỗi cái nhận đúng một.

Bạn có thể tự hỏi: "Vậy thì có gì hay?" Câu trả lời nằm ở phần tiếp theo.

### Partial Application — Sức mạnh thực sự

Vì function được curry, bạn có thể **cho trước một số tham số mà chưa cần cho hết** — và nhận lại một function mới đang "chờ" phần còn lại. Kỹ thuật này gọi là partial application, và nó cực kỳ hữu ích trong thực tế:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Function 2 tham số
multiply = \a, b -> a * b

main =
    # Partial application — cho trước 1 tham số
    double = multiply 2       # → \b -> 2 * b
    triple = multiply 3       # → \b -> 3 * b

    Stdout.line! "double 5 = $(Num.toStr (double 5))"   # → 10
    Stdout.line! "triple 5 = $(Num.toStr (triple 5))"   # → 15

    # Dùng trong List.map — rất gọn!
    numbers = [1, 2, 3, 4, 5]
    doubled = List.map numbers (multiply 2)
    tripled = List.map numbers (multiply 3)

    Stdout.line! "Doubled: $(Inspect.toStr doubled)"   # → [2, 4, 6, 8, 10]
    Stdout.line! "Tripled: $(Inspect.toStr tripled)"   # → [3, 6, 9, 12, 15]
```

Partial application giúp **tái sử dụng** function theo cách rất tự nhiên. Bạn không cần tạo class mới, không cần override method — chỉ cần "điền sẵn" một vài tham số và có ngay function mới sẵn sàng dùng. Trong ví dụ trên, `multiply 2` và `multiply 3` là hai functions hoàn toàn khác nhau, sinh ra từ cùng một "khuôn".

Để thấy rõ hơn giá trị thực tế, hãy xem một ví dụ gần với công việc hàng ngày hơn:

### Ví dụ thực tế: Format tiền

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Function format giá tiền theo đơn vị
formatPrice = \currency, amount ->
    "$(Num.toStr amount) $(currency)"

main =
    # Partial application → tạo formatter cụ thể
    formatVND = formatPrice "VNĐ"
    formatUSD = formatPrice "USD"

    prices = [45000, 50000, 35000]

    # Dùng formatter trong pipeline
    vndPrices = List.map prices formatVND
    List.forEach vndPrices \p -> Stdout.line! "  $(p)"
    # Output:
    #   45000 VNĐ
    #   50000 VNĐ
    #   35000 VNĐ
```

Thay vì viết hai hàm `formatVND` và `formatUSD` riêng biệt, bạn chỉ cần một hàm `formatPrice` và "fix" tham số đầu tiên. Code gọn hơn, dễ mở rộng hơn — muốn thêm EUR hay JPY chỉ cần thêm một dòng.

---

## ✅ Checkpoint 6.1–6.3

> Đến đây bạn phải hiểu:
> 1. Mọi function trong Roc đều là **lambda** (`\x -> ...`)
> 2. **Currying** = function nhiều tham số thực ra là nhiều function lồng nhau
> 3. **Partial application** = cho trước một số tham số → nhận function mới
>
> **Test nhanh**: `multiply 2` có type gì nếu `multiply : Num a, Num a -> Num a`?
> <details><summary>Đáp án</summary>`Num a -> Num a` — nhận 1 số, trả 1 số (đã "nhớ" sẵn tham số đầu là 2).</details>

Nếu bạn trả lời được câu hỏi trong checkpoint, bạn đã nắm vững phần cơ bản. Nhưng functions riêng lẻ thì chưa thực sự mạnh — sức mạnh thật sự đến khi bạn **nối chúng lại với nhau**. Phần tiếp theo sẽ giới thiệu công cụ quan trọng nhất của chapter này.

---

## 6.4 — Pipeline `|>` — Data chảy qua functions

Đây là lúc mọi thứ bắt đầu "click" — nơi mà currying, partial application, và cách Roc thiết kế functions kết hợp lại thành một thứ thật sự mạnh mẽ.

### Vấn đề: Nested function calls

Bạn muốn lấy một chuỗi, nối thêm phần đuôi, chuyển thành chữ hoa, rồi cắt khoảng trắng thừa. Viết theo cách truyền thống sẽ thành thế này:

```roc
# Đọc từ TRONG ra NGOÀI — khó hiểu!
result = Str.trim (Str.toUppercase (Str.concat "hello" " world"))
# Thứ tự thực hiện: concat → toUppercase → trim
# Nhưng bạn đọc: trim → toUppercase → concat 🤯
```

Để hiểu dòng này, bạn phải đọc **từ trong ra ngoài** — `concat` chạy trước, rồi `toUppercase`, rồi `trim`. Nhưng mắt bạn đọc từ trái sang phải: `trim` → `toUppercase` → `concat`. Hai hướng ngược nhau. Khi chỉ có 3 bước thì còn chịu được, nhưng 5-6 bước lồng nhau sẽ trở thành mớ ngoặc đơn khó đọc.

Roc giải quyết chuyện này bằng **pipeline operator** `|>`.

### Giải pháp: Pipeline `|>`

```roc
# Đọc từ TRÊN xuống DƯỚI — theo thứ tự thực hiện!
result =
    "hello"
    |> Str.concat " world"      # "hello world"
    |> Str.toUppercase           # "HELLO WORLD"  (nếu có)
    |> Str.trim                  # "HELLO WORLD"
```

Cùng logic, nhưng giờ bạn đọc từ trên xuống dưới, đúng thứ tự thực hiện. Dòng đầu là data gốc, mỗi dòng tiếp theo là một bước biến đổi.

Quy tắc hoạt động của `|>` rất đơn giản: **lấy kết quả bên trái, đưa vào hàm bên phải như tham số cuối cùng.**

```
a |> f      ≡  f a
a |> f b    ≡  f b a      (a vào vị trí tham số cuối)
```

Đó là lý do tại sao thứ tự tham số quan trọng trong FP. Khi thiết kế function, tham số "chính" (cái bạn muốn pipeline) nên đặt ở cuối. Các hàm trong thư viện chuẩn của Roc — `List.map`, `List.keepIf`, `Str.trim` — đều tuân theo quy ước này.

### Ví dụ thực tế: Xử lý data

Xem pipeline xử lý một list số — mỗi dòng là một bước, dễ theo dõi như đọc checklist:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Pipeline — data chảy từ trên xuống dưới
    result =
        [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        |> List.keepIf \x -> x % 2 == 0      # giữ số chẵn → [2,4,6,8,10]
        |> List.map \x -> x * x               # bình phương → [4,16,36,64,100]
        |> List.walk 0 \s, x -> s + x         # tổng → 220

    Stdout.line! "Tổng bình phương số chẵn: $(Num.toStr result)"
    # Output: Tổng bình phương số chẵn: 220

    # Pipeline với chuỗi
    cleaned =
        "  Hello,  World!  "
        |> Str.trim                           # "Hello,  World!"
        |> Str.split ","                      # ["Hello", "  World!"]
        |> List.map Str.trim                  # ["Hello", "World!"]
        |> Str.joinWith " & "                 # "Hello & World!"

    Stdout.line! cleaned
    # Output: Hello & World!
```

Mỗi dòng trong pipeline là một bước transformation nhỏ. Muốn bỏ bước nào? Comment nó ra. Muốn thêm bước? Thêm một dòng `|>`. Đây là sức mạnh thật sự — code tự giải thích, dễ thay đổi, và bạn có thể debug bằng cách comment từng dòng từ dưới lên.

Kết hợp pipeline với partial application — thứ bạn đã học ở section trước — và code trở nên gọn đến mức như đang viết pseudo-code:

### Pipeline + Partial Application = Magic

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

addSuffix = \suffix, text -> "$(text)$(suffix)"
addPrefix = \prefix, text -> "$(prefix)$(text)"

main =
    # Combine partial application + pipeline
    result =
        "roc"
        |> addPrefix "I love "       # "I love roc"
        |> addSuffix "!"             # "I love roc!"
        |> addSuffix " 🎉"           # "I love roc! 🎉"

    Stdout.line! result
    # Output: I love roc! 🎉
```

Pipeline không chỉ giúp code gọn hơn — nó thay đổi cách bạn **suy nghĩ** về vấn đề. Thay vì nghĩ "tôi cần gọi hàm nào trước", bạn nghĩ "data này cần đi qua những bước nào". Đó chính là tư duy FP.

---

## 6.5 — Backpassing `<-` — Callback gọn gàng

Phần này ngắn, nhưng bạn cần biết vì sẽ gặp nó thường xuyên khi làm việc với Tasks ở Chapter 12. Vấn đề nó giải quyết rất cụ thể: **callback hell**.

### Vấn đề: Callback hell

Khi bạn cần chạy nhiều tác vụ IO nối tiếp nhau — đọc input, xử lý, rồi in kết quả — code sẽ bị thụt vào sâu dần:

```roc
# Dài dòng — nested callbacks
main =
    Task.await Stdin.line \input ->
        Task.await (process input) \result ->
            Stdout.line result
```

Chỉ có 3 bước mà đã thụt vào 3 cấp. Hình dung nếu có 5-6 bước — code sẽ trôi dần sang bên phải.

Backpassing `<-` cho phép bạn viết cùng logic nhưng **phẳng**:

### Giải pháp: Backpassing `<-`

```roc
# Gọn gàng — backpassing
main =
    input <- Stdin.line |> Task.await
    result <- process input |> Task.await
    Stdout.line result
```

Cú pháp `x <- f |> g` đọc là: "chạy `f`, pipe qua `g`, gán kết quả cho `x`, rồi tiếp tục phía dưới". Kết quả hoàn toàn giống phiên bản nested — chỉ gọn và dễ đọc hơn.

Một điều quan trọng: backpassing chỉ là **cú pháp** — nó không thay đổi logic chương trình, chỉ thay đổi cách bạn viết. Compiler biến hai phiên bản thành cùng một thứ. Bạn sẽ dùng nó nhiều khi làm việc với Tasks ở Chapter 12, nên giờ chỉ cần nắm ý tưởng là đủ.

Ví dụ sau không dùng Task (vì chưa học), nhưng cho bạn thấy cú pháp callback quen thuộc:

### Ví dụ thực tế

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Backpassing với List.walk — callback gọn hơn
    numbers = [1, 2, 3, 4, 5]

    # Thay vì:
    # total = List.walk numbers 0 (\state, x -> state + x)

    # Dùng backpassing:
    total = List.walk numbers 0 \state, x -> state + x

    Stdout.line! "Tổng: $(Num.toStr total)"
```

> **💡 Lưu ý**: Backpassing là **cú pháp** — không thay đổi logic. Nó chỉ giúp code dễ đọc hơn. Bạn sẽ dùng nhiều khi làm việc với Tasks (Chapter 12).

---

## 6.6 — Function Composition

Đến đây, bạn đã biết cách viết function, curry chúng, nối chúng bằng pipeline, và rút gọn callback. Phần cuối cùng này kết hợp tất cả lại thành một triết lý lập trình: **xây dựng chương trình từ những mảnh nhỏ**.

Ý tưởng đơn giản: thay vì viết một function lớn làm mọi thứ, bạn viết nhiều function nhỏ, mỗi cái làm đúng một việc, rồi nối chúng lại thành pipeline. Nếu một bước sai, bạn sửa đúng function đó mà không ảnh hưởng phần còn lại.

### Ghép functions nhỏ thành function lớn

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Các bước xử lý nhỏ
normalize = \text -> Str.trim text
validate = \text ->
    if Str.isEmpty text then Err EmptyInput
    else Ok text
formatOutput = \text -> "✅ Hợp lệ: $(text)"

# Compose thành pipeline
processInput = \rawInput ->
    rawInput
    |> normalize
    |> validate
    |> Result.map formatOutput

main =
    # Test với input hợp lệ
    when processInput "  Hello Roc  " is
        Ok msg -> Stdout.line! msg
        Err EmptyInput -> Stdout.line! "❌ Input rỗng!"

    # Test với input rỗng
    when processInput "   " is
        Ok msg -> Stdout.line! msg
        Err EmptyInput -> Stdout.line! "❌ Input rỗng!"

    # Output:
    # ✅ Hợp lệ: Hello Roc
    # ❌ Input rỗng!
```

Quan sát cách `processInput` được xây dựng: nó không chứa logic phức tạp — chỉ gọi 3 function nhỏ nối tiếp nhau. `normalize` lo chuyện khoảng trắng, `validate` lo chuyện input rỗng, `formatOutput` lo chuyện hiển thị. Mỗi function làm đúng một việc duy nhất, dễ test riêng, dễ thay thế.

### Pattern: Transform pipeline

Đây là pattern cốt lõi nhất của FP — bạn sẽ gặp nó lặp đi lặp lại trong suốt cuốn sách:

```
Raw Input → Normalize → Validate → Transform → Format → Output
```

Thiết kế chương trình theo cách này có một lợi thế rất lớn: khi có bug, bạn biết **chính xác** nó nằm ở bước nào. Nếu data đã normalize đúng nhưng validate sai, bug nằm trong hàm `validate`. Không cần đọc cả chương trình.

Ví dụ sau là một ứng dụng nhỏ để bạn thấy pattern này trong thực tế: hệ thống tính điểm học sinh. Dữ liệu đầu vào là một chuỗi CSV, đầu ra là xếp loại — mỗi bước xử lý được tách thành function riêng:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Mỗi function làm 1 việc duy nhất
parseScores = \csv ->
    csv
    |> Str.split ","
    |> List.keepOks Str.toI64

average = \numbers ->
    if List.isEmpty numbers then
        0
    else
        total = List.walk numbers 0 \s, x -> s + x
        total // (Num.toI64 (List.len numbers))

grade = \avg ->
    if avg >= 90 then "A"
    else if avg >= 80 then "B"
    else if avg >= 70 then "C"
    else "D"

# Pipeline: CSV → scores → average → grade
processStudent = \name, csv ->
    scores = parseScores csv
    avg = average scores
    letterGrade = grade avg
    "$(name): $(Num.toStr avg) điểm → $(letterGrade)"

main =
    Stdout.line! (processStudent "An" "85,92,78,95")
    Stdout.line! (processStudent "Bình" "70,65,72,68")
    # Output:
    # An: 87 điểm → B
    # Bình: 68 điểm → D
```

Nhìn vào `processStudent`: nó chỉ gọi 3 function — `parseScores`, `average`, `grade` — rồi format kết quả. Mỗi function có thể test độc lập: `parseScores "85,92"` phải trả `[85, 92]`, `average [80, 90]` phải trả `85`, `grade 87` phải trả `"B"`. Nếu đầu ra sai, bạn biết chính xác function nào cần sửa.

---

## 🏋️ Bài tập

Ba bài tập sau đi từ dễ đến khó, mỗi bài tập trung vào một kỹ thuật chính bạn vừa học. Hãy thử làm trước khi xem lời giải — việc tự viết code (dù sai) giúp bạn nhớ lâu hơn rất nhiều so với việc đọc lời giải.

**Bài 1** (5 phút): Pipeline cơ bản

Viết pipeline biến `[5, 3, 8, 1, 9, 2]` thành tổng 3 số lớn nhất:

```roc
# Kết quả mong đợi: 8 + 9 + 5 = 22
```

<details><summary>💡 Gợi ý</summary>

Sort (giảm dần) → lấy 3 phần tử đầu → tính tổng

</details>

<details><summary>✅ Lời giải</summary>

```roc
result =
    [5, 3, 8, 1, 9, 2]
    |> List.sortWith \a, b -> Num.compare b a  # sort giảm dần
    |> List.takeFirst 3                         # lấy 3 phần tử đầu
    |> List.walk 0 \s, x -> s + x              # tổng
# → 22
```

</details>

---

**Bài 2** (10 phút): Partial Application

Bài này kiểm tra xem bạn đã nắm được partial application chưa. Tạo "factory" functions — functions sinh ra functions khác bằng cách fix sẵn một tham số:

```roc
# a) Tạo discounter — giảm giá theo %
# discount30 = createDiscount 30
# discount30 100000  → 70000

# b) Tạo greeting formatter
# greetVietnamese = createGreeting "Xin chào"
# greetVietnamese "An"  → "Xin chào An!"
```

<details><summary>✅ Lời giải</summary>

```roc
# a)
createDiscount = \percent, price ->
    price - (price * percent // 100)

discount30 = createDiscount 30
# discount30 100000  → 70000

# b)
createGreeting = \greeting, name ->
    "$(greeting) $(name)!"

greetVietnamese = createGreeting "Xin chào"
# greetVietnamese "An"  → "Xin chào An!"
```

</details>

---

**Bài 3** (15 phút): Data processing pipeline

Bài cuối đòi hỏi kết hợp tất cả: pipeline, anonymous functions, và `List.walk`. Cho danh sách giao dịch, viết pipeline tính tổng tiền chi cho một category:

```roc
transactions = [
    { item: "Phở", amount: 45000, category: "Food" },
    { item: "Cà phê", amount: 25000, category: "Drink" },
    { item: "Bún bò", amount: 50000, category: "Food" },
    { item: "Trà đá", amount: 5000, category: "Drink" },
    { item: "Cơm tấm", amount: 40000, category: "Food" },
]

# Yêu cầu: Tính tổng tiền chi cho "Food" bằng pipeline
# Kết quả mong đợi: 135000
```

<details><summary>✅ Lời giải</summary>

```roc
foodTotal =
    transactions
    |> List.keepIf \t -> t.category == "Food"     # lọc Food
    |> List.map \t -> t.amount                     # lấy amount
    |> List.walk 0 \s, x -> s + x                 # tổng
# → 135000
```

</details>

---

## 🔧 Troubleshooting

Nếu bạn gặp lỗi khi viết code cho bài tập, đây là những vấn đề phổ biến nhất và cách sửa:

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `Too many args` | Gọi function với nhiều tham số hơn nó nhận | Kiểm tra signature — có thể cần dấu ngoặc |
| Pipeline type mismatch | Output bước trước không khớp input bước sau | Kiểm tra type từng bước — comment bước cuối, xem kết quả trung gian |
| Partial application không hoạt động | Thứ tự tham số sai | Tham số muốn "fix" phải ở ĐẦU — thiết kế function accordingly |
| Backpassing confusing | Chưa quen cú pháp `<-` | Viết dạng bình thường trước, rồi refactor sang `<-` |

---

## Tóm tắt

Chapter này đã trang bị cho bạn bộ công cụ cơ bản nhất của FP: functions. Từ cách viết đơn giản nhất cho đến kết hợp chúng thành pipeline — mỗi kỹ thuật sẽ xuất hiện lặp lại trong những chapter tiếp theo.

- ✅ **Mọi function = lambda** — `add = \a, b -> a + b`. Named chỉ là gán lambda cho tên.
- ✅ **Currying** = function nhiều tham số = nhiều function lồng nhau, mỗi cái nhận 1 tham số.
- ✅ **Partial application** = cho trước 1+ tham số → nhận function mới. Rất mạnh khi kết hợp `List.map`.
- ✅ **Pipeline `|>`** = data chảy từ trên xuống qua nhiều bước. Đọc theo thứ tự thực hiện, dễ sửa.
- ✅ **Backpassing `<-`** = callback gọn gàng. Dùng nhiều với Tasks.
- ✅ **Transform pipeline** = pattern FP cốt lõi: Raw → Normalize → Validate → Transform → Output.

Những kỹ thuật này sẽ xuất hiện liên tục trong những chapter tiếp theo. Pipeline đặc biệt quan trọng — bạn sẽ thấy nó ở mọi nơi, từ xử lý dữ liệu đơn giản cho đến thiết kế domain logic phức tạp.

## Tiếp theo

→ Chapter 7: **Records & Tag Unions** ⭐ — deep dive vào hai công cụ thiết kế quan trọng nhất của Roc. Structural records, tag unions, record update syntax, open vs closed types, và "making illegal states unrepresentable" trong thực tế.
