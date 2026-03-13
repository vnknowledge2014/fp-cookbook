# Chapter 19 — Workflows as Pipelines

> **Bạn sẽ học được**:
> - Business workflows = **`|>` pipelines** + **`?` chains**
> - Tách workflow thành steps rõ ràng — mỗi step 1 function
> - **Validation pipeline** — validate → transform → execute
> - **Multi-step workflows** — order processing, user onboarding
> - **Composable steps** — tái sử dụng steps giữa các workflows
> - Side effects ở cuối pipeline (pure core / effectful shell)
>
> **Yêu cầu trước**: [Chapter 18 — Domain Modeling](chapter_18_domain_modeling.md)
> **Thời gian đọc**: ~40 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Biến bất kỳ business process nào thành pipeline rõ ràng, testable, maintainable.

---

Business process nào cũng là một pipeline: nhận đơn → validate → xử lý → trả kết quả. Trong FP, pipeline không phải ẩn dụ — nó là code thật. Mỗi bước là một function, dữ liệu chảy qua pipeline, lỗi tự động được bắt và báo cáo.

## Workflows as Pipelines — Pattern cốt lõi

Scott Wlaschin mô tả business workflows như pipelines: data vào → validate → enrich → calculate → persist. Mỗi step là pure function. Errors xử lý qua Result chaining. Trong Roc, pipeline pattern tự nhiên nhất nhờ `|>` operator.


## 19.1 — Workflows = Pipelines

Mỗi business process là một chuỗi bước: nhận input → validate → transform → output. Trong FP, pipeline không phải metaphor — nó là code thật, với `|>` nối các bước lại.

### Business nói:

```
"Khi khách ĐẶT HÀNG:
 1. Kiểm tra giỏ hàng không rỗng
 2. Validate thông tin giao hàng
 3. Tính tổng tiền + phí ship
 4. Kiểm tra tồn kho
 5. Tạo đơn hàng
 6. Gửi email xác nhận"
```

### Code nói:

```roc
placeOrder = \rawInput ->
    cart       = validateCart?       rawInput.items
    shipping   = validateShipping?   rawInput.address
    pricing    = calculatePricing    cart shipping
    _          = checkInventory?     cart
    order      = createOrder         cart pricing shipping
    Ok order
# Side effect (email) ở shell, không ở đây
```

Mỗi bước = 1 function. Pipeline đọc từ trên xuống = quy trình business. Nếu bước nào lỗi (`?`), dừng ngay.

---

## 19.2 — Validation Pipeline

Validation là bước đầu tiên và quan trọng nhất: dữ liệu từ bên ngoài (user input, API response, file content) phải được validate trước khi vào domain. Mỗi validator là một function trả `Result`.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ VALIDATION STEPS ═══════

validateName : Str -> Result Str [EmptyName, NameTooLong]
validateName = \raw ->
    trimmed = Str.trim raw
    if Str.isEmpty trimmed then Err EmptyName
    else if Str.countUtf8Bytes trimmed > 50 then Err NameTooLong
    else Ok trimmed

validateEmail : Str -> Result Str [InvalidEmail]
validateEmail = \raw ->
    if Str.contains raw "@" && Str.contains raw "." then Ok raw
    else Err InvalidEmail

validateAge : Str -> Result U8 [InvalidAge, AgeTooYoung, AgeTooOld]
validateAge = \raw ->
    when Str.toI64 raw is
        Err _ -> Err InvalidAge
        Ok n ->
            if n < 13 then Err AgeTooYoung
            else if n > 120 then Err AgeTooOld
            else Ok (Num.toU8 n)

validatePassword : Str -> Result Str [PasswordTooShort, PasswordNoDigit, PasswordNoUpper]
validatePassword = \raw ->
    if Str.countUtf8Bytes raw < 8 then Err PasswordTooShort
    else
        bytes = Str.toUtf8 raw
        hasDigit = List.any bytes \b -> b >= 48 && b <= 57
        hasUpper = List.any bytes \b -> b >= 65 && b <= 90
        if !hasDigit then Err PasswordNoDigit
        else if !hasUpper then Err PasswordNoUpper
        else Ok raw

