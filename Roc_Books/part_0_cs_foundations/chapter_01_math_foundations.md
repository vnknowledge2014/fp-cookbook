# Chapter 1 — Math Foundations for FP

> **Bạn sẽ học được**:
> - Lambda Calculus là gì — và tại sao nó đơn giản hơn bạn tưởng
> - Curry-Howard Correspondence — compiler chính là "thầy giáo" kiểm tra bài cho bạn
> - Dùng phép cộng và phép nhân để đếm số trạng thái chương trình (**Algebraic Type Sizes**)
> - Tại sao "make illegal states unrepresentable" giúp code hết bug
>
> **Yêu cầu trước**: Đọc [Chapter 0 — Roc in 10 Minutes](chapter_00_roc_in_10_minutes.md) nếu chưa biết cú pháp Roc.
> **Thời gian đọc**: ~40-50 phút | **Level**: CS Foundations
> **Kết quả cuối cùng**: Nhìn vào bất kỳ data model nào và tính được "có bao nhiêu trạng thái sai có thể xảy ra" — rồi sửa thiết kế để trạng thái sai = 0.

---

## Trước khi bắt đầu: Bạn không cần "giỏi toán"

Nếu bạn đọc tên chapter — "Math Foundations" — và thấy hơi lo, đừng lo. Chương này **không** yêu cầu bạn giải phương trình hay nhớ công thức phức tạp.

Toán ở đây giống như cách bạn đếm tiền thối ở quán trà sữa hơn là toán trong sách giáo khoa. Mình sẽ đi từng bước nhỏ, với rất nhiều ví dụ từ đời thường.

Nếu bạn hiểu được:
- 2 + 3 = 5
- 2 × 3 = 6
- "Nếu trời mưa **THÌ** mang ô"

...thì bạn đủ nền tảng cho toàn bộ chapter này rồi.

---

## 1.1 — Lambda Calculus: Máy biến đổi giá trị

Lambda calculus là nền tảng toán học của FP — mọi function trong Roc đều là lambda.

### Câu chuyện mở đầu

Hãy tưởng tượng bạn có một **chiếc máy** rất đơn giản. Máy này chỉ làm một việc: bạn cho thứ gì đó vào, nó nhả ra kết quả.

```
🥚 → [Máy luộc trứng] → 🥚🔥 (trứng chín)
```

Bạn cho trứng sống vào, máy trả ra trứng chín. Luôn luôn như vậy. Không bao giờ cho trứng vào mà ra bánh mì.

Trong lập trình, chiếc máy này gọi là **function**. Và năm 1936, một nhà toán học tên **Alonzo Church** phát minh ra cách viết tắt cho chiếc máy:

```
λx. x + 1
```

Đọc là: *"Cho tôi một thứ gọi là x, tôi trả lại x cộng 1."*

Hệ thống này gọi là **Lambda Calculus**. Nghe fancy nhưng ý tưởng rất đơn giản: **mọi thứ đều là máy biến đổi** (function). Không cần biến thay đổi, không cần vòng lặp, không cần gì khác.

Church đặt câu hỏi: "Nếu ta chỉ dùng functions — liệu ta có thể thực hiện MỌI phép tính không?" Câu trả lời: **Có!**

Roc được xây dựng trực tiếp trên triết lý này. Trong Roc, mọi thứ xoay quanh functions. Không có biến thay đổi được, không có vòng lặp kiểu `for`. Chỉ có functions — giống hệt Lambda Calculus.

### Lambda trong Roc

Trong toán, Lambda Calculus viết function thế này:

```
λx. x + 1
```

Đọc từng phần:
- `λ` (lambda) = "đây là một cái máy"
- `x` = "bỏ cái gì đó vào, gọi nó là x"
- `.` = "rồi"
- `x + 1` = "trả ra x cộng 1"

Ghép lại: **"Đây là cái máy, bỏ x vào, ra x+1"**.

Trong Roc, viết gần như y hệt:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Chiếc máy: cho x vào, trả ra x + 1
    addOne = \x -> x + 1

    # Cho số 5 vào máy
    result = addOne 5

    Stdout.line! "Cho 5 vào máy cộng 1: ra $(Num.toStr result)"
    # Output: Cho 5 vào máy cộng 1: ra 6
