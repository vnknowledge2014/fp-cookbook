# Chapter 27 — Database Fundamentals & SQL

> **Bạn sẽ học được**:
> - **Relational model** — tables, columns, keys, relationships
> - **SQL** — SELECT, INSERT, UPDATE, DELETE, JOIN, CTEs
> - **Normalization** — 1NF, 2NF, 3NF — tại sao cần
> - **Indexing** — B-Tree, composite indexes, khi nào dùng
> - **Transactions** — ACID, isolation levels
> - Kết nối Roc với SQLite qua platform
>
> **Yêu cầu trước**: [Chapter 26 — Capstone](../part_5_testing_and_apps/chapter_26_capstone.md)
> **Thời gian đọc**: ~50 phút | **Level**: Principal
> **Kết quả cuối cùng**: Hiểu database fundamentals, viết SQL thành thạo, kết nối từ Roc app.

> [!NOTE]
> **📢 Về Part VI**: Roc ecosystem đang phát triển nhanh. Nhiều patterns trong Part VI tập trung vào **pure domain logic** (testable, portable) trong khi platform integrations (DB drivers, crypto, etc.) mở rộng theo thời gian. Code ở đây ưu tiên minh họa **tư duy kiến trúc** — cách tách pure logic khỏi infrastructure — hơn là production-ready libraries.

---

Mọi ứng dụng thực tế đều cần lưu trữ dữ liệu lâu dài. Chapter này dạy SQL và relational model từ góc nhìn FP — bạn sẽ thấy SQL queries giống pipeline (SELECT → WHERE → ORDER BY) và database schema giống domain model.

## Database Fundamentals — Persistence cho Roc

Roc applications cần lưu data — thông qua platform capabilities cho database access. Chapter này dạy SQL fundamentals và cách Roc interact với databases qua platform model, giữ application code pure.


## 27.1 — Relational Model

Relational model tổ chức data thành tables (relations), rows (tuples), columns (attributes). Mỗi row là unique, mỗi column có type cố định — giống records trong Roc.

### Tại sao cần database?

```
Roc app                         Database
┌────────────┐     save     ┌──────────────┐
│ Dict/List  │ ──────────→  │ Tables        │
│ (in memory)│ ←──────────  │ (on disk)     │
└────────────┘     load     │ Durable!      │
  ↑ mất khi                 │ Queryable!    │
  restart                   │ Concurrent!   │
                            └──────────────┘
```

In-memory (Dict, List) → mất khi app restart. Database → dữ liệu *durable*, *queryable*, *concurrent*.

### Tables = Records có cấu trúc

```sql
-- Mỗi table = List of records có cùng schema
-- Mỗi row = 1 record
-- Mỗi column = 1 field

CREATE TABLE customers (
    id INTEGER PRIMARY KEY,    -- Primary Key: identify duy nhất
    name TEXT NOT NULL,         -- NOT NULL: bắt buộc
    email TEXT UNIQUE,          -- UNIQUE: không trùng
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

Tương đương Roc:

```roc
# Tương tự 1 record trong List
customer = { id: 1, name: "An", email: "an@mail.com", createdAt: "2024-03-15" }
```

### Keys & Relationships

```sql
-- Primary Key: identity (như Entity ID trong DDD)
-- Foreign Key: relationship (đơn hàng THUỘC VỀ khách hàng)

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,       -- FK → customers
    total INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft',
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- 1 Customer → N Orders (One-to-Many)
-- 1 Order → 1 Customer
```

```
┌──────────┐       ┌──────────┐       ┌────────────┐
│ customers│ 1───N │  orders  │ 1───N │ order_items │
│ id, name │       │ id, cust │       │ id, order  │
│ email    │       │ total    │       │ product    │
└──────────┘       └──────────┘       │ qty, price │
                                       └────────────┘
```

---

## 27.2 — SQL CRUD

SQL là ngôn ngữ declarative — bạn nói "muốn gì" không nói "làm thế nào". SELECT giống `List.map`, WHERE giống `List.keepIf`, GROUP BY giống `List.walk` — FP mindset áp dụng trực tiếp.

### CREATE (INSERT)

```sql
-- Thêm 1 row
INSERT INTO customers (name, email) VALUES ('An', 'an@mail.com');
INSERT INTO customers (name, email) VALUES ('Bình', 'binh@mail.com');

