# Chapter 18 — Domain Modeling

> **Bạn sẽ học được**:
> - Biến domain analysis thành code Roc — **step by step**
> - Opaque types = **Value Objects**
> - Tag unions = **State machines** và **Domain events**
> - Records = **Aggregates**
> - Modules = **Bounded Contexts**
> - Xây hệ thống thực tế từ phân tích domain đến code
>
> **Yêu cầu trước**: [Chapter 17 — Introduction to DDD](chapter_17_introduction_to_ddd.md)
> **Thời gian đọc**: ~50 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Xây dựng domain model hoàn chỉnh cho bất kỳ bài toán nào — từ phân tích đến code.

---

Một domain model tốt biến business rules thành compiler errors. Đơn hàng chưa thanh toán không thể giao? Type system bắt lỗi đó trước khi code chạy. Chapter này đi từ phân tích domain đến viết types — quy trình bạn sẽ dùng cho mọi dự án thực tế.

## 18.1 — Quy trình: Domain → Types → Code

Quy trình 3 bước: (1) phân tích domain với domain expert, (2) chuyển concepts thành types, (3) viết business logic dưới dạng pure functions trên types đó.

### 4 bước

```
1. PHÂN TÍCH  → Ubiquitous Language, danh sách concepts
2. THIẾT KẾ   → Mapping: concept → Roc type
3. CODE       → Modules, opaque types, functions
4. VALIDATE   → Tests, review với domain expert
```

### Mapping table

| Domain concept | Roc construct | Ví dụ |
|---------------|---------------|-------|
| Giá trị đơn giản | Opaque type | `Money := U64` |
| Giá trị phức hợp | Opaque type + record | `Address := { ... }` |
| Danh sách cố định | Tag union | `[S, M, L, XL]` |
| Trạng thái | Tag union | `[Draft, Active, Closed]` |
| Sự kiện | Tag union + payload | `[OrderPlaced { ... }]` |
| Đối tượng có ID | Opaque type + ID field | `Customer := { id, ... }` |
| Nhóm liên quan | Record (aggregate) | `Order := { items, total, ... }` |
| Ranh giới | Module | `Catalog/Product.roc` |
| Business rule | Function | `placeOrder : Cart -> Result Order Error` |
| Workflow | Pipeline `\|>` | `validate \|> price \|> ship` |

---

## 18.2 — Case Study: Hệ thống đặt bàn nhà hàng

Making illegal states unrepresentable — thiết kế types sao cho dữ liệu sai **không thể tồn tại**. Email không có @ → type system từ chối, không cần validate ở runtime.

### Bước 1: Phân tích domain

```
Cuộc trò chuyện với chủ nhà hàng:

"Khách gọi ĐẶT BÀN, cho biết SỐ KHÁCH, NGÀY GIỜ, và TÊN.
 Nhân viên kiểm tra LỊCH CÓ TRỐNG không.
 Nếu trống thì XÁC NHẬN, gửi mã đặt bàn.
 Khách có thể HỦY trước 2 tiếng.
 Khi khách đến, nhân viên CHECK-IN.
 Nếu khách không đến trong 30 phút → TỪ CHỐI (no-show)."
```

Ubiquitous Language rút ra:
- **Reservation** (đặt bàn)
- **Guest** (khách) — tên + số điện thoại
- **Party size** (số khách)
- **Time slot** (khung giờ)
- **Confirm, Cancel, Check-in, No-show** (trạng thái)

### Bước 2: Thiết kế types

```roc
# Value Objects
ReservationId := U64
GuestName := Str           # validate: không rỗng
PhoneNumber := Str          # validate: bắt đầu "0", 10 ký tự
PartySize := U8             # validate: 1-20
TimeSlot := { hour : U8, minute : U8 }  # validate: 10:00-22:00

# State Machine (tag union)
ReservationStatus : [
    Pending,                # chờ xác nhận
    Confirmed,              # đã xác nhận
    CheckedIn,              # khách đã đến
    Cancelled { reason : Str },  # đã hủy
    NoShow,                 # khách không đến
]

# Entity (aggregate)
Reservation := {
    id : ReservationId,
    guest : { name : GuestName, phone : PhoneNumber },
    partySize : PartySize,
    timeSlot : TimeSlot,
    date : Str,
    status : ReservationStatus,
}
```