```

Dấu `\` trong Roc thay cho `λ` — vì bàn phím không có phím lambda!

Hãy so sánh:

| Toán (Lambda Calculus) | Roc | Ý nghĩa |
|---|---|---|
| `λx. x + 1` | `\x -> x + 1` | Cho x, trả x+1 |
| `λx. x * 2` | `\x -> x * 2` | Cho x, trả x nhân đôi |
| `λx. λy. x + y` | `\x, y -> x + y` | Cho x và y, trả tổng |

Bạn thấy không? Chỉ đổi `λ` thành `\` và `.` thành `->`. Vậy thôi.

> **💡 So sánh ngôn ngữ**: Nếu biết Elm thì syntax giống hệt. Nếu biết Rust thì `|x| x + 1` cùng ý nghĩa, khác cú pháp. Nếu biết Python thì `lambda x: x + 1`.

### β-reduction: Cho giá trị vào máy

Khi bạn cho một giá trị cụ thể vào "chiếc máy", quá trình đó gọi là **β-reduction** (đọc: "beta reduction"). Nghe khó nhưng thực ra là thay số vào rồi tính:

```
Máy:           λx. x + 1
Bỏ 5 vào:     (λx. x + 1) 5
Thay x bằng 5: 5 + 1
Kết quả:       6
```

Giống như bạn bấm máy tính vậy: nhập 5, bấm "+1", ra 6.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    addOne = \x -> x + 1

    # β-reduction lần 1: thay x = 5 → ra 6
    Stdout.line! "addOne 5 = $(Num.toStr (addOne 5))"

    # β-reduction lần 2: thay x = 100 → ra 101
    Stdout.line! "addOne 100 = $(Num.toStr (addOne 100))"

    # β-reduction lần 3: thay x = 0 → ra 1
    Stdout.line! "addOne 0 = $(Num.toStr (addOne 0))"
    # Output:
    # addOne 5 = 6
    # addOne 100 = 101
    # addOne 0 = 1
```

**Mỗi lần bạn gọi function trong Roc, bạn đang làm β-reduction.** Bạn đã làm hàng ngàn lần mà không biết nó có tên.

### Máy nhận máy: Higher-Order Functions

Đây là lúc thú vị. Lambda Calculus cho phép **máy nhận máy khác làm đầu vào**.

Ẩn dụ đời thường: Nghĩ về **dây chuyền sản xuất** trong nhà máy.

- Máy A: cắt vải
- Máy B: may vải thành áo
- Dây chuyền: cho vải vào Máy A, kết quả đưa vào Máy B → ra áo

"Dây chuyền" ở đây là một "máy" nhận 2 "máy" khác (A, B) rồi nối chúng lại.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Máy 1: cộng 10
    addTen = \x -> x + 10

    # Máy 2: nhân đôi
    double = \x -> x * 2

    # "Dây chuyền": nhận một máy và chạy nó 2 lần
    applyTwice = \f, x ->
        first = f x       # chạy máy lần 1
        f first            # lấy kết quả, chạy lần 2

    # Chạy máy "cộng 10" hai lần: 5 → 15 → 25
    r1 = applyTwice addTen 5
    Stdout.line! "Cộng 10 hai lần: 5 → $(Num.toStr r1)"

    # Chạy máy "nhân đôi" hai lần: 3 → 6 → 12
    r2 = applyTwice double 3
    Stdout.line! "Nhân đôi hai lần: 3 → $(Num.toStr r2)"
    # Output:
    # Cộng 10 hai lần: 5 → 25
    # Nhân đôi hai lần: 3 → 12
