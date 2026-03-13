# 🐫 Domain-Driven FP with OCaml (Jane Street) — Combined Book Outline

> Kết hợp: DDD Functional + FP Made Easier + Learn Go with Tests + F# Fun & Profit
> Coverage: **~90%** FP/DDD · Full CS foundations included
> Approach: Foundations → Beginner → Intermediate → Advanced → Principal
> Tổng: **44 chapters** + Appendices

---

## Part 0: CS Foundations (Pre-requisite)

> *Nền tảng CS — ví dụ bằng OCaml*

### Chapter 1 — Math Foundations for FP
**Lambda Calculus**: `fun x -> x+1`, β-reduction. **Curry-Howard**: Types = Propositions, Programs = Proofs. **Discrete Math**: Product (records) = AND, Sum (variants) = OR. **Algebraic type sizes**.

### Chapter 2 — Algorithmic Thinking & Complexity
**Big-O**: O(1)→O(n²). **Tail recursion** + accumulator — sống còn trong OCaml. Divide & conquer, mergesort, binary search. Amortized analysis.

### Chapter 3 — Functional Data Structures
**Persistent**: cons lists, balanced trees, structural sharing. **Okasaki's Queue**: 2 lists. `Base.Map` = balanced BST. **Graph**: `Map.t`. **Big-O**: `List.cons` O(1), `Map.find` O(log n).

---

## Part I: OCaml Fundamentals (Beginner)

### Chapter 4 — Getting Started
`opam`, `dune`, `utop`. Jane Street `Base`. Project structure.

### Chapter 5 — Values, Types & Expressions
`let` bindings (immutable). Type inference. Primitives. Expressions. `ref` cho mutation (hiếm).

### Chapter 6 — Control Flow & Pattern Matching
`if/then/else` (expression). `match...with` — exhaustive. Guards `when`. Wildcards.

### Chapter 7 — Functions
Auto-currying. Partial application. `|>`. Labeled `~name`. Optional `?opt`. `let rec`. `fun x -> ...`.

### Chapter 8 — Core Data Types
Tuples, lists, arrays. `Option`, `Result`. Records. Copy-update `{ r with ... }`.

### Chapter 9 — Variants & Algebraic Types
`type shape = Circle of float | Rectangle of float * float`. Polymorphic variants. Recursive types.

### Chapter 10 — Modules & Signatures
`module M = struct ... end`. `.mli`. `include`, `open`. Modules = namespaces + encapsulation.

---

## Part II: Thinking Functionally (Intermediate)

### Chapter 11 — Purity, Immutability & Referential Transparency
Immutable by default. Side-effects qua explicit types. Memoization. Pure function composition.

### Chapter 12 — Higher-Order Functions & `Base`
`List.map`, `filter`, `fold`. `Base` API: `choose`, `filter_map`, `find`, `exists`, `for_all`, `group_by`.

### Chapter 13 — Functors (Module Functors)
`module type Comparable`, `module Make(Ord: Comparable)`. `Map.Make`, `Set.Make`. **≠ CT functors**.

### Chapter 14 — Error Handling & `Or_error`
`Result.t`, `Or_error.t`. `ok_exn`, `error_s`. `Result.bind`, `Result.map`. ROP tự nhiên.

### Chapter 15 — PPX & Deriving
`[@@deriving sexp, compare, equal, hash]`. `ppx_jane`. Custom derivers.

---

## Part III: Design Patterns — FP & Classical (Advanced)

### Chapter 16 — GoF → FP Translation
**Strategy** = HOF. **Command** = variant. **Visitor** = pattern match. **Factory** = smart constructor. **Decorator** = composition. **Iterator** = `Sequence.t`.

### Chapter 17 — CQRS & Event Sourcing
Tách read/write. Event Sourcing: events, `List.fold` rebuild. Projections, snapshots.

---

## Part IV: Domain-Driven Design (Advanced)

### Chapter 18 — Introduction to DDD
Ubiquitous Language, Bounded Contexts, Event Storming.

### Chapter 19 — Functional Architecture
Onion Architecture. `.mli` = bounded context contracts. `Async` push IO to edges.

### Chapter 20 — Domain Modeling with OCaml Types
Wrapper types. Abstract types via `.mli`. Smart constructors. State machines. Near 1:1 with F#.

### Chapter 21 — Workflows as Pipelines
`|>` native. `validate |> price |> acknowledge`. `Result` chaining.

### Chapter 22 — ROP with `let%bind` and `let%map` ⭐
`ppx_let`: monadic bind syntax. Do-notation equivalent. `Deferred.Or_error` for async ROP.

### Chapter 23 — Serialization
`ppx_sexp_conv`, `ppx_yojson_conv`, `Jsonaf`. Domain ↔ DTO.

### Chapter 24 — Persistence & Side Effects
Repository module signatures. DI via functors or first-class modules. CQS. `Async`.

---

## Part V: Deep FP Patterns (Advanced → Principal)

### Chapter 25 — Abstract Algebra
Semigroup, Monoid via module signatures. `Base.Comparable`, `Base.Hashable`.

