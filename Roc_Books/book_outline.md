# 🔷 Pure Functional Domain Design with Roc — Combined Book Outline

> Kết hợp: DDD Functional + FP Made Easier + Learn Go with Tests + F# Fun & Profit
> Coverage: **~90%** FP/DDD · Full CS foundations included
> Approach: Foundations → Beginner → Intermediate → Advanced → Principal
> Tổng: **33 chapters** + Appendices
> Note: Roc = Elm philosophy (no abstract Functor/Monad), practical FP only

---

## [Chapter 0 — Roc in 10 Minutes (Quick Primer)](part_0_cs_foundations/chapter_00_roc_in_10_minutes.md)

> *Đọc trước khi vào Part 0. Chỉ dạy đủ syntax để đọc hiểu code examples — Part I sẽ dạy lại sâu hơn.*

Install Roc, `roc run`. `app [main] { cli: platform ... }`. Values: numbers, strings, booleans. `\x -> x + 1` (lambda). `when...is` (pattern matching). Records `{ name: "An", age: 25 }`. Tags `Ok`, `Err`, `Confirmed`. `Stdout.line!`. `List.map`, `List.walk` ở mức "đọc hiểu". **Không dạy sâu** — để dành cho Part I.

---

## Part 0: CS Foundations (Pre-requisite)

> *Nền tảng CS cần thiết — không phụ thuộc ngôn ngữ nhưng ví dụ bằng Roc. Đọc Chapter 0 trước nếu chưa biết Roc.*

### [Chapter 1 — Math Foundations for FP](part_0_cs_foundations/chapter_01_math_foundations.md)
Lambda Calculus: `\x -> x+1`. Curry-Howard: Roc enforces natively. Discrete Math: Product (records) = AND, Tag unions = OR. Algebraic type sizes.

### [Chapter 2 — Algorithmic Thinking & Complexity](part_0_cs_foundations/chapter_02_algorithmic_thinking.md)
Big-O: `List.prepend` O(1), `List.append` O(n). Tail-call optimization. `List.walk` = fold. Binary search, divide & conquer.

### [Chapter 3 — Functional Data Structures](part_0_cs_foundations/chapter_03_functional_data_structures.md)
Roc Lists = reference-counted, unique ownership = in-place mutation khi possible. `Dict`, `Set`. Graph: `Dict Str (List Str)`. **Persistent but fast**: best of both worlds.

---

## Part I: Roc Fundamentals (Beginner)

### [Chapter 4 — Getting Started](part_1_fundamentals/chapter_04_getting_started.md)
Install Roc, `roc run`. REPL. **Platform concept**.

### [Chapter 5 — Values & Types](part_1_fundamentals/chapter_05_values_and_types.md)
Numbers, strings, booleans. Structural typing. Type inference.

### [Chapter 6 — Functions & Pipelines](part_1_fundamentals/chapter_06_functions_and_pipelines.md)
`add = \a, b -> a + b`. Auto-currying. `|>`. Backpassing `<-`.

### [Chapter 7 — Records & Tag Unions ⭐](part_1_fundamentals/chapter_07_records_and_tag_unions.md)
Structural records + structural sum types. No prior declaration.

### [Chapter 8 — Pattern Matching](part_1_fundamentals/chapter_08_pattern_matching.md)
`when value is`. Exhaustive. Destructuring. Guards.

### [Chapter 9 — Lists & Standard Functions](part_1_fundamentals/chapter_09_lists_and_standard_functions.md)
`List.map`, `List.filter`, `List.walk`. Dict, Set.

### [Chapter 10 — Opaque Types & Modules](part_1_fundamentals/chapter_10_opaque_types_and_modules.md)
`Email := Str`. Opaque = smart constructors. `exposes`, `imports`.

---

## Part II: Thinking Functionally (Intermediate)

### [Chapter 11 — Purity by Default](part_2_thinking_functionally/chapter_11_purity_by_default.md)
All functions pure. Side effects only through `Task`. No mutation. No exceptions.

### [Chapter 12 — Tasks & Effects ⭐](part_2_thinking_functionally/chapter_12_tasks_and_effects.md)
`Task ok err`. `Task.await`, `Task.map`. Platform handles execution.

### [Chapter 13 — Error Handling with `Result`](part_2_thinking_functionally/chapter_13_error_handling.md)
`Result ok err`. `Result.try` = bind. Chaining.

### [Chapter 14 — Abilities](part_2_thinking_functionally/chapter_14_abilities.md)
Built-in: `Eq`, `Hash`, `Inspect`, `Encode`, `Decode`. No user-defined.

---

## Part III: Design Patterns (Advanced)

### [Chapter 15 — FP Patterns](part_3_design_patterns/chapter_15_fp_patterns.md)
Strategy = fn value. Command = tag union. Middleware = `Task -> Task` composition.

### [Chapter 16 — Event Sourcing](part_3_design_patterns/chapter_16_event_sourcing.md)
Events as tag unions. `List.walk` rebuild. Projections.

---

## Part IV: Domain-Driven Design (Advanced)

