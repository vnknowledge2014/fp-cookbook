# ⚡ Practical Domain Design with Zig — Combined Book Outline

> Kết hợp: DDD Functional + FP Made Easier + Learn Go with Tests + F# Fun & Profit
> Coverage: **~30%** FP/DDD · Full CS foundations included
> Approach: Foundations → Beginner → Intermediate → Advanced → Principal
> Tổng: **34 chapters** + Appendices
> Note: Zig = imperative systems lang — sách tập trung DDD + systems + comptime

---

## Part 0: CS Foundations (Pre-requisite)

### Chapter 1 — Math & Logic Foundations
Discrete Math. Set theory, boolean algebra. Type theory: Product (structs) = AND, Sum (tagged unions) = OR. Binary & bit operations.

### Chapter 2 — Algorithmic Thinking & Complexity
Big-O: stack O(1), heap O(allocator). Memory complexity. Binary search, sorting. Cache-friendly DS: AoS vs SoA. Amortized: `ArrayList.append`.

### Chapter 3 — Data Structures — Systems Perspective
Contiguous: arrays, slices, `ArrayList`. Hash tables: `std.HashMap`. Trees: binary, B-tree. Graph: `[][]u32`. **Allocator-aware DS**. Arena allocator.

---

## Part I: Zig Fundamentals (Beginner)

### Chapter 4 — Getting Started
Install Zig, `zig init-exe`, build system. Philosophy: no hidden control flow.

### Chapter 5 — Values, Types & Operators
Integers, floats, bool, `?T`. `const` vs `var`. Comptime-known vs runtime.

### Chapter 6 — Control Flow
`if/else`, `switch`, `while`, `for`. `orelse`, `catch`. Labels, blocks.

### Chapter 7 — Functions
Declarations, function pointers `fn(i32) i32`, inline, `anytype`.

### Chapter 8 — Data Structures
Arrays, slices, structs. Tagged unions `union(enum)`. Packed/extern structs.

### Chapter 9 — Memory & Allocators ⭐
Stack vs heap. `std.mem.Allocator`, `GPA`, `ArenaAllocator`, `FixedBufferAllocator`.

### Chapter 10 — Error Handling
Error unions `!`. Error sets. `try`, `catch`, `orelse`. Error return traces.

---

## Part II: Zig Systems Patterns (Intermediate)

### Chapter 11 — Comptime ⭐
`comptime`. Generics: `fn max(comptime T: type, a: T, b: T) T`. Code generation.

### Chapter 12 — Comptime Generics & Type Functions
`fn ArrayList(comptime T: type) type`. `@TypeInfo` introspection.

### Chapter 13 — Interfaces via Comptime
`anytype` + comptime checks. Compile error = "interface not satisfied".

### Chapter 14 — Iterator Pattern
`fn next(self: *Self) ?T`. Manual map/filter. Lazy evaluation.

---

## Part III: Design Patterns (Advanced)

### Chapter 15 — GoF → Zig
Strategy = fn pointer. Observer = callback array. Command = tagged union. Factory = `init()`. Iterator = `next() ?T`.

### Chapter 16 — Domain Patterns in Systems Code
Newtype structs. State machine = tagged union. Error-driven workflows. Repository: comptime interface.

---

## Part IV: Domain-Driven Design (Advanced)

### Chapter 17 — Introduction to DDD
Ubiquitous Language, Bounded Contexts. Methodology cho mọi ngôn ngữ.

### Chapter 18 — Domain Modeling
Structs = products. Tagged unions = sums. Smart constructors: `pub fn init(raw: []const u8) !Email`.

### Chapter 19 — Error-Driven Workflows
Error unions. `try` chaining. Workflow = try sequence.

### Chapter 20 — Architecture & Modules
File = module. `pub`. `@import`. `build.zig.zon`. Bounded contexts = directories.

### Chapter 21 — Serialization
`std.json`. Struct ↔ JSON.

### Chapter 22 — Persistence & IO
`std.fs`, `std.net`. Allocator-aware persistence.

---

## Part V: Testing & Systems (Principal)

### Chapter 23 — Testing
`test "name" { ... }`. `std.testing.expect`. Test allocator = leak detection.

### Chapter 24 — Concurrency
`std.Thread`. `@atomicRmw`. Channels. Async I/O.

### Chapter 25 — HTTP & Web
`std.http`. `zap`/`http.zig`. JSON APIs.

### Chapter 26 — FFI & C Interop ⭐
`@cImport` — seamless C header import. Zig as "better C".

### Chapter 27 — Capstone Part 1: Domain Model ⭐
DDD model (structs + tagged unions), error-driven workflow, JSON, tests.

---

## Part VI: Production Engineering (Principal)

> *Database, Security, Distributed Systems, System Design — self-contained*

### Chapter 28 — Database Fundamentals & SQL
**Relational model**: tables, keys. **SQL**: `SELECT`, `JOIN`, `GROUP BY`, CTEs. **Normalization**. **Indexing**: B-Tree (implement in Zig!). **Transactions**: ACID, isolation levels. **Zig**: `zig-sqlite` (C binding via `@cImport`), PostgreSQL via `libpq` binding. **Custom B-tree storage**: Zig's allocator control = build your own storage engine.

### Chapter 29 — Advanced Data Patterns
**Migrations**: file-based SQL scripts. **Event Store**: append-only file, mmap. **Embedded DB**: SQLite (via `@cImport`), LMDB, RocksDB. **Caching**: in-process `HashMap`, allocator-aware LRU cache. **Memory-mapped files** for large datasets.

### Chapter 30 — Security Essentials
**Auth**: Password hashing (`std.crypto` — built-in argon2, bcrypt). **Tokens**: JWT manual implementation (Zig has fine-grained control). **TLS**: via `std.crypto` or `bearssl`/`wolfssl` C binding. **Zig advantage**: no buffer overflows (bounds checking), no use-after-free (manual but explicit).

### Chapter 31 — Application Security & Hardening
**OWASP Top 10**: buffer overflow impossible (bounds check), SQL injection (parameterized), input validation (comptime). **Fuzzing**: `zig test --fuzz` (built-in fuzzer!). **Memory safety**: test allocator catches leaks, use-after-free. **Address Sanitizer**: `zig build -Dsanitize=true`. **Zig security model**: explicit > implicit = fewer attack surfaces.

### Chapter 32 — Distributed Systems Fundamentals
**CAP Theorem**: CP vs AP. **Consistency models**. **Replication**: leader-follower. **Partitioning**: consistent hashing. **Consensus**: Raft (implement in Zig — good exercise). **Message passing**: TCP sockets (`std.net`), custom protocols. **Patterns**: Saga, Circuit Breaker, Retry. **Observability**: structured logging (custom), metrics. **io_uring**: Zig's async I/O = high-performance networking.

### Chapter 33 — System Design Thinking
**Capacity estimation**. **Load balancing**: implement L4 balancer in Zig. **Caching layers**: in-process, file-mapped, external. **Protocol design**: binary protocols (Zig excels), MessagePack, custom formats. **Microservices**: Zig as high-perf service in polyglot architecture. **Embedding**: Zig in C/C++ projects, WASM target. **Design exercises**: rate limiter, key-value store, custom protocol server.

### Chapter 34 — Capstone Part 2: Production Deployment ⭐
CLI tool + SQLite persistence, in-process cache, TLS, structured logging, fuzzing, Docker (minimal image), CI/CD, cross-compilation.

---

## Appendices
### A — Zig Comptime Patterns Cookbook
### B — From Rust/Go to Zig Translation Table
