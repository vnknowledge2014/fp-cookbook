# Chapter 31 — Distributed Systems Fundamentals

> **Bạn sẽ học được**:
> - **CAP Theorem** — Consistency, Availability, Partition Tolerance
> - **Consistency models** — strong, eventual, causal
> - **Saga pattern** — distributed transactions qua pure orchestration
> - **Circuit Breaker** — pure state machine cho failure handling
> - **Retry pattern** — exponential backoff
> - **Structured logging** — `Inspect` ability cho observability
>
> **Yêu cầu trước**: [Chapter 30 — Application Security](chapter_30_application_security.md)
> **Thời gian đọc**: ~45 phút | **Level**: Principal
> **Kết quả cuối cùng**: Tư duy distributed systems — patterns là pure logic, transport là platform concern.

---

Khi hệ thống chạy trên nhiều máy, mọi thứ đều có thể thất bại — network lag, server crash, dữ liệu không đồng bộ. CAP theorem nói bạn không thể có tất cả. Chapter này trang bị kiến thức nền tảng để thiết kế hệ thống chịu lỗi.

## Distributed Systems — Khi một server không đủ

CAP theorem, consistency models, message queues, saga pattern — mental models cho distributed architecture. Chapter này trang bị kiến thức để bạn đánh giá trade-offs khi scale application.


## 31.1 — CAP Theorem

CAP theorem: trong hệ thống phân tán, khi network partition xảy ra, bạn chọn Consistency (mọi node cùng data) hoặc Availability (mọi request có response). Không được cả hai.

### 3 đặc tính — chỉ chọn được 2

```
      Consistency (C)
         /\
        /  \
       /    \
      / CP    CA \
     /   ↑      \
    /  CHỌN 2   \
   /____________\
 Availability    Partition
     (A)        Tolerance (P)
                    AP
```

| | Mô tả | Ví dụ |
|---|---|---|
| **C** (Consistency) | Mọi read trả về write mới nhất | Tài khoản ngân hàng |
| **A** (Availability) | Mọi request đều có response | Social media feed |
| **P** (Partition Tolerance) | Hệ thống vẫn chạy khi mạng lỗi | Mọi hệ thống distributed |

### Trong thực tế

```roc
# CAP trade-offs = domain decisions
# Mỗi bounded context có thể chọn khác nhau!

# Banking: CP (nhất quán > availability)
# → Chuyển tiền phải đúng, cho dù chậm
accountTransfer = \from, to, amount ->
    # Consistency: strong — read ALWAYS latest
    # Nếu partition → reject transaction (unavailable)
    if from.balance < amount then Err InsufficientFunds
    else Ok { from: { from & balance: from.balance - amount }, to: { to & balance: to.balance + amount } }

# Social feed: AP (available > consistent)
# → Show cũ OK, miễn luôn có response
# → Eventually consistent: update propagates over time
```

---

## 31.2 — Consistency Models

Eventual consistency: data sẽ đồng bộ **cuối cùng**, nhưng tạm thời có thể khác nhau giữa các nodes. Phù hợp cho social media, e-commerce. Không phù hợp cho banking.

```roc
# ═══════ Strong Consistency ═══════
# Sau khi write → tất cả reads thấy giá trị mới
# Giống single database (ACID)
# Trade-off: chậm, cần coordination

# ═══════ Eventual Consistency ═══════
# Sau khi write → reads CÓ THỂ thấy giá trị cũ tạm thời
# Cuối cùng → tất cả nodes hội tụ (converge)
# Trade-off: nhanh, nhưng cần handle stale data

# ═══════ Trong Roc: Pure logic handles both ═══════

# Merge function cho eventual consistency
# Khi 2 nodes có data khác nhau → merge
mergeCounters = \counterA, counterB ->
    # Last-Write-Wins: lấy giá trị mới hơn
    if counterA.updatedAt > counterB.updatedAt then counterA
    else counterB

# CRDT-style: merge mà không mất data
mergeSets = \setA, setB ->
    # Union: giữ tất cả phần tử của cả 2
    List.concat setA setB
    |> deduplicate

deduplicate = \list ->
    List.walk list [] \acc, item ->
        if List.contains acc item then acc else List.append acc item

# Tests
expect
    a = { value: 10, updatedAt: 100 }
    b = { value: 20, updatedAt: 200 }
    (mergeCounters a b).value == 20    # b wins (newer)

expect
    a = [1, 2, 3]
    b = [2, 3, 4, 5]
    merged = mergeSets a b
    List.len merged == 5    # {1,2,3,4,5}
```

