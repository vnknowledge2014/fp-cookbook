# Chapter 20 — Platform Separation ⭐

> **Bạn sẽ học được**:
> - Roc's **Platform model** — App = pure domain, Platform = IO
> - **Onion Architecture** — enforced by the language, not by discipline
> - Tại sao App code **KHÔNG THỂ** import IO — compiler cấm
> - So sánh với Clean Architecture, Hexagonal Architecture
> - Cách tổ chức dự án lớn theo Platform model
> - **Dependency inversion** tự nhiên — không cần DI framework
>
> **Yêu cầu trước**: [Chapter 19 — Workflows as Pipelines](chapter_19_workflows_as_pipelines.md)
> **Thời gian đọc**: ~35 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Hiểu sâu kiến trúc Roc — tại sao "pure core / effectful shell" được ngôn ngữ **enforce**, không chỉ khuyến khích.

---

Roc có một thiết kế mà không ngôn ngữ nào khác có: **platform model**. Application code và platform code tách hoàn toàn — app chỉ chứa pure logic, platform lo IO. Đây là Onion Architecture được enforce ở cấp độ ngôn ngữ, không phải convention.

## Platform Separation — Kiến trúc độc đáo của Roc

Roc có concept platform/application không giống language nào khác. Platform cung cấp I/O capabilities (file, network, terminal). Application viết pure logic dùng capabilities đó. Tách biệt này cho phép same application code chạy trên nhiều platforms khác nhau — CLI, web, embedded.


## 20.1 — Onion Architecture: Review nhanh

Onion Architecture đặt domain logic ở trung tâm, IO ở lớp ngoài. Trong Roc, platform model **enforce** pattern này ở cấp ngôn ngữ — không phải convention.

### Trong các ngôn ngữ khác

```
┌─────────────────────────────────────┐
│           Infrastructure            │  ← DB, HTTP, File IO
│  ┌─────────────────────────────┐   │
│  │      Application Layer      │   │  ← Use cases, orchestration
│  │  ┌─────────────────────┐   │   │
│  │  │    Domain Layer     │   │   │  ← Business rules, entities
│  │  └─────────────────────┘   │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

Quy tắc: Dependencies point INWARD
- Domain → không phụ thuộc gì
- Application → phụ thuộc Domain
- Infrastructure → phụ thuộc Application + Domain
```

**Vấn đề**: Trong Java/Python/Go, quy tắc này chỉ là **convention**. Ai cũng có thể `import database` từ Domain layer. CI/CD kiểm tra? Linter? Code review? Dễ bỏ sót.

### Trong Roc: Compiler enforce

```
┌─────────────────────────────────────┐
│           Platform (basic-cli)      │  ← IO: File, HTTP, Stdout
│  ┌─────────────────────────────┐   │
│  │           App (your code)   │   │  ← Pure functions + Tasks
│  │  ┌─────────────────────┐   │   │
│  │  │    Domain Modules    │   │   │  ← Pure: types, logic
│  │  └─────────────────────┘   │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

Quy tắc: Domain modules KHÔNG THỂ import platform IO
→ Compiler error nếu thử
→ Không cần linter, không cần code review cho điều này
```

---

## 20.2 — Platform Model chi tiết

Application code chỉ chứa pure logic — business rules, validation, transformations. Tất cả IO (database, network, filesystem) thuộc về platform.

### App vs Platform

```roc
# filename: main.roc

# Dòng 1: Khai báo App — cho platform biết entry point
app [main] {
    cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br"
}

# Platform cung cấp IO capabilities
import cli.Stdout      # ← IO capability từ platform
import cli.File        # ← IO capability từ platform
import cli.Stdin       # ← IO capability từ platform

# App modules — KHÔNG import IO
import Domain.Order    # ← pure module
import Domain.Payment  # ← pure module
```

### Domain module — PURE, không biết IO tồn tại

```roc
# filename: Domain/Order.roc
module [Order, create, addItem, total, validate]

# Module này KHÔNG ĐƯỢC import cli.Stdout, cli.File, etc.
# Compiler sẽ báo lỗi nếu thử
# → BẮT BUỘC pure

Order := {
    items : List { name : Str, price : U64, qty : U64 },
    status : [Draft, Placed, Paid, Shipped],
}

create : Order
create = @Order { items: [], status: Draft }

addItem : Order, Str, U64, U64 -> Order
addItem = \@Order order, name, price, qty ->
    @Order { order & items: List.append order.items { name, price, qty } }

total : Order -> U64
total = \@Order order ->
    List.walk order.items 0 \s, item -> s + item.price * item.qty

validate : Order -> Result Order [EmptyOrder, ZeroTotal]
validate = \@Order order ->
    if List.isEmpty order.items then Err EmptyOrder
    else if total (@Order order) == 0 then Err ZeroTotal
    else Ok (@Order order)
```

### main.roc — Shell, orchestrate IO + domain

