# Appendix G — Concurrency & Parallelism in Roc

> Cách Roc xử lý concurrency — platform-managed, pure by design.

---

## G.1 — Roc's approach: Purity enables parallelism

### Tại sao concurrency trong Roc khác?

Trong hầu hết ngôn ngữ, concurrency = **nightmare**:

```
Traditional (Python/Java/Go):
  Thread 1: read X   → compute → write X
  Thread 2: read X   → compute → write X
  → RACE CONDITION! 💥

  Solution: locks, mutexes, channels, async/await
  Complexity: rất cao
```

Trong Roc, **purity giải quyết vấn đề gốc**:

```
Roc:
  Pure function A: input → output (NO shared state)
  Pure function B: input → output (NO shared state)
  → CAN ALWAYS RUN IN PARALLEL! ✅

  Không locks, không mutexes, không data races
  Vì: không có shared mutable state
```

> **💡 Key insight**: Nếu function không có side effects, chạy song song luôn an toàn. Roc enforce purity → concurrency safety **miễn phí**.

---

## G.2 — Platform handles concurrency

### App code = pure, Platform decides execution

```
┌─────────────────────────────────────┐
│          Your App (pure)            │
│  processOrder : Order -> Receipt    │  ← Pure — có thể parallelize
│  validateUser : User -> Result      │  ← Pure — có thể parallelize
│  calculatePrice : Item -> U64       │  ← Pure — có thể parallelize
├─────────────────────────────────────┤
│        Platform (runtime)           │
│  Decides: sequential or parallel?   │  ← Platform quyết định
│  Manages: threads, async, IO        │
│  Optimizes: based on CPU cores      │
└─────────────────────────────────────┘
```

App code **KHÔNG biết** nó chạy 1 thread hay 1000 threads — đó là chuyện của Platform.

---

## G.3 — Sequential vs Concurrent Tasks

### Sequential — mặc định

```roc
# Tasks chạy LẦN LƯỢT — dòng trước xong, dòng sau mới chạy
main =
    Stdout.line! "Step 1"        # → chạy xong
    content = File.readUtf8! p   # → rồi mới chạy
    Stdout.line! content         # → rồi mới chạy
```

### Concurrent HTTP requests (concept)

```roc
# ═══════ Cách tiếp cận: batch processing ═══════

# Pure: tạo danh sách requests
buildRequests = \urls ->
    List.map urls \url -> { method: Get, url, headers: [], body: [] }

# Shell: gửi lần lượt (basic-cli hiện tại)
fetchAll! = \urls ->
    List.walk urls [] \results, url ->
        result = Http.send { method: Get, url, headers: [], body: [] }
            |> Task.attempt!
        List.append results result

# ═══════ Tương lai: platform có thể parallel ═══════
# Khi platform hỗ trợ, cùng code nhưng platform dispatch song song
# App code KHÔNG CẦN ĐỔI — platform tự optimize!
```

---

## G.4 — Concurrent Patterns bằng Pure Logic

### Pattern 1: Map-Reduce (tự nhiên trong FP)

```roc
# Map-Reduce = parallel-ready by design!
# Vì map và reduce đều pure → platform CÓ THỂ chạy song song

# Step 1: Map — xử lý từng item (parallel-safe)
processedItems = List.map items \item ->
    { item & price: applyDiscount item.price item.category }

# Step 2: Reduce — gom kết quả
totalRevenue = List.walk processedItems 0 \s, item -> s + item.price

# Roc runtime CÓ THỂ song song hóa List.map khi items lớn
# App code không cần thay đổi gì!
```

### Pattern 2: Fork-Join via Task combinators

```roc
# Fetch nhiều resources, combine kết quả
# Hiện tại sequential — tương lai platform có thể parallel

buildDashboard! = \userId ->
    # Các task này KHÔNG phụ thuộc nhau → có thể song song
    profile = fetchProfile! userId
    orders = fetchOrders! userId
    stats = fetchStats! userId

    # Combine (pure)
    Ok {
        name: profile.name,
        orderCount: List.len orders,
        totalSpent: stats.totalSpent,
    }
```

### Pattern 3: Pipeline parallelism

```roc
# Pipeline stages có thể overlap (pipelining)
# Stage 1 cho item N chạy đồng thời Stage 2 cho item N-1

processOrders = \orders ->
    orders
    |> List.map validate          # Stage 1: validate
    |> List.keepOks \x -> x       # Filter valid
    |> List.map calculatePricing  # Stage 2: pricing
    |> List.map generateInvoice   # Stage 3: invoicing
    # Mỗi stage pure → platform CÓ THỂ pipeline
```

---

## G.5 — Event-Driven Concurrency

### Event processing — naturally concurrent

```roc
# Events đã pure → xử lý song song an toàn

# Mỗi event handler = pure function
handleEvent = \state, event ->
    when event is
        OrderPlaced order -> { state & orders: Dict.insert state.orders order.id order }
        PaymentReceived payment -> updatePayment state payment
        ItemShipped shipment -> updateShipping state shipment

# Batch process events
# Nếu events KHÔNG conflict, platform có thể process song song
processEvents = \initialState, events ->
    List.walk events initialState handleEvent
```

### Actor model (concept)

```roc
# Roc phù hợp Actor model vì:
# 1. Mỗi actor = pure function: state + message -> new state
# 2. Không shared mutable state
# 3. Platform quản lý message passing

# Actor = pure state machine
actorUpdate = \state, message ->
    when message is
        Increment -> { state & count: state.count + 1 }
        Decrement -> { state & count: state.count - 1 }
        Reset -> { state & count: 0 }

# Platform handles: scheduling, message delivery, fairness
```

---

## G.6 — So sánh Concurrency Models

| Ngôn ngữ | Model | Complexity | Safety |
|-----------|-------|-----------|--------|
| Python | GIL + asyncio | ⭐⭐⭐ | ⭐⭐ |
| JavaScript | Event loop | ⭐⭐ | ⭐⭐ |
| Go | Goroutines + channels | ⭐⭐⭐ | ⭐⭐⭐ |
| Rust | Ownership + Send/Sync | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Erlang/Elixir | Actor model (BEAM) | ⭐⭐ | ⭐⭐⭐⭐ |
| **Roc** | **Purity + Platform** | **⭐** | **⭐⭐⭐⭐⭐** |

### Tại sao Roc đơn giản nhất?

```
Bạn KHÔNG CẦN nghĩ về:
  ❌ Locks / Mutexes        → không shared mutable state
  ❌ Thread safety          → pure functions luôn safe
  ❌ Race conditions        → không side effects
  ❌ Deadlocks             → không locks
  ❌ async/await coloring   → platform handles
  ❌ Actor supervision      → platform handles

Bạn CHỈ CẦN:
  ✅ Viết pure functions
  ✅ Dùng Task cho IO
  ✅ Để platform lo phần còn lại
```

---

## G.7 — Thực tế: Roc ecosystem hiện tại

| Feature | Status | Ghi chú |
|---------|--------|---------|
| Sequential tasks | ✅ Stable | `!` chaining |
| Task.attempt | ✅ Stable | Error recovery |
| Parallel List.map | 🔄 Future | Runtime optimization |
| Concurrent HTTP | 🔄 Future | Platform enhancement |
| Worker pools | 🔄 Future | Platform feature |
| Channels/messaging | 🔄 Future | Community platforms |

> **💡 Takeaway**: Viết pure code hôm nay → tự động hưởng concurrency khi platform hỗ trợ. **Không cần refactor code!** Đây là sức mạnh của purity.
