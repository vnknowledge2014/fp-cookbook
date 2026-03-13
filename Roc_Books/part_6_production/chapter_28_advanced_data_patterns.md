# Chapter 28 — Advanced Data Patterns

> **Bạn sẽ học được**:
> - **Migrations** — versioned SQL changes, up/down
> - **Event Store** — append-only database cho Event Sourcing
> - **CQRS** — Command Query Responsibility Segregation
> - **Caching** — strategies, invalidation, Roc patterns
> - **Repository Pattern** — tách domain khỏi storage
>
> **Yêu cầu trước**: [Chapter 27 — Database Fundamentals](chapter_27_database_fundamentals.md)
> **Thời gian đọc**: ~45 phút | **Level**: Principal
> **Kết quả cuối cùng**: Áp dụng data patterns nâng cao — event store, CQRS, caching — trong kiến trúc Roc.

---

Schema thay đổi — đó là chuyện tất yếu. Từ migration strategies đến CQRS, từ materialized views đến event store, chapter này trang bị những patterns bạn cần khi dự án vượt qua giai đoạn prototype.

## Advanced Data Patterns — Vượt qua CRUD

Migrations, caching, event stores, CQRS — patterns cho production data management. Roc's immutability làm event sourcing implementation tự nhiên. Chapter này mở rộng toolkit data của bạn.


## 28.1 — Database Migrations

Schema thay đổi theo thời gian — thêm column, đổi type, tạo index. Migrations quản lý thay đổi này một cách có version — mỗi migration có up (áp dụng) và down (rollback).

### Vấn đề: Schema thay đổi theo thời gian

```
v1: customers (id, name)
v2: customers (id, name, email)          ← thêm email
v3: customers (id, name, email, phone)   ← thêm phone
```

### Migration = versioned SQL files

```
migrations/
├── 001_create_customers.sql
├── 002_add_email.sql
├── 003_add_phone.sql
└── 004_create_orders.sql
```

```sql
-- 001_create_customers.sql
-- UP
CREATE TABLE customers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- DOWN
DROP TABLE customers;
```

```sql
-- 002_add_email.sql
-- UP
ALTER TABLE customers ADD COLUMN email TEXT UNIQUE;

-- DOWN
-- SQLite không hỗ trợ DROP COLUMN trực tiếp
-- Cần tạo table mới → copy data → đổi tên
```

### Migration tracker trong Roc

```roc
# ═══════ PURE: Migration tracker ═══════

Migration : { version : U64, name : Str, upSql : Str, downSql : Str }

migrations = [
    {
        version: 1,
        name: "create_customers",
        upSql: "CREATE TABLE customers (id INTEGER PRIMARY KEY, name TEXT NOT NULL);",
        downSql: "DROP TABLE customers;",
    },
    {
        version: 2,
        name: "add_email",
        upSql: "ALTER TABLE customers ADD COLUMN email TEXT;",
        downSql: "",  # not reversible in SQLite
    },
    {
        version: 3,
        name: "create_orders",
        upSql:
            """
            CREATE TABLE orders (
                id INTEGER PRIMARY KEY,
                customer_id INTEGER REFERENCES customers(id),
                total INTEGER NOT NULL DEFAULT 0,
                status TEXT NOT NULL DEFAULT 'draft'
            );
            """,
        downSql: "DROP TABLE orders;",
    },
]

# Tìm migrations chưa chạy
pendingMigrations = \allMigrations, currentVersion ->
    List.keepIf allMigrations \m -> m.version > currentVersion

# Tests
expect
    pending = pendingMigrations migrations 0
    List.len pending == 3

expect
    pending = pendingMigrations migrations 2
    List.len pending == 1

expect
    pending = pendingMigrations migrations 3
    List.len pending == 0
```

---

## 28.2 — Event Store: Append-Only Database

Thay vì UPDATE row, event store chỉ INSERT events mới. State hiện tại = replay tất cả events từ đầu. Pattern này cho phép audit trail, time travel, và rebuild state bất kỳ lúc nào.

### Event Sourcing + Database

```sql
-- Event Store = append-only table
-- KHÔNG BAO GIỜ UPDATE hoặc DELETE

CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    stream_id TEXT NOT NULL,           -- "order-123", "account-456"
    event_type TEXT NOT NULL,          -- "OrderPlaced", "MoneyDeposited"
    event_data TEXT NOT NULL,          -- JSON payload
    metadata TEXT,                      -- timestamp, user, version
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_events_stream ON events(stream_id);
CREATE INDEX idx_events_type ON events(event_type);

-- Append event
INSERT INTO events (stream_id, event_type, event_data)
VALUES ('order-1', 'OrderPlaced', '{"items":["Phở","Cà phê"],"total":70000}');

INSERT INTO events (stream_id, event_type, event_data)
VALUES ('order-1', 'OrderConfirmed', '{}');

-- Read events for a stream (ordered!)
SELECT * FROM events WHERE stream_id = 'order-1' ORDER BY id;
```