```

`applyTwice` là **higher-order function** — function nhận function khác làm đầu vào.

Chú ý: Roc không cần bạn ghi kiểu cho `f` hay `x`. Compiler tự đoán ra `f` phải là function vì bạn gọi `f x`. Đây gọi là **type inference** — compiler thông minh hơn bạn tưởng.

> **💡 Nếu bạn là học sinh nhỏ tuổi**: Hãy nghĩ function như **hộp ma thuật**. Bạn bỏ thứ gì vào, nó phun ra thứ khác. Higher-order function = hộp ma thuật to hơn, có thể chứa hộp ma thuật nhỏ bên trong!

> **💡 Tip**: Trong Roc, `List.map`, `List.keepIf`, `List.walk` — tất cả đều là "dây chuyền nhận máy biến đổi". Bạn sẽ dùng chúng rất nhiều từ Chapter 9.

### Church Encoding — Trò ảo thuật: Mọi thứ đều là function

Alonzo Church còn chứng minh một điều kỳ diệu: **chỉ cần functions thôi, bạn có thể tạo ra số, boolean, if/else, và mọi thứ khác.**

Ví dụ, TRUE và FALSE có thể viết bằng function:

- **TRUE** = "Cho tôi 2 thứ, tôi chọn **cái đầu tiên**"
- **FALSE** = "Cho tôi 2 thứ, tôi chọn **cái thứ hai**"

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # TRUE = luôn chọn cái đầu tiên
    churchTrue = \a, _b -> a

    # FALSE = luôn chọn cái thứ hai
    churchFalse = \_a, b -> b

    # Thử nghiệm: hỏi "Hôm nay trời nắng?"
    # Nếu ĐÚNG → chọn "đi biển", nếu SAI → chọn "ở nhà"
    Stdout.line! "Trời nắng (đúng): $(churchTrue "đi biển" "ở nhà")"
    Stdout.line! "Trời nắng (sai):  $(churchFalse "đi biển" "ở nhà")"
    # Output:
    # Trời nắng (đúng): đi biển
    # Trời nắng (sai):  ở nhà
```

Đây là **if/else viết bằng function thuần túy!** Bạn không cần nhớ Church Encoding để code Roc, nhưng nó cho thấy một ý tưởng quan trọng: **functions là viên gạch cơ bản nhất** — mạnh đến mức có thể xây dựng mọi thứ từ nó. Roc đặt functions làm trung tâm vì lý do đó.

---

## ✅ Checkpoint 1.1

> Đến đây bạn phải hiểu:
> 1. **Function = chiếc máy biến đổi**: cho đầu vào, nhận đầu ra
> 2. **`\x -> x + 1`** trong Roc = `λx. x + 1` trong Lambda Calculus
> 3. **β-reduction** = cho giá trị vào function và tính kết quả (bạn làm hàng ngày)
> 4. **Higher-order function** = function nhận function khác làm đầu vào
>
> **Test nhanh**: `(\x -> \y -> x + y) 3 5` cho kết quả bao nhiêu?
> <details><summary>Đáp án</summary>8 — thay x=3, y=5, tính 3+5=8. Giống như gọi `\x, y -> x + y` với (3, 5).</details>

---

## 1.2 — Curry-Howard: Types là lời hứa

Curry-Howard correspondence: types = logical propositions, programs = proofs. Function type `a -> b` = "nếu có a, tôi đảm bảo cho b".

### Câu chuyện: Bao bì trên hộp sữa

Khi bạn cầm hộp sữa tươi ở siêu thị, trên hộp ghi "Sữa tươi 100% — 1 lít". Bạn tin rằng:
- Bên trong **đúng là sữa** (không phải nước lọc)
- Dung tích **đúng 1 lít** (không phải 500ml)

Nếu bên trong là nước cam, đó là gian lận. Hệ thống nhà nước sẽ phạt.

**Types trong lập trình hoạt động giống y như nhãn dán trên hộp sữa.** Chúng là **lời hứa** về nội dung bên trong. Và "hệ thống nhà nước" ở đây chính là **compiler** — nó kiểm tra xem lời hứa có đúng không.

### Curry-Howard Correspondence

Hai nhà toán học, Haskell Curry và William Howard, phát hiện một điều bất ngờ: **lập trình và logic toán học thực ra là cùng một thứ**, chỉ viết khác nhau.

Nếu bạn chưa học logic toán học, đừng lo. Hãy nghĩ đơn giản thế này:

| Đời thường | Trong Roc | Trong logic |
|---|---|---|
| "Nếu có gạo **thì** nấu cơm được" | `cook : Rice -> Meal` | A → B (A kéo theo B) |
| "Mang **cả** áo **và** quần" | `{ shirt: Shirt, pants: Pants }` | A AND B |
| "Trả bằng tiền mặt **hoặc** thẻ" | `[Cash, Card]` | A OR B |

Ý nghĩa thực tế: **Nếu bạn viết được function `foo : A -> B` chạy đúng**, bạn đã chứng minh rằng "từ A có thể suy ra B".

### Types = Lời hứa trong code