---

## 31.3 — Saga Pattern: Distributed Transactions

Message queues tách producer và consumer: service A gửi message, service B xử lý khi sẵn sàng. Retry tự động, không mất message nếu consumer down. RabbitMQ, Kafka.

### Vấn đề: Transaction qua nhiều services

```
Đặt vé máy bay:
1. Reserve seat     ← Service A
2. Charge payment   ← Service B
3. Send confirmation ← Service C

Nếu step 2 fail? → phải UNDO step 1 (release seat)
```

### Saga = orchestrated steps + compensating actions

```roc
# ═══════ PURE: Saga definition ═══════

SagaStep : {
    name : Str,
    action : Str,           # description of action
    compensation : Str,     # how to undo
}

# Define saga steps (PURE — chỉ mô tả, chưa execute)
bookFlightSaga = [
    {
        name: "Reserve Seat",
        action: "POST /api/flights/reserve",
        compensation: "POST /api/flights/release",
    },
    {
        name: "Charge Payment",
        action: "POST /api/payments/charge",
        compensation: "POST /api/payments/refund",
    },
    {
        name: "Send Confirmation",
        action: "POST /api/notifications/send",
        compensation: "POST /api/notifications/cancel",
    },
]

# ═══════ PURE: Saga executor logic ═══════

SagaState : [
    Running { completed : List Str, current : U64 },
    Compensating { toUndo : List Str, reason : Str },
    Completed,
    Failed { reason : Str },
]

executeSagaStep = \state, stepResult ->
    when state is
        Running { completed, current } ->
            when stepResult is
                Ok -> Running { completed: List.append completed (Num.toStr current), current: current + 1 }
                Err reason -> Compensating { toUndo: completed, reason }
        Compensating { toUndo, reason } ->
            when toUndo is
                [.. as rest, _last] -> Compensating { toUndo: rest, reason }
                [] -> Failed { reason }
        _ -> state

# Test happy path
expect
    state = Running { completed: [], current: 0 }
    s1 = executeSagaStep state Ok
    s2 = executeSagaStep s1 Ok
    s3 = executeSagaStep s2 Ok
    when s3 is
        Running { completed, .. } -> List.len completed == 3
        _ -> Bool.false

# Test failure → compensate
expect
    state = Running { completed: [], current: 0 }
    s1 = executeSagaStep state Ok       # step 1 OK
    s2 = executeSagaStep s1 (Err "Payment failed")  # step 2 FAIL
    when s2 is
        Compensating { toUndo, reason } ->
            List.len toUndo == 1 && reason == "Payment failed"
        _ -> Bool.false
```

---

## 31.4 — Circuit Breaker: Pure State Machine

Saga pattern quản lý transaction phân tán: nếu bước 3 thất bại, compensate bước 2 và bước 1. Không có ACID transaction globally — thay bằng eventually consistent workflow.

### Concept

```
Normal operation → failures count up → OPEN circuit → wait → HALF-OPEN → test → CLOSED (or OPEN again)

┌────────┐  failures ≥ threshold  ┌────────┐
│ CLOSED │ ──────────────────────→│  OPEN  │
│(normal)│                        │(reject)│
└────┬───┘ ←──────────────────── └───┬────┘
     │      success in half-open     │
     │                    timeout    │
     │                               │
     │      ┌───────────┐           │
     └──────│ HALF-OPEN │←──────────┘
            │ (test 1)  │
            └───────────┘
```