-- Thêm nhiều rows
INSERT INTO orders (customer_id, total, status) VALUES
    (1, 150000, 'confirmed'),
    (1, 75000, 'draft'),
    (2, 200000, 'completed');
```

### READ (SELECT)

```sql
-- Lấy tất cả
SELECT * FROM customers;

-- Lọc
SELECT name, email FROM customers WHERE id = 1;

-- Sắp xếp
SELECT * FROM orders ORDER BY total DESC;

-- Giới hạn
SELECT * FROM orders LIMIT 10 OFFSET 0;

-- Đếm
SELECT COUNT(*) AS total_orders FROM orders;
SELECT status, COUNT(*) AS count FROM orders GROUP BY status;
-- status   | count
-- draft    | 1
-- confirmed| 1
-- completed| 1

-- Tổng
SELECT SUM(total) AS revenue FROM orders WHERE status = 'completed';
```

### UPDATE

```sql
-- Sửa 1 row
UPDATE orders SET status = 'confirmed' WHERE id = 2;

-- Sửa nhiều rows
UPDATE orders SET status = 'cancelled' WHERE status = 'draft' AND customer_id = 1;
```

### DELETE

```sql
-- Xóa 1 row
DELETE FROM orders WHERE id = 3;

-- Xóa hàng loạt (cẩn thận!)
DELETE FROM orders WHERE status = 'cancelled';
```

---

## 27.3 — JOIN: Kết hợp tables

JOIN kết nối data từ nhiều tables. INNER JOIN = chỉ giữ rows khớp, LEFT JOIN = giữ tất cả rows bên trái. Giống `List.keepOks` vs `List.map` — pattern quen thuộc.

```sql
-- INNER JOIN: chỉ rows MATCH cả 2 bảng
SELECT
    c.name AS customer,
    o.id AS order_id,
    o.total,
    o.status
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;

-- Kết quả:
-- customer | order_id | total  | status
-- An       | 1        | 150000 | confirmed
-- An       | 2        | 75000  | draft
-- Bình     | 3        | 200000 | completed

-- LEFT JOIN: tất cả customers, kể cả không có orders
SELECT
    c.name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id;
```

### Trong FP, JOIN ≈ merge 2 lists

```roc
# Tương đương INNER JOIN trong Roc
innerJoin = \customers, orders ->
    List.joinMap orders \order ->
        List.keepOks customers \customer ->
            if customer.id == order.customerId then
                Ok { customer: customer.name, orderId: order.id, total: order.total }
            else
                Err NotMatch
```

---

## 27.4 — CTEs: Tổ chức query phức tạp

Indexes tăng tốc truy vấn — từ O(n) scan toàn bộ table xuống O(log n) lookup. Giống Dict lookup thay vì List.findFirst — tradeoff: tốn bộ nhớ, tăng tốc đọc.

```sql
-- CTE = "let binding" trong SQL
-- Giống Roc: name = expression

-- Bước 1: Tính revenue per customer
-- Bước 2: Xếp hạng
-- Bước 3: Lấy top 5

WITH customer_revenue AS (
    -- Bước 1
    SELECT
        c.id,
        c.name,
        SUM(o.total) AS total_revenue,
        COUNT(o.id) AS order_count
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.id
),
ranked AS (
    -- Bước 2
    SELECT *,
        ROW_NUMBER() OVER (ORDER BY total_revenue DESC) AS rank
    FROM customer_revenue
)
-- Bước 3
SELECT name, total_revenue, order_count, rank
FROM ranked
WHERE rank <= 5;
```

> **💡 CTE trong Roc**: Giống hệt pipeline! Mỗi `WITH` clause = 1 step.

```roc
# Tương đương CTE
topCustomers = \customers, orders ->
    # Step 1: revenue per customer (= CTE customer_revenue)
    customerRevenue = List.map customers \c ->
        customerOrders = List.keepIf orders \o -> o.customerId == c.id && o.status == Completed
        revenue = List.walk customerOrders 0 \s, o -> s + o.total
        count = List.len customerOrders
        { name: c.name, totalRevenue: revenue, orderCount: count }

    # Step 2: rank (= CTE ranked)
    ranked = customerRevenue
        |> List.sortWith \a, b -> Num.compare b.totalRevenue a.totalRevenue

    # Step 3: top 5
    List.takeFirst ranked 5
