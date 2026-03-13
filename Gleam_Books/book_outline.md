# ✨ Functional Domain Design with Gleam — Combined Book Outline

> Kết hợp: DDD Functional + FP Made Easier + Learn Go with Tests + F# Fun & Profit
> Coverage: **~55%** FP/DDD · Full CS foundations included
> Approach: Foundations → Beginner → Intermediate → Advanced → Principal
> Tổng: **34 chapters** + Appendices

---

## Part 0: CS Foundations (Pre-requisite)

### Chapter 1 — Math Foundations for FP
Lambda Calculus: `fn(x) { x + 1 }`, β-reduction. Curry-Howard. Discrete Math: Product (custom types) = AND, Variants = OR. Algebraic type sizes.

### Chapter 2 — Algorithmic Thinking & Complexity
Big-O: `List` prepend O(1), append O(n). **Recursion bắt buộc** — Gleam không có loops. Tail recursion, accumulator. Binary search. Mergesort recursive.

### Chapter 3 — Functional Data Structures
Persistent linked list: cons O(1), random access O(n). `Dict` = immutable map. Okasaki's Queue. Graph: `Dict(String, List(String))`. BEAM structural sharing.

---

## Part I: Gleam Fundamentals (Beginner)

### Chapter 4 — Getting Started
Install Gleam, Erlang/OTP. `gleam new`, `gleam run`, `gleam test`. BEAM + JavaScript targets.

### Chapter 5 — Values, Types & Expressions
`let` (always immutable). Ints, Floats, Strings, Bools. No `null`. Type inference.

### Chapter 6 — Functions & Pipelines
`fn add(a: Int, b: Int) -> Int`. Anonymous functions. `|>`. Labeled arguments.

### Chapter 7 — Custom Types — ADTs
Records + Variants. **Sum + Product native**.

### Chapter 8 — Pattern Matching
`case value { ... }`. Exhaustive. Destructuring, guards.

### Chapter 9 — Lists, Tuples & Stdlib
`List`, `Dict`. `list.map`, `list.filter`, `list.fold`. `gleam/result`, `gleam/option`.

### Chapter 10 — Modules & Opaque Types
`pub`. **Opaque types** = smart constructors: `pub opaque type Email { Email(String) }`.

---

## Part II: Thinking Functionally (Intermediate)

### Chapter 11 — Immutability & the Gleam Way
Everything immutable. Update = create new. Simplicity > power.

### Chapter 12 — Recursion & Tail Calls
No loops. TCO. Recursive list processing. Accumulator pattern.

### Chapter 13 — Higher-Order Functions
`list.map`, `list.filter`, `list.fold`, `list.flat_map`. Closures.

### Chapter 14 — Error Handling with `Result`
`Result(Ok, Error)`. `use user <- result.try(find_user(id))`.

### Chapter 15 — The `use` Expression ⭐
Desugars to callback. **Gleam's answer to monads**. Error handling, resource management.

---

## Part III: Design Patterns (Advanced)

### Chapter 16 — FP Patterns in Gleam
Strategy = HOF. Command = custom type. Factory = opaque constructor. Middleware = `fn(Request, fn(Request) -> Response) -> Response`.

### Chapter 17 — Event Sourcing
Events as custom types. `list.fold` rebuild. Projections.

---

## Part IV: Domain-Driven Design (Advanced)

### Chapter 18 — Introduction to DDD
Ubiquitous Language, Bounded Contexts, Event Storming.

### Chapter 19 — Domain Modeling
Custom types = domain model. Opaque = smart constructors. State machines.

### Chapter 20 — Workflows as Pipelines
`|>` + `Result` + `use`.

### Chapter 21 — Serialization
`gleam/json` encoder/decoder. Domain ↔ DTO.

### Chapter 22 — Architecture & Side Effects
OTP processes for IO. Functional core + imperative shell.

---

## Part V: OTP, Testing & Web (Principal)

### Chapter 23 — OTP Fundamentals
Processes, `gleam/otp/actor`. Supervisors. Actors.

### Chapter 24 — Building HTTP Services
`wisp` / `mist`. Routing, middleware, JSON.

### Chapter 25 — Testing
`gleam test`. TDD cycle. PBT via Erlang `proper`.

### Chapter 26 — Gleam on JavaScript + Lustre
JS target. `lustre` = Elm-like MVU frontend. SSR.

### Chapter 27 — Capstone Part 1: Domain Model ⭐
DDD model, pipelines, event sourcing, wisp API, OTP actors, tests.

---

## Part VI: Production Engineering (Principal)

> *Database, Security, Distributed Systems, System Design — self-contained*

### Chapter 28 — Database Fundamentals & SQL
**Relational model**: tables, keys. **SQL**: `SELECT`, `JOIN`, `GROUP BY`, CTEs. **Normalization** 1NF→BCNF. **Indexing**: B-Tree, composite. **Transactions**: ACID, isolation levels. **Gleam**: `gleam_pgo` (PostgreSQL), `sqlight` (SQLite). **Ecto patterns** via Elixir interop.

### Chapter 29 — Advanced Data Patterns
**Migrations**: schema versioning. **Event Store**: append-only. **NoSQL**: Redis, ETS (BEAM built-in in-memory). **Caching**: ETS tables (BEAM native, zero-cost), Redis via Erlang libraries. **Mnesia**: BEAM's built-in distributed DB.

### Chapter 30 — Security Essentials
**Auth**: Password hashing (`argon2`/`bcrypt` via Erlang). **Sessions vs Tokens**: JWT, session cookies via `wisp`. **OAuth 2.0**: PKCE. **Authorization**: RBAC, middleware guards.

### Chapter 31 — Application Security & Hardening
**OWASP Top 10**: SQL injection (parameterized queries), XSS (template auto-escape), CSRF. **Input validation**: type-driven (Gleam's type system!). **TLS**: BEAM SSL. **CORS**: middleware. **Rate limiting**: token bucket. **Logging**: `gleam/erlang/process` + structured logging.

### Chapter 32 — Distributed Systems — BEAM Advantage ⭐
**CAP Theorem**. **BEAM distribution native**: `Node.connect`, distributed processes, `pg` (process groups). **CRDTs**: built on BEAM. **Partitioning**: consistent hashing. **Message Queues**: BEAM mailboxes = built-in queues. External: RabbitMQ (`amqp` Erlang lib). **Patterns**: Saga (via OTP), Circuit Breaker, Retry. **Observability**: `telemetry` (BEAM ecosystem), OpenTelemetry. **BEAM advantage**: WhatsApp, Discord, RabbitMQ — battle-tested distributed systems.

### Chapter 33 — System Design Thinking
**Capacity estimation**. **Load balancing**: BEAM schedulers auto-balance, Nginx frontend. **Caching**: ETS (blazing fast), CDN. **API design**: REST (wisp) vs WebSocket (BEAM native). **Clustering**: BEAM clustering = horizontal scaling built-in. **Hot code upgrades**. **Design exercises**: chat system (WhatsApp-like), rate limiter, notification service.

### Chapter 34 — Capstone Part 2: Production Deployment ⭐
Gleam + PostgreSQL (`gleam_pgo`), ETS cache, JWT auth, TLS, rate limiting, `telemetry`, Docker, clustering, Fly.io (BEAM-optimized deployment).

---

## Appendices
### A — From F# to Gleam Translation Table
### B — Gleam + BEAM Ecosystem Map