Hãy xem types hoạt động như lời hứa thế nào:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Lời hứa 1: "Cho tôi một số, tôi trả lại số gấp đôi"
# Rõ ràng, đơn giản, không thể hiểu nhầm
doubleIt = \x -> x * 2

# Lời hứa 2: "Cho tôi hai số, tôi trả kết quả chia HOẶC lỗi"
# Result = "thành công hoặc thất bại — tôi nói rõ cho bạn"
safeDivide = \a, b ->
    if b == 0 then
        Err DivisionByZero    # lỗi — tag tự tạo!
    else
        Ok (a / b)            # kết quả

main =
    # Lời hứa 1: luôn trả số
    Stdout.line! "Gấp đôi 5: $(Num.toStr (doubleIt 5))"

    # Lời hứa 2: compiler BẮT BUỘC xử lý cả Ok lẫn Err
    when safeDivide 10 3 is
        Ok result -> Stdout.line! "10 ÷ 3 = $(Num.toStr result)"
        Err DivisionByZero -> Stdout.line! "Lỗi: chia cho 0!"

    when safeDivide 10 0 is
        Ok result -> Stdout.line! "10 ÷ 0 = $(Num.toStr result)"
        Err DivisionByZero -> Stdout.line! "Lỗi: chia cho 0!"
    # Output:
    # Gấp đôi 5: 10
    # 10 ÷ 3 = 3
    # Lỗi: chia cho 0!
```

Nếu bạn quên xử lý trường hợp `Err`, compiler sẽ **từ chối chạy chương trình**. Đây không phải compiler khó tính — đây là compiler đang bảo vệ bạn, giống như thầy giáo kiểm tra bài trước khi cho nộp.

Chú ý: `DivisionByZero` là một **tag** — trong Roc, bạn không cần khai báo gì trước. Cứ viết `Err DivisionByZero`, compiler tự hiểu. Đây là **structural typing** — type được xác định bởi *cấu trúc*, không phải *tên* (→ chi tiết ở Chapter 7).

### Tại sao điều này quan trọng?

Vì nó cho ta nguyên tắc: **nếu type đúng, chương trình rất khó sai**.

Roc đặc biệt hơn nhiều ngôn ngữ: **mọi function đều pure** (không tạo side effects — không đọc file, không gửi HTTP). Side effects chỉ xảy ra qua `Task` và được platform xử lý (→ Chapter 12). Types mạnh + purity = gần như không thể viết bug.

> **💡 Tip**: Khi compiler báo lỗi, đừng khó chịu. Nghĩ rằng: "Compiler đang nhắc mình chưa xử lý hết trường hợp." Đó là Curry-Howard đang bảo vệ bạn.

---

## ✅ Checkpoint 1.2

> Đến đây bạn phải hiểu:
> 1. **Types = lời hứa** về đầu vào và đầu ra
> 2. `Result ok err` = "thành công hoặc lỗi" — bắt buộc bạn xử lý cả hai
> 3. Compiler kiểm tra types = kiểm tra lời hứa có nhất quán không
> 4. Roc dùng **structural typing** — tags tự do, không cần khai báo trước
>
> **Test nhanh**: `safeDivide : F64, F64 -> Result F64 [DivisionByZero]` — function này "hứa" điều gì?
> <details><summary>Đáp án</summary>"Cho tôi hai số F64, tôi sẽ trả lại F64 hoặc lỗi DivisionByZero — tôi nói rõ cả hai trường hợp, và bạn phải xử lý cả hai."</details>

---

## 1.3 — Phép nhân và phép cộng trong Types

Record = phép nhân (A AND B AND C). Tag union = phép cộng (A OR B OR C). Hai phép toán này đủ để mô tả mọi domain.

### Ẩn dụ: Menu quán cà phê

Bạn vào quán cà phê và thấy menu:

**Combo bữa sáng** (chọn **tất cả**):
- Bánh mì **VÀ** cà phê **VÀ** trái cây

**Đồ uống** (chọn **một**):
- Trà **HOẶC** cà phê **HOẶC** nước cam

Hai kiểu chọn này tương ứng với hai loại type:
- **VÀ** (chọn tất cả) → **Product type** → Record `{ a, b }` trong Roc
- **HOẶC** (chọn một) → **Sum type** → Tag union `[A, B]` trong Roc

### Record = VÀ = Phép nhân

Khi bạn kết hợp nhiều thứ bằng **VÀ**, bạn có **product type**.

Tại sao gọi là "phép nhân"? Hãy đếm:

```
Nếu quán có:   3 món chính × 2 đồ uống × 3 tráng miệng
Tổng combo:    3 × 2 × 3 = 18 combo khác nhau
```

Giống hệt phép nhân trong toán!

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    # Record = product type = chọn TẤT CẢ = phép NHÂN
    # Không cần khai báo type trước — structural typing!
    myCombo = {
        mainDish: Pho,       # VÀ (chọn 1 trong 3)
        drink: CaPhe,        # VÀ (chọn 1 trong 2)
        dessert: Flan,       # (chọn 1 trong 3)
    }

    mainStr = when myCombo.mainDish is
        Pho -> "Phở"
        BunBo -> "Bún bò"
        ComTam -> "Cơm tấm"

    drinkStr = when myCombo.drink is
        TraDa -> "Trà đá"
        CaPhe -> "Cà phê"

    dessertStr = when myCombo.dessert is
        Che -> "Chè"
        KemDua -> "Kem dừa"
        Flan -> "Flan"

    Stdout.line! "Combo: $(mainStr) + $(drinkStr) + $(dessertStr)"
    # Output: Combo: Phở + Cà phê + Flan

    # Tổng combo = 3 × 2 × 3 = 18
    Stdout.line! "Tổng combo có thể: $(Num.toStr (3 * 2 * 3))"
    # Output: Tổng combo có thể: 18
```

