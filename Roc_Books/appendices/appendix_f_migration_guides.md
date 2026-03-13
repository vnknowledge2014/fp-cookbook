# Appendix F — Migration Guides

> Hướng dẫn chuyển đổi từ Python, JavaScript, và Rust sang Roc.
> (Elm/Gleam → Roc xem [Appendix B](appendix_b_elm_gleam_translation.md))

---

## F.1 — Python → Roc

### Mindset shift

| Python | Roc | Ghi chú |
|--------|-----|---------|
| Everything mutable | Everything immutable | Record update tạo bản mới |
| Classes + methods | Records + functions | Không có OOP |
| `try/except` | `Result ok err` + `?` | Lỗi explicit trong type |
| `None` | Không tồn tại | Dùng `Result` hoặc tag union |
| `import os; os.read(...)` | Qua Platform + `!` | App không trực tiếp access IO |
| Duck typing | Structural typing | Compiler check tại compile time |
| `pip install` | Platform URL | Không có package manager riêng |

### Syntax side-by-side

```python
# Python
def process_order(items, discount=0):
    subtotal = sum(item["price"] * item["qty"] for item in items)
    if discount > 0:
        total = subtotal * (1 - discount / 100)
    else:
        total = subtotal
    
    if total <= 0:
        raise ValueError("Invalid total")
    
    return {"items": items, "total": total, "status": "confirmed"}
```

```roc
# Roc
processOrder = \items, discount ->
    subtotal = List.walk items 0 \s, item -> s + item.price * item.qty

    total =
        if discount > 0 then
            subtotal * (100 - discount) // 100
        else
            subtotal

    if total <= 0 then
        Err InvalidTotal
    else
        Ok { items, total, status: Confirmed }

# Tests — không cần pytest!
expect
    items = [{ price: 100, qty: 2 }, { price: 50, qty: 1 }]
    processOrder items 10 == Ok { items, total: 225, status: Confirmed }

expect
    processOrder [] 0 == Ok { items: [], total: 0, status: Confirmed }
    |> Result.isErr    # total = 0 → Err InvalidTotal
```

### Common Python patterns → Roc

```python
# Python: list comprehension
evens = [x for x in range(10) if x % 2 == 0]

# Roc: List.keepIf
evens = List.range { start: At 0, end: Before 10 }
    |> List.keepIf \x -> x % 2 == 0

# Python: dict
config = {"host": "localhost", "port": 8080}
host = config.get("host", "default")

# Roc: Dict
config = Dict.fromList [("host", "localhost"), ("port", "8080")]
host = Dict.get config "host" |> Result.withDefault "default"

# Python: class
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
    
    def greet(self):
        return f"Hello, {self.name}!"

# Roc: record + function
createUser = \name, email -> { name, email }
greet = \user -> "Hello, $(user.name)!"

# Python: try/except
try:
    result = int(input("Number: "))
except ValueError:
    result = 0

# Roc: Result
result = Str.toI64 inputStr |> Result.withDefault 0
```

---

## F.2 — JavaScript/TypeScript → Roc

### Mindset shift

| JS/TS | Roc | Ghi chú |
|-------|-----|---------|
| `let`/`const`/`var` | `x = 5` (luôn immutable) | Không có `let mut` |
| `null`/`undefined` | Không tồn tại | `Result` thay `null` |
| `throw Error` | `Err tag` | Lỗi trong type system |
| `async/await` | `Task` + `!` | Platform handles async |
| `Promise` | `Task ok err` | Tương tự, nhưng typed errors |
| `this` keyword | Không có | Functions standalone |
| `npm install` | Platform URL | Dependencies qua URL |
| Runtime errors | Compile-time errors | Compiler bắt hầu hết lỗi |

### Syntax side-by-side

```javascript
// JavaScript
const processPayment = async (amount, method) => {
  if (amount <= 0) throw new Error("Invalid amount");
  
  switch (method) {
    case "card":
      const result = await chargeCard(amount);
      return { ...result, status: "paid" };
    case "cash":
      return { amount, status: "paid", change: 0 };
    default:
      throw new Error(`Unknown method: ${method}`);
  }
};
```

```roc
# Roc
processPayment = \amount, method ->
    if amount <= 0 then
        Err InvalidAmount
    else
        when method is
            Card ->
                result = chargeCard? amount    # ? = early return on Err
                Ok { result & status: Paid }
            Cash ->
                Ok { amount, status: Paid, change: 0 }
            # Compiler FORCES you to handle all cases!
            # No "default" → no forgotten cases
```

### Common JS patterns → Roc

