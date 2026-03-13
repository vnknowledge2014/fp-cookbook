# Chapter 26 — Capstone: Café Management System ⭐

> **Đây là bài tổng hợp** — áp dụng TẤT CẢ kiến thức từ Chapter 0 đến 25.
>
> **Bạn sẽ xây dựng**: Hệ thống quản lý quán café hoàn chỉnh
> - Domain modeling (DDD) — opaque types, tag unions, state machines
> - Event sourcing — commands, events, projections
> - Workflows as pipelines — `?` chaining
> - Testing — `expect` cho domain logic
> - CLI interface — interactive menu
>
> **Thời gian**: ~60 phút | **Level**: Principal
> **Kết quả cuối cùng**: Hệ thống hoàn chỉnh, testable, extensible — proof of mastery.

---

Đây là lúc mọi thứ kết hợp lại. 21 chapters kiến thức — từ values đến DDD, từ testing đến web services — gộp vào một dự án hoàn chỉnh: hệ thống quản lý quán cà phê. Không phải bài tập nhỏ lẻ nữa, mà là sản phẩm có kiến trúc rõ ràng.

## Capstone Project — Tất cả cùng nhau

Mọi concept từ 25 chapters trước kết hợp vào một dự án hoàn chỉnh. Domain modeling, workflows, error handling, serialization, testing — tất cả áp dụng vào bài toán thực tế. Đây là blueprint cho production Roc applications.


## 26.1 — Domain Analysis

Bước đầu tiên của DDD: nói chuyện với domain expert (chủ quán). Từ cuộc hội thoại, rút ra ubiquitous language — bảng ánh xạ giữa thuật ngữ business và code.

### Cuộc trò chuyện với chủ quán

```
"Quán tôi bán cà phê, trà, và sinh tố. Mỗi đồ uống có SIZE (S/M/L).

Quy trình: Khách ĐẶT MÓN → nhân viên XÁC NHẬN → barista PHA CHẾ → phục vụ GIAO → khách NHẬN.
Đơn có thể bị HỦY bất kỳ lúc nào trước khi pha.

Tôi cần biết: doanh thu ngày, đồ uống bán chạy, báo cáo theo giờ."
```

### Ubiquitous Language

| Thuật ngữ | Code |
|-----------|------|
| Thức uống | `Drink` |
| Kích cỡ | `Size` |
| Đơn hàng | `CafeOrder` |
| Đặt món | `placeOrder` |
| Xác nhận | `confirmOrder` |
| Pha chế | `startPreparing` |
| Giao | `markReady` |
| Nhận | `complete` |
| Hủy | `cancelOrder` |
| Doanh thu | `revenue` |

---

## 26.2 — Value Objects

Từ domain analysis sang code: mỗi khái niệm business trở thành type, mỗi quy tắc business trở thành function. Đây là phần dài nhất — toàn bộ hệ thống nằm ở đây.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout
import cli.Stdin

# ═══════════════════════════════════════
#               VALUE OBJECTS
# ═══════════════════════════════════════

# Drinks — tag union
# Mỗi drink có default price, size multiplier áp dụng sau

drinkName = \drink ->
    when drink is
        Espresso -> "Espresso"
        Latte -> "Latte"
        Cappuccino -> "Cappuccino"
        GreenTea -> "Trà xanh"
        MilkTea -> "Trà sữa"
        MangoSmoothie -> "Sinh tố xoài"

drinkBasePrice = \drink ->
    when drink is
        Espresso -> 35000
        Latte -> 45000
        Cappuccino -> 45000
        GreenTea -> 30000
        MilkTea -> 40000
        MangoSmoothie -> 50000

# Size — tag union + multiplier
sizeLabel = \size ->
    when size is
        S -> "S"
        M -> "M"
        L -> "L"

sizeMultiplier = \size ->
    when size is
        S -> 100    # ×1.0
        M -> 130    # ×1.3
        L -> 160    # ×1.6

# Calculate price: base × multiplier / 100
calculatePrice = \drink, size ->
    drinkBasePrice drink * sizeMultiplier size // 100

# ═══════ TESTS ═══════

expect calculatePrice Espresso S == 35000
expect calculatePrice Espresso M == 45500
expect calculatePrice Latte L == 72000
expect calculatePrice GreenTea S == 30000

# ═══════════════════════════════════════
#              ORDER ITEM
# ═══════════════════════════════════════

formatItem = \item ->
    price = calculatePrice item.drink item.size
    "$(drinkName item.drink) ($(sizeLabel item.size)) — $(Num.toStr price)đ"