```

---

## 27.5 — Normalization

Transactions đảm bảo consistency — nhóm nhiều operations thành 1 đơn vị: tất cả thành công hoặc tất cả rollback. ACID properties: Atomicity, Consistency, Isolation, Durability.

### 1NF: Không list trong cell

```sql
-- ❌ Vi phạm 1NF
-- | id | name | phones                |
-- | 1  | An   | 0901234567,0912345678 |  ← LIST trong 1 cell!!

-- ✅ 1NF — tách table
CREATE TABLE phones (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    phone TEXT NOT NULL
);
```

### 2NF: Mọi column phụ thuộc TOÀN BỘ primary key

```sql
-- ❌ Vi phạm 2NF (composite key nhưng product_name chỉ phụ thuộc product_id)
-- | order_id | product_id | product_name | qty |

-- ✅ 2NF — tách
-- orders: order_id, ...
-- products: product_id, product_name
-- order_items: order_id, product_id, qty
```

### 3NF: Không phụ thuộc bắc cầu

```sql
-- ❌ Vi phạm 3NF (city_name phụ thuộc zip_code, zip_code phụ thuộc customer)
-- | id | name | zip_code | city_name |

-- ✅ 3NF — tách
-- customers: id, name, zip_code
-- zip_codes: zip_code, city_name
```

> **💡 Quy tắc**: "Mỗi column phụ thuộc the key, the whole key, and nothing but the key."

---

## 27.6 — Indexing

Normalization giảm data duplication. 1NF: no nested data. 2NF: every column depends on full key. 3NF: no transitive dependencies. Trade-off: normalize = ít duplicate, nhiều JOINs.

### B-Tree Index — tìm nhanh

```sql
-- Không index: scan TOÀN BỘ table (O(n))
SELECT * FROM orders WHERE customer_id = 1;
-- Có index: lookup O(log n)

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);

-- Composite index: tìm theo NHIỀU columns
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
-- Dùng cho: WHERE customer_id = 1 AND status = 'completed'
```

### Khi nào tạo index?

| Nên index | KHÔNG nên index |
|-----------|-----------------|
| Columns trong WHERE | Columns ít giá trị (bool) |
| Columns trong JOIN ON | Tables nhỏ (< 1000 rows) |
| Columns trong ORDER BY | Columns thường xuyên UPDATE |
| Foreign keys | Tables insert rất nhiều |

---

## 27.7 — Transactions & ACID

Connection pooling tái sử dụng database connections thay vì tạo mới mỗi request. Roc app giữ pool connections, mỗi Task lấy 1 connection, dùng xong trả lại.

### ACID = 4 đặc tính

```sql
-- Atomicity: tất cả hoặc không gì
-- Consistency: data luôn hợp lệ
-- Isolation: transactions không ảnh hưởng nhau
-- Durability: committed = stored safely

BEGIN TRANSACTION;

-- Chuyển tiền: trừ A, cộng B
UPDATE accounts SET balance = balance - 500000 WHERE id = 1;
UPDATE accounts SET balance = balance + 500000 WHERE id = 2;

-- Nếu OK → commit
COMMIT;

-- Nếu lỗi → rollback (tất cả hoàn tác)
-- ROLLBACK;
```

### Trong Roc: Transaction = pure function

```roc
# Transaction logic = pure!
transferMoney = \fromAccount, toAccount, amount ->
    if fromAccount.balance < amount then
        Err InsufficientFunds
    else
        newFrom = { fromAccount & balance: fromAccount.balance - amount }
        newTo = { toAccount & balance: toAccount.balance + amount }
        Ok (newFrom, newTo)