### Roc: Event Store helpers

```roc
# ═══════ PURE: Event serialization ═══════

eventToSQL = \streamId, event ->
    { eventType, eventData } = serializeEvent event
    buildInsert "events"
        ["stream_id", "event_type", "event_data"]
        [streamId, eventType, eventData]

serializeEvent = \event ->
    when event is
        OrderPlaced { items, total } ->
            itemsJson = List.map items \i -> "\"$(i)\""
                |> Str.joinWith ","
            { eventType: "OrderPlaced", eventData: "{\"items\":[$(itemsJson)],\"total\":$(Num.toStr total)}" }
        OrderConfirmed ->
            { eventType: "OrderConfirmed", eventData: "{}" }
        OrderShipped { trackingCode } ->
            { eventType: "OrderShipped", eventData: "{\"tracking\":\"$(trackingCode)\"}" }
        OrderCompleted ->
            { eventType: "OrderCompleted", eventData: "{}" }

# Test
expect
    sql = eventToSQL "order-42" (OrderPlaced { items: ["Phở"], total: 45000 })
    Str.contains sql "order-42" && Str.contains sql "OrderPlaced"

# ═══════ PURE: Read model builder ═══════

buildStreamQuery = \streamId ->
    "SELECT event_type, event_data FROM events WHERE stream_id = '$(streamId)' ORDER BY id;"

buildAllStreamsQuery = \eventType ->
    "SELECT DISTINCT stream_id FROM events WHERE event_type = '$(eventType)';"

expect
    buildStreamQuery "order-1"
    == "SELECT event_type, event_data FROM events WHERE stream_id = 'order-1' ORDER BY id;"
```

---

## 28.3 — CQRS: Command Query Separation

CQRS tách read và write thành hai paths khác nhau. Commands thay đổi state (write), Queries đọc state (read). Tách rời cho phép optimize mỗi path riêng — write model cho consistency, read model cho performance.

### Pattern: Write path ≠ Read path

```
                    ┌─────────────┐
Commands ──────────→│ Event Store  │ (append-only, normalized)
(write)             │ (source of  │
                    │   truth)    │
                    └──────┬──────┘
                           │ project
                    ┌──────▼──────┐
Queries ←──────────│ Read Models  │ (denormalized, fast)
(read)              │ (projections)│
                    └─────────────┘
```

```sql
-- WRITE: Events table (normalized, append-only)
INSERT INTO events (stream_id, event_type, event_data)
VALUES ('order-1', 'OrderPlaced', '{"total":70000}');

-- READ: Projection table (denormalized, optimized for queries)
-- Rebuilt from events

CREATE TABLE order_summary (
    order_id TEXT PRIMARY KEY,
    customer_name TEXT,
    total INTEGER,
    item_count INTEGER,
    status TEXT,
    last_updated TEXT
);

-- Projection: Doanh thu theo ngày
CREATE TABLE daily_revenue (
    date TEXT PRIMARY KEY,
    total_revenue INTEGER DEFAULT 0,
    order_count INTEGER DEFAULT 0
);
```

### Roc: CQRS pattern

```roc
# ═══════ WRITE SIDE: Commands → Events ═══════

processCommand = \state, command ->
    when command is
        PlaceOrder { items, total } ->
            if List.isEmpty items then Err EmptyOrder
            else Ok (OrderPlaced { items, total })
        ConfirmOrder ->
            if state.status != Placed then Err (InvalidStatus state.status)
            else Ok OrderConfirmed
        ShipOrder { trackingCode } ->
            if state.status != Confirmed then Err (InvalidStatus state.status)
            else Ok (OrderShipped { trackingCode })

# ═══════ READ SIDE: Events → Projections ═══════

# Projection 1: Order summary (cho UI)
projectOrderSummary = \events ->
    List.walk events { status: "unknown", total: 0, itemCount: 0 } \summary, event ->
        when event is
            OrderPlaced { items, total } ->
                { summary & status: "placed", total, itemCount: List.len items }
            OrderConfirmed ->
                { summary & status: "confirmed" }
            OrderShipped _ ->
                { summary & status: "shipped" }
            OrderCompleted ->
                { summary & status: "completed" }

# Projection 2: Daily stats (cho báo cáo)
projectDailyStats = \events ->
    completed = List.keepIf events \e ->
        when e is
            OrderCompleted -> Bool.true
            _ -> Bool.false
    { completedOrders: List.len completed }

# Tests
expect
    events = [OrderPlaced { items: ["Phở", "Cà phê"], total: 70000 }, OrderConfirmed]
    summary = projectOrderSummary events
    summary.status == "confirmed" && summary.total == 70000 && summary.itemCount == 2
```