# ═══════════════════════════════════════
#           STATE MACHINE: Order
# ═══════════════════════════════════════

# States: Placed → Confirmed → Preparing → Ready → Completed
#                                                  ↗
#         Placed → Cancelled (bất kỳ lúc nào trước Preparing)
#         Confirmed → Cancelled

statusIcon = \status ->
    when status is
        Placed -> "📝"
        Confirmed -> "✅"
        Preparing -> "☕"
        Ready -> "🔔"
        Completed -> "🎉"
        Cancelled _ -> "❌"

statusLabel = \status ->
    when status is
        Placed -> "Đặt món"
        Confirmed -> "Đã xác nhận"
        Preparing -> "Đang pha"
        Ready -> "Sẵn sàng"
        Completed -> "Hoàn thành"
        Cancelled reason -> "Hủy: $(reason)"

# ═══════════════════════════════════════
#            STATE TRANSITIONS
# ═══════════════════════════════════════

confirmOrder = \order ->
    when order.status is
        Placed -> Ok { order & status: Confirmed }
        _ -> Err (InvalidTransition "Chỉ xác nhận đơn vừa đặt")

startPreparing = \order ->
    when order.status is
        Confirmed -> Ok { order & status: Preparing }
        _ -> Err (InvalidTransition "Chỉ pha khi đã xác nhận")

markReady = \order ->
    when order.status is
        Preparing -> Ok { order & status: Ready }
        _ -> Err (InvalidTransition "Chỉ sẵn sàng khi đang pha")

completeOrder = \order ->
    when order.status is
        Ready -> Ok { order & status: Completed }
        _ -> Err (InvalidTransition "Chỉ hoàn thành khi đã sẵn sàng")

cancelOrder = \order, reason ->
    when order.status is
        Placed -> Ok { order & status: Cancelled reason }
        Confirmed -> Ok { order & status: Cancelled reason }
        _ -> Err (InvalidTransition "Không thể hủy đơn đang pha hoặc đã xong")

# ═══════ STATE TRANSITION TESTS ═══════

expect
    order = { id: 1, items: [], status: Placed, total: 0 }
    Result.isOk (confirmOrder order)

expect
    order = { id: 1, items: [], status: Confirmed, total: 0 }
    Result.isOk (startPreparing order)

expect
    order = { id: 1, items: [], status: Preparing, total: 0 }
    Result.isErr (cancelOrder order "Đổi ý")

expect
    order = { id: 1, items: [], status: Placed, total: 0 }
    confirmed = confirmOrder order |> Result.withDefault order
    preparing = startPreparing confirmed |> Result.withDefault confirmed
    ready = markReady preparing |> Result.withDefault preparing
    completed = completeOrder ready |> Result.withDefault ready
    completed.status == Completed

# ═══════════════════════════════════════
#              EVENTS (Event Sourcing)
# ═══════════════════════════════════════

applyEvent = \state, event ->
    when event is
        OrderPlaced { id, items, total } ->
            order = { id, items, status: Placed, total }
            { state & orders: Dict.insert state.orders id order, nextId: id + 1 }
        OrderConfirmed { id } ->
            updateOrder state id \o -> { o & status: Confirmed }
        PreparationStarted { id } ->
            updateOrder state id \o -> { o & status: Preparing }
        OrderReady { id } ->
            updateOrder state id \o -> { o & status: Ready }
        OrderCompleted { id } ->
            updateOrder state id \o -> { o & status: Completed }
        OrderCancelled { id, reason } ->
            updateOrder state id \o -> { o & status: Cancelled reason }

updateOrder = \state, id, transform ->
    when Dict.get state.orders id is
        Ok order -> { state & orders: Dict.insert state.orders id (transform order) }
        Err _ -> state

emptyState = { orders: Dict.empty {}, nextId: 1 }

rebuildState = \events -> List.walk events emptyState applyEvent

# ═══════════════════════════════════════
#           WORKFLOW PIPELINE
# ═══════════════════════════════════════

# Place order pipeline: validate → calculate → create event
placeOrderWorkflow = \rawItems ->
    items = validateItems? rawItems
    total = List.walk items 0 \s, item -> s + calculatePrice item.drink item.size
    Ok { items, total }

validateItems = \rawItems ->
    if List.isEmpty rawItems then Err EmptyOrder
    else Ok rawItems

# ═══════ WORKFLOW TESTS ═══════

expect
    items = [{ drink: Latte, size: M }, { drink: GreenTea, size: S }]
    when placeOrderWorkflow items is
        Ok result -> result.total == 45500 + 30000  # 75500
        Err _ -> Bool.false

expect placeOrderWorkflow [] == Err EmptyOrder

