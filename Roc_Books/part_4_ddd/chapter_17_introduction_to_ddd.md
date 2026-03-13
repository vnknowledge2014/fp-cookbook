# Chapter 17 — Introduction to DDD

> **Bạn sẽ học được**:
> - **Domain-Driven Design** là gì — và tại sao nó quan trọng
> - **Ubiquitous Language** — ngôn ngữ chung giữa dev và domain experts
> - **Bounded Contexts** — ranh giới rõ ràng giữa các phần hệ thống
> - **Value Objects** — opaque types trong Roc
> - **Entities vs Value Objects** — khi nào dùng gì
> - **Aggregates** — nhóm dữ liệu liên quan
>
> **Yêu cầu trước**: [Chapter 16 — Event Sourcing](../part_3_design_patterns/chapter_16_event_sourcing.md)
> **Thời gian đọc**: ~40 phút | **Level**: Advanced
> **Kết quả cuối cùng**: Tư duy "types represent domain concepts" — nền tảng cho domain modeling.

---

Phần lớn dự án phần mềm thất bại không phải vì bug kỹ thuật — mà vì developer hiểu sai yêu cầu. Domain-Driven Design (DDD) giải quyết chuyện này bằng cách bắt code "nói cùng ngôn ngữ" với business. Trong FP, DDD đặc biệt mạnh vì types có thể biểu diễn chính xác domain rules.

## 17.1 — DDD là gì?

DDD là cách tiếp cận phát triển phần mềm đặt domain (nghiệp vụ) làm trung tâm. Code phản ánh cách business nghĩ — không phải cách developer nghĩ.

### Vấn đề: Code và Business nói tiếng khác

```python
# Dev code — thuật ngữ kỹ thuật
class UserManager:
    def process_transaction(self, user_id, data):
        record = db.get(user_id)
        record.field3 += data["amount"]  # field3 là gì???
        db.save(record)
```

```
# Business nói:
# "Khi khách hàng ĐẶT HÀNG, hệ thống cần XÁC NHẬN tồn kho,
#  TÍNH TỔNG gồm phí vận chuyển, rồi TẠO ĐƠN."
```

Code nói `process_transaction` + `field3`. Business nói `đặt hàng` + `xác nhận tồn kho`. **Hai ngôn ngữ khác nhau** → bugs, hiểu sai, features sai.

### DDD: Code IS the domain

```roc
# Roc — code ĐỌC NHƯ business rules
placeOrder = \cart, inventory ->
    verifiedItems = verifyStock? cart.items inventory    # xác nhận tồn kho
    subtotal = calculateSubtotal verifiedItems           # tính tổng
    shipping = calculateShipping cart.address             # phí vận chuyển
    order = createOrder verifiedItems (subtotal + shipping)  # tạo đơn
    Ok order
```

Đọc code = đọc business requirements. Đây là mục tiêu của DDD.

> **💡 Eric Evans** (tác giả DDD): *"The model is the code. The code is the model."*

---

## 17.2 — Ubiquitous Language

Ubiquitous language bắt buộc developer và domain expert dùng cùng từ vựng. "Order" trong code phải là "Order" trong cuộc họp — không phải "OrderEntity" hay "OrderDTO".

### Quy tắc: Dùng CÙNG từ ngữ

Dev và domain expert dùng **cùng thuật ngữ** trong mọi nơi: code, tài liệu, meeting, test:

```roc
# ❌ Thuật ngữ kỹ thuật — domain expert không hiểu
processEntity = \record, opType ->
    when opType is
        1 -> updateField record "status" "active"
        2 -> updateField record "status" "inactive"
        _ -> record

# ✅ Ubiquitous Language — đọc hiểu ngay
activateAccount = \account -> { account & status: Active }
suspendAccount = \account, reason -> { account & status: Suspended reason }
```

### Bảng ví dụ: E-commerce domain

| Domain term | Code (Roc) | KHÔNG dùng |
|-------------|-----------|------------|
| Đơn hàng | `Order` | `Record`, `Entity`, `DataObject` |
| Đặt hàng | `placeOrder` | `createRecord`, `processData` |
| Xác nhận | `confirmOrder` | `updateStatus` |
| Hủy đơn | `cancelOrder` | `deleteRecord`, `setFlag` |
| Tồn kho | `Inventory` | `StockTable`, `warehouse_data` |
| Mặt hàng | `LineItem` | `Item`, `row` |
| Khách hàng | `Customer` | `User`, `Actor` |
| Thanh toán | `Payment` | `Transaction` |
| Hóa đơn | `Invoice` | `BillingRecord` |

---

## 17.3 — Value Objects: Opaque Types trong Roc