**Quy tắc nhớ: Record = VÀ = nhân.**

Điểm đặc biệt của Roc: bạn viết `{ mainDish: Pho, drink: CaPhe }` mà **không cần khai báo type trước**. Compiler tự suy ra. Đây là **structural typing** — khác với Rust hay Gleam nơi bạn phải khai báo `struct` hoặc type alias.

### Tag Union = HOẶC = Phép cộng

Khi bạn chọn **một trong nhiều** lựa chọn, bạn có **sum type**.

Tại sao gọi là "phép cộng"? Hãy đếm:

```
Thanh toán bằng:  Tiền mặt HOẶC Thẻ HOẶC Ví điện tử
Tổng cách trả:   1 + 1 + 1 = 3 cách
```

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Tag union = sum type = chọn MỘT = phép CỘNG
describePayment = \payment ->
    when payment is
        Cash -> "💵 Trả tiền mặt"
        Card { number } -> "💳 Trả thẻ: $(number)"
        EWallet { name } -> "📱 Trả ví: $(name)"

main =
    Stdout.line! (describePayment Cash)
    Stdout.line! (describePayment (Card { number: "****1234" }))
    Stdout.line! (describePayment (EWallet { name: "MoMo" }))
    # Output:
    # 💵 Trả tiền mặt
    # 💳 Trả thẻ: ****1234
    # 📱 Trả ví: MoMo
```

**Quy tắc nhớ: Tag union = HOẶC = cộng.**

Tag union trong Roc là **structural** — bạn không cần khai báo type trước. Cứ viết `Cash`, compiler tự hiểu. Đây khác biệt lớn so với `enum` trong Rust hay Gleam, nơi bạn phải khai báo tên type.

> **💡 Mẹo ghi nhớ cho bạn nhỏ tuổi**:
> - **Record** giống **bộ đồ chơi Lego**: bạn cần TẤT CẢ các miếng ghép (VÀ)
> - **Tag union** giống **tiệm kem**: bạn chỉ chọn MỘT vị (HOẶC)

### Bảng tóm tắt nhanh

| Loại | Trong Roc | Nghĩa | Phép tính | Ví dụ |
|------|-----------|-------|-----------|-------|
| Product | Record `{ a, b }` | VÀ (tất cả) | Nhân × | 2 bánh × 3 nước = **6** |
| Sum | Tag union `[A, B]` | HOẶC (một) | Cộng + | 2 bánh + 3 nước = **5** |

---

## ✅ Checkpoint 1.3

> Đến đây bạn phải hiểu:
> 1. Record `{ a, b }` = VÀ → **nhân** số trạng thái
> 2. Tag union `[A, B]` = HOẶC → **cộng** số trạng thái
> 3. Bạn đếm được tổng trạng thái bằng phép nhân và cộng đơn giản
>
> **Test nhanh**: Một record có 2 field kiểu `Bool` có bao nhiêu trạng thái?
> <details><summary>Đáp án</summary>2 × 2 = 4 trạng thái. Bool có 2 giá trị, record nhân chúng lại.</details>

---

## 1.4 — Đếm trạng thái: Vũ khí bí mật chống bugs

Đây là phần thực hành nhất — và là kỹ năng bạn sẽ dùng xuyên suốt cuốn sách.

### Bài toán: Đơn hàng quán cà phê

Bạn xây hệ thống quản lý đơn hàng. Một đơn hàng có các trạng thái: mới tạo → đang pha → sẵn sàng → khách nhận.

#### Cách 1: Dùng nhiều boolean (❌ SAI)

Cách "tự nhiên" nhất là tạo một `Bool` cho mỗi trạng thái:

```roc
# ❌ CÁCH SAI: mỗi bước là một cờ Bool
badOrder = {
    isConfirmed: Bool.true,     # đã xác nhận chưa?
    isPreparing: Bool.false,    # đang pha chế chưa?
    isReady: Bool.false,        # đã sẵn sàng chưa?
    isDelivered: Bool.false,    # đã giao chưa?
}