```roc
# filename: main.roc
app [main] { cli: platform "..." }

import cli.Stdout
import cli.File
import cli.Path
import Domain.Order exposing [Order]

main =
    # IO: đọc input
    Stdout.line! "Nhập tên món:"
    itemName = Stdin.line!

    # PURE: domain logic
    order =
        Order.create
        |> Order.addItem itemName 45000 1

    when Order.validate order is
        Ok validOrder ->
            # PURE: tính toán
            orderTotal = Order.total validOrder
            receipt = formatReceipt validOrder orderTotal

            # IO: output
            Stdout.line! receipt
            File.writeUtf8! (Path.fromStr "receipt.txt") receipt
        Err EmptyOrder ->
            Stdout.line! "❌ Đơn hàng rỗng"
        Err ZeroTotal ->
            Stdout.line! "❌ Tổng = 0"

# Helper — vẫn pure
formatReceipt = \order, total ->
    "=== Hóa đơn ===\nTổng: $(Num.toStr total)đ"
```

---

## 20.3 — So sánh kiến trúc

Testing pure code đơn giản: input → output, không mock. Platform code test riêng với integration tests. Tách biệt rõ ràng = testing dễ dàng.

| | Clean/Hexagonal | Roc Platform |
|---|---|---|
| Enforcement | Convention + discipline | **Compiler** |
| DI Framework | Cần (Spring, etc.) | **Không cần** |
| Interface/Port | Phải tạo thủ công | **Platform cung cấp** |
| Test doubles | Mocking framework | **Gọi pure functions trực tiếp** |
| Complexity | Nhiều layers, interfaces | **2 layers: App + Platform** |
| Có thể bypass? | Có (ai cũng import được) | **Không** (compiler cấm) |

### Dependency Inversion — tự nhiên

```
OOP Clean Architecture:
  Domain ← Interface ← Infrastructure
  OrderService ← OrderRepository (interface) ← PostgresOrderRepository (class)
  → Cần DI container để wire

Roc:
  Domain modules → pure functions
  Platform → IO
  main.roc → import cả hai, connect
  → Không cần DI, vì domain KHÔNG BIẾT IO tồn tại
```

---

## 20.4 — Tổ chức dự án lớn

Dependency injection trong FP: truyền functions làm tham số thay vì inject services. Pure functions nhận mọi dependency qua arguments.

```
my-saas-app/
├── main.roc                      # Entry point — shell
│
├── Domain/                       # Pure domain — NO IO
│   ├── User.roc                  # User entity + business rules
│   ├── Order.roc                 # Order aggregate
│   ├── Payment.roc               # Payment value objects
│   ├── Inventory.roc             # Stock management
│   └── Pricing.roc               # Pricing strategies
│
├── Workflows/                    # Pure workflows — NO IO
│   ├── PlaceOrder.roc            # Order placement pipeline
│   ├── ProcessPayment.roc        # Payment pipeline
│   └── UserOnboarding.roc        # Registration pipeline
│
├── Adapters/                     # IO adapters — uses platform
│   ├── Database.roc              # File-based persistence
│   ├── EmailSender.roc           # Email via platform HTTP
│   └── JsonParser.roc            # Parse JSON input
│
└── tests/
    ├── UserTests.roc             # expect tests — pure
    ├── OrderTests.roc
    └── WorkflowTests.roc
```

### Quy tắc

