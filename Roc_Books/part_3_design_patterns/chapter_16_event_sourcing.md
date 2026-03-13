# Chapter 16 — Event Sourcing

> **Bạn sẽ học được**:
> - **Event Sourcing** — lưu sự kiện thay vì trạng thái hiện tại
> - Events as **tag unions** — hoàn hảo cho Roc
> - `List.walk` = engine rebuild state từ events
> - **Projections** — nhiều "góc nhìn" từ cùng event stream
> - **Snapshots** — tối ưu hiệu năng khi event list lớn
> - So sánh Event Sourcing vs CRUD truyền thống
>
> **Yêu cầu trước**: [Chapter 15 — FP Patterns](chapter_15_fp_patterns.md)
> **Thời gian đọc**: ~40 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Xây dựng hệ thống event-sourced hoàn chỉnh — bank account, e-commerce cart, hoặc task tracker.

---

Hầu hết ứng dụng lưu trạng thái hiện tại: số dư tài khoản là 500k. Nhưng nếu có tranh chấp — "tôi đã nạp tiền mà sao không thấy?" — bạn không có bằng chứng. Event sourcing lưu **mọi sự kiện** từ đầu: nạp 200k, rút 100k, chuyển 50k. Số dư hiện tại = replay tất cả events. FP và event sourcing ăn khớp hoàn hảo — cả hai đều dựa trên immutable data.

## Event Sourcing — Lưu sự kiện, không lưu trạng thái

Thay vì lưu trạng thái hiện tại (`balance = 800k`), event sourcing lưu **chuỗi sự kiện** (`Deposit 1M → Withdraw 200k`). Trạng thái hiện tại được **tính lại** từ events. Lợi ích: full audit trail, time travel (xem state ở bất kỳ thời điểm), event replay cho debugging.

Roc's immutability và pure functions đặc biệt phù hợp cho event sourcing: mỗi event handler là pure function nhận state + event → trả state mới.


## 16.1 — CRUD vs Event Sourcing

CRUD ghi đè trạng thái cũ — `UPDATE balance SET amount = 500`. Event sourcing ghi lại sự kiện — `Deposited 200`, `Withdrew 100`. Sự khác biệt: CRUD mất lịch sử, event sourcing giữ tất cả.

### CRUD: Lưu trạng thái hiện tại

```
# Truyền thống — chỉ biết TẠI THỜI ĐIỂM NÀY
account = { balance: 750000 }

# Hỏi: "Tại sao balance là 750k?" → 🤷 KHÔNG BIẾT
# Hỏi: "Hôm qua balance là bao nhiêu?" → 🤷 KHÔNG BIẾT
```

### Event Sourcing: Lưu sự kiện

```
# Event Sourcing — biết MỌI THỨ đã xảy ra
events = [
    AccountCreated { owner: "An" },           # +0
    Deposited { amount: 1000000 },             # +1000000
    Withdrawn { amount: 200000 },              # -200000
    Deposited { amount: 50000 },               # +50000
    TransferSent { to: "Bình", amount: 100000 }, # -100000
]
# → Balance = 1000000 - 200000 + 50000 - 100000 = 750000 ✅

# Hỏi: "Tại sao 750k?" → replay events!
# Hỏi: "Hôm qua?" → replay đến sự kiện hôm qua!
```

> **💡 Ẩn dụ**: CRUD giống **ảnh chụp** — bạn thấy hiện tại, mất quá khứ. Event Sourcing giống **video** — xem lại bất kỳ khung hình nào.

### Tại sao Event Sourcing phù hợp FP?

1. **Events = tag unions** — type-safe, exhaustive
2. **Rebuild = `List.walk`** — fold events thành state
3. **Immutable** — events không bao giờ sửa/xóa
4. **Pure** — rebuild state là pure function, dễ test

---

## 16.2 — Ví dụ: Bank Account

Events là facts — "đã xảy ra, không thể thay đổi". Mỗi event là một tag union value, immutable, có timestamp. Domain logic quyết định event nào hợp lệ.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ EVENTS — tag unions ═══════