# Tests — no DB needed!
expect
    from = { id: 1, balance: 1000000 }
    to = { id: 2, balance: 500000 }
    when transferMoney from to 300000 is
        Ok (newFrom, newTo) ->
            newFrom.balance == 700000 && newTo.balance == 800000
        Err _ -> Bool.false

expect
    from = { id: 1, balance: 100 }
    to = { id: 2, balance: 500 }
    transferMoney from to 300 == Err InsufficientFunds
```

### Isolation Levels

| Level | Dirty Read | Non-repeatable | Phantom |
|-------|-----------|----------------|---------|
| Read Uncommitted | ✅ có | ✅ có | ✅ có |
| Read Committed | ❌ không | ✅ có | ✅ có |
| Repeatable Read | ❌ không | ❌ không | ✅ có |
| Serializable | ❌ không | ❌ không | ❌ không |

---

## 27.8 — Roc + SQLite

SQL injection — lỗ hổng nguy hiểm nhất với databases. Luôn dùng parameterized queries, không bao giờ nối string trực tiếp vào SQL. Roc's type system giúp nhưng không thay thế best practices.

```roc
# filename: main.roc
app [main] { cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br" }

import cli.Stdout

# ═══════ PURE CORE: SQL builders ═══════

# Type-safe SQL builder — không string concatenation!
buildInsert = \table, columns, values ->
    cols = Str.joinWith columns ", "
    vals = List.map values \v -> "'$(v)'"
        |> Str.joinWith ", "
    "INSERT INTO $(table) ($(cols)) VALUES ($(vals));"

buildSelect = \table, conditions ->
    base = "SELECT * FROM $(table)"
    if Str.isEmpty conditions then
        "$(base);"
    else
        "$(base) WHERE $(conditions);"

buildUpdate = \table, sets, conditions ->
    setClause = List.map sets \(col, val) -> "$(col) = '$(val)'"
        |> Str.joinWith ", "
    "UPDATE $(table) SET $(setClause) WHERE $(conditions);"

# ═══════ TESTS ═══════

expect
    buildInsert "customers" ["name", "email"] ["An", "an@mail.com"]
    == "INSERT INTO customers (name, email) VALUES ('An', 'an@mail.com');"

expect
    buildSelect "orders" "status = 'completed'"
    == "SELECT * FROM orders WHERE status = 'completed';"

expect
    buildUpdate "orders" [("status", "shipped")] "id = 1"
    == "UPDATE orders SET status = 'shipped' WHERE id = 1;"

# ═══════ ARCHITECTURE ═══════

# Pure domain: SQL builder, data mapping
# Platform shell: actual DB connection
#
# main =
#     sql = buildInsert "customers" ["name", "email"] ["An", "an@mail.com"]
#     SQLite.execute! db sql    # ← platform provides this
#
# Domain code test bằng expect — không cần DB!

main =
    # Demo SQL generation
    Stdout.line! "=== SQL Builder Demo ==="

    sql1 = buildInsert "customers" ["name", "email"] ["An", "an@mail.com"]
    Stdout.line! sql1

    sql2 = buildSelect "orders" "customer_id = 1 AND status = 'completed'"
    Stdout.line! sql2

    sql3 = buildUpdate "orders" [("status", "shipped"), ("updated_at", "2024-03-15")] "id = 42"
    Stdout.line! sql3

    Stdout.line! "\n=== Report Queries ==="
    reportSql =
        """
        WITH daily_revenue AS (
            SELECT date(created_at) AS day, SUM(total) AS revenue
            FROM orders WHERE status = 'completed'
            GROUP BY day
        )
        SELECT day, revenue FROM daily_revenue ORDER BY day DESC LIMIT 7;
        """
    Stdout.line! reportSql
```

---


## ✅ Checkpoint 27

> Đến đây bạn phải hiểu:
> 1. Relational model: tables + keys (PK, FK) + normalization
> 2. SQL: `SELECT`, `JOIN`, `GROUP BY`, CTEs — ngôn ngữ query universal
> 3. Roc truy cập DB qua platform bindings (SQLite via basic-cli)
>
> **Test nhanh**: Tại sao cần normalization?
> <details><summary>Đáp án</summary>Tránh data duplication và update anomalies — mỗi fact lưu đúng 1 chỗ.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Schema design

Thiết kế database schema cho hệ thống thư viện:

```
Entities: Book, Author, Member, Loan
Relationships: Book N-N Author, Member 1-N Loan, Book 1-N Loan
```

<details><summary>✅ Lời giải</summary>

```sql
CREATE TABLE authors (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT NOT NULL, isbn TEXT UNIQUE);
CREATE TABLE book_authors (book_id INTEGER REFERENCES books, author_id INTEGER REFERENCES authors);
CREATE TABLE members (id INTEGER PRIMARY KEY, name TEXT NOT NULL, email TEXT UNIQUE);
CREATE TABLE loans (
    id INTEGER PRIMARY KEY,
    book_id INTEGER REFERENCES books,
    member_id INTEGER REFERENCES members,
    borrowed_at TEXT NOT NULL,
    due_at TEXT NOT NULL,
    returned_at TEXT
);
```

</details>

---

**Bài 2** (15 phút): SQL queries

Viết queries cho library schema:

```sql
-- 1. Sách đang được mượn (chưa trả)
-- 2. Member mượn nhiều nhất
-- 3. Sách quá hạn > 7 ngày
-- 4. Top 5 sách phổ biến nhất
```

<details><summary>✅ Lời giải</summary>

```sql
-- 1. Sách đang mượn
SELECT b.title, m.name, l.due_at
FROM loans l JOIN books b ON l.book_id = b.id JOIN members m ON l.member_id = m.id
WHERE l.returned_at IS NULL;

-- 2. Top borrower
SELECT m.name, COUNT(*) AS loan_count
FROM loans l JOIN members m ON l.member_id = m.id
GROUP BY m.id ORDER BY loan_count DESC LIMIT 1;

-- 3. Quá hạn
SELECT b.title, m.name, l.due_at,
    CAST(julianday('now') - julianday(l.due_at) AS INTEGER) AS days_overdue
FROM loans l JOIN books b ON l.book_id = b.id JOIN members m ON l.member_id = m.id
WHERE l.returned_at IS NULL AND julianday('now') - julianday(l.due_at) > 7;

-- 4. Sách phổ biến
SELECT b.title, COUNT(*) AS times_borrowed
FROM loans l JOIN books b ON l.book_id = b.id
GROUP BY b.id ORDER BY times_borrowed DESC LIMIT 5;
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| SQL injection | String concatenation | Dùng parameterized queries: `?` placeholders |
| Query chậm | Thiếu index | `EXPLAIN QUERY PLAN` → thêm index |
| Data bị mất | Quên transaction | Wrap thay đổi quan trọng trong `BEGIN...COMMIT` |
| Normalization quá mức | Quá nhiều JOINs | Denormalize cho read queries, normalize cho write |

---

## Tóm tắt

- ✅ **Relational model** = tables (rows + columns), keys (PK, FK), relationships (1-1, 1-N, N-N).
- ✅ **SQL CRUD** = INSERT, SELECT, UPDATE, DELETE. WHERE/ORDER BY/GROUP BY/LIMIT.
- ✅ **JOIN** = kết hợp tables. INNER (match cả 2) vs LEFT (tất cả bên trái).
- ✅ **CTEs** = `WITH ... AS (...)` — "let bindings" cho SQL, giống Roc pipelines.
- ✅ **Normalization** = 1NF (no lists), 2NF (full key dependency), 3NF (no transitive).
- ✅ **Indexing** = B-Tree, composite. `O(n) → O(log n)`. Dùng cho WHERE/JOIN/ORDER BY.
- ✅ **ACID** = Atomicity, Consistency, Isolation, Durability. Transactions = all or nothing.
- ✅ **Roc + DB** = pure SQL builders + platform IO. Domain logic test không cần database.

## Tiếp theo

→ Chapter 28: **Advanced Data Patterns** — migrations, event store (append-only), caching strategies, và platform-side IO management.