### Bước 3: Code

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ VALUE OBJECTS ═══════

validatePartySize : U8 -> Result U8 [TooFew, TooMany]
validatePartySize = \n ->
    if n < 1 then Err TooFew
    else if n > 20 then Err TooMany
    else Ok n

validateTimeSlot : U8, U8 -> Result { hour : U8, minute : U8 } [TooEarly, TooLate, InvalidMinute]
validateTimeSlot = \hour, minute ->
    if minute >= 60 then Err InvalidMinute
    else if hour < 10 then Err TooEarly
    else if hour >= 22 then Err TooLate
    else Ok { hour, minute }

validateGuestName : Str -> Result Str [EmptyName]
validateGuestName = \name ->
    trimmed = Str.trim name
    if Str.isEmpty trimmed then Err EmptyName else Ok trimmed

# ═══════ STATE MACHINE ═══════

# Mỗi transition = 1 function, chỉ hoạt động ở trạng thái đúng

confirmReservation = \reservation ->
    when reservation.status is
        Pending -> Ok { reservation & status: Confirmed }
        _ -> Err (InvalidTransition "Chỉ có thể xác nhận đặt bàn đang chờ")

checkIn = \reservation ->
    when reservation.status is
        Confirmed -> Ok { reservation & status: CheckedIn }
        _ -> Err (InvalidTransition "Chỉ check-in khi đã xác nhận")

cancelReservation = \reservation, reason ->
    when reservation.status is
        Pending -> Ok { reservation & status: Cancelled { reason } }
        Confirmed -> Ok { reservation & status: Cancelled { reason } }
        _ -> Err (InvalidTransition "Không thể hủy đặt bàn ở trạng thái này")

markNoShow = \reservation ->
    when reservation.status is
        Confirmed -> Ok { reservation & status: NoShow }
        _ -> Err (InvalidTransition "Chỉ đánh dấu no-show khi đã xác nhận")

# ═══════ WORKFLOW ═══════

createReservation = \rawName, rawPartySize, rawHour, rawMinute, date ->
    name = validateGuestName? rawName
    partySize = validatePartySize? rawPartySize
    timeSlot = validateTimeSlot? rawHour rawMinute
    Ok {
        id: 1,
        guest: { name, phone: "0901234567" },
        partySize,
        timeSlot,
        date,
        status: Pending,
    }

# ═══════ DISPLAY ═══════

statusIcon = \status ->
    when status is
        Pending -> "⏳"
        Confirmed -> "✅"
        CheckedIn -> "🎉"
        Cancelled _ -> "❌"
        NoShow -> "👻"

describeReservation = \r ->
    time = "$(Num.toStr r.timeSlot.hour):$(if r.timeSlot.minute < 10 then "0" else "")$(Num.toStr r.timeSlot.minute)"
    "$(statusIcon r.status) #$(Num.toStr r.id) | $(r.guest.name) | $(Num.toStr r.partySize) khách | $(r.date) $(time)"