# Mỗi event = 1 sự kiện đã xảy ra (quá khứ, immutable)
# Tên viết ở THỜI QUÁ KHỨ: "Deposited" không phải "Deposit"

# ═══════ STATE — record ═══════

emptyAccount = { owner: "", balance: 0, isActive: Bool.false }

# ═══════ REDUCER — pure function ═══════

applyEvent = \state, event ->
    when event is
        AccountOpened { owner } ->
            { state & owner, isActive: Bool.true }
        Deposited { amount } ->
            { state & balance: state.balance + amount }
        Withdrawn { amount } ->
            { state & balance: state.balance - amount }
        TransferSent { amount, to: _ } ->
            { state & balance: state.balance - amount }
        TransferReceived { amount, from: _ } ->
            { state & balance: state.balance + amount }
        AccountClosed ->
            { state & isActive: Bool.false }

# Rebuild state = List.walk!
rebuildState = \events ->
    List.walk events emptyAccount applyEvent

# ═══════ VALIDATION — pure ═══════

validateCommand = \state, command ->
    when command is
        Withdraw { amount } ->
            if !(state.isActive) then Err AccountInactive
            else if amount > state.balance then Err InsufficientFunds
            else Ok (Withdrawn { amount })
        Deposit { amount } ->
            if !(state.isActive) then Err AccountInactive
            else if amount <= 0 then Err InvalidAmount
            else Ok (Deposited { amount })
        SendTransfer { amount, to } ->
            if !(state.isActive) then Err AccountInactive
            else if amount > state.balance then Err InsufficientFunds
            else Ok (TransferSent { amount, to })
        CloseAccount ->
            if state.balance != 0 then Err BalanceNotZero
            else Ok AccountClosed

main =
    events = [
        AccountOpened { owner: "An" },
        Deposited { amount: 1000000 },
        Withdrawn { amount: 200000 },
        Deposited { amount: 50000 },
        TransferSent { amount: 100000, to: "Bình" },
    ]

    account = rebuildState events

    Stdout.line! "=== Tài khoản $(account.owner) ==="
    Stdout.line! "Số dư: $(Num.toStr account.balance)đ"
    Stdout.line! "Active: $(Inspect.toStr account.isActive)"

    # Replay từng bước — xem balance thay đổi theo thời gian
    Stdout.line! "\n=== Lịch sử ==="
    List.walk events { state: emptyAccount, index: 0 } \ctx, event ->
        newState = applyEvent ctx.state event
        label = when event is
            AccountOpened { owner } -> "Mở TK: $(owner)"
            Deposited { amount } -> "Nạp $(Num.toStr amount)đ"
            Withdrawn { amount } -> "Rút $(Num.toStr amount)đ"
            TransferSent { amount, to } -> "Chuyển $(Num.toStr amount)đ → $(to)"
            TransferReceived { amount, from } -> "Nhận $(Num.toStr amount)đ ← $(from)"
            AccountClosed -> "Đóng TK"
        Stdout.line! "  $(Num.toStr (ctx.index + 1)). $(label) → Balance: $(Num.toStr newState.balance)đ"
        { state: newState, index: ctx.index + 1 }

    # Validate command mới
    Stdout.line! "\n=== Test commands ==="
    when validateCommand account (Withdraw { amount: 500000 }) is
        Ok _ -> Stdout.line! "✅ Rút 500k: OK"
        Err err -> Stdout.line! "❌ Rút 500k: $(Inspect.toStr err)"

    when validateCommand account (Withdraw { amount: 2000000 }) is
        Ok _ -> Stdout.line! "✅ Rút 2M: OK"
        Err InsufficientFunds -> Stdout.line! "❌ Rút 2M: Không đủ tiền"
        Err _ -> Stdout.line! "❌ Lỗi khác"
    # Output:
    # === Tài khoản An ===
    # Số dư: 750000đ
    # Active: Bool.true
    #
    # === Lịch sử ===
    #   1. Mở TK: An → Balance: 0đ
    #   2. Nạp 1000000đ → Balance: 1000000đ
    #   3. Rút 200000đ → Balance: 800000đ
    #   4. Nạp 50000đ → Balance: 850000đ
    #   5. Chuyển 100000đ → Bình → Balance: 750000đ
    #
    # === Test commands ===
    # ✅ Rút 500k: OK
    # ❌ Rút 2M: Không đủ tiền
