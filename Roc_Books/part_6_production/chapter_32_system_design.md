# Chapter 32 — System Design Thinking

> **Bạn sẽ học được**:
> - **Capacity estimation** — back-of-envelope calculations
> - **API design** — REST principles, versioning, pagination
> - **Caching strategy** — multi-level caching, invalidation
> - **Architecture patterns** — Roc as pure domain core trong polyglot system
> - **Serverless** — Roc → WASM → edge functions
> - **Design exercises** — URL shortener, rate limiter
>
> **Yêu cầu trước**: [Chapter 31 — Distributed Systems](chapter_31_distributed_systems.md)
> **Thời gian đọc**: ~45 phút | **Level**: Principal
> **Kết quả cuối cùng**: Tư duy thiết kế hệ thống — từ requirements đến architecture decisions.

---

"Thiết kế hệ thống cho 1 triệu users" — câu hỏi phỏng vấn kinh điển. Nhưng quan trọng hơn phỏng vấn, đây là kỹ năng bạn cần khi hệ thống thật sự mở rộng. Capacity estimation, load balancing, caching, sharding — từ lý thuyết đến thực hành.

## System Design — Kiến trúc ở mức hệ thống

Capacity estimation, load balancing, caching layers, API design, microservices. Framework tư duy cho system design — áp dụng cho interview và real-world architecture.


## 32.1 — Capacity Estimation

Bước 1 của system design: ước lượng. 1 triệu users, 10% active daily = 100k DAU. Mỗi user 10 requests/ngày = 1M requests/ngày ≈ 12 requests/giây. Từ số này → chọn architecture.

### Back-of-envelope calculations

```roc
# ═══════ PURE: Capacity calculator ═══════

# Useful constants
bytesPerKB = 1024
bytesPerMB = 1024 * 1024
bytesPerGB = 1024 * 1024 * 1024
secondsPerDay = 86400
secondsPerMonth = 2592000

# Example: Social media platform
# 10M daily active users
# Each user: 5 reads + 1 write per day

estimateLoad = \config ->
    dailyReads = config.dailyActiveUsers * config.readsPerUser
    dailyWrites = config.dailyActiveUsers * config.writesPerUser
    readsPerSecond = dailyReads // secondsPerDay
    writesPerSecond = dailyWrites // secondsPerDay
    { dailyReads, dailyWrites, readsPerSecond, writesPerSecond }

estimateStorage = \config ->
    dailyNewData = config.dailyActiveUsers * config.writesPerUser * config.avgWriteSize
    monthlyData = dailyNewData * 30
    yearlyData = dailyNewData * 365
    { dailyBytes: dailyNewData, monthlyBytes: monthlyData, yearlyBytes: yearlyData }

formatBytes = \bytes ->
    if bytes >= bytesPerGB then "$(Num.toStr (bytes // bytesPerGB)) GB"
    else if bytes >= bytesPerMB then "$(Num.toStr (bytes // bytesPerMB)) MB"
    else if bytes >= bytesPerKB then "$(Num.toStr (bytes // bytesPerKB)) KB"
    else "$(Num.toStr bytes) B"

# Test
expect
    config = { dailyActiveUsers: 10000000, readsPerUser: 5, writesPerUser: 1 }
    load = estimateLoad config
    load.readsPerSecond == 578    # ~580 reads/sec
    && load.writesPerSecond == 115  # ~115 writes/sec

expect
    config = { dailyActiveUsers: 10000000, writesPerUser: 1, avgWriteSize: 500 }
    storage = estimateStorage config
    storage.dailyBytes == 5000000000    # ~5 GB/day
```

### Quy tắc ngón cái

| Metric | Rule of Thumb |
|--------|--------------|
| 1 server | ~1000 concurrent connections |
| SSD read | ~250μs = 0.25ms |
| Network (same DC) | ~0.5ms |
| Network (cross DC) | ~50-150ms |
| SSD write | ~1ms |
| DB query (indexed) | ~1-5ms |
| HTTP request | ~50-200ms |

---

## 32.2 — API Design

Load balancing phân phối requests đều giữa các servers. Round-robin đơn giản nhất, least-connections thông minh hơn, consistent hashing cho stateful services.