```javascript
// JS: array methods
const result = items
  .filter(x => x.active)
  .map(x => x.price * x.qty)
  .reduce((sum, x) => sum + x, 0);

// Roc: same pipeline style!
result = items
    |> List.keepIf \x -> x.active
    |> List.map \x -> x.price * x.qty
    |> List.walk 0 \sum, x -> sum + x

// JS: optional chaining
const name = user?.profile?.name ?? "Anonymous";

// Roc: explicit Result handling
name =
    when Dict.get users userId is
        Ok user ->
            when Dict.get user.profile "name" is
                Ok n -> n
                Err _ -> "Anonymous"
        Err _ -> "Anonymous"

// TS: interface
interface User {
  name: string;
  age: number;
  email?: string;
}

// Roc: record type (structural!)
User : { name : Str, age : U32, email : Result Str [NoEmail] }
# Hoặc đơn giản hơn:
user = { name: "An", age: 25 }
# Compiler tự suy type!
```

---

## F.3 — Rust → Roc

### Mindset shift

| Rust | Roc | Ghi chú |
|------|-----|---------|
| `let mut` / `&mut` | Tự động (ref counting) | Roc auto in-place khi unique |
| `enum` (nominal) | Tag union (structural) | Không cần `enum` khai báo |
| `struct` (nominal) | Record (structural) | Không cần `struct` khai báo |
| `trait` | Ability (limited) | Chỉ built-in: Eq, Hash, Inspect, Encode, Decode |
| `impl` blocks | Functions standalone | Không method syntax |
| `Result<T, E>` | `Result ok err` | Tương tự! |
| `Option<T>` | `Result a [NotFound]` | Roc không có Option |
| `match` (exhaustive) | `when...is` (exhaustive) | Rất giống! |
| Lifetime annotations | Không cần | Ref counting handles |
| `cargo` | `roc run/build/test` | Simpler toolchain |
| `println!` macro | `Stdout.line!` task | `!` = effectful |

### Syntax side-by-side

```rust
// Rust
use std::collections::HashMap;

#[derive(Debug)]
enum OrderStatus {
    Draft,
    Confirmed,
    Shipped,
    Delivered,
}

struct Order {
    id: u64,
    items: Vec<String>,
    status: OrderStatus,
    total: u64,
}

impl Order {
    fn new(id: u64) -> Self {
        Order { id, items: vec![], status: OrderStatus::Draft, total: 0 }
    }

    fn add_item(&mut self, item: String, price: u64) {
        self.items.push(item);
        self.total += price;
    }

    fn confirm(&mut self) -> Result<(), String> {
        match self.status {
            OrderStatus::Draft => {
                self.status = OrderStatus::Confirmed;
                Ok(())
            }
            _ => Err("Can only confirm draft orders".to_string()),
        }
    }
}
```

```roc
# Roc — NO struct/enum declarations needed!

createOrder = \id ->
    { id, items: [], status: Draft, total: 0 }

addItem = \order, item, price ->
    { order &
        items: List.append order.items item,
        total: order.total + price,
    }

confirmOrder = \order ->
    when order.status is
        Draft -> Ok { order & status: Confirmed }
        _ -> Err (InvalidTransition "Can only confirm draft orders")

# Tests — inline!
expect
    order = createOrder 1
        |> addItem "Phở" 45000
        |> addItem "Cà phê" 25000
    order.total == 70000

expect
    order = createOrder 1
    confirmOrder order |> Result.isOk

expect
    order = { (createOrder 1) & status: Confirmed }
    confirmOrder order |> Result.isErr
```

### Key differences for Rust developers

```
1. NO lifetimes     — Roc uses reference counting
2. NO mut           — immutable by default, in-place update automatic
3. NO trait impls   — abilities are derived, not implemented
4. NO match arms ;  — just indentation
5. NO enum/struct   — structural types, just write the data
6. NO cargo.toml    — platforms via URL in `app` declaration
7. NO std::io       — IO only through platform
```

---

## F.4 — Cheat Sheet tổng hợp

```
Python dev:
  ✅ Cú pháp dễ quen — ít ceremony hơn Python!
  ⚠️ Không class/OOP — dùng records + functions
  ⚠️ Không mutable — record update tạo bản mới
  ⭐ Bắt đầu: Ch0 → Part I → Ch11 (Purity = game changer)

JS/TS dev:
  ✅ Pipeline |> giống pipe operator proposal
  ✅ Pattern matching giống TC39 proposal
  ⚠️ Không null/undefined — dùng Result
  ⚠️ Không throw — lỗi trong type system
  ⭐ Bắt đầu: Ch0 → Ch7 (Tags) → Ch12 (Tasks = typed Promises)

Rust dev:
  ✅ Result/match rất giống — feel at home
  ✅ Không lifetimes — ref counting tự động
  ⚠️ Structural typing — không cần declare enums/structs
  ⚠️ Limited abilities — không custom traits
  ⭐ Bắt đầu: Ch0 → Ch7 (structural types) → Ch20 (Platform = unique)
```