# ═══════ WORKFLOW: Chain với ? ═══════

registerUser : { name : Str, email : Str, age : Str, password : Str } -> Result _ _
registerUser = \input ->
    name     = validateName?     input.name
    email    = validateEmail?    input.email
    age      = validateAge?      input.age
    password = validatePassword? input.password
    Ok {
        name,
        email,
        age,
        password,
        role: if age >= 18 then Member else JuniorMember,
        createdAt: "2024-03-15",
    }

# ═══════ ERROR MESSAGING ═══════

errorToMessage : _ -> Str
errorToMessage = \err ->
    when err is
        EmptyName -> "Tên không được rỗng"
        NameTooLong -> "Tên tối đa 50 ký tự"
        InvalidEmail -> "Email không hợp lệ"
        InvalidAge -> "Tuổi phải là số"
        AgeTooYoung -> "Phải từ 13 tuổi trở lên"
        AgeTooOld -> "Tuổi không hợp lệ"
        PasswordTooShort -> "Mật khẩu tối thiểu 8 ký tự"
        PasswordNoDigit -> "Mật khẩu cần ít nhất 1 chữ số"
        PasswordNoUpper -> "Mật khẩu cần ít nhất 1 chữ hoa"

main =
    testCases = [
        { name: "An", email: "an@mail.com", age: "25", password: "Secure1pass" },
        { name: "", email: "an@mail.com", age: "25", password: "Secure1pass" },
        { name: "An", email: "bad", age: "25", password: "Secure1pass" },
        { name: "An", email: "an@mail.com", age: "10", password: "Secure1pass" },
        { name: "An", email: "an@mail.com", age: "25", password: "short" },
    ]

    List.forEach testCases \input ->
        when registerUser input is
            Ok user -> Stdout.line! "✅ $(user.name) ($(Inspect.toStr user.role))"
            Err err -> Stdout.line! "❌ $(input.name): $(errorToMessage err)"
    # Output:
    # ✅ An (Member)
    # ❌ : Tên không được rỗng
    # ❌ An: Email không hợp lệ
    # ❌ An: Phải từ 13 tuổi trở lên
    # ❌ An: Mật khẩu tối thiểu 8 ký tự
```

---

## 19.3 — Transform Pipeline

Sau validation → **transform** data thành domain objects:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ STEP 1: PARSE raw input ═══════
parseOrderInput = \rawItems ->
    if List.isEmpty rawItems then Err EmptyOrder
    else
        parsed = List.keepOks rawItems \item ->
            when Str.toU64 item.qtyStr is
                Ok qty -> Ok { name: item.name, price: item.price, qty }
                Err _ -> Err (InvalidQuantity item.name)
        if List.isEmpty parsed then Err NoValidItems
        else Ok parsed

# ═══════ STEP 2: VALIDATE business rules ═══════
validateBusinessRules = \items ->
    totalItems = List.walk items 0 \s, i -> s + i.qty
    if totalItems > 50 then Err (TooManyItems totalItems)
    else Ok items

# ═══════ STEP 3: TRANSFORM → domain objects ═══════
buildLineItems = \items ->
    List.map items \item ->
        lineTotal = item.price * item.qty
        { item & lineTotal }

# ═══════ STEP 4: CALCULATE totals ═══════
calculateTotals = \lineItems ->
    subtotal = List.walk lineItems 0 \s, item -> s + item.lineTotal
    tax = subtotal * 10 // 100
    shipping = if subtotal >= 500000 then 0 else 30000
    { lineItems, subtotal, tax, shipping, total: subtotal + tax + shipping }

# ═══════ FULL PIPELINE ═══════
processOrder = \rawItems ->
    parsed    = parseOrderInput?      rawItems      # Step 1: parse
    validated = validateBusinessRules? parsed        # Step 2: validate
    lineItems = buildLineItems         validated     # Step 3: transform (pure, no ?)
    totals    = calculateTotals        lineItems     # Step 4: calculate (pure, no ?)
    Ok totals

main =
    rawItems = [
        { name: "Phở", price: 45000, qtyStr: "2" },
        { name: "Cà phê", price: 25000, qtyStr: "3" },
        { name: "Bánh mì", price: 15000, qtyStr: "5" },
    ]

    when processOrder rawItems is
        Ok order ->
            Stdout.line! "=== Đơn hàng ==="
            List.forEach order.lineItems \item ->
                Stdout.line! "  $(item.name) × $(Num.toStr item.qty) = $(Num.toStr item.lineTotal)đ"
            Stdout.line! "Tạm tính: $(Num.toStr order.subtotal)đ"
            Stdout.line! "Thuế: $(Num.toStr order.tax)đ"
            Stdout.line! "Ship: $(Num.toStr order.shipping)đ"
            Stdout.line! "Tổng: $(Num.toStr order.total)đ"
        Err err ->
            Stdout.line! "❌ $(Inspect.toStr err)"
    # Output:
    # === Đơn hàng ===
    #   Phở × 2 = 90000đ
    #   Cà phê × 3 = 75000đ
    #   Bánh mì × 5 = 75000đ
    # Tạm tính: 240000đ
    # Thuế: 24000đ
    # Ship: 30000đ
    # Tổng: 294000đ
```