```roc
# ═══════ PURE: Circuit Breaker State Machine ═══════

CircuitState : [
    Closed { failures : U64 },
    Open { openedAt : U64 },
    HalfOpen,
]

defaultConfig = { failureThreshold: 5, resetTimeout: 30 }

# Record success
onSuccess = \state ->
    when state is
        Closed _ -> Closed { failures: 0 }
        HalfOpen -> Closed { failures: 0 }     # recovery!
        Open _ -> state                          # shouldn't happen

# Record failure
onFailure = \state, config ->
    when state is
        Closed { failures } ->
            newFailures = failures + 1
            if newFailures >= config.failureThreshold then
                Open { openedAt: 0 }    # trip circuit
            else
                Closed { failures: newFailures }
        HalfOpen ->
            Open { openedAt: 0 }    # back to open
        Open _ ->
            state

# Should we allow request?
canExecute = \state, currentTime, config ->
    when state is
        Closed _ -> Bool.true
        HalfOpen -> Bool.true
        Open { openedAt } ->
            # After timeout → allow 1 test request (half-open)
            currentTime - openedAt >= config.resetTimeout

# Transition to half-open
tryReset = \state, currentTime, config ->
    when state is
        Open { openedAt } ->
            if currentTime - openedAt >= config.resetTimeout then HalfOpen
            else state
        _ -> state

# Tests
expect
    state = Closed { failures: 0 }
    s1 = onFailure state defaultConfig
    s2 = onFailure s1 defaultConfig
    s3 = onFailure s2 defaultConfig
    s4 = onFailure s3 defaultConfig
    s5 = onFailure s4 defaultConfig    # 5th failure → OPEN
    when s5 is
        Open _ -> Bool.true
        _ -> Bool.false

expect
    state = Closed { failures: 3 }
    recovered = onSuccess state
    recovered == Closed { failures: 0 }    # reset counter

expect
    state = HalfOpen
    recovered = onSuccess state
    recovered == Closed { failures: 0 }    # circuit closed again
```

---

## 31.5 — Retry Pattern

Circuit breaker ngăn cascading failure: nếu service B chậm, service A ngừng gọi B tạm thời thay vì chờ timeout. Open → Half-Open → Closed — state machine quen thuộc.

```roc
# ═══════ PURE: Retry configuration ═══════

RetryConfig : {
    maxRetries : U64,
    initialDelayMs : U64,
    backoffMultiplier : U64,    # ×100 (150 = 1.5x)
    maxDelayMs : U64,
}

defaultRetryConfig = {
    maxRetries: 3,
    initialDelayMs: 1000,
    backoffMultiplier: 200,    # 2x
    maxDelayMs: 30000,
}

# Calculate delay for attempt n (exponential backoff)
calculateDelay = \config, attempt ->
    base = config.initialDelayMs
    multiplier = List.range { start: At 0, end: Before attempt }
        |> List.walk 100 \acc, _ -> acc * config.backoffMultiplier // 100
    delay = base * multiplier // 100
    Num.min delay config.maxDelayMs

# Should retry?
shouldRetry = \config, attempt, error ->
    if attempt >= config.maxRetries then Bool.false
    else
        when error is
            Timeout -> Bool.true
            ConnectionError -> Bool.true
            ServerError _ -> Bool.true
            ClientError _ -> Bool.false    # 4xx = don't retry
            _ -> Bool.false

# Tests
expect calculateDelay defaultRetryConfig 0 == 1000     # 1s
expect calculateDelay defaultRetryConfig 1 == 2000     # 2s
expect calculateDelay defaultRetryConfig 2 == 4000     # 4s
expect calculateDelay defaultRetryConfig 10 <= 30000   # capped

expect shouldRetry defaultRetryConfig 0 Timeout == Bool.true
expect shouldRetry defaultRetryConfig 3 Timeout == Bool.false   # max reached
expect shouldRetry defaultRetryConfig 0 (ClientError 404) == Bool.false
```

---

## 31.6 — Structured Logging

Observability = logs + metrics + traces. Logs nói "chuyện gì xảy ra", metrics nói "hệ thống khỏe không", traces nói "request đi qua đâu". Cần cả 3 để debug production.

```roc
# ═══════ PURE: Log entry builder ═══════

LogLevel : [Debug, Info, Warn, Error]

logLevelStr = \level ->
    when level is
        Debug -> "DEBUG"
        Info -> "INFO"
        Warn -> "WARN"
        Error -> "ERROR"

# Structured log = record, not string
LogEntry : {
    level : LogLevel,
    message : Str,
    context : Dict Str Str,
    timestamp : Str,
}

createLog = \level, message, timestamp ->
    { level, message, context: Dict.empty {}, timestamp }

withField = \entry, key, value ->
    { entry & context: Dict.insert entry.context key value }

# Format as JSON
formatLogJson = \entry ->
    fields = [
        ("level", "\"$(logLevelStr entry.level)\""),
        ("message", "\"$(entry.message)\""),
        ("timestamp", "\"$(entry.timestamp)\""),
    ]
    contextFields = Dict.walk entry.context [] \list, k, v ->
        List.append list ("$(k)", "\"$(v)\"")
    allFields = List.concat fields contextFields
        |> List.map \(k, v) -> "\"$(k)\":$(v)"
    "{$(Str.joinWith allFields ",")}"

# Usage
expect
    log = createLog Info "Order placed" "2024-03-15T10:00:00Z"
        |> withField "orderId" "42"
        |> withField "userId" "user-1"
        |> withField "total" "75000"
        |> formatLogJson
    Str.contains log "\"level\":\"INFO\""
    && Str.contains log "\"orderId\":\"42\""
```