Bounded context chia hệ thống lớn thành vùng nhỏ, mỗi vùng có ngôn ngữ riêng. "Product" trong Catalog có nghĩa khác "Product" trong Shipping — và đó là thiết kế đúng.

### Value Object = giá trị được định nghĩa bởi attributes, không identity

```roc
# filename: Domain.roc

# Value Objects — so sánh bằng GIÁ TRỊ (attributes)
# 100.000đ = 100.000đ (dù là 2 tờ tiền khác nhau)

Money := { amount : U64, currency : [VND, USD, EUR] } implements [Eq, Inspect]

fromVND : U64 -> Money
fromVND = \amount -> @Money { amount, currency: VND }

fromUSD : U64 -> Money
fromUSD = \amount -> @Money { amount, currency: USD }

add : Money, Money -> Result Money [CurrencyMismatch]
add = \@Money a, @Money b ->
    if a.currency != b.currency then Err CurrencyMismatch
    else Ok (@Money { amount: a.amount + b.amount, currency: a.currency })

toStr : Money -> Str
toStr = \@Money m ->
    symbol = when m.currency is
        VND -> "đ"
        USD -> "$"
        EUR -> "€"
    "$(Num.toStr m.amount)$(symbol)"
```

### Đặc điểm Value Objects

1. **Immutable** — Roc tự đảm bảo
2. **So sánh bằng giá trị** — `100VND == 100VND` ← `Eq`
3. **Không có identity** — không cần ID
4. **Self-validating** — smart constructor validate

Ví dụ thêm:

```roc
# Tất cả đều là Value Objects
EmailAddress := Str implements [Eq, Hash, Inspect]
PhoneNumber := Str implements [Eq, Hash, Inspect]
Quantity := U32 implements [Eq, Inspect]

# Address — value object phức tạp
Address := {
    street : Str,
    city : Str,
    district : Str,
    postalCode : Str,
} implements [Eq, Inspect]
```

---

## 17.4 — Entities: Có identity

Value Object là giá trị được so sánh bằng nội dung — `Money 50000` bằng `Money 50000`. Entity có identity — hai Order dù cùng items vẫn là hai đơn khác nhau.

### Entity = đối tượng có lifecycle, nhận diện bằng ID

```roc
# Entity — 2 customer cùng tên KHÁC NHAU vì ID khác
CustomerId := U64 implements [Eq, Hash, Inspect]

Customer := {
    id : CustomerId,
    name : Str,
    email : EmailAddress,
    status : [Active, Suspended Str, Banned],
} implements [Inspect]

# So sánh entities bằng ID, không phải attributes
customersEqual = \@Customer a, @Customer b ->
    a.id == b.id
```

### Entity vs Value Object

| | Value Object | Entity |
|---|---|---|
| Identity | Không (so sánh bằng giá trị) | Có (so sánh bằng ID) |
| Mutable? | Không (luôn tạo mới) | "Có" (tạo version mới) |
| Ví dụ | Money, Address, Email | Customer, Order, Product |
| Roc | Opaque type + Eq | Opaque type + ID field |

---

## 17.5 — Bounded Contexts: Ranh giới rõ ràng

Aggregate gom một nhóm objects liên quan, chỉ truy cập qua aggregate root. Order là aggregate root, OrderItem chỉ tồn tại trong context của Order.

### Cùng từ nhưng nghĩa khác

```
Từ "Product":
- Catalog team: { name, description, images, category }
- Inventory team: { sku, quantity, warehouse }
- Pricing team: { basePrice, discounts, taxRate }
- Shipping team: { weight, dimensions, isFragile }
```

Nếu nhét hết vào 1 type `Product` → **monster record** 50 fields. Thay vào đó: mỗi team có **Bounded Context** riêng.

### Trong Roc: Mỗi context = 1 module

```roc
# filename: Catalog/Product.roc
module [Product, create, describe]

Product := { id : U64, name : Str, description : Str, category : Str }

# filename: Inventory/StockItem.roc
module [StockItem, create, available]

StockItem := { productId : U64, quantity : U32, warehouse : Str }

# filename: Pricing/PricedItem.roc
module [PricedItem, create, finalPrice]

PricedItem := { productId : U64, basePrice : U64, discount : U8, taxRate : U8 }
```

Cùng `productId` nhưng **contexts khác nhau**, **data khác nhau**, **modules khác nhau**.

```
my-shop/
├── Catalog/
│   ├── Product.roc
│   └── Category.roc
├── Inventory/
│   ├── StockItem.roc
│   └── Warehouse.roc
├── Pricing/
│   ├── PricedItem.roc
│   └── Discount.roc
├── Ordering/
│   ├── Order.roc
│   ├── LineItem.roc
│   └── OrderStatus.roc
└── main.roc
```