```

---

## 16.3 — Projections: Nhiều góc nhìn từ cùng events

**Projection** = walk cùng event list nhưng tạo **state khác nhau**:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

main =
    events = [
        ItemAdded { name: "Phở", price: 45000, qty: 1 },
        ItemAdded { name: "Cà phê", price: 25000, qty: 2 },
        ItemRemoved { name: "Cà phê", qty: 1 },
        ItemAdded { name: "Bánh mì", price: 15000, qty: 3 },
        DiscountApplied { percent: 10 },
        ItemAdded { name: "Phở", price: 45000, qty: 1 },
    ]

    # Projection 1: Giỏ hàng hiện tại
    cart = List.walk events (Dict.empty {}) \state, event ->
        when event is
            ItemAdded { name, price, qty } ->
                existing = Dict.get state name |> Result.withDefault { price, qty: 0 }
                Dict.insert state name { price, qty: existing.qty + qty }
            ItemRemoved { name, qty } ->
                when Dict.get state name is
                    Ok item ->
                        newQty = item.qty - qty
                        if newQty <= 0 then Dict.remove state name
                        else Dict.insert state name { item & qty: newQty }
                    Err _ -> state
            _ -> state

    Stdout.line! "=== Giỏ hàng ==="
    # {} = unit accumulator (chỉ in, không tích lũy giá trị)
    Dict.walk cart {} \_, name, item ->
        Stdout.line! "  $(name) × $(Num.toStr item.qty) = $(Num.toStr (item.price * Num.toU64 item.qty))đ"

    # Projection 2: Thống kê
    stats = List.walk events { totalAdded: 0, totalRemoved: 0, discountPercent: 0 } \s, event ->
        when event is
            ItemAdded { qty, .. } -> { s & totalAdded: s.totalAdded + qty }
            ItemRemoved { qty, .. } -> { s & totalRemoved: s.totalRemoved + qty }
            DiscountApplied { percent } -> { s & discountPercent: percent }

    Stdout.line! "\n=== Thống kê ==="
    Stdout.line! "  Thêm: $(Num.toStr stats.totalAdded) lần"
    Stdout.line! "  Xóa: $(Num.toStr stats.totalRemoved) lần"
    Stdout.line! "  Giảm giá: $(Num.toStr stats.discountPercent)%"

    # Projection 3: Timeline
    Stdout.line! "\n=== Timeline ==="
    List.walkWithIndex events {} \_, event, i ->
        desc = when event is
            ItemAdded { name, qty, .. } -> "➕ $(name) ×$(Num.toStr qty)"
            ItemRemoved { name, qty } -> "➖ $(name) ×$(Num.toStr qty)"
            DiscountApplied { percent } -> "🏷️ Giảm $(Num.toStr percent)%"
        Stdout.line! "  $(Num.toStr (i + 1)). $(desc)"
```

Cùng event list → 3 projections hoàn toàn khác nhau. Thêm projection mới = viết 1 `List.walk` mới — **không sửa code cũ**.

---

## 16.4 — Snapshots: Tối ưu hiệu năng

Khi event list **quá dài** (hàng triệu events), rebuild từ đầu chậm. Giải pháp: **snapshot** — lưu state tại 1 thời điểm:

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

applyEvent = \state, event ->
    when event is
        Deposited { amount } -> { state & balance: state.balance + amount }
        Withdrawn { amount } -> { state & balance: state.balance - amount }

# Rebuild từ snapshot + recent events
rebuildFromSnapshot = \snapshot, recentEvents ->
    List.walk recentEvents snapshot applyEvent

main =
    # Giả sử đã có snapshot từ event #1000
    snapshot = { owner: "An", balance: 5000000, isActive: Bool.true }

    # Chỉ cần apply events MỚI (sau snapshot)
    recentEvents = [
        Deposited { amount: 200000 },
        Withdrawn { amount: 50000 },
        Deposited { amount: 100000 },
    ]

    current = rebuildFromSnapshot snapshot recentEvents
    Stdout.line! "Balance: $(Num.toStr current.balance)đ"
    # 5000000 + 200000 - 50000 + 100000 = 5250000đ

    # Tạo snapshot mới khi cần
    Stdout.line! "Snapshot events saved: $(Num.toStr (List.len recentEvents))"