main =
    # Happy path: Tạo → Xác nhận → Check-in
    when createReservation "Nguyễn An" 4 19 30 "2024-03-15" is
        Err err -> Stdout.line! "❌ Tạo thất bại: $(Inspect.toStr err)"
        Ok reservation ->
            Stdout.line! (describeReservation reservation)

            when confirmReservation reservation is
                Err _ -> Stdout.line! "❌ Xác nhận thất bại"
                Ok confirmed ->
                    Stdout.line! (describeReservation confirmed)

                    when checkIn confirmed is
                        Err _ -> Stdout.line! "❌ Check-in thất bại"
                        Ok checkedIn ->
                            Stdout.line! (describeReservation checkedIn)

    # Hủy
    Stdout.line! "\n=== Hủy đặt bàn ==="
    when createReservation "Trần Bình" 2 20 0 "2024-03-15" is
        Ok r ->
            Stdout.line! (describeReservation r)
            when cancelReservation r "Thay đổi kế hoạch" is
                Ok cancelled -> Stdout.line! (describeReservation cancelled)
                Err _ -> Stdout.line! "❌"
        Err _ -> Stdout.line! "❌"

    # Validation failures
    Stdout.line! "\n=== Validation ==="
    when createReservation "" 4 19 30 "2024-03-15" is
        Err EmptyName -> Stdout.line! "❌ Tên rỗng"
        _ -> {}
    when createReservation "An" 25 19 30 "2024-03-15" is
        Err TooMany -> Stdout.line! "❌ Quá 20 khách"
        _ -> {}
    when createReservation "An" 4 8 0 "2024-03-15" is
        Err TooEarly -> Stdout.line! "❌ Trước 10h"
        _ -> {}
    # Output:
    # ⏳ #1 | Nguyễn An | 4 khách | 2024-03-15 19:30
    # ✅ #1 | Nguyễn An | 4 khách | 2024-03-15 19:30
    # 🎉 #1 | Nguyễn An | 4 khách | 2024-03-15 19:30
    #
    # === Hủy đặt bàn ===
    # ⏳ #1 | Trần Bình | 2 khách | 2024-03-15 20:00
    # ❌ #1 | Trần Bình | 2 khách | 2024-03-15 20:00
    #
    # === Validation ===
    # ❌ Tên rỗng
    # ❌ Quá 20 khách
    # ❌ Trước 10h
```

---

## 18.3 — State Machine Patterns

State machines biến quy trình business thành code an toàn. Đơn hàng chỉ có thể đi từ Pending → Confirmed → Shipped — compiler bắt lỗi nếu bạn cố ship đơn chưa confirm.

### Pattern 1: Tag union cho trạng thái

Đã thấy ở trên — mỗi status là 1 tag, mỗi transition là 1 function.

### Pattern 2: Mỗi trạng thái mang data khác nhau

```roc
# Mỗi state chứa CHÍNH XÁC data phù hợp
# → Không có field "null" hay "optional vô nghĩa"

# Draft — chỉ có items
# Submitted — thêm submittedAt
# Paid — thêm paymentMethod, paidAt
# Shipped — thêm trackingCode, shippedAt

describeOrder = \order ->
    when order is
        Draft { items } ->
            "📝 Nháp: $(Num.toStr (List.len items)) món"
        Submitted { items, submittedAt } ->
            "📨 Đã gửi lúc $(submittedAt)"
        Paid { items, paymentMethod, paidAt } ->
            "💰 Thanh toán qua $(paymentMethod) lúc $(paidAt)"
        Shipped { trackingCode, shippedAt, .. } ->
            "🚚 Đã giao — mã: $(trackingCode)"
        Delivered { deliveredAt, .. } ->
            "✅ Nhận hàng lúc $(deliveredAt)"

submitOrder = \order ->
    when order is
        Draft { items } ->
            if List.isEmpty items then Err EmptyOrder
            else Ok (Submitted { items, submittedAt: "2024-03-15 10:00" })
        _ -> Err (InvalidTransition "Chỉ draft mới submit được")

payOrder = \order, method ->
    when order is
        Submitted { items, submittedAt } ->
            Ok (Paid { items, submittedAt, paymentMethod: method, paidAt: "2024-03-15 10:05" })
        _ -> Err (InvalidTransition "Phải submit trước khi thanh toán")
```

**Ưu điểm**: Mỗi state biết chính xác data nó có. Không bao giờ `order.trackingCode` trên đơn chưa ship → **compiler error, không phải runtime bug**.

### Pattern 3: Transition table

```roc
# Tất cả transitions hợp lệ, dễ đọc
validTransitions = \from, to ->
    when (from, to) is
        (Pending, Confirmed) -> Bool.true
        (Pending, Cancelled _) -> Bool.true
        (Confirmed, CheckedIn) -> Bool.true
        (Confirmed, Cancelled _) -> Bool.true
        (Confirmed, NoShow) -> Bool.true
        _ -> Bool.false