### REST Principles

```roc
# ═══════ PURE: RESTful route design ═══════

# Resources = nouns, HTTP methods = verbs
# /api/v1/resources

Route : { method : [Get, Post, Put, Delete], path : Str, description : Str }

apiRoutes = [
    # Orders CRUD
    { method: Get, path: "/api/v1/orders", description: "List orders (paginated)" },
    { method: Get, path: "/api/v1/orders/:id", description: "Get order by ID" },
    { method: Post, path: "/api/v1/orders", description: "Create order" },
    { method: Put, path: "/api/v1/orders/:id", description: "Update order" },
    { method: Delete, path: "/api/v1/orders/:id", description: "Delete order" },

    # Nested resources
    { method: Get, path: "/api/v1/orders/:id/items", description: "List items in order" },
    { method: Post, path: "/api/v1/orders/:id/items", description: "Add item to order" },

    # Actions (RPC-style for complex operations)
    { method: Post, path: "/api/v1/orders/:id/confirm", description: "Confirm order" },
    { method: Post, path: "/api/v1/orders/:id/cancel", description: "Cancel order" },
]
```

### Pagination

```roc
# ═══════ PURE: Pagination logic ═══════

PaginationParams : { page : U64, perPage : U64 }

defaultPagination = { page: 1, perPage: 20 }

paginate : List a, PaginationParams -> { items : List a, meta : { page : U64, perPage : U64, totalItems : U64, totalPages : U64 } }
paginate = \items, params ->
    total = List.len items
    totalPages = if total == 0 then 1 else (total + params.perPage - 1) // params.perPage
    safePage = Num.max 1 (Num.min params.page totalPages)
    offset = (safePage - 1) * params.perPage
    pageItems = List.sublist items { start: offset, len: params.perPage }
    { items: pageItems, meta: { page: safePage, perPage: params.perPage, totalItems: total, totalPages } }

# Tests
expect
    items = List.range { start: At 1, end: At 100 }
    result = paginate items { page: 1, perPage: 10 }
    List.len result.items == 10
    && result.meta.totalPages == 10
    && result.meta.totalItems == 100

expect
    items = List.range { start: At 1, end: At 5 }
    result = paginate items { page: 2, perPage: 3 }
    result.items == [4, 5]
    && result.meta.page == 2

expect
    result = paginate ([] : List U64) { page: 1, perPage: 10 }
    List.isEmpty result.items && result.meta.totalPages == 1

# JSON response format
paginationToJson = \meta ->
    """
    {"page":$(Num.toStr meta.page),"perPage":$(Num.toStr meta.perPage),"totalItems":$(Num.toStr meta.totalItems),"totalPages":$(Num.toStr meta.totalPages)}
    """
```

### API Versioning

```roc
# Strategy 1: URL-based (recommended for REST)
# /api/v1/orders    ← version 1
# /api/v2/orders    ← version 2

# Strategy 2: Header-based
# Accept: application/vnd.myapi.v2+json

# Router with versioning
routeRequest = \method, url ->
    when Str.splitFirst url "/api/" is
        Ok { after, .. } ->
            when Str.splitFirst after "/" is
                Ok { before: version, after: path } ->
                    Ok { version, path, method }
                Err _ -> Err InvalidRoute
        Err _ -> Err InvalidRoute

expect
    routeRequest Get "/api/v1/orders"
    == Ok { version: "v1", path: "orders", method: Get }
```

---

## 32.3 — Multi-Level Caching

Caching giảm load database: cache hit = trả ngay từ memory (microseconds), cache miss = query database (milliseconds). Redis phổ biến nhất. Cache invalidation là bài toán khó nhất.

```
Request → L1 (In-Memory) → L2 (Redis/HTTP) → L3 (Database)
            ~1μs              ~1ms              ~5ms

Cache hit ratio target: 90%+ for L1, 95%+ for L1+L2
```

