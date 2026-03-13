# Appendix B — From Elm/Gleam to Roc Translation Table

> Bảng chuyển đổi nhanh cho developers từ Elm hoặc Gleam sang Roc.

---

## B.1 — Syntax Comparison

### Functions

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Define function | `add a b = a + b` | `fn add(a, b) { a + b }` | `add = \a, b -> a + b` |
| Type annotation | `add : Int -> Int -> Int` | `fn add(a: Int, b: Int) -> Int` | `add : I64, I64 -> I64` |
| Lambda | `\x -> x + 1` | `fn(x) { x + 1 }` | `\x -> x + 1` |
| Pipe | `x \|> f \|> g` | `x \|> f \|> g` | `x \|> f \|> g` |
| Apply | `f x y` | `f(x, y)` | `f x y` |

### Types

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Record | `{ name = "An" }` | N/A (custom types) | `{ name: "An" }` |
| Record access | `.name record` | `record.name` | `record.name` |
| Record update | `{ r \| name = "B" }` | `User(..user, name: "B")` | `{ r & name: "B" }` |
| Custom type | `type Color = Red \| Blue` | `type Color { Red Blue }` | `Color : [Red, Blue]` (tag union) |
| With payload | `type Msg = Click Int` | `type Msg { Click(Int) }` | `Msg : [Click I64]` |
| Opaque type | `type Email = Email String` | `opaque type Email { Email(String) }` | `Email := Str` |
| Type alias | `type alias User = { ... }` | `type User { ... }` | `User : { ... }` |
| Tuple | `(1, "a")` | `#(1, "a")` | `(1, "a")` |

### Pattern Matching

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Match | `case x of` | `case x {` | `when x is` |
| Wildcard | `_` | `_` | `_` |
| Destructure | `Just v -> ...` | `Some(v) -> ...` | `Ok v -> ...` |
| Guard | N/A | `if cond ->` | `x if cond ->` |
| Exhaustive | ✅ Required | ✅ Required | ✅ Required |

### Control Flow

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| If | `if c then a else b` | `case c { True -> ... }` | `if c then a else b` |
| Let | `let x = 1 in ...` | `let x = 1` | `x = 1` |
| Block | Implicit | `{ ... }` | Indentation-based |

---

## B.2 — Module System

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Define module | `module Main exposing (..)` | `pub fn ...` | `module [fn1, Type1]` |
| Import | `import List` | `import gleam/list` | `import cli.Stdout` |
| Qualified | `List.map` | `list.map` | `List.map` |
| Private | Not exported | No `pub` | Not in `module [...]` |
| File = Module | ✅ | ✅ | ✅ |

---

## B.3 — Error Handling

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Result type | `Result a b` | `Result(a, b)` | `Result a b` |
| Success | `Ok value` | `Ok(value)` | `Ok value` |
| Failure | `Err error` | `Error(error)` | `Err error` |
| Chain | `Result.andThen` | `result.try` | `Result.try` or `?` |
| Map | `Result.map` | `result.map` | `Result.map` |
| Early return | N/A | `use x <- result.try(r)` | `x = r?` (backpassing) |
| Maybe/Option | `Maybe a` | `Option(a)` | `Result a [NotFound]` (no Maybe!) |

### Roc's `?` operator (unique!)

```roc
# Elm: Result.andThen chains
# validate input |> Result.andThen process |> Result.andThen save

# Gleam: use keyword
# use validated <- result.try(validate(input))
# use processed <- result.try(process(validated))
# save(processed)

# Roc: ? suffix (most concise)
validated = validate? input
processed = process? validated
save processed
```

---

## B.4 — Effects & IO

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Side effects | `Cmd msg` | Direct (impure) | `Task ok err` |
| Pure by default | ✅ | ❌ | ✅ |
| IO execution | Runtime (TEA) | Direct | Platform |
| Print | `Debug.log` | `io.println` | `Stdout.line!` |
| HTTP | `Http.get` → Cmd | `httpc.request` | `Http.send` → Task |
| File read | N/A (browser) | `file.read` | `File.readUtf8!` |
| Effect marker | `Cmd`/`Sub` types | None | `!` suffix on functions |

### Key difference: Roc's `!` bang suffix