```

---

## 18.4 — Event-Sourced Domain Model

Kết hợp DDD + Event Sourcing (Chapter 16):

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Domain Events — PAST TENSE
# Command: "PlaceOrder" → Event: "OrderPlaced"

applyEvent = \state, event ->
    when event is
        AccountOpened { name, initialDeposit } ->
            { state & name, balance: initialDeposit, status: Active, transactionCount: 0 }
        MoneyDeposited { amount, description: _ } ->
            { state & balance: state.balance + amount, transactionCount: state.transactionCount + 1 }
        MoneyWithdrawn { amount, description: _ } ->
            { state & balance: state.balance - amount, transactionCount: state.transactionCount + 1 }
        AccountFrozen { reason: _ } ->
            { state & status: Frozen }
        AccountUnfrozen ->
            { state & status: Active }
        AccountClosed ->
            { state & status: Closed }

# Commands → validate → produce Events
processCommand = \state, command ->
    when command is
        Deposit { amount, description } ->
            if state.status != Active then Err AccountNotActive
            else if amount <= 0 then Err InvalidAmount
            else Ok (MoneyDeposited { amount, description })
        Withdraw { amount, description } ->
            if state.status != Active then Err AccountNotActive
            else if amount <= 0 then Err InvalidAmount
            else if amount > state.balance then Err InsufficientFunds
            else Ok (MoneyWithdrawn { amount, description })
        Freeze { reason } ->
            if state.status != Active then Err AccountNotActive
            else Ok (AccountFrozen { reason })
        Close ->
            if state.balance != 0 then Err BalanceNotZero
            else Ok AccountClosed

emptyState = { name: "", balance: 0, status: Inactive, transactionCount: 0 }

main =
    events = [
        AccountOpened { name: "Nguyễn An", initialDeposit: 500000 },
        MoneyDeposited { amount: 1000000, description: "Lương tháng 3" },
        MoneyWithdrawn { amount: 200000, description: "Tiền ăn" },
        MoneyDeposited { amount: 50000, description: "Cashback" },
        MoneyWithdrawn { amount: 100000, description: "Đổ xăng" },
    ]

    account = List.walk events emptyState applyEvent

    Stdout.line! "=== $(account.name) ==="
    Stdout.line! "Số dư: $(Num.toStr account.balance)đ"
    Stdout.line! "Giao dịch: $(Num.toStr account.transactionCount)"
    Stdout.line! "Trạng thái: $(Inspect.toStr account.status)"

    # Test command validation
    Stdout.line! "\n=== Test Commands ==="

    when processCommand account (Withdraw { amount: 500000, description: "Mua sắm" }) is
        Ok event -> Stdout.line! "✅ $(Inspect.toStr event)"
        Err err -> Stdout.line! "❌ $(Inspect.toStr err)"

    when processCommand account (Withdraw { amount: 9999999, description: "Rút hết" }) is
        Ok _ -> Stdout.line! "✅ OK"
        Err InsufficientFunds -> Stdout.line! "❌ Không đủ tiền"
        Err _ -> Stdout.line! "❌ Lỗi khác"

    # Freeze → thử giao dịch
    frozenAccount = applyEvent account (AccountFrozen { reason: "Nghi ngờ gian lận" })
    when processCommand frozenAccount (Deposit { amount: 100, description: "Test" }) is
        Err AccountNotActive -> Stdout.line! "❌ Tài khoản bị đóng băng — không giao dịch được"
        _ -> {}
```

---

## 18.5 — Checklist: Domain Model đúng?

Trước khi code, kiểm tra:

### ✅ Checklist

- [ ] **Ubiquitous Language**: Tên types/functions = thuật ngữ domain?
- [ ] **Value Objects**: Giá trị quan trọng có opaque type riêng? (Money, Email, PhoneNumber)
- [ ] **Smart Constructors**: Mọi value object validate khi tạo?
- [ ] **State Machine**: Trạng thái dùng tag union? Transitions explicit?
- [ ] **Invalid states impossible**: Có thể tạo data vô nghĩa không?
- [ ] **Aggregate invariants**: Data luôn nhất quán? (total = sum of items)
- [ ] **Bounded Contexts**: Modules tách biệt? Không share internal types?
- [ ] **Events past tense**: `OrderPlaced` không phải `PlaceOrder`?
- [ ] **Testable**: Core logic pure? Dùng `expect` test được?

---


## ✅ Checkpoint 18