# ═══════════════════════════════════════
#              PROJECTIONS
# ═══════════════════════════════════════

# Projection 1: Doanh thu
totalRevenue = \state ->
    Dict.walk state.orders 0 \revenue, _, order ->
        when order.status is
            Completed -> revenue + order.total
            _ -> revenue

# Projection 2: Thống kê trạng thái
orderStats = \state ->
    Dict.walk state.orders { placed: 0, confirmed: 0, preparing: 0, ready: 0, completed: 0, cancelled: 0 } \stats, _, order ->
        when order.status is
            Placed -> { stats & placed: stats.placed + 1 }
            Confirmed -> { stats & confirmed: stats.confirmed + 1 }
            Preparing -> { stats & preparing: stats.preparing + 1 }
            Ready -> { stats & ready: stats.ready + 1 }
            Completed -> { stats & completed: stats.completed + 1 }
            Cancelled _ -> { stats & cancelled: stats.cancelled + 1 }

# Projection 3: Đồ uống bán chạy
drinkPopularity = \state ->
    Dict.walk state.orders (Dict.empty {}) \counts, _, order ->
        when order.status is
            Completed ->
                List.walk order.items counts \c, item ->
                    name = drinkName item.drink
                    current = Dict.get c name |> Result.withDefault 0
                    Dict.insert c name (current + 1)
            _ -> counts

# ═══════ PROJECTION TESTS ═══════

expect
    events = [
        OrderPlaced { id: 1, items: [{ drink: Latte, size: M }], total: 45500 },
        OrderConfirmed { id: 1 },
        PreparationStarted { id: 1 },
        OrderReady { id: 1 },
        OrderCompleted { id: 1 },
    ]
    state = rebuildState events
    totalRevenue state == 45500

expect
    events = [
        OrderPlaced { id: 1, items: [], total: 50000 },
        OrderPlaced { id: 2, items: [], total: 30000 },
        OrderCompleted { id: 1 },
        OrderCancelled { id: 2, reason: "Hết hàng" },
    ]
    state = rebuildState events
    stats = orderStats state
    stats.completed == 1 && stats.cancelled == 1

# ═══════════════════════════════════════
#              CLI INTERFACE
# ═══════════════════════════════════════

showMainMenu =
    """

    ╔══════════════════════════════════╗
    ║       ☕ Café Manager v1.0       ║
    ╠══════════════════════════════════╣
    ║  1. Đặt món                      ║
    ║  2. Xem đơn hàng                 ║
    ║  3. Cập nhật trạng thái          ║
    ║  4. Hủy đơn                      ║
    ║  5. Báo cáo doanh thu            ║
    ║  6. Thoát                        ║
    ╚══════════════════════════════════╝
    Chọn (1-6):
    """

showDrinkMenu =
    """
    Chọn thức uống:
    1. Espresso    (35k)
    2. Latte       (45k)
    3. Cappuccino  (45k)
    4. Trà xanh    (30k)
    5. Trà sữa     (40k)
    6. Sinh tố xoài (50k)
    """

parseDrink = \choice ->
    when choice is
        "1" -> Ok Espresso
        "2" -> Ok Latte
        "3" -> Ok Cappuccino
        "4" -> Ok GreenTea
        "5" -> Ok MilkTea
        "6" -> Ok MangoSmoothie
        _ -> Err InvalidChoice

parseSize = \choice ->
    when choice is
        "1" -> Ok S
        "2" -> Ok M
        "3" -> Ok L
        _ -> Err InvalidChoice

# ═══════════════════════════════════════
#                 MAIN
# ═══════════════════════════════════════

main =
    Stdout.line! "☕ Chào mừng đến Café Manager!"
    mainLoop! emptyState