### Chapter 26 — Functors (Category Theory)
`Base.Monad.Make`. `map` interface. Functor laws. **Distinguish from module functors**.

### Chapter 27 — Monads & `Base.Monad`
`bind`, `return`, `map`, `join`. `let%bind`/`let%map`. `Option`, `Result`, `Deferred`.

### Chapter 28 — Applicative & Validation
`Base.Applicative`. `let%map x = a and y = b in ...`. Collect ALL errors. `Validate`.

### Chapter 29 — Monad Transformers via Module Functors ⭐
`Deferred.Or_error.t` = stacked. Module functors naturally stack monads.

### Chapter 30 — Computation Expressions — OCaml Way
`let%bind`, `let%map`, `match%bind`, `if%bind`. Compare F# CE, Haskell do.

### Chapter 31 — Parser Combinators with Angstrom ⭐
`Angstrom`. `>>|`, `>>=`, `<|>`, `many`, `many1`. JSON parser.

### Chapter 32 — Recursive Types & Catamorphisms
Recursive variants. `fold` for trees. Catamorphisms. Tail recursion.

---

## Part VI: Testing & Web (Principal)

### Chapter 33 — Testing with `ppx_expect` & `ppx_inline_test`
`let%expect_test`. Inline tests. Snapshot testing.

### Chapter 34 — Property-Based Testing
`ppx_quickcheck`. `Base_quickcheck`. Generator, shrinker.

### Chapter 35 — Async & Concurrency
`Async`. `Deferred.t`. `Pipe`. `Tcp`. **Multicore OCaml 5.0**: domains, effects. `Eio`.

### Chapter 36 — Building Web Applications
`Dream`, `Cohttp`. REST APIs. `Bonsai` / `Incr_dom` frontend. `js_of_ocaml`.

### Chapter 37 — Capstone Part 1: Domain Model ⭐
Order-Taking System: domain model (variants + records), pipeline (`let%bind`), CQRS, tests.

---

## Part VII: Production Engineering (Principal)

> *Database, Security, Distributed Systems, System Design — self-contained*

### Chapter 38 — Database Fundamentals & SQL
**Relational model**: tables, keys. **SQL**: `SELECT`, `JOIN`, `GROUP BY`, CTEs. **Normalization** 1NF→BCNF. **Indexing**: B-Tree, composite, `EXPLAIN ANALYZE`. **Transactions**: ACID, isolation levels, locking. **OCaml**: `caqti` (generic DB interface), `pgx` (PostgreSQL), `irmin` (Git-like store). Connection pooling.

### Chapter 39 — Advanced Data Patterns
**Migrations**: schema versioning. **CQRS persistence**: separate read/write DBs. **Event Store**: append-only. **NoSQL**: Document, KV, Column, Graph — khi nào dùng. **Caching**: strategies, invalidation. **OCaml**: `irmin` CRDT store, `redis` bindings.

### Chapter 40 — Security Essentials
**Auth**: Password hashing (`argon2`/`bcrypt`), salt. **Sessions vs Tokens**: JWT, refresh tokens, PASETO. **OAuth 2.0**: Authorization Code + PKCE. **Authorization**: RBAC, ABAC, capability-based. **OCaml**: `mirage-crypto`, `x509`, `dream` sessions.

### Chapter 41 — Application Security & Hardening
**OWASP Top 10**: Injection, XSS, CSRF, SSRF. **Input validation**: type-driven (OCaml's type system helps!). **TLS**: `tls` library, `ocaml-letsencrypt`. **Headers**: CSP, HSTS. **Secrets**: env vars, `dotenv`. **Rate limiting**. **Audit logging**.

### Chapter 42 — Distributed Systems Fundamentals
**CAP Theorem**: CP vs AP. **Consistency models**: strong, eventual, causal. **Replication**: leader-follower, leaderless. CRDTs. **Partitioning**: hash, range, consistent hashing. **Consensus**: Raft basics. **Message Queues**: delivery guarantees. **Patterns**: Saga, Circuit Breaker, Retry+backoff, Outbox, CDC. **Observability**: structured logging, metrics, tracing. **OCaml**: MirageOS unikernels, `Irmin` CRDTs, `amqp-client` (RabbitMQ).

### Chapter 43 — System Design Thinking
**Capacity estimation**: QPS, storage. **Load balancing**: L4/L7, consistent hashing. **Caching layers**: client, CDN, app, DB. **API design**: REST vs gRPC vs GraphQL. **Microservices**: Conway's Law, monolith-first. **Database scaling**: replicas, sharding. **Design exercises**: URL shortener, chat, rate limiter.

### Chapter 44 — Capstone Part 2: Production Deployment ⭐
Order-Taking + PostgreSQL (`caqti`), caching, JWT auth, TLS, logging, Docker, CI/CD.

---

## Appendices
### A — Jane Street Ecosystem Map
`Base` → `Core` → `Async`, `ppx_jane`, `Expect_test`, `Incremental`, `Bonsai`, `Angstrom`.
### B — From F#/PureScript to OCaml Translation Table