> Đến đây bạn phải hiểu:
> 1. **Opaque types** = domain values: `Email := Str`, `OrderId := U64`
> 2. **Smart constructors** = validate khi tạo, luôn valid sau đó
> 3. **Tag unions** = state machines: `[Draft, Confirmed, Shipped, Delivered]`
>
> **Test nhanh**: `Email := Str` — tại sao không dùng `Str` trực tiếp?
> <details><summary>Đáp án</summary>Opaque type ngăn truyền nhầm. Smart constructor đảm bảo "nếu có Email thì nó valid".</details>

---

## 🏋️ Bài tập

**Bài 1** (15 phút): Library domain model

Phân tích và model hệ thống thư viện:

```
Concepts: Book, Member, Loan
States: Available, OnLoan, Reserved, Lost
Transitions: borrow, return, reserve, reportLost
Rules: Max 5 sách/member, loan 14 ngày, chỉ borrow khi Available
```

<details><summary>✅ Lời giải</summary>

```roc
# Value Objects
BookId := U64 implements [Eq, Hash, Inspect]
MemberId := U64 implements [Eq, Hash, Inspect]
ISBN := Str implements [Eq, Hash, Inspect]

# State Machine
BookStatus : [
    Available,
    OnLoan { memberId : U64, dueDate : Str },
    Reserved { memberId : U64 },
    Lost,
]

# Transitions
borrowBook = \book, memberId, dueDate ->
    when book.status is
        Available -> Ok { book & status: OnLoan { memberId, dueDate } }
        Reserved { memberId: reservedBy } ->
            if reservedBy == memberId then Ok { book & status: OnLoan { memberId, dueDate } }
            else Err ReservedByOther
        OnLoan _ -> Err AlreadyOnLoan
        Lost -> Err BookLost

returnBook = \book ->
    when book.status is
        OnLoan _ -> Ok { book & status: Available }
        _ -> Err NotOnLoan

# Simplified: thực tế nên lưu cả OnLoan info bên cạnh reservation
reserveBook = \book, memberId ->
    when book.status is
        OnLoan _ -> Ok { book & status: Reserved { memberId } }
        _ -> Err CannotReserve

reportLost = \book ->
    when book.status is
        OnLoan _ -> Ok { book & status: Lost }
        _ -> Err NotOnLoan
```

</details>

---

**Bài 2** (20 phút): E-commerce domain model

Model đầy đủ cho hệ thống e-commerce nhỏ:

```
Bounded Contexts:
1. Catalog: Product (id, name, price, category)
2. Cart: CartItem, ShoppingCart (add, remove, updateQty)
3. Ordering: Order state machine (Draft→Placed→Paid→Shipped→Delivered)

Requirements:
- Cart max 20 items
- Order phải có ít nhất 1 item
- Chỉ ship khi đã paid
```

<details><summary>💡 Gợi ý</summary>

Tạo 3 modules riêng biệt. Dùng opaque types cho ProductId, OrderId, Money. State machine bằng tag union cho Order status. Smart constructors validate mọi thứ.

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Model quá phức tạp | Cố model mọi thứ cùng lúc | Bắt đầu từ core domain, thêm dần |
| State transitions lặp code | Copy-paste when...is | Extract helper functions cho shared logic |
| Domain expert không hiểu code | Dùng thuật ngữ kỹ thuật | Review code cùng domain expert, dùng TỪ CỦA HỌ |
| Quá nhiều opaque types | Mọi string đều wrap | Chỉ wrap values có business rules (validate, compare khác, combine) |

---

## Tóm tắt

- ✅ **Quy trình**: Phân tích → Mapping → Code → Validate. Bảng mapping concept → Roc construct.
- ✅ **Value Objects** = opaque types + smart constructors. Validate khi tạo, immutable, compare by value.
- ✅ **State Machines** = tag unions. Transitions = functions. Compiler enforce valid transitions.
- ✅ **Event Sourcing + DDD** = commands validate → produce events → events rebuild state.
- ✅ **Mỗi state chứa data riêng** — `Draft { items }` vs `Shipped { trackingCode }`. Không null fields.
- ✅ **Checklist** — 9 điểm kiểm tra trước khi ship domain model.

## Tiếp theo

→ Chapter 19: **Workflows as Pipelines** — biến business workflows thành `|>` pipelines + `Result.try` chains. Validate → Transform → Execute — mỗi bước rõ ràng, compose tự nhiên.