```roc
# ═══════ PURE: Multi-level cache ═══════

CacheResult : [L1Hit, L2Hit, L3Hit, Miss]

lookupMultiLevel = \key, l1Cache, l2Cache, database ->
    when Dict.get l1Cache key is
        Ok value -> { result: L1Hit, value, l1Cache, l2Cache }
        Err _ ->
            when Dict.get l2Cache key is
                Ok value ->
                    # Promote to L1
                    newL1 = Dict.insert l1Cache key value
                    { result: L2Hit, value, l1Cache: newL1, l2Cache }
                Err _ ->
                    when Dict.get database key is
                        Ok value ->
                            # Promote to L1 and L2
                            newL1 = Dict.insert l1Cache key value
                            newL2 = Dict.insert l2Cache key value
                            { result: L3Hit, value, l1Cache: newL1, l2Cache: newL2 }
                        Err _ ->
                            { result: Miss, value: "", l1Cache, l2Cache }

# Tests
expect
    l1 = Dict.fromList [("key1", "val1")]
    l2 = Dict.empty {}
    db = Dict.empty {}
    result = lookupMultiLevel "key1" l1 l2 db
    result.result == L1Hit && result.value == "val1"

expect
    l1 = Dict.empty {}
    l2 = Dict.fromList [("key2", "val2")]
    db = Dict.empty {}
    result = lookupMultiLevel "key2" l1 l2 db
    result.result == L2Hit
    && Dict.get result.l1Cache "key2" == Ok "val2"    # promoted!
```

---

## 32.4 — Architecture: Roc in Polyglot Systems

Database sharding chia data thành nhiều databases nhỏ. Shard by user ID: users 1-1M ở shard 1, 1M-2M ở shard 2. Scale write performance, nhưng cross-shard queries phức tạp.

```
┌──────────────────────────────────────────────┐
│              Production System                │
│                                               │
│  ┌──────────┐  HTTP   ┌──────────────────┐  │
│  │ Frontend │ ──────→ │   API Gateway    │  │
│  │ (React)  │         │   (nginx/Go)     │  │
│  └──────────┘         └────────┬─────────┘  │
│                                │              │
│         ┌──────────────────────┼──────┐      │
│         │                      │      │      │
│    ┌────▼─────┐  ┌────────▼──┐  ┌──▼────┐  │
│    │ Order    │  │ Payment  │  │ User  │  │
│    │ Service  │  │ Service  │  │Service│  │
│    │ (Roc)   │  │ (Roc)   │  │(Roc)  │  │
│    │ pure    │  │ pure    │  │pure   │  │
│    │ domain  │  │ domain  │  │domain │  │
│    └────┬─────┘  └────┬────┘  └──┬────┘  │
│         │              │          │        │
│    ┌────▼──────────────▼──────────▼────┐  │
│    │        Database / Message Queue     │  │
│    │        (PostgreSQL / RabbitMQ)      │  │
│    └────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

### Roc's role: Pure Domain Core

```roc
# Each microservice:
# 1. Platform handles HTTP/DB/Queue (IO)
# 2. Roc app = pure domain logic (testable, fast, correct)
# 3. No runtime, single binary → small container image

# Benefits:
# - Domain logic portable between services
# - Testing: no mocks needed for domain code
# - Performance: compiled, no GC pauses
# - Security: platform isolation
```

### Serverless: Roc → WASM

```roc
# Roc compiles to WASM → runs on edge
# Use case: validation, transformation, routing at CDN edge

# Edge function pattern:
# 1. Receive request (platform provides)
# 2. Pure transform (Roc)
# 3. Return response (platform sends)

# Advantages:
# - Cold start: ~1ms (Roc binary is tiny)
# - Memory: minimal (no runtime)
# - Latency: runs at edge, close to user
```

---

## 32.5 — Design Exercises

CDN đưa static content (images, CSS, JS) gần user hơn. User ở Việt Nam truy cập server ở Singapore thay vì US — latency giảm từ 200ms xuống 20ms.

### Exercise 1: URL Shortener

```roc
# ═══════ PURE: URL Shortener domain ═══════

# Requirements:
# - Shorten long URL → short code (6 chars)
# - Redirect short code → original URL
# - Count clicks per URL
# - Expire links after N days

# Domain model
URLEntry : {
    shortCode : Str,
    originalUrl : Str,
    clickCount : U64,
    createdAt : U64,
    expiresAt : U64,
}