```

```
Không snapshot:  [e1, e2, e3, ... e999, e1000, e1001, e1002]
                 ← walk toàn bộ 1002 events

Có snapshot:     [snapshot@1000] + [e1001, e1002]
                 ← walk chỉ 2 events!
```

---

## 16.5 — Ví dụ hoàn chỉnh: Task Tracker

Kết hợp mọi concept: commands tạo events, events được lưu, projections rebuild state từ events. Ứng dụng Task Tracker hoàn chỉnh.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Events
# TaskCreated { id, title }
# TaskCompleted { id }
# TaskReopened { id }
# TaskDeleted { id }
# PrioritySet { id, priority: [Low, Medium, High] }

applyTaskEvent = \state, event ->
    when event is
        TaskCreated { id, title } ->
            task = { id, title, status: Open, priority: Medium }
            { state & tasks: Dict.insert state.tasks id task }
        TaskCompleted { id } ->
            when Dict.get state.tasks id is
                Ok task -> { state & tasks: Dict.insert state.tasks id { task & status: Done } }
                Err _ -> state
        TaskReopened { id } ->
            when Dict.get state.tasks id is
                Ok task -> { state & tasks: Dict.insert state.tasks id { task & status: Open } }
                Err _ -> state
        TaskDeleted { id } ->
            { state & tasks: Dict.remove state.tasks id }
        PrioritySet { id, priority } ->
            when Dict.get state.tasks id is
                Ok task -> { state & tasks: Dict.insert state.tasks id { task & priority } }
                Err _ -> state

emptyState = { tasks: Dict.empty {} }

rebuildTasks = \events -> List.walk events emptyState applyTaskEvent

# Projections
countByStatus = \state ->
    Dict.walk state.tasks { open: 0, done: 0 } \counts, _, task ->
        when task.status is
            Open -> { counts & open: counts.open + 1 }
            Done -> { counts & done: counts.done + 1 }

highPriorityOpen = \state ->
    Dict.walk state.tasks [] \items, _, task ->
        if task.priority == High && task.status == Open then
            List.append items task.title
        else
            items

main =
    events = [
        TaskCreated { id: 1, title: "Setup dự án" },
        TaskCreated { id: 2, title: "Viết tests" },
        TaskCreated { id: 3, title: "Fix bug login" },
        PrioritySet { id: 3, priority: High },
        TaskCompleted { id: 1 },
        TaskCreated { id: 4, title: "Deploy production" },
        PrioritySet { id: 4, priority: High },
        TaskCompleted { id: 3 },
        TaskReopened { id: 3 },
    ]

    state = rebuildTasks events

    # Projection: Danh sách tasks
    Stdout.line! "=== Tasks ==="
    Dict.walk state.tasks {} \_, id, task ->
        statusIcon = when task.status is
            Open -> "⬜"
            Done -> "✅"
        priorityIcon = when task.priority is
            High -> "🔴"
            Medium -> "🟡"
            Low -> "🟢"
        Stdout.line! "  $(statusIcon) $(priorityIcon) #$(Num.toStr id): $(task.title)"

    # Projection: Thống kê
    counts = countByStatus state
    Stdout.line! "\n📊 Open: $(Num.toStr counts.open) | Done: $(Num.toStr counts.done)"

    # Projection: High priority open
    urgent = highPriorityOpen state
    Stdout.line! "\n🔴 Urgent:"
    List.forEach urgent \title -> Stdout.line! "  → $(title)"
    # Output:
    # === Tasks ===
    #   ✅ 🟡 #1: Setup dự án
    #   ⬜ 🟡 #2: Viết tests
    #   ⬜ 🔴 #3: Fix bug login
    #   ⬜ 🔴 #4: Deploy production
    #
    # 📊 Open: 3 | Done: 1
    #
    # 🔴 Urgent:
    #   → Fix bug login
    #   → Deploy production
```

---