---

## 28.4 — Caching Patterns

Database queries chậm khi scale. Cache — lưu kết quả query tạm thời — giảm load database và tăng tốc response. 3 patterns chính: cache-aside, write-through, write-behind.

### Cache = tránh tính toán lại hoặc query lại

```roc
# ═══════ PURE: Cache as Dict ═══════

# Cache = Dict key value
# Lookup: Dict.get cache key
# Miss: compute, store, return
# Hit: return cached value

lookupOrCompute = \cache, key, computeFn ->
    when Dict.get cache key is
        Ok value -> { cache, value }          # HIT
        Err _ ->
            value = computeFn key
            newCache = Dict.insert cache key value
            { cache: newCache, value }        # MISS → computed

# Test
expect
    cache = Dict.empty {}
    expensive = \k -> k * k * k    # giả sử tính toán tốn kém

    # Miss lần đầu
    r1 = lookupOrCompute cache 5 expensive
    r1.value == 125

    # Hit lần sau
    && Dict.get r1.cache 5 == Ok 125
```

### Cache Strategies

```roc
# ═══════ Strategy 1: LRU (Least Recently Used) ═══════

# LRU cache = Dict + access order
# Khi đầy → xóa phần tử ít dùng nhất

# ═══════ Strategy 2: TTL (Time To Live) ═══════

# Mỗi entry có expiry time
# Khi lookup: kiểm tra expired?

isExpired = \entry, currentTime ->
    currentTime > entry.expiresAt

lookupWithTTL = \cache, key, currentTime ->
    when Dict.get cache key is
        Ok entry ->
            if isExpired entry currentTime then
                Err Expired   # cần recompute
            else
                Ok entry.value
        Err _ ->
            Err NotFound

# ═══════ Strategy 3: Write-Through ═══════

# Write → update cache AND database
# Read → always from cache (fast)

# ═══════ Strategy 4: Cache-Aside ═══════

# Read: check cache → miss? → query DB → store in cache
# Write: update DB → invalidate cache
```

### Khi nào cache?

| Nên cache | KHÔNG nên cache |
|-----------|-----------------|
| Query phức tạp, chạy thường xuyên | Data thay đổi liên tục |
| Kết quả ít thay đổi | Mỗi request data khác nhau |
| Tính toán tốn kém | Tính toán đơn giản |
| Read >> Write | Write >> Read |

---

## 28.5 — Repository Pattern

Repository ẩn chi tiết storage đằng sau một interface đơn giản: `findById`, `save`, `delete`. Domain logic không biết data ở database, file, hay API — nó chỉ gọi repository.

### Tách domain khỏi storage

```roc
# ═══════ PURE: Repository interface (concepts) ═══════

# Trong Roc không có interface, nhưng dùng records of functions

# "Repository" = record chứa các functions
# Mỗi implementation cung cấp functions khác nhau

# In-Memory Repository (cho testing)
inMemoryOrderRepo = \initialOrders -> {
    findById: \orders, id ->
        List.findFirst orders \o -> o.id == id
        |> Result.mapErr \_ -> NotFound,

    findAll: \orders -> orders,

    findByStatus: \orders, status ->
        List.keepIf orders \o -> o.status == status,

    save: \orders, order ->
        existing = List.keepIf orders \o -> o.id != order.id
        List.append existing order,

    delete: \orders, id ->
        List.keepIf orders \o -> o.id != id,
}

# Tests — dùng in-memory, không cần DB!
expect
    repo = inMemoryOrderRepo []
    orders = []

    order1 = { id: 1, status: "placed", total: 50000 }
    orders1 = repo.save orders order1

    List.len orders1 == 1

expect
    repo = inMemoryOrderRepo []
    orders = [
        { id: 1, status: "placed", total: 50000 },
        { id: 2, status: "completed", total: 30000 },
        { id: 3, status: "placed", total: 70000 },
    ]

    placed = repo.findByStatus orders "placed"
    List.len placed == 2
```

### Architecture

```
┌──────────────────────────────────────┐
│              main.roc (Shell)         │
│                                       │
│  repo = sqliteOrderRepo db            │
│  orders = repo.findAll!               │
│  processed = processOrders orders     │  ← pure!
│  repo.save! processed                 │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│        Domain (Pure)                  │
│  processOrders = \orders -> ...       │
│  Tested with inMemoryRepo            │
└──────────────────────────────────────┘
```