# Bao nhiêu trạng thái có thể xảy ra?
# Mỗi Bool có 2 giá trị → 2 × 2 × 2 × 2 = 16 trạng thái
#
# Nhưng chỉ có ~4 trạng thái HỢP LÝ:
# 1. Mới xác nhận:  {true,  false, false, false}
# 2. Đang pha:      {true,  true,  false, false}
# 3. Sẵn sàng:      {true,  true,  true,  false}
# 4. Đã giao:       {true,  true,  true,  true}
#
# 16 - 4 = 12 trạng thái VÔ NGHĨA!
#
# Ví dụ: isDelivered = true nhưng isConfirmed = false
# → 🤯 Giao cho ai khi chưa ai đặt?!
```

**Vấn đề**: 16 trạng thái nhưng chỉ 4 trạng thái hợp lệ. **12 trạng thái còn lại là bugs ngồi chờ bạn mắc sai lầm.** Năm nay không bug, nhưng ai đó vào sửa code năm sau thì... 💥

#### Cách 2: Dùng tag union (✅ ĐÚNG)

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ✅ CÁCH ĐÚNG: liệt kê chính xác các trạng thái hợp lệ
advanceOrder = \status ->
    when status is
        Confirmed -> Preparing
        Preparing -> Ready
        Ready -> Delivered
        Delivered -> Delivered    # đã giao rồi thì giữ nguyên

statusToStr = \status ->
    when status is
        Confirmed -> "📝 Đã xác nhận"
        Preparing -> "☕ Đang pha"
        Ready -> "✅ Sẵn sàng"
        Delivered -> "🎉 Đã giao"

main =
    # Chỉ có 4 trạng thái — TẤT CẢ đều hợp lệ!
    s0 = Confirmed
    s1 = advanceOrder s0    # → Đang pha
    s2 = advanceOrder s1    # → Sẵn sàng
    s3 = advanceOrder s2    # → Đã giao

    Stdout.line! "Bước 0: $(statusToStr s0)"
    Stdout.line! "Bước 1: $(statusToStr s1)"
    Stdout.line! "Bước 2: $(statusToStr s2)"
    Stdout.line! "Bước 3: $(statusToStr s3)"
    # Output:
    # Bước 0: 📝 Đã xác nhận
    # Bước 1: ☕ Đang pha
    # Bước 2: ✅ Sẵn sàng
    # Bước 3: 🎉 Đã giao
```

**So sánh:**

| | Cách 1 (bool flags) | Cách 2 (tag union) |
|---|---|---|
| Tổng trạng thái | 16 | 4 |
| Trạng thái hợp lệ | 4 | 4 |
| Trạng thái thừa (= bugs) | **12** | **0** |
| Compiler bảo vệ? | ❌ Không | ✅ Có |

Và đây là điểm đẹp của Roc: bạn không cần khai báo `OrderStatus` đâu cả. Chỉ cần dùng `Confirmed`, `Preparing`, `Ready`, `Delivered` như các tags. Compiler tự suy ra tag union `[Confirmed, Preparing, Ready, Delivered]`.

> **💡 Nguyên tắc vàng**: Nếu "tổng trạng thái" > "trạng thái hợp lệ", thiết kế cần sửa.

### Bảng công thức đếm nhanh