# Generate short code from ID (base62)
base62Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

idToShortCode = \id ->
    if id == 0 then "000000"
    else
        digits = generateBase62Digits id []
        padded = List.repeat "0" (6 - List.len digits) |> List.concat digits
        Str.joinWith padded ""

generateBase62Digits = \n, acc ->
    if n == 0 then acc
    else
        remainder = Num.rem n 62 |> Num.toU64
        char = Str.graphemes base62Chars
            |> List.get remainder
            |> Result.withDefault "0"
        generateBase62Digits (n // 62) (List.prepend acc char)

# Resolve short code → URL
resolveUrl = \store, shortCode, currentTime ->
    when Dict.get store shortCode is
        Ok entry ->
            if currentTime > entry.expiresAt then Err Expired
            else Ok { entry & clickCount: entry.clickCount + 1 }
        Err _ -> Err NotFound

# Tests
expect idToShortCode 1 == "000001"
expect Str.countUtf8Bytes (idToShortCode 999999) == 6

expect
    store = Dict.fromList [("abc123", { shortCode: "abc123", originalUrl: "https://example.com", clickCount: 0, createdAt: 100, expiresAt: 1000 })]
    when resolveUrl store "abc123" 500 is
        Ok entry -> entry.clickCount == 1 && entry.originalUrl == "https://example.com"
        Err _ -> Bool.false

expect
    store = Dict.fromList [("abc123", { shortCode: "abc123", originalUrl: "https://example.com", clickCount: 0, createdAt: 100, expiresAt: 1000 })]
    resolveUrl store "abc123" 2000 == Err Expired
```

### Exercise 2: Rate Limiter

```roc
# ═══════ PURE: Rate Limiter (Token Bucket) ═══════

# Token Bucket algorithm:
# - Bucket starts full (maxTokens)
# - Each request consumes 1 token
# - Tokens replenish at fixed rate
# - Empty bucket → reject request

RateLimitState : { tokens : U64, lastRefill : U64 }

RateLimitConfig : { maxTokens : U64, refillRate : U64, refillInterval : U64 }

# Refill tokens based on time elapsed
refillTokens = \state, config, currentTime ->
    elapsed = currentTime - state.lastRefill
    tokensToAdd = elapsed * config.refillRate // config.refillInterval
    newTokens = Num.min config.maxTokens (state.tokens + tokensToAdd)
    { tokens: newTokens, lastRefill: currentTime }

# Try to consume a token
tryConsume = \state, config, currentTime ->
    refilled = refillTokens state config currentTime
    if refilled.tokens > 0 then
        Ok { tokens: refilled.tokens - 1, lastRefill: refilled.lastRefill }
    else
        Err RateLimited

# Tests
expect
    config = { maxTokens: 10, refillRate: 1, refillInterval: 1 }
    state = { tokens: 10, lastRefill: 0 }
    # Consume 1 token
    when tryConsume state config 0 is
        Ok newState -> newState.tokens == 9
        Err _ -> Bool.false

expect
    config = { maxTokens: 10, refillRate: 1, refillInterval: 1 }
    state = { tokens: 0, lastRefill: 0 }
    # No tokens → rejected
    tryConsume state config 0 == Err RateLimited

expect
    config = { maxTokens: 10, refillRate: 1, refillInterval: 1 }
    state = { tokens: 0, lastRefill: 0 }
    # After 5 seconds → 5 new tokens
    when tryConsume state config 5 is
        Ok newState -> newState.tokens == 4    # 5 refilled - 1 consumed
        Err _ -> Bool.false

# Per-user rate limiting
checkRateLimit = \limits, userId, config, currentTime ->
    state = Dict.get limits userId |> Result.withDefault { tokens: config.maxTokens, lastRefill: currentTime }
    when tryConsume state config currentTime is
        Ok newState ->
            Ok (Dict.insert limits userId newState)
        Err RateLimited ->
            Err RateLimited
```

---

## 32.6 — Design Process Checklist

Tổng hợp: thiết kế URL shortener (như bit.ly). Capacity estimation → API design → database schema → caching → scaling. Mỗi quyết định có trade-off — document tại sao chọn approach này.

```
1. REQUIREMENTS
   □ Functional requirements (user stories)
   □ Non-functional (latency, throughput, availability)
   □ Scale estimates (users, requests/sec, data size)

2. HIGH-LEVEL DESIGN
   □ Components (services, databases, caches)
   □ Data flow (request → response)
   □ API contracts (routes, payloads)

3. DETAILED DESIGN
   □ Data model (tables, indexes, relationships)
   □ Domain model (types, state machines, events)
   □ Algorithms (caching, rate limiting, ranking)

4. SCALE & RELIABILITY
   □ Bottlenecks (read-heavy? write-heavy?)
   □ Caching strategy (what, where, TTL)
   □ Failure handling (circuit breaker, retry, saga)

5. ROC-SPECIFIC
   □ Pure domain functions (testable)
   □ Platform handles IO (DB, HTTP, Queue)
   □ Opaque types for domain invariants
   □ Tag unions for exhaustive states
```


## ✅ Checkpoint 32

> Đến đây bạn phải hiểu:
> 1. **Capacity estimation** = back-of-envelope (users × actions × size = QPS, storage)
> 2. **API design** = REST + pagination + versioning. Roc route = pure functions
> 3. **Multi-level caching** = L1 memory → L2 Redis → L3 DB. Promote on miss
>
> **Test nhanh**: Token bucket rate limiter hết tokens → refill sau 5 giây. Kết quả?
> <details><summary>Đáp án</summary>`refillTokens` tính tokens mới dựa trên elapsed time. Sau 5 giây, `tokensToAdd = 5 * refillRate / refillInterval`.</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Thiết kế API cho hệ thống quản lý tasks (todo app). Liệt kê 5 REST endpoints với method, path, và description. Implement `paginate` function cho task list.

<details><summary>✅ Gợi ý</summary>

```roc
taskRoutes = [
    { method: Get, path: "/api/v1/tasks", description: "List tasks (paginated)" },
    { method: Post, path: "/api/v1/tasks", description: "Create task" },
    { method: Put, path: "/api/v1/tasks/:id", description: "Update task" },
    { method: Delete, path: "/api/v1/tasks/:id", description: "Delete task" },
    { method: Post, path: "/api/v1/tasks/:id/complete", description: "Mark complete" },
]
```

</details>

---

**Bài 2** (15 phút): Implement LRU cache với max size. Khi cache đầy, loại bỏ item ít dùng nhất. Viết tests cho: cache hit, cache miss, eviction.

<details><summary>✅ Gợi ý</summary>

```roc
# LRU = Least Recently Used
# Dùng Dict + List (access order)
LRUCache : { items : Dict Str Str, order : List Str, maxSize : U64 }

put : LRUCache, Str, Str -> LRUCache
put = \cache, key, value ->
    # 1. Remove key from order if exists
    # 2. If full, remove oldest (first in order)
    # 3. Add to end of order + dict
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| API quá slow | Thiếu pagination, thiếu cache | Paginate mặc định, cache hot data |
| Rate limiter bypass | Per-IP chỉ → shared IP | Kết hợp IP + user ID + API key |
| Short URL collision | Sequential ID hết | Dùng random ID + collision check |
| Capacity underestimate | Quên peak hours | Multiply average × 10 cho peak |

---

## Tóm tắt

- ✅ **Capacity estimation** = back-of-envelope. Users × actions × size = storage/bandwidth.
- ✅ **API design** = REST resources, versioning (URL-based), pagination (offset + metadata).
- ✅ **Caching** = multi-level (L1 memory → L2 Redis → L3 DB). Promote on miss.
- ✅ **Architecture** = Roc as pure domain core. Platform handles all IO. Polyglot-friendly.
- ✅ **Serverless** = Roc → WASM → edge. Tiny binary, no runtime, ~1ms cold start.
- ✅ **URL Shortener** = base62 encoding, expiration, click counting (pure logic).
- ✅ **Rate Limiter** = Token Bucket algorithm. Per-user state. Refill over time.

## Tiếp theo

→ Chapter 33: **Capstone Part 2: Production Deployment** ⭐ — Roc webserver + SQLite + JWT auth + logging + Docker + CI/CD. Chương cuối cùng!
