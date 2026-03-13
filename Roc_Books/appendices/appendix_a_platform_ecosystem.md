# Appendix A — Roc Platform Ecosystem

> Tổng hợp platforms, packages, và resources cho Roc developer.

---

## A.1 — Official Platforms

### basic-cli — CLI Applications

```roc
app [main] {
    cli: platform "https://github.com/roc-lang/basic-cli/releases/download/0.19.0/Hj-J_zxz7V9YurCSTFcFdu6cQJie4guzsPMUi5kBYUk.tar.br"
}
```

| Module | Chức năng |
|--------|-----------|
| `Stdout`, `Stderr` | In ra terminal |
| `Stdin` | Đọc input từ user |
| `File` | Đọc/ghi file (readUtf8, writeUtf8) |
| `Path` | Path manipulation |
| `Dir` | Directory operations |
| `Env` | Environment variables |
| `Arg` | Command-line arguments |
| `Http` | HTTP client (GET, POST, etc.) |
| `Sleep` | Delay execution |
| `Cmd` | Run external commands |
| `Utc` | UTC timestamps |

**Use case**: CLI tools, scripts, file processing, HTTP clients, automation.

---

### basic-webserver — Web Services

```roc
app [Model, server] {
    webserver: platform "https://github.com/roc-lang/basic-webserver/releases/download/0.10.0/lWg35nBCAzFBqEhIM0eBI_6hWM3e7h4iGjM6bBCwals.tar.br"
}
```

| Concept | Mô tả |
|---------|--------|
| `Model` | Server state type |
| `server.init` | Initialize model |
| `server.respond` | Handle requests |
| `Request` | `{ method, url, headers, body }` |
| `Response` | `{ status, headers, body }` |

**Use case**: REST APIs, JSON services, webhooks, web applications.

---

## A.2 — Platform Architecture

```
┌─────────────────────────────────────────┐
│              Your Roc App                │
│  ┌──────────────────────────────────┐   │
│  │         Pure Domain Code          │   │
│  │  (functions, types, logic)        │   │
│  │  → Testable via `expect`          │   │
│  └──────────────┬───────────────────┘   │
│                 │ calls                  │
│  ┌──────────────▼───────────────────┐   │
│  │         Platform API              │   │
│  │  (Stdout, File, Http, ...)        │   │
│  │  → Provided by platform           │   │
│  └──────────────┬───────────────────┘   │
└─────────────────┼───────────────────────┘
                  │ executes
┌─────────────────▼───────────────────────┐
│           Platform Runtime               │
│  (Rust/Zig/C compiled binary)            │
│  → Handles IO, memory, threading        │
│  → App CANNOT bypass this               │
└─────────────────────────────────────────┘
```

### Key properties

| Property | Giải thích |
|----------|-----------|
| **Isolation** | App không thể access IO mà platform không cung cấp |
| **Purity** | App code thuần — side effects chỉ qua platform |
| **Portability** | Cùng app code → chạy trên nhiều platforms |
| **Security** | Capability-based — platform cấp quyền |

---

## A.3 — Community Platforms

| Platform | Mô tả | Use case |
|----------|--------|----------|
| `roc-lang/basic-cli` | CLI toolkit | Scripts, tools |
| `roc-lang/basic-webserver` | HTTP server | REST APIs |
| `roc-lang/basic-ssg` | Static site generator | Blogs, docs |
| Community packages | Growing ecosystem | Various |

---

## A.4 — Package System

### Sử dụng packages

```roc
app [main] {
    cli: platform "...",
    json: "https://github.com/lukewilliamboswell/roc-json/releases/download/0.12.0/1trwx8sltQ-e9Y2rOB0LLmSs_R3TaGEB9ocnYeS4fms.tar.br",
}

import json.Json
```

### Tạo package

```roc
# Package = module + roc.toml
package [MyModule] {}

# MyModule.roc
module [functionA, functionB, TypeA]

# Expose qua package name
```

---

## A.5 — Tooling

| Tool | Command | Mô tả |
|------|---------|--------|
| Run | `roc run main.roc` | Chạy trực tiếp |
| Build | `roc build main.roc` | Compile binary |
| Test | `roc test main.roc` | Chạy `expect` tests |
| Format | `roc format main.roc` | Auto-format code |
| Check | `roc check main.roc` | Type-check không build |
| REPL | `roc repl` | Interactive REPL |
| Docs | `roc docs main.roc` | Generate docs |

---

## A.6 — Resources

| Resource | URL |
|----------|-----|
| Official site | [roc-lang.org](https://www.roc-lang.org) |
| GitHub | [github.com/roc-lang/roc](https://github.com/roc-lang/roc) |
| Tutorial | [roc-lang.org/tutorial](https://www.roc-lang.org/tutorial) |
| Zulip chat | [roc.zulipchat.com](https://roc.zulipchat.com) |
| Examples | [roc-lang/examples](https://github.com/roc-lang/examples) |
| Package index | [roc-lang.org/packages](https://www.roc-lang.org/packages) |

---

## A.7 — Roc vs Other FP Languages

| Feature | Roc | Elm | Haskell | OCaml |
|---------|-----|-----|---------|-------|
| Platform model | ✅ Built-in | ✅ (ports) | ❌ | ❌ |
| Pure by default | ✅ | ✅ | ✅ (lazy) | ❌ |
| No runtime exceptions | ✅ | ✅ | ❌ | ❌ |
| Type inference | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| Typeclasses/Abilities | ✅ (limited) | ❌ | ✅ (full) | ✅ (modules) |
| Compile to binary | ✅ | ❌ (JS) | ✅ | ✅ |
| GC | ❌ (ref-count) | ✅ (JS GC) | ✅ | ✅ |
| WASM support | ✅ | ❌ | Partial | ✅ |
| Learning curve | Low | Low | High | Medium |