1. **Domain/** — KHÔNG import `cli.*`. Chỉ import Domain modules khác
2. **Workflows/** — KHÔNG import `cli.*`. Import Domain modules
3. **Adapters/** — CÓ THỂ import `cli.*`. Connect Domain với IO
4. **main.roc** — Shell: import Adapters + Workflows, orchestrate

### Ví dụ flow

```roc
# main.roc
main =
    # 1. IO: Đọc request
    rawInput = Stdin.line!

    # 2. Adapter: Parse
    orderInput = JsonParser.parseOrderInput rawInput

    # 3. Pure Workflow: Process
    when PlaceOrder.execute orderInput is
        Ok order ->
            # 4. Adapter: Save
            Database.saveOrder! order
            # 5. Adapter: Notify
            EmailSender.sendConfirmation! order
            # 6. IO: Respond
            Stdout.line! "✅ Order #$(Num.toStr order.id) created"
        Err err ->
            Stdout.line! "❌ $(PlaceOrder.errorMessage err)"
```

---

## 20.5 — Testing: Pure Core = dễ test

Port/Adapter pattern: pure domain định nghĩa "interface" (port), platform cung cấp implementation (adapter). Thay database? Chỉ thay adapter, domain code không đổi.

### Domain tests — không cần mock

```roc
# filename: tests/OrderTests.roc

import Domain.Order

# Test domain logic — KHÔNG CẦN mock IO, database, hay HTTP
expect
    order =
        Order.create
        |> Order.addItem "Phở" 45000 2
        |> Order.addItem "Cà phê" 25000 1
    Order.total order == 115000

expect
    emptyOrder = Order.create
    Order.validate emptyOrder == Err EmptyOrder

expect
    order = Order.create |> Order.addItem "Phở" 45000 1
    Result.isOk (Order.validate order)
```

### Workflow tests — không cần mock

```roc
# filename: tests/PlaceOrderTests.roc

import Workflows.PlaceOrder

expect
    input = { items: [{ name: "Phở", price: 45000, qty: 1 }], address: "123 ABC" }
    result = PlaceOrder.execute input
    Result.isOk result

expect
    input = { items: [], address: "123 ABC" }
    PlaceOrder.execute input == Err EmptyOrder
```

**Không mock = test nhanh, đáng tin, dễ viết.**

---

## 20.6 — Available Platforms

Roc có nhiều platforms cho nhiều mục đích:

| Platform | Mục đích | IO capabilities |
|----------|---------|-----------------|
| `basic-cli` | CLI apps | Stdout, Stdin, File, HTTP, Env, Arg |
| `basic-webserver` | Web services | HTTP server, routing, JSON |
| `basic-ssg` | Static site generator | File, Markdown |
| Community platforms | Games, WASM, etc. | Tùy platform |

**Cùng Domain code, khác Platform:**

```roc
# Domain logic KHÔNG ĐỔI — chạy trên BẤT KỲ platform nào

# CLI app:
app [main] { cli: platform "basic-cli" }
# Domain.Order, Domain.Payment... → SAME!

# Web server:
app [main] { web: platform "basic-webserver" }
# Domain.Order, Domain.Payment... → SAME!

# Chỉ shell (main.roc) thay đổi
```

---


## ✅ Checkpoint 20

> Đến đây bạn phải hiểu:
> 1. Platform separation = **Onion Architecture miễn phí** ở language level
> 2. App = pure domain logic, Platform = IO (DB, HTTP, file)
> 3. App KHÔNG THỂ access IO mà platform không cung cấp — capability-based security
>
> **Test nhanh**: App Roc có thể gọi trực tiếp OS API không?
> <details><summary>Đáp án</summary>Không bao giờ! App chỉ gọi platform API. Platform quyết định cho phép gì.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Phân loại modules

Cho danh sách modules, phân loại vào Domain, Workflow, hoặc Adapter:

```
a) User.roc — create, validate, changePassword
b) SendEmail.roc — gửi email qua SMTP
c) ProcessRefund.roc — validate → calculate → createRefund
d) Product.roc — opaque type, pricing logic
e) FileStorage.roc — save/load file
f) RegisterUser.roc — validate → create → assign plan
g) Money.roc — add, subtract, currency conversion
```

<details><summary>✅ Lời giải</summary>

```
Domain:   a) User, d) Product, g) Money
Workflow: c) ProcessRefund, f) RegisterUser
Adapter:  b) SendEmail, e) FileStorage
```

</details>

---

**Bài 2** (15 phút): Refactor thành Platform Separation

Tách code dưới thành Domain + Shell:

```roc
# ❌ Logic + IO trộn lẫn
main =
    Stdout.line! "Nhập điểm:"
    scoreStr = Stdin.line!
    when Str.toI64 scoreStr is
        Ok score ->
            grade = if score >= 90 then "A"
                    else if score >= 80 then "B"
                    else if score >= 70 then "C"
                    else "F"
            Stdout.line! "Điểm: $(Num.toStr score) → $(grade)"
            File.writeUtf8! (Path.fromStr "result.txt") "$(Num.toStr score),$(grade)"
        Err _ ->
            Stdout.line! "❌ Nhập số!"
```

<details><summary>✅ Lời giải</summary>

```roc
# Domain (pure)
classifyScore = \score ->
    if score >= 90 then "A"
    else if score >= 80 then "B"
    else if score >= 70 then "C"
    else "F"

parseScore = \scoreStr ->
    Str.toI64 scoreStr |> Result.mapErr \_ -> InvalidInput

formatResult = \score, grade ->
    "$(Num.toStr score),$(grade)"

# Shell (IO)
main =
    Stdout.line! "Nhập điểm:"
    scoreStr = Stdin.line!
    when parseScore scoreStr is
        Ok score ->
            grade = classifyScore score
            Stdout.line! "Điểm: $(Num.toStr score) → $(grade)"
            File.writeUtf8! (Path.fromStr "result.txt") (formatResult score grade)
        Err InvalidInput ->
            Stdout.line! "❌ Nhập số!"
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Domain module cần IO | Thiết kế sai | Truyền data VÀO domain, không để domain ĐỌC data |
| Quá nhiều params truyền vào domain | Shell đọc quá nhiều IO cùng lúc | Nhóm thành record hoặc tách sub-workflows |
| "Tôi cần current time trong domain" | Time = side effect | Shell đọc time → truyền vào domain: `classify now event` |

---

## Tóm tắt

- ✅ **Platform model** = App (pure domain) + Platform (IO). **Compiler enforce** — không bypass được.
- ✅ **Onion Architecture by language** — không cần DI framework, linter, hay discipline.
- ✅ **Domain modules** = KHÔNG import IO. Pure functions, easy to test.
- ✅ **main.roc** = Shell. Import platform IO + domain, orchestrate.
- ✅ **Cùng domain, khác platform** — CLI, Web, SSG dùng chung Domain code.
- ✅ **Testing** = gọi pure functions trực tiếp, không mock IO.

## Tiếp theo

→ Chapter 21: **Serialization** — `Encode`/`Decode` abilities, JSON via platform, parse external data vào domain types an toàn.