---

## 17.6 — Aggregates: Nhóm liên quan

Domain Event thông báo "điều gì đã xảy ra" trong domain. `OrderPlaced`, `PaymentReceived`, `ItemShipped` — các module khác lắng nghe và phản ứng.

### Aggregate = nhóm entities + value objects cùng thay đổi

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# Order Aggregate — gồm Order (entity) + LineItems (value objects)
# Order là "aggregate root" — mọi thay đổi phải qua Order

addLineItem = \order, product, quantity ->
    newItem = { product, quantity, unitPrice: product.price }
    items = List.append order.items newItem
    recalculate { order & items }

removeLineItem = \order, productId ->
    items = List.keepIf order.items \item -> item.product.id != productId
    recalculate { order & items }

# Logic NỘI BỘ — mọi thay đổi items → recalculate
recalculate = \order ->
    subtotal = List.walk order.items 0 \s, item ->
        s + item.unitPrice * Num.toU64 item.quantity
    tax = subtotal * 10 // 100
    { order & subtotal, tax, total: subtotal + tax }

# Validate invariants — luôn đúng, không có trạng thái sai
validateOrder = \order ->
    if List.isEmpty order.items then
        Err EmptyOrder
    else if order.total == 0 then
        Err ZeroTotal
    else
        Ok order

main =
    emptyOrder = { id: 1, items: [], subtotal: 0, tax: 0, total: 0, status: Draft }

    phone = { id: 101, name: "iPhone", price: 25000000 }
    case_ = { id: 102, name: "Ốp lưng", price: 200000 }

    order =
        emptyOrder
        |> addLineItem phone 1
        |> addLineItem case_ 2

    Stdout.line! "=== Đơn hàng #$(Num.toStr order.id) ==="
    List.forEach order.items \item ->
        lineTotal = item.unitPrice * Num.toU64 item.quantity
        Stdout.line! "  $(item.product.name) × $(Num.toStr item.quantity) = $(Num.toStr lineTotal)đ"
    Stdout.line! "Tạm tính: $(Num.toStr order.subtotal)đ"
    Stdout.line! "Thuế (10%): $(Num.toStr order.tax)đ"
    Stdout.line! "Tổng: $(Num.toStr order.total)đ"

    # Xóa 1 item → tự recalculate
    order2 = removeLineItem order 102
    Stdout.line! "\n=== Sau khi xóa ốp lưng ==="
    Stdout.line! "Tổng: $(Num.toStr order2.total)đ"
    # Output:
    # === Đơn hàng #1 ===
    #   iPhone × 1 = 25000000đ
    #   Ốp lưng × 2 = 400000đ
    # Tạm tính: 25400000đ
    # Thuế (10%): 2540000đ
    # Tổng: 27940000đ
    #
    # === Sau khi xóa ốp lưng ===
    # Tổng: 27500000đ  (25000000 + 10% tax = 27500000)
```

### Quy tắc Aggregates

1. **Aggregate Root** — chỉ thao tác qua root (Order), không sửa LineItem trực tiếp
2. **Invariants** — aggregate đảm bảo dữ liệu luôn hợp lệ (total = sum of items + tax)
3. **Consistency boundary** — thay đổi trong aggregate phải nhất quán
4. **Nhỏ nhất có thể** — aggregate nên nhỏ, chỉ gồm data PHẢI thay đổi cùng nhau

---

## 17.7 — DDD + Roc Type System

FP và DDD ăn khớp hoàn hảo: opaque types cho value objects, tag unions cho domain events, pure functions cho business rules. Types biểu diễn domain, compiler enforce rules.

### Types encode business rules

```roc
# Business rule: Đơn hàng chỉ có thể ship khi đã thanh toán
# → Dùng type system enforce!

# ❌ Dùng string/bool — compiler không giúp
shipOrder = \order ->
    if order.isPaid then ...   # quên check → bug

# ✅ Dùng types — KHÔNG THỂ gọi sai
# order phải ở trạng thái Paid MỚI có function ship
shipPaidOrder : PaidOrder -> Result ShippedOrder [OutOfStock]

# Không có function: shipDraftOrder hay shipCancelledOrder
# → Compiler CẤM bạn ship đơn chưa thanh toán!
```

```roc
# State machine bằng types
# Draft → chỉ có thể: addItem, removeItem, submitOrder
# Submitted → chỉ có thể: confirmOrder, cancelOrder
# Confirmed → chỉ có thể: markPaid
# Paid → chỉ có thể: shipOrder
# Shipped → chỉ có thể: markDelivered