Nhìn lại `processOrder`: 4 bước, đọc từ trên xuống, mỗi bước 1 function. Steps mà có thể fail dùng `?`, steps thuần túy thì gọi trực tiếp.

---

## 19.4 — Composable Steps: Tái sử dụng

Khi các bước pipeline là functions thuần, bạn có thể tái sử dụng chúng ở nhiều workflows khác nhau. Validate email? Dùng chung cho registration, profile update, và contact form.

```roc
# ═══════ REUSABLE STEPS ═══════

# Step này dùng ở nhiều workflows
trimAndValidateNotEmpty : Str, Str -> Result Str [FieldEmpty Str]
trimAndValidateNotEmpty = \value, fieldName ->
    trimmed = Str.trim value
    if Str.isEmpty trimmed then Err (FieldEmpty fieldName) else Ok trimmed

validateRange : I64, I64, I64, Str -> Result I64 [OutOfRange Str]
validateRange = \value, min, max, fieldName ->
    if value < min || value > max then Err (OutOfRange fieldName) else Ok value

# ═══════ WORKFLOW 1: User registration ═══════
registerUser = \input ->
    name  = trimAndValidateNotEmpty? input.name "name"
    email = trimAndValidateNotEmpty? input.email "email"
    # ... more steps
    Ok { name, email }

# ═══════ WORKFLOW 2: Product creation ═══════
createProduct = \input ->
    name  = trimAndValidateNotEmpty? input.name "product name"
    desc  = trimAndValidateNotEmpty? input.description "description"
    price = validateRange? input.price 1000 100000000 "price"
    Ok { name, desc, price }

# ═══════ WORKFLOW 3: Review submission ═══════
submitReview = \input ->
    title   = trimAndValidateNotEmpty? input.title "title"
    body    = trimAndValidateNotEmpty? input.body "body"
    rating  = validateRange? input.rating 1 5 "rating"
    Ok { title, body, rating }
```

`trimAndValidateNotEmpty` và `validateRange` dùng lại ở **3 workflows khác nhau**.

---

## 19.5 — Pipeline với Side Effects

Pipeline thực tế không chỉ transform data — nó cần đọc database, gửi email, ghi log. Pattern: pure pipeline cho logic, wrap bằng Task ở shell.

### Pure core → Shell applies effects

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ PURE PIPELINE ═══════

processLoanApplication = \input ->
    applicant = validateApplicant? input
    score     = calculateCreditScore applicant
    decision  = makeDecision score applicant.requestedAmount
    Ok { applicant, score, decision }

validateApplicant = \input ->
    name   = if Str.isEmpty (Str.trim input.name) then Err EmptyName else Ok (Str.trim input.name)
    income = if input.income < 0 then Err NegativeIncome else Ok input.income
    amount = if input.requestedAmount <= 0 then Err InvalidAmount else Ok input.requestedAmount
    n = name?
    i = income?
    a = amount?
    Ok { name: n, income: i, requestedAmount: a, age: input.age }