| Type | Số trạng thái | Ví dụ Roc |
|------|--------------|-----------|
| `{}` (empty record) | 1 | Chỉ có 1 giá trị |
| `Bool` | 2 | `Bool.true` hoặc `Bool.false` |
| Tag union N tags | N | `[Spring, Summer, Fall, Winter]` → 4 |
| Record `{ a, b }` | a × b | `{ x: Bool, y: Bool }` → 2×2 = 4 |
| Tag union with data | cộng từng tag | `[Circle Bool, Rect Bool Bool]` → 2 + 4 = 6 |
| `Result ok err` | \|ok\| + \|err\| | `Result Bool [NotFound]` → 2+1 = 3 |

### Thực hành cùng nhau

Hãy cùng thiết kế hệ thống xác thực tài khoản và đếm trạng thái:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Hệ thống đăng nhập — tag union:
#   Anonymous                                 → 1 trạng thái
#   LoggedIn { userId: U8, isAdmin: Bool }    → 256 × 2 = 512
#   Banned [Spam, Fraud, Harassment]          → 3
# Tổng: 1 + 512 + 3 = 516 trạng thái — TẤT CẢ hợp lệ!

greetUser = \account ->
    when account is
        Anonymous ->
            "👤 Xin chào khách! Hãy đăng nhập."

        LoggedIn { userId, isAdmin } ->
            if isAdmin then
                "👑 Chào Admin #$(Num.toStr userId)"
            else
                "😊 Chào User #$(Num.toStr userId)"

        Banned reason ->
            reasonStr = when reason is
                Spam -> "Spam"
                Fraud -> "Lừa đảo"
                Harassment -> "Quấy rối"
            "🚫 Tài khoản bị cấm: $(reasonStr)"

main =
    users = [
        Anonymous,
        LoggedIn { userId: 42, isAdmin: Bool.false },
        LoggedIn { userId: 1, isAdmin: Bool.true },
        Banned Spam,
    ]

    List.forEach users \account ->
        Stdout.line! (greetUser account)

    Stdout.line! "\nTổng trạng thái: 1 + 512 + 3 = $(Num.toStr (1 + 512 + 3))"
    # Output:
    # 👤 Xin chào khách! Hãy đăng nhập.
    # 😊 Chào User #42
    # 👑 Chào Admin #1
    # 🚫 Tài khoản bị cấm: Spam
    #
    # Tổng trạng thái: 1 + 512 + 3 = 516
```

Không thể tạo user vừa `Anonymous` vừa `LoggedIn`. Không thể tạo user `Banned` nhưng vẫn có `isAdmin`. Compiler ngăn chặn hết.

---

## 🏋️ Bài tập

**Bài 1** (5 phút): Đếm trạng thái

Tính số trạng thái của mỗi type:

```roc
# a) Record có 2 field Bool
# { x: Bool, y: Bool }

# b) Tag union 3 tags
# [Red, Yellow, Green]

# c) Record chứa 2 tag unions
# { north: [Red, Yellow, Green], east: [Red, Yellow, Green] }

# d) Tag union với data bên trong
# [Circle Bool, Rectangle Bool Bool]
```

<details><summary>✅ Lời giải Bài 1</summary>

```
a) Record (nhân): 2 × 2 = 4
b) Tag union (cộng): 3
c) Record (nhân): 3 × 3 = 9
d) Tag union (cộng): Circle có 2 + Rectangle có 2×2=4 → 2 + 4 = 6
```

</details>

---

**Bài 2** (10 phút): Sửa thiết kế sai

Record sau có bao nhiêu trạng thái? Bao nhiêu hợp lệ? Hãy chuyển thành tag union:

```roc
# ❌ Thiết kế sai
account = {
    isActive: Bool.true,
    isVerified: Bool.false,
    isSuspended: Bool.false,
}
```

<details><summary>💡 Gợi ý</summary>Lifecycle tài khoản: đăng ký → xác minh → hoạt động → (có thể bị khóa). Dùng tag union cho lifecycle.</details>

<details><summary>✅ Lời giải Bài 2</summary>

```roc
# Cũ: 2 × 2 × 2 = 8 trạng thái, hợp lệ chỉ ~4
# → 4 trạng thái thừa = bugs