---


## ✅ Checkpoint 28

> Đến đây bạn phải hiểu:
> 1. Migrations = schema versioning — evolve database structure safely
> 2. Event Store = append-only — rebuild state bằng `List.walk` (FP style!)
> 3. Caching = platform concern — app stays pure, platform manages cache
>
> **Test nhanh**: Event Store và traditional DB khác nhau thế nào?
> <details><summary>Đáp án</summary>Traditional DB lưu state cuối, Event Store lưu CHUỖI events. State = fold events.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Migration plan

Thiết kế migrations cho e-commerce:

```
v1: products (id, name, price)
v2: + categories table, products.category_id FK
v3: + order_items table (order_id, product_id, qty, price)
v4: + products.stock_count column
```

<details><summary>✅ Lời giải</summary>

```roc
ecommerceMigrations = [
    { version: 1, name: "create_products",
      upSql: "CREATE TABLE products (id INTEGER PRIMARY KEY, name TEXT, price INTEGER);",
      downSql: "DROP TABLE products;" },
    { version: 2, name: "add_categories",
      upSql: "CREATE TABLE categories (id INTEGER PRIMARY KEY, name TEXT); ALTER TABLE products ADD COLUMN category_id INTEGER REFERENCES categories(id);",
      downSql: "DROP TABLE categories;" },
    { version: 3, name: "create_order_items",
      upSql: "CREATE TABLE order_items (id INTEGER PRIMARY KEY, order_id INTEGER REFERENCES orders(id), product_id INTEGER REFERENCES products(id), qty INTEGER, price INTEGER);",
      downSql: "DROP TABLE order_items;" },
    { version: 4, name: "add_stock",
      upSql: "ALTER TABLE products ADD COLUMN stock_count INTEGER DEFAULT 0;",
      downSql: "" },
]
```

</details>

---

**Bài 2** (15 phút): Event Store + Projections

Viết event store cho bank account với 2 projections:

```roc
# Events: AccountOpened, Deposited, Withdrawn, AccountClosed
# Projection 1: Current balance
# Projection 2: Transaction history (list of {type, amount, runningBalance})
```

<details><summary>✅ Lời giải</summary>

```roc
projectBalance = \events ->
    List.walk events 0 \bal, event ->
        when event is
            AccountOpened { deposit } -> deposit
            Deposited { amount } -> bal + amount
            Withdrawn { amount } -> bal - amount
            AccountClosed -> 0

projectHistory = \events ->
    List.walk events { balance: 0, history: [] } \state, event ->
        when event is
            Deposited { amount } ->
                newBal = state.balance + amount
                { balance: newBal, history: List.append state.history { type: "deposit", amount, runningBalance: newBal } }
            Withdrawn { amount } ->
                newBal = state.balance - amount
                { balance: newBal, history: List.append state.history { type: "withdraw", amount, runningBalance: newBal } }
            AccountOpened { deposit } ->
                { balance: deposit, history: [{ type: "opened", amount: deposit, runningBalance: deposit }] }
            AccountClosed ->
                { state & history: List.append state.history { type: "closed", amount: 0, runningBalance: 0 } }
    |> .history

expect
    events = [AccountOpened { deposit: 1000 }, Deposited { amount: 500 }, Withdrawn { amount: 200 }]
    projectBalance events == 1300
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Migration chạy 2 lần | Thiếu tracking table | Lưu applied versions trong `schema_migrations` table |
| Event Store quá lớn | Millions of events | Dùng snapshots (Chapter 16) |
| Cache stale | Quên invalidate | Write-through hoặc TTL ngắn |
| CQRS read model lỗi | Projection bug | Rebuild từ events — source of truth |

---

## Tóm tắt

- ✅ **Migrations** = versioned SQL files, up/down, tracked. Schema evolves safely.
- ✅ **Event Store** = append-only table. stream_id + event_type + event_data. Không UPDATE/DELETE.
- ✅ **CQRS** = write path (commands → events) ≠ read path (events → projections). Optimized separately.
- ✅ **Caching** = Dict-based in Roc. Strategies: LRU, TTL, write-through, cache-aside.
- ✅ **Repository** = records of functions. In-memory cho tests, SQLite cho production.
- ✅ **Roc advantage**: platform handles IO (DB, cache). Domain stays **pure** and **testable**.

## Tiếp theo

→ Chapter 29: **Security Essentials** — password hashing, JWT tokens, OAuth 2.0, type-safe security.