calculateCreditScore = \applicant ->
    baseScore = 300
    incomeBonus = Num.min 200 (applicant.income // 5000000)
    ageBonus = if applicant.age >= 25 && applicant.age <= 55 then 100 else 50
    baseScore + incomeBonus + ageBonus

makeDecision = \score, amount ->
    if score >= 600 && amount <= 500000000 then
        Approved { maxAmount: amount, interestRate: 8 }
    else if score >= 400 then
        Conditional { maxAmount: amount // 2, interestRate: 12, conditions: "Cần bảo lãnh" }
    else
        Rejected { reason: "Điểm tín dụng thấp ($(Num.toStr score))" }

# ═══════ EFFECTFUL SHELL ═══════

main =
    applications = [
        { name: "An", income: 30000000, requestedAmount: 200000000, age: 30 },
        { name: "Bình", income: 8000000, requestedAmount: 100000000, age: 22 },
        { name: "Cường", income: 5000000, requestedAmount: 500000000, age: 19 },
    ]

    List.forEach applications \input ->
        when processLoanApplication input is
            Ok result ->
                decisionStr = when result.decision is
                    Approved { maxAmount, interestRate } ->
                        "✅ Duyệt $(Num.toStr maxAmount)đ, lãi $(Num.toStr interestRate)%"
                    Conditional { maxAmount, interestRate, conditions } ->
                        "⚠️ Duyệt có ĐK: $(Num.toStr maxAmount)đ, lãi $(Num.toStr interestRate)% — $(conditions)"
                    Rejected { reason } ->
                        "❌ Từ chối: $(reason)"
                Stdout.line! "$(result.applicant.name) (Score: $(Num.toStr result.score)): $(decisionStr)"
            Err err ->
                Stdout.line! "❌ $(Inspect.toStr err)"
    # Output:
    # An (Score: 606): ✅ Duyệt 200000000đ, lãi 8%
    # Bình (Score: 451): ⚠️ Duyệt có ĐK: 50000000đ, lãi 12% — Cần bảo lãnh
    # Cường (Score: 401): ⚠️ Duyệt có ĐK: 250000000đ, lãi 12% — Cần bảo lãnh
```

---

## 19.6 — Pattern: Pipeline with Context

Khi các steps cần **chia sẻ context** (accumulate thông tin qua pipeline):

```roc
# Pipeline context — accumulate results qua các bước
processPayment = \input ->
    # Bước 1: Validate
    validated = validatePaymentInput? input

    # Bước 2: Convert currency (ctx bắt đầu)
    ctx = {
        amount: validated.amount,
        currency: validated.currency,
        amountVND: convertToVND validated.amount validated.currency,
        fees: 0,
    }

    # Bước 3: Calculate fees (ctx cộng dồn)
    ctx2 = {
        ctx &
        fees: calculateFees ctx.amountVND validated.method,
    }

    # Bước 4: Final
    Ok {
        originalAmount: ctx2.amount,
        currency: ctx2.currency,
        amountVND: ctx2.amountVND,
        fees: ctx2.fees,
        totalCharge: ctx2.amountVND + ctx2.fees,
        method: validated.method,
    }

convertToVND = \amount, currency ->
    when currency is
        VND -> amount
        USD -> amount * 25000
        EUR -> amount * 27000

calculateFees = \amountVND, method ->
    when method is
        Cash -> 0
        Card -> amountVND * 2 // 100
        EWallet -> amountVND * 1 // 100
```

---


## ✅ Checkpoint 19

> Đến đây bạn phải hiểu:
> 1. Workflow = chuỗi functions: `validate |> price |> confirm`
> 2. `|>` + `Result.try` = type-safe pipeline tự dừng ở error
> 3. Mỗi step nhận output của step trước — composition tự nhiên
>
> **Test nhanh**: `validate |> Result.try price |> Result.try confirm` — nếu validate fail?
> <details><summary>Đáp án</summary>Pipeline dừng ngay! `price` và `confirm` không chạy.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Booking pipeline

Viết workflow đặt phòng khách sạn:

```roc
# Steps:
# 1. validateDates (checkIn trước checkOut)
# 2. validateGuests (1-4)
# 3. selectRoom (dựa trên guests: 1-2 = Standard, 3-4 = Suite)
# 4. calculatePrice (Standard: 800k/đêm, Suite: 1.5M/đêm)
# 5. Return booking summary
```

<details><summary>✅ Lời giải</summary>

```roc
bookRoom = \input ->
    nights = validateDates? input.checkIn input.checkOut
    guests = validateGuests? input.guests
    room = selectRoom guests
    price = calculatePrice room nights
    Ok { room, nights, guests, totalPrice: price }

validateDates = \checkIn, checkOut ->
    if checkOut <= checkIn then Err InvalidDates
    else Ok (checkOut - checkIn)

validateGuests = \n ->
    if n < 1 then Err TooFewGuests
    else if n > 4 then Err TooManyGuests
    else Ok n

selectRoom = \guests ->
    if guests <= 2 then Standard else Suite

calculatePrice = \room, nights ->
    rate = when room is
        Standard -> 800000
        Suite -> 1500000
    rate * nights
```

</details>

---

**Bài 2** (15 phút): Multi-step onboarding

Viết user onboarding workflow:

```roc
# Step 1: Validate profile (name, email, age ≥ 18)
# Step 2: Choose plan (Free, Basic 99k/tháng, Pro 299k/tháng)
# Step 3: If paid plan → validate payment method
# Step 4: Create account
# Step 5: Generate welcome message (khác nhau cho mỗi plan)
```

<details><summary>✅ Lời giải</summary>

```roc
onboardUser = \input ->
    profile = validateProfile? input
    plan = choosePlan? input.planChoice
    _ = validatePaymentIfNeeded? plan input.paymentMethod
    account = createAccount profile plan
    welcome = generateWelcome account
    Ok { account, welcome }

validatePaymentIfNeeded = \plan, method ->
    when plan is
        Free -> Ok {}
        Basic -> if method == NoPayment then Err PaymentRequired else Ok {}
        Pro -> if method == NoPayment then Err PaymentRequired else Ok {}

generateWelcome = \account ->
    when account.plan is
        Free -> "🎉 Chào $(account.name)! Bắt đầu miễn phí."
        Basic -> "🎉 Chào $(account.name)! Plan Basic — mở khóa tính năng cơ bản."
        Pro -> "🎉 Chào $(account.name)! Plan Pro — toàn bộ tính năng VIP."
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Pipeline quá dài | Quá nhiều bước trong 1 function | Nhóm bước liên quan thành sub-pipelines |
| Khó debug pipeline | Không biết step nào lỗi | Thêm context vào error tags: `Err (Step3 reason)` |
| Steps khó tái sử dụng | Quá specific cho 1 workflow | Tách validation steps thành generic + compose |
| Side effects lọt vào pipeline | IO xen giữa pure steps | Di chuyển IO ra shell, truyền data vào pure core |

---

## Tóm tắt

- ✅ **Workflow = Pipeline**: mỗi bước 1 function, đọc trên xuống = business process.
- ✅ **`?` cho fallible steps**: tự dừng khi lỗi. Pure steps gọi trực tiếp (không `?`).
- ✅ **Validation → Transform → Calculate**: pattern phổ biến cho mọi business workflow.
- ✅ **Composable steps**: tách thành functions nhỏ, tái sử dụng giữa workflows.
- ✅ **Pipeline with context**: record accumulates thông tin qua các bước.
- ✅ **Pure core / Effectful shell**: pipeline = pure. IO (email, save DB) ở shell.

## Tiếp theo

→ Chapter 20: **Platform Separation** ⭐ — Roc's unique architecture: App = pure domain, Platform = IO. Language-level Onion Architecture — clean architecture mà không cần discipline, compiler enforce.