# ✅ Mới: chính xác 4 trạng thái, tất cả hợp lệ
# [Registered, Verified, Active, Suspended Str]
#
# Registered              → chưa xác minh
# Verified                → đã xác minh email
# Active                  → tài khoản active
# Suspended "vi phạm"     → bị tạm khóa (kèm lý do)
```

</details>

---

**Bài 3** (15 phút): Thiết kế domain model cho quán cà phê

Thiết kế hệ thống types cho quy trình gọi đồ uống:
- 3 loại đồ uống: cà phê, trà, sinh tố
- 3 sizes: S, M, L
- 3 topping: không, trân châu, thạch
- 4 trạng thái đơn: mới → đang pha → sẵn sàng → khách nhận

Yêu cầu: (1) Viết types bằng tags + records, (2) Tính tổng combo, (3) Viết function `describeOrder` dùng `when...is`

<details><summary>💡 Gợi ý</summary>

- Drink, Size, Topping → tag union (sum types) — chọn MỘT
- Đơn hàng = Drink + Size + Topping + Status → record (product type) — cần TẤT CẢ
- Tổng combo đồ uống = 3 × 3 × 3 = ?

</details>

<details><summary>✅ Lời giải Bài 3</summary>

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

describeOrder = \order ->
    drinkName = when order.drink is
        Coffee -> "Cà phê"
        Tea -> "Trà"
        Smoothie -> "Sinh tố"
    sizeStr = when order.size is
        S -> "nhỏ"
        M -> "vừa"
        L -> "lớn"
    toppingStr = when order.topping is
        None -> ""
        BobaPearl -> " + trân châu"
        Jelly -> " + thạch"
    icon = when order.status is
        New -> "📝"
        Preparing -> "☕"
        Ready -> "✅"
        PickedUp -> "🎉"
    "$(icon) $(drinkName) size $(sizeStr)$(toppingStr)"

main =
    # Tổng combo: 3 × 3 × 3 = 27 loại đồ uống
    # Kể cả trạng thái: 27 × 4 = 108
    Stdout.line! "Tổng combo: $(Num.toStr (3 * 3 * 3))"
    Stdout.line! "Tổng trạng thái: $(Num.toStr (3 * 3 * 3 * 4))"

    order = {
        drink: Coffee,
        size: M,
        topping: BobaPearl,
        status: Preparing,
    }
    Stdout.line! (describeOrder order)
    # Output:
    # Tổng combo: 27
    # Tổng trạng thái: 108
    # ☕ Cà phê size vừa + trân châu
```

</details>

---

## 🔧 Troubleshooting

| Lỗi thường gặp | Nguyên nhân | Cách sửa |
|---|---|---|
| `does not cover all possibilities` | Quên xử lý một tag trong `when...is` | Thêm nhánh còn thiếu — Roc bắt buộc xử lý hết |
| `Type mismatch` | Tên field/tag sai hoặc type không khớp | Kiểm tra chính tả — Roc phân biệt hoa/thường |
| "Tính sai số trạng thái" | Nhầm cộng vs nhân | Record → **nhân**, Tag union → **cộng** |
| "Không biết dùng record hay tag union" | Chưa rõ quan hệ VÀ hay HOẶC | Cần **tất cả** → record. Chọn **một** → tag union |

---

## Tóm tắt

- ✅ **Lambda Calculus** = mọi tính toán đều là function. `\x -> x + 1` trong Roc = `λx. x + 1`. Gọi function = β-reduction.
- ✅ **Curry-Howard** = Types là lời hứa. Compiler kiểm tra types = kiểm tra logic. `Result ok err` hứa rõ ràng "thành công hoặc lỗi".
- ✅ **Records** = VÀ = nhân số trạng thái. **Tag unions** = HOẶC = cộng số trạng thái.
- ✅ **Đếm trạng thái** = vũ khí chống bugs. Nếu tổng trạng thái > trạng thái hợp lệ → thiết kế cần sửa. Dùng tag union thay boolean flags.
- ✅ **Roc structural typing**: Không cần khai báo type — tags và records tự tồn tại. Compiler suy ra type từ cách bạn dùng.

## Tiếp theo

→ Chapter 2: **Algorithmic Thinking & Complexity** — bạn sẽ học Big-O (tại sao code này chạy nhanh hơn code kia), recursion (function gọi chính nó), và đặc biệt `List.walk` (fold) — cách Roc thay thế vòng lặp `for`.