### [Chapter 17 — Introduction to DDD](part_4_ddd/chapter_17_introduction_to_ddd.md)
Ubiquitous Language, Bounded Contexts, Event Storming.

### [Chapter 18 — Domain Modeling](part_4_ddd/chapter_18_domain_modeling.md)
Opaque types = domain values. Tag unions = state machines.

### [Chapter 19 — Workflows as Pipelines](part_4_ddd/chapter_19_workflows_as_pipelines.md)
`|>` + `Result.try`. Natural composition.

### [Chapter 20 — Platform Separation ⭐](part_4_ddd/chapter_20_platform_separation.md)
App = pure domain. Platform = IO. **Language-level Onion Architecture**.

### [Chapter 21 — Serialization](part_4_ddd/chapter_21_serialization.md)
`Encode`/`Decode` abilities. JSON via platform.

---

## Part V: Testing & Apps (Principal)

### [Chapter 22 — Testing](part_5_testing_and_apps/chapter_22_testing.md)
`expect` keyword. TDD cycle.

### [Chapter 23 — PBT Concepts](part_5_testing_and_apps/chapter_23_pbt_concepts.md)
Properties: round-trip, idempotent, metamorphic. PBT thinking.

### [Chapter 24 — CLI Applications](part_5_testing_and_apps/chapter_24_cli_applications.md)
`basic-cli`. Arguments. File I/O. HTTP.

### [Chapter 25 — Web Services](part_5_testing_and_apps/chapter_25_web_services.md)
`basic-webserver`. Routing, JSON API.

### [Chapter 26 — Capstone Part 1: Domain Model ⭐](part_5_testing_and_apps/chapter_26_capstone.md)
DDD model, pipelines, event sourcing, tests.

---

## Part VI: Production Engineering (Principal)

> *Database, Security, Distributed Systems, System Design — self-contained*

### [Chapter 27 — Database Fundamentals & SQL](part_6_production/chapter_27_database_fundamentals.md)
**Relational model**: tables, keys. **SQL**: `SELECT`, `JOIN`, CTEs. **Normalization**. **Indexing**: B-Tree, composite. **Transactions**: ACID, isolation levels. **Roc**: via `basic-cli` platform + SQLite binding. Platform-provided DB drivers.

### [Chapter 28 — Advanced Data Patterns](part_6_production/chapter_28_advanced_data_patterns.md)
**Migrations**: SQL files, versioning. **Event Store**: append-only. **Caching**: in-memory via platform, Redis via HTTP API. **Roc note**: platform handles IO, app stays pure — caching = platform concern.

### [Chapter 29 — Security Essentials](part_6_production/chapter_29_security_essentials.md)
**Auth**: Password hashing (via platform crypto). **Tokens**: JWT (encode/decode via abilities). **OAuth 2.0**: HTTP-based implementation. **Roc advantage**: pure functions = no state leaks, opaque types = encapsulation. Type system prevents many security bugs.

### [Chapter 30 — Application Security](part_6_production/chapter_30_application_security.md)
**OWASP Top 10**: injection (parameterized queries), XSS (template auto-escape), CSRF. **Roc security model**: purity = no side-channel attacks from pure code, opaque types = cannot inspect internals. **Input validation**: type-driven (parse via `Decode`). **Platform isolation**: app cannot access IO except through platform — capability-based security built-in!

### [Chapter 31 — Distributed Systems Fundamentals](part_6_production/chapter_31_distributed_systems.md)
**CAP Theorem**. **Consistency models**: strong, eventual. **Replication**: concepts. **Message passing**: HTTP-based messaging via platform. **Patterns**: Saga (pure orchestration logic + platform-side effects), Circuit Breaker (pure state machine), Retry. **Observability**: structured logging via `Inspect` ability. **Roc insight**: distributed logic = pure transformations + platform-handled transport.

### [Chapter 32 — System Design Thinking](part_6_production/chapter_32_system_design.md)
**Capacity estimation**. **Load balancing**: platform concern. **Caching**. **API design**: REST (webserver platform). **Architecture**: Roc as pure domain core in polyglot system — other languages handle infrastructure. **Serverless**: Roc → WASM → edge functions. **Design exercises**: URL shortener, rate limiter (pure state machine).

### [Chapter 33 — Capstone Part 2: Production Deployment ⭐](part_6_production/chapter_33_capstone_production.md)
Roc webserver + SQLite, caching, JWT auth, structured logging, Docker (minimal), CI/CD.

---

## Appendices
### [A — Roc Platform Ecosystem](appendices/appendix_a_platform_ecosystem.md)
### [B — From Elm/Gleam to Roc Translation Table](appendices/appendix_b_elm_gleam_translation.md)
### [C — Bảng thuật ngữ Anh-Việt (Glossary)](appendices/appendix_c_glossary.md)
### [D — Exercise Solutions Index](appendices/appendix_d_exercise_index.md)
### [E — Debugging & Tooling](appendices/appendix_e_debugging_and_tooling.md)
### [F — Migration Guides (Python/JS/Rust → Roc)](appendices/appendix_f_migration_guides.md)
### [G — Concurrency & Parallelism in Roc](appendices/appendix_g_concurrency.md)