mainLoop! = \state ->
    Stdout.line! showMainMenu
    choice = Stdin.line!

    when choice is
        "1" -> # Đặt món
            Stdout.line! showDrinkMenu
            drinkChoice = Stdin.line!
            when parseDrink drinkChoice is
                Err _ ->
                    Stdout.line! "❓ Chọn 1-6"
                    mainLoop! state
                Ok drink ->
                    Stdout.line! "Size (1=S, 2=M, 3=L):"
                    sizeChoice = Stdin.line!
                    when parseSize sizeChoice is
                        Err _ ->
                            Stdout.line! "❓ Chọn 1-3"
                            mainLoop! state
                        Ok size ->
                            item = { drink, size }
                            price = calculatePrice drink size
                            event = OrderPlaced {
                                id: state.nextId,
                                items: [item],
                                total: price,
                            }
                            newState = applyEvent state event
                            Stdout.line! "✅ Đơn #$(Num.toStr state.nextId): $(formatItem item) = $(Num.toStr price)đ"
                            mainLoop! newState

        "2" -> # Xem đơn hàng
            if Dict.isEmpty state.orders then
                Stdout.line! "📭 Chưa có đơn nào"
            else
                Stdout.line! "=== Đơn hàng ==="
                Dict.walk state.orders {} \_, id, order ->
                    icon = statusIcon order.status
                    label = statusLabel order.status
                    Stdout.line! "  $(icon) #$(Num.toStr id) — $(Num.toStr order.total)đ — $(label)"
            mainLoop! state

        "3" -> # Cập nhật trạng thái
            Stdout.line! "Nhập mã đơn:"
            idStr = Stdin.line!
            when Str.toU64 idStr is
                Err _ ->
                    Stdout.line! "❌ Mã không hợp lệ"
                    mainLoop! state
                Ok id ->
                    when Dict.get state.orders id is
                        Err _ ->
                            Stdout.line! "❌ Không tìm thấy đơn #$(idStr)"
                            mainLoop! state
                        Ok order ->
                            nextAction = when order.status is
                                Placed -> "xác nhận"
                                Confirmed -> "bắt đầu pha"
                                Preparing -> "sẵn sàng"
                                Ready -> "hoàn thành"
                                _ -> "không thể cập nhật"
                            Stdout.line! "$(statusIcon order.status) #$(idStr): $(statusLabel order.status) → $(nextAction)? (y/n)"
                            confirm = Stdin.line!
                            if confirm == "y" then
                                event = when order.status is
                                    Placed -> Ok (OrderConfirmed { id })
                                    Confirmed -> Ok (PreparationStarted { id })
                                    Preparing -> Ok (OrderReady { id })
                                    Ready -> Ok (OrderCompleted { id })
                                    _ -> Err "Không thể cập nhật"
                                when event is
                                    Ok ev ->
                                        newState = applyEvent state ev
                                        Stdout.line! "✅ Đã cập nhật"
                                        mainLoop! newState
                                    Err msg ->
                                        Stdout.line! "❌ $(msg)"
                                        mainLoop! state
                            else
                                mainLoop! state

        "4" -> # Hủy đơn
            Stdout.line! "Nhập mã đơn cần hủy:"
            idStr = Stdin.line!
            when Str.toU64 idStr is
                Err _ ->
                    Stdout.line! "❌ Mã không hợp lệ"
                    mainLoop! state
                Ok id ->
                    when Dict.get state.orders id is
                        Err _ ->
                            Stdout.line! "❌ Không tìm thấy"
                            mainLoop! state
                        Ok order ->
                            when cancelOrder order "Khách hủy" is
                                Ok _ ->
                                    newState = applyEvent state (OrderCancelled { id, reason: "Khách hủy" })
                                    Stdout.line! "❌ Đã hủy đơn #$(idStr)"
                                    mainLoop! newState
                                Err (InvalidTransition msg) ->
                                    Stdout.line! "❌ $(msg)"
                                    mainLoop! state

        "5" -> # Báo cáo
            revenue = totalRevenue state
            stats = orderStats state
            popularity = drinkPopularity state

            Stdout.line! "╔══════════════════════════════════╗"
            Stdout.line! "║         📊 BÁO CÁO              ║"
            Stdout.line! "╠══════════════════════════════════╣"
            Stdout.line! "║ 💰 Doanh thu: $(Num.toStr revenue)đ"
            Stdout.line! "║"
            Stdout.line! "║ 📋 Trạng thái:"
            Stdout.line! "║   📝 Đặt: $(Num.toStr stats.placed)"
            Stdout.line! "║   ✅ Xác nhận: $(Num.toStr stats.confirmed)"
            Stdout.line! "║   ☕ Đang pha: $(Num.toStr stats.preparing)"
            Stdout.line! "║   🔔 Sẵn sàng: $(Num.toStr stats.ready)"
            Stdout.line! "║   🎉 Hoàn thành: $(Num.toStr stats.completed)"
            Stdout.line! "║   ❌ Hủy: $(Num.toStr stats.cancelled)"
            Stdout.line! "║"
            Stdout.line! "║ 🏆 Bán chạy:"
            Dict.walk popularity {} \_, name, count ->
                Stdout.line! "║   $(name): $(Num.toStr count) ly"
            Stdout.line! "╚══════════════════════════════════╝"
            mainLoop! state

        "6" ->
            stats = orderStats state
            Stdout.line! "👋 Tạm biệt! Hôm nay: $(Num.toStr stats.completed) đơn hoàn thành."

        _ ->
            Stdout.line! "❓ Chọn 1-6"
            mainLoop! state