## ✅ Checkpoint 16

> Đến đây bạn phải hiểu:
> 1. Event Sourcing = lưu **events** thay vì state — rebuild bằng `List.walk`
> 2. Events là tag union: `[ItemAdded, ItemRemoved, OrderPlaced, ...]`
> 3. `fold events initialState applyEvent` = rebuild bất kỳ state nào
>
> **Test nhanh**: Tại sao lưu events tốt hơn lưu state cuối?
> <details><summary>Đáp án</summary>Full audit trail, time travel, và có thể tạo nhiều views (projections) khác nhau.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Shopping cart event sourcing

Viết event-sourced shopping cart với events:

```roc
# ItemAdded { product, price, qty }
# ItemRemoved { product }
# QuantityChanged { product, newQty }
# CouponApplied { code, discountPercent }

# Rebuild: { items: Dict, coupon: ... }
# Projection: tính totalPrice (sau coupon)
```

<details><summary>✅ Lời giải</summary>

```roc
applyCartEvent = \state, event ->
    when event is
        ItemAdded { product, price, qty } ->
            { state & items: Dict.insert state.items product { price, qty } }
        ItemRemoved { product } ->
            { state & items: Dict.remove state.items product }
        QuantityChanged { product, newQty } ->
            when Dict.get state.items product is
                Ok item -> { state & items: Dict.insert state.items product { item & qty: newQty } }
                Err _ -> state
        CouponApplied { discountPercent, .. } ->
            { state & discountPercent }

totalPrice = \state ->
    subtotal = Dict.walk state.items 0 \s, _, item ->
        s + item.price * Num.toU64 item.qty
    subtotal - (subtotal * state.discountPercent // 100)
```

</details>

---

**Bài 2** (15 phút): Time travel

Viết function `stateAtEvent` trả state tại một event cụ thể:

```roc
# stateAtEvent events 3 → state sau 3 events đầu tiên
# stateAtEvent events 0 → emptyState
```

Và `diff` — so sánh state tại 2 thời điểm khác nhau:

<details><summary>✅ Lời giải</summary>

```roc
stateAtEvent = \events, n ->
    events
    |> List.takeFirst n
    |> List.walk emptyState applyEvent

# Diff balance
balanceDiff = \events, fromN, toN ->
    stateFrom = stateAtEvent events fromN
    stateTo = stateAtEvent events toN
    stateTo.balance - stateFrom.balance
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Rebuild chậm | Quá nhiều events | Dùng snapshots (16.4) |
| Events thiếu data để rebuild | Event design quá sparse | Mỗi event phải chứa ĐỦ data để rebuild — không rely on external state |
| State sai sau replay | Bug trong `applyEvent` | Test `applyEvent` cho từng event type |
| Projection trả kết quả khác nhau | `List.walk` initial state sai | Kiểm tra giá trị khởi tạo cho mỗi projection |

---

## Tóm tắt

- ✅ **Event Sourcing** = lưu sự kiện (video) thay vì trạng thái (ảnh). Biết MỌI THỨ đã xảy ra.
- ✅ **Events = tag unions** — type-safe, exhaustive matching, immutable.
- ✅ **Rebuild = `List.walk events initialState applyEvent`** — pure function!
- ✅ **Projections** = nhiều `List.walk` khác nhau trên cùng events → nhiều góc nhìn.
- ✅ **Snapshots** = lưu state tại 1 thời điểm → chỉ replay events mới.
- ✅ **Validation** = `validateCommand state cmd → Result Event Error` — kiểm tra trước khi tạo event.
- ✅ **Time travel** = `List.takeFirst events n |> rebuild` — xem state tại bất kỳ thời điểm nào.

## 🎉 Kết thúc Part III — Design Patterns!

| Chapter | Pattern |
|---------|---------|
| 15 | Strategy, Command, Middleware, Builder, Interpreter |
| **16** | **Event Sourcing, Projections, Snapshots, Time Travel** |

## Tiếp theo

→ **Part IV: Domain-Driven Design** — Chapter 17: **Introduction to DDD** — Ubiquitous Language, Bounded Contexts, Aggregates, và cách áp dụng DDD với Roc's type system.