```roc
# Roc marks effectful calls with !
content = File.readUtf8! path    # ← ! means "performs effect"
Stdout.line! content             # ← ! means "performs effect"

# Pure functions: no !
result = List.map items transform  # ← no !, pure
```

---

## B.5 — Data Structures

| Structure | Elm | Gleam | Roc |
|-----------|-----|-------|-----|
| List | `List a` | `List(a)` | `List a` |
| Dict/Map | `Dict k v` | `dict.Dict(k, v)` | `Dict k v` |
| Set | `Set a` | `set.Set(a)` | `Set a` |
| String | `String` | `String` | `Str` |
| Int | `Int` | `Int` | `I64`, `U64`, `I32`, etc. |
| Float | `Float` | `Float` | `F64`, `F32`, `Dec` |
| Bool | `Bool` | `Bool` | `Bool` |
| Char | `Char` | N/A | N/A (use `U8`) |

### Numbers: Roc is more specific

```roc
# Elm:  Int, Float (2 types)
# Gleam: Int, Float (2 types)
# Roc:  I8, I16, I32, I64, I128,
#       U8, U16, U32, U64, U128,
#       F32, F64, Dec
#       (13 numeric types!)

# Default: Num * (inferred from usage)
x = 42        # Num * → becomes I64, U64, etc.
y = 3.14      # Frac * → becomes F64, Dec, etc.
```

---

## B.6 — Abilities / Typeclasses / Traits

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Mechanism | `comparable`, `number` | N/A | Abilities |
| User-defined | ❌ | ❌ | ❌ (built-in only) |
| Equality | `==` (structural) | `==` | `Eq` ability |
| Hashing | N/A | N/A | `Hash` ability |
| Debug print | `Debug.toString` | `string.inspect` | `Inspect` ability |
| Serialize | `Json.Encode` | N/A | `Encode`/`Decode` abilities |

```roc
# Opaque type với abilities
Email := Str implements [Eq, Hash, Inspect]

# Tự động derive cho:
# - Eq: so sánh == !=
# - Hash: dùng trong Dict/Set
# - Inspect: debug print
```

---

## B.7 — Testing

| Concept | Elm | Gleam | Roc |
|---------|-----|-------|-----|
| Framework | `elm-test` | `gleeunit` | Built-in! |
| Define test | `test "name" <\| ...` | `pub fn test_name()` | `expect expression` |
| Assertion | `Expect.equal a b` | `should.equal(a, b)` | `expect a == b` |
| Run | `elm-test` | `gleam test` | `roc test file.roc` |
| Inline | ❌ (separate file) | ❌ (test/ dir) | ✅ (same file!) |

```roc
# Roc: tests live WITH the code
add = \a, b -> a + b

expect add 2 3 == 5    # ← right here, next to the function!
expect add 0 0 == 0
```

---

## B.8 — Architecture Paradigm

| Paradigm | Elm | Gleam | Roc |
|----------|-----|-------|-----|
| Architecture | TEA (Model-View-Update) | OTP-inspired | Platform Separation |
| State | `Model` + messages | Processes + state | `Model` (webserver) or pure |
| UI | Virtual DOM | Lustre (optional) | N/A (backend focus) |
| Concurrency | Single-threaded | BEAM processes | Platform handles |
| Compilation | → JavaScript | → JavaScript / Erlang | → Native binary / WASM |

---

## B.9 — Quick Cheat Sheet

```
Elm developer → Roc:
  - Cú pháp tương tự, dùng \ thay cho anonymous functions
  - Result.andThen → dùng ? operator (ngắn hơn nhiều!)
  - Cmd/Sub → Task (tương tự concept)
  - Không có Maybe → dùng Result với custom error tag
  - Record syntax: { name = "An" } → { name: "An" }
  - Record update: { r | x = 1 } → { r & x = 1 }
  - Module: exposing (...) → module [...]
  - Platform = concept hoàn toàn mới (không có trong Elm)

Gleam developer → Roc:
  - Pure by default! (Gleam cho phép side effects tự do)
  - \ lambda syntax thay vì fn()
  - Tag unions structural (không cần declare type trước)
  - Records built-in (Gleam dùng custom types)
  - use keyword → ? operator
  - pub → module [...] list
  - BEAM runtime → native binary (fast cold start!)
```