```

---

## 26.3 — Kiến thức đã áp dụng

Nhìn lại: 21 chapters kiến thức gộp vào 1 dự án. Bảng dưới đây map mỗi tính năng trong capstone về chapter bạn đã học.

| Chapter | Đã dùng |
|---------|---------|
| 5 (Types) | `U64`, `Str`, `Bool` cho domain values |
| 6 (Control Flow) | `when...is`, `if...then...else` |
| 7 (Records & Tags) | Records cho Order, tag unions cho Drink/Size/Status |
| 8 (Pattern Matching) | Exhaustive matching trên status, drink, size |
| 9 (Lists) | `List.walk`, `List.map`, `List.keepIf` |
| 10 (Opaque Types) | Concepts cho value objects |
| 11 (Purity) | Pure core (calculatePrice, formatItem) vs shell (mainLoop!) |
| 12 (Tasks) | `Stdout.line!`, `Stdin.line!`, interactive loop |
| 13 (Result) | `?` operator trong workflow, error handling |
| 15 (FP Patterns) | Command pattern (events), Strategy (drinkBasePrice) |
| 16 (Event Sourcing) | Events → `applyEvent` → `rebuildState`, projections |
| 17-18 (DDD) | Ubiquitous Language, value objects, state machine, aggregates |
| 19 (Pipelines) | `placeOrderWorkflow` validation pipeline |
| 20 (Platform) | Pure domain + effectful shell separation |
| 22-23 (Testing) | `expect` tests cho prices, transitions, projections |

---

## 26.4 — Mở rộng (bài tập)

Hệ thống café hiện tại là nền tảng. Những bài tập sau thêm tính năng mới — mỗi bài tập trung vào một pattern bạn đã học.

### Bài 1: Thêm toppings

```roc
# Thêm Topping: [NoTopping, BubblePearl (+10k), WhippedCream (+8k), ExtraShot (+15k)]
# Thay đổi: item có field topping, calculatePrice cộng thêm topping price
```

### Bài 2: Nhiều items per order

```roc
# Cho phép đặt nhiều items trong 1 đơn
# placeOrderWorkflow nhận List { drink, size, topping }
# Tính total = sum of all items
```

### Bài 3: Báo cáo theo giờ

```roc
# Thêm timestamp cho events
# Projection: revenue by hour — "10:00-11:00: 450000đ, 11:00-12:00: 680000đ"
```

### Bài 4: REST API

```roc
# Chuyển sang basic-webserver platform
# POST /api/orders — đặt món
# GET /api/orders — danh sách
# PUT /api/orders/:id/next — chuyển trạng thái
# GET /api/reports/revenue — doanh thu
# Domain code KHÔNG ĐỔI — chỉ đổi shell!
```

---

## 🎉 Chúc mừng! Bạn đã hoàn thành Part V!

### Recap hành trình đến đây

```
Part 0: CS Foundations
  └─ Thuật toán, cấu trúc dữ liệu, tư duy lập trình

Part I: Roc Fundamentals
  └─ Syntax, types, functions, collections, modules

Part II: Thinking Functionally
  └─ Purity, Tasks, Result, Abilities

Part III: Design Patterns
  └─ FP Patterns, Event Sourcing

Part IV: Domain-Driven Design
  └─ DDD, Domain Modeling, Pipelines, Platform Separation

Part V: Testing & Apps ← BẠN Ở ĐÂY! ✅
  └─ TDD, PBT, CLI Apps, Web Services, Capstone Café System
```

> **💡 Bạn đã master**: Từ "Hello World" đến xây hệ thống event-sourced, domain-driven, fully tested. Đây là nền tảng vững chắc — bây giờ hãy mang nó vào production!

### Bước tiếp theo: Part VI — Production Engineering

Part VI sẽ dạy bạn những kỹ năng cần thiết để đưa ứng dụng Roc vào **thế giới thực**:

- **Database** — SQL, normalization, ACID, indexing
- **Security** — JWT, OAuth, OWASP Top 10
- **Distributed Systems** — CAP, Saga, Circuit Breaker
- **System Design** — capacity planning, API design, architecture
- **Capstone Production** — deploy Roc app với Docker, CI/CD

> **💡 Roc advantage trong production**: Domain code bạn viết ở Capstone trên **KHÔNG CẦN ĐỔI** khi đưa vào production — chỉ thêm infrastructure layer (database, auth, logging).

→ **Tiếp theo**: Chapter 27 — **Database Fundamentals & SQL**