submitOrder : DraftOrder -> Result SubmittedOrder [EmptyOrder]
confirmOrder : SubmittedOrder -> ConfirmedOrder
markPaid : ConfirmedOrder -> PaidOrder
shipOrder : PaidOrder -> Result ShippedOrder [OutOfStock]
markDelivered : ShippedOrder -> DeliveredOrder
```

> **💡 Nguyên tắc DDD-FP**: "Nếu business rule quan trọng, hãy encode bằng type. Compiler sẽ enforce rule thay bạn."

---


## ✅ Checkpoint 17

> Đến đây bạn phải hiểu:
> 1. **Ubiquitous Language** = dev và business dùng cùng thuật ngữ
> 2. **Bounded Context** = phạm vi áp dụng model — cùng từ có thể khác nghĩa
> 3. **Event Storming** = workshop tìm Commands, Events, Aggregates
>
> **Test nhanh**: "Customer" trong Order context và Support context có nghĩa giống nhau không?
> <details><summary>Đáp án</summary>Không! Cùng từ nhưng khác Bounded Context = khác model.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Value Objects

Tạo các value objects cho domain "Nhà hàng":

```roc
# TableNumber — 1-50
# NumberOfGuests — 1-20
# ReservationTime — record { hour: U8, minute: U8 }
# Mỗi cái có smart constructor validate
```

<details><summary>✅ Lời giải</summary>

```roc
TableNumber := U8 implements [Eq, Hash, Inspect]

tableNumber : U8 -> Result TableNumber [InvalidTable]
tableNumber = \n ->
    if n >= 1 && n <= 50 then Ok (@TableNumber n) else Err InvalidTable

NumberOfGuests := U8 implements [Eq, Inspect]

numberOfGuests : U8 -> Result NumberOfGuests [TooFew, TooMany]
numberOfGuests = \n ->
    if n < 1 then Err TooFew
    else if n > 20 then Err TooMany
    else Ok (@NumberOfGuests n)

ReservationTime := { hour : U8, minute : U8 } implements [Eq, Inspect]

reservationTime : U8, U8 -> Result ReservationTime [InvalidTime]
reservationTime = \h, m ->
    if h >= 10 && h <= 22 && m < 60 then Ok (@ReservationTime { hour: h, minute: m })
    else Err InvalidTime
```

</details>

---

**Bài 2** (15 phút): Bounded Contexts

Thiết kế bounded contexts cho hệ thống "Bệnh viện":

```
Context 1: Đăng ký khám — Patient (name, phone, insurance)
Context 2: Khám bệnh — Patient (id, symptoms, history)
Context 3: Thanh toán — Patient (id, services, totalDue)
```

Tạo 3 modules với types khác nhau cho "Patient".

<details><summary>✅ Lời giải</summary>

```roc
# filename: Registration/Patient.roc
module [Patient, register]
Patient := { name : Str, phone : Str, insuranceId : Str }

# filename: Examination/Patient.roc
module [Patient, startExam]
Patient := { id : U64, symptoms : List Str, history : List Str }

# filename: Billing/Patient.roc
module [Patient, createBill]
Patient := { id : U64, services : List { name : Str, cost : U64 }, totalDue : U64 }
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| "Không biết đặt tên gì" | Thiếu domain knowledge | Nói chuyện với domain expert, dùng TỪ NGỮ CỦA HỌ |
| Aggregate quá lớn | Nhồi quá nhiều entities | Tách thành nhiều aggregates nhỏ hơn |
| Bounded contexts trùng lặp code | Cùng concept nhưng khác context | OK — đây là TÍNH NĂNG, không phải bug. Mỗi context có needs riêng |
| Types quá phức tạp | Over-engineering | Bắt đầu đơn giản, thêm complexity khi cần |

---

## Tóm tắt

- ✅ **DDD** = code phản ánh domain. "Model is code, code is model."
- ✅ **Ubiquitous Language** = dev + business dùng cùng thuật ngữ. `placeOrder` không phải `processData`.
- ✅ **Value Objects** = opaque types + smart constructors. So sánh bằng giá trị, immutable, self-validating.
- ✅ **Entities** = có identity (ID). Lifecycle, có thể thay đổi state.
- ✅ **Bounded Contexts** = ranh giới. Cùng "Product" nhưng nghĩa khác ở Catalog vs Inventory.
- ✅ **Aggregates** = nhóm entity + value objects. Thay đổi qua root, đảm bảo invariants.
- ✅ **Types encode business rules** = compiler enforce domain logic.

## Tiếp theo

→ Chapter 18: **Domain Modeling** — biến domain analysis thành code Roc. Opaque types = value objects, tag unions = state machines, modules = bounded contexts. Xây hệ thống thực tế từ A→Z.