---


## ✅ Checkpoint 31

> Đến đây bạn phải hiểu:
> 1. CAP theorem: Consistency + Availability + Partition tolerance — pick 2
> 2. Saga pattern = orchestrate distributed transactions (compensating actions)
> 3. Circuit Breaker = pure state machine: Closed → Open → HalfOpen
>
> **Test nhanh**: Hệ thống ngân hàng chọn CP hay AP?
> <details><summary>Đáp án</summary>CP (Consistency + Partition tolerance) — tiền phải chính xác.</details>

---

## 🏋️ Bài tập

**Bài 1** (15 phút): Saga cho e-commerce

Thiết kế saga cho checkout flow:

```roc
# Steps: ValidateCart → ReserveInventory → ProcessPayment → CreateShipment
# Compensations: -, ReleaseInventory, RefundPayment, CancelShipment
```

<details><summary>✅ Lời giải</summary>

```roc
checkoutSaga = [
    { name: "Validate Cart", action: "validateCart", compensation: "" },
    { name: "Reserve Inventory", action: "reserveInventory", compensation: "releaseInventory" },
    { name: "Process Payment", action: "processPayment", compensation: "refundPayment" },
    { name: "Create Shipment", action: "createShipment", compensation: "cancelShipment" },
]

# Simulate execution
simulateCheckout = \stepResults ->
    List.walkUntil (List.map2 checkoutSaga stepResults \step, result -> (step, result))
        { completed: [], failed: Bool.false, failReason: "" } \state, (step, result) ->
        when result is
            Ok ->
                Continue { state & completed: List.append state.completed step.name }
            Err reason ->
                Break { state & failed: Bool.true, failReason: reason }

expect
    result = simulateCheckout [Ok, Ok, Err "Insufficient funds", Ok]
    result.failed && result.failReason == "Insufficient funds"
    && List.len result.completed == 2
```

</details>

---

**Bài 2** (10 phút): Health check aggregator

```roc
# Aggregate health status từ multiple services
# All healthy → Healthy | Any degraded → Degraded | Any down → Unhealthy
```

<details><summary>✅ Lời giải</summary>

```roc
ServiceHealth : [Healthy, Degraded, Down]

aggregateHealth = \services ->
    List.walk services Healthy \overall, service ->
        when (overall, service.status) is
            (_, Down) -> Down
            (Down, _) -> Down
            (_, Degraded) -> Degraded
            (Degraded, _) -> Degraded
            (Healthy, Healthy) -> Healthy

expect
    services = [
        { name: "API", status: Healthy },
        { name: "DB", status: Healthy },
    ]
    aggregateHealth services == Healthy

expect
    services = [
        { name: "API", status: Healthy },
        { name: "DB", status: Down },
    ]
    aggregateHealth services == Down
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| CAP trade-off sai | Dùng AP cho financial data | Banking = CP (consistency first) |
| Saga compensation thiếu | Quên undo step | Mỗi action PHẢI có compensation |
| Circuit breaker quá sensitive | Threshold quá thấp | Tăng threshold, tăng reset timeout |
| Retry gây thundering herd | Tất cả retry cùng lúc | Thêm jitter (random delay) |

---

## Tóm tắt

- ✅ **CAP Theorem** — chọn 2 trong 3: Consistency, Availability, Partition Tolerance.
- ✅ **Consistency models** — strong (banking) vs eventual (social feed). Merge functions cho convergence.
- ✅ **Saga** = distributed transaction. Steps + compensations. Pure orchestration logic.
- ✅ **Circuit Breaker** = pure state machine: Closed→Open→HalfOpen. Protect against cascade failures.
- ✅ **Retry** = exponential backoff + max retries. Don't retry client errors.
- ✅ **Structured logging** = JSON log entries, `withField` builder pattern.
- ✅ **Roc insight** = distributed logic = pure transformations. Platform handles transport.

## Tiếp theo

→ Chapter 32: **System Design Thinking** — capacity estimation, API design, caching strategy, architecture patterns, design exercises.
