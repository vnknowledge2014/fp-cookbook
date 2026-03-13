# Chapter 25 — Web Services

> **Bạn sẽ học được**:
> - `basic-webserver` platform — setup web server trong Roc
> - **Routing** — pattern matching trên method + path
> - **JSON API** — parse request body, trả JSON response
> - **In-memory state** — quản lý data trong server
> - REST API patterns — CRUD endpoints
> - Xây API hoàn chỉnh: Todo API
>
> **Yêu cầu trước**: [Chapter 24 — CLI Applications](chapter_24_cli_applications.md)
> **Thời gian đọc**: ~45 phút | **Level**: Principal
> **Kết quả cuối cùng**: Xây REST API hoàn chỉnh — routing, JSON, CRUD, error handling.

---

Web service là ứng dụng phổ biến nhất trên thế giới — từ REST API đến GraphQL. Roc xử lý HTTP requests theo cách FP: mỗi request là input, response là output, middleware là function composition. Không class, không annotation, không magic.

## Web Services — Roc trên server

Roc platform model cho phép web development: `roc-server` platform cung cấp HTTP handling. Bạn viết pure request handlers, platform quản lý server infrastructure. Chapter này xây dựng REST API.


## 25.1 — basic-webserver Platform

Platform `basic-webserver` biến Roc function thành HTTP server. Bạn viết function nhận Request trả Response — platform lo phần socket, concurrency, parsing.

### Setup

```roc
# filename: main.roc
app [Model, server] {
    webserver: platform "https://github.com/roc-lang/basic-webserver/releases/download/0.10.0/lWg35nBCAzFBqEhIM0eBI_6hWM3e7h4iGjM6bBCwals.tar.br"
}

import webserver.Webserver exposing [Request, Response]

# Model = trạng thái server (in-memory)
Model : {}

# server = { init, respond }
server = { init: Task.ok {}, respond }

respond : Request, Model -> Task Response [ServerErr Str]_
respond = \req, _model ->
    Task.ok { status: 200, headers: [], body: Str.toUtf8 "Hello from Roc! 🚀" }
```

```bash
roc run main.roc
# Server listening on http://localhost:8000

curl http://localhost:8000
# Hello from Roc! 🚀
```

---

## 25.2 — Routing

Routing map URL paths tới handler functions. `GET /api/users` → `listUsers`, `POST /api/users` → `createUser`. Pattern matching trên HTTP method + path — FP style.

```roc
# filename: main.roc
app [Model, server] {
    webserver: platform "https://github.com/roc-lang/basic-webserver/releases/download/0.10.0/lWg35nBCAzFBqEhIM0eBI_6hWM3e7h4iGjM6bBCwals.tar.br"
}

import webserver.Webserver exposing [Request, Response]

Model : {}
server = { init: Task.ok {}, respond }

respond : Request, Model -> Task Response [ServerErr Str]_
respond = \req, _model ->
    # Routing = pattern matching trên method + url
    when (req.method, req.url) is
        (Get, "/") ->
            ok200 "🏠 Trang chủ"

        (Get, "/about") ->
            ok200 "ℹ️ About: Roc Web Server v1.0"

        (Get, "/health") ->
            jsonResponse 200 "{\"status\":\"ok\",\"uptime\":\"running\"}"

        (Get, url) if Str.startsWith url "/hello/" ->
            name = Str.replaceFirst url "/hello/" ""
            ok200 "👋 Xin chào, $(name)!"

        (_, "/") ->
            methodNotAllowed

        _ ->
            notFound req.url

# ═══════ RESPONSE HELPERS ═══════

ok200 = \body -> Task.ok {
    status: 200,
    headers: [{ name: "Content-Type", value: Str.toUtf8 "text/plain; charset=utf-8" }],
    body: Str.toUtf8 body,
}

jsonResponse = \status, body -> Task.ok {
    status,
    headers: [{ name: "Content-Type", value: Str.toUtf8 "application/json" }],
    body: Str.toUtf8 body,
}

notFound = \url -> Task.ok {
    status: 404,
    headers: [],
    body: Str.toUtf8 "404 Not Found: $(url)",
}

methodNotAllowed = Task.ok {
    status: 405,
    headers: [],
    body: Str.toUtf8 "405 Method Not Allowed",
}
```

```bash
curl http://localhost:8000/
# 🏠 Trang chủ

curl http://localhost:8000/hello/An
# 👋 Xin chào, An!

curl http://localhost:8000/health
# {"status":"ok","uptime":"running"}

curl http://localhost:8000/unknown
# 404 Not Found: /unknown
```

---

## 25.3 — JSON Request/Response

Middleware trong FP là function composition: `logging(auth(rateLimit(handler)))`. Mỗi middleware nhận handler và trả handler mới — pipeline tự nhiên.

```roc
# ═══════ PURE: JSON helpers ═══════

# Simple JSON builder
jsonObject = \pairs ->
    fields = List.map pairs \(key, value) ->
        "\"$(key)\":$(value)"
    "{$(Str.joinWith fields ",")}"

jsonStr = \s -> "\"$(s)\""
jsonNum = \n -> Num.toStr n
jsonBool = \b -> if b then "true" else "false"

jsonList = \items ->
    "[$(Str.joinWith items ",")]"

# Ví dụ sử dụng
# jsonObject [("name", jsonStr "An"), ("age", jsonNum 25), ("active", jsonBool Bool.true)]
# → {"name":"An","age":"25","active":true}

# ═══════ PURE: Parse request body ═══════

# Simplified JSON parser cho key=value
parseJsonField : Str, Str -> Result Str [FieldNotFound]
parseJsonField = \json, field ->
    pattern = "\"$(field)\":"
    when Str.splitFirst json pattern is
        Ok { after, .. } ->
            trimmed = Str.trim after
            if Str.startsWith trimmed "\"" then
                rest = Str.replaceFirst trimmed "\"" ""
                when Str.splitFirst rest "\"" is
                    Ok { before, .. } -> Ok before
                    Err _ -> Err FieldNotFound
            else
                when Str.splitFirst trimmed "," is
                    Ok { before, .. } -> Ok (Str.trim before)
                    Err _ ->
                        when Str.splitFirst trimmed "}" is
                            Ok { before, .. } -> Ok (Str.trim before)
                            Err _ -> Err FieldNotFound
        Err _ -> Err FieldNotFound
```

---

## 25.4 — REST API: Todo App

Request parsing: extract path params, query params, body JSON. Mỗi operation trả Result — lỗi parse tự động trở thành 400 Bad Request response.

```roc
# filename: main.roc
app [Model, server] {
    webserver: platform "https://github.com/roc-lang/basic-webserver/releases/download/0.10.0/lWg35nBCAzFBqEhIM0eBI_6hWM3e7h4iGjM6bBCwals.tar.br"
}

import webserver.Webserver exposing [Request, Response]

# ═══════ MODEL ═══════

Model : {
    todos: List { id : U64, title : Str, done : Bool },
    nextId : U64,
}

server = {
    init: Task.ok { todos: [], nextId: 1 },
    respond,
}

# ═══════ PURE CORE ═══════

addTodo : Model, Str -> Model
addTodo = \model, title ->
    todo = { id: model.nextId, title, done: Bool.false }
    { todos: List.append model.todos todo, nextId: model.nextId + 1 }

toggleTodo : Model, U64 -> Model
toggleTodo = \model, id ->
    todos = List.map model.todos \t ->
        if t.id == id then { t & done: !(t.done) } else t
    { model & todos }

deleteTodo : Model, U64 -> Model
deleteTodo = \model, id ->
    { model & todos: List.keepIf model.todos \t -> t.id != id }

findTodo : Model, U64 -> Result { id : U64, title : Str, done : Bool } [NotFound]
findTodo = \model, id ->
    List.findFirst model.todos \t -> t.id == id
    |> Result.mapErr \_ -> NotFound

# ═══════ JSON SERIALIZATION ═══════

todoToJson = \todo ->
    jsonObject [
        ("id", jsonNum todo.id),
        ("title", jsonStr todo.title),
        ("done", jsonBool todo.done),
    ]

todosToJson = \todos ->
    items = List.map todos todoToJson
    jsonList items

jsonObject = \pairs ->
    fields = List.map pairs \(key, value) -> "\"$(key)\":$(value)"
    "{$(Str.joinWith fields ",")}"

jsonStr = \s -> "\"$(s)\""
jsonNum = \n -> Num.toStr n
jsonBool = \b -> if b then "true" else "false"
jsonList = \items -> "[$(Str.joinWith items ",")]"

# ═══════ ROUTER ═══════

respond : Request, Model -> Task Response [ServerErr Str]_
respond = \req, model ->
    when (req.method, req.url) is
        # GET /api/todos — list all
        (Get, "/api/todos") ->
            body = todosToJson model.todos
            jsonResponse 200 body

        # GET /api/todos/:id — get one
        (Get, url) if Str.startsWith url "/api/todos/" ->
            idStr = Str.replaceFirst url "/api/todos/" ""
            when Str.toU64 idStr is
                Ok id ->
                    when findTodo model id is
                        Ok todo -> jsonResponse 200 (todoToJson todo)
                        Err NotFound -> jsonResponse 404 "{\"error\":\"Todo not found\"}"
                Err _ ->
                    jsonResponse 400 "{\"error\":\"Invalid ID\"}"

        # POST /api/todos — create
        (Post, "/api/todos") ->
            bodyStr = req.body |> Str.fromUtf8 |> Result.withDefault ""
            when parseJsonField bodyStr "title" is
                Ok title ->
                    newModel = addTodo model title
                    newTodo = List.last newModel.todos |> Result.withDefault { id: 0, title: "", done: Bool.false }
                    jsonResponse 201 (todoToJson newTodo)
                Err _ ->
                    jsonResponse 400 "{\"error\":\"Missing 'title' field\"}"

        # PUT /api/todos/:id/toggle — toggle done
        (Put, url) if Str.startsWith url "/api/todos/" && Str.endsWith url "/toggle" ->
            idStr = url
                |> Str.replaceFirst "/api/todos/" ""
                |> Str.replaceFirst "/toggle" ""
            when Str.toU64 idStr is
                Ok id ->
                    when findTodo model id is
                        Ok _ ->
                            _newModel = toggleTodo model id
                            jsonResponse 200 "{\"message\":\"Toggled\"}"
                        Err NotFound ->
                            jsonResponse 404 "{\"error\":\"Todo not found\"}"
                Err _ ->
                    jsonResponse 400 "{\"error\":\"Invalid ID\"}"

        # DELETE /api/todos/:id — delete
        (Delete, url) if Str.startsWith url "/api/todos/" ->
            idStr = Str.replaceFirst url "/api/todos/" ""
            when Str.toU64 idStr is
                Ok id ->
                    when findTodo model id is
                        Ok _ ->
                            _newModel = deleteTodo model id
                            jsonResponse 200 "{\"message\":\"Deleted\"}"
                        Err NotFound ->
                            jsonResponse 404 "{\"error\":\"Todo not found\"}"
                Err _ ->
                    jsonResponse 400 "{\"error\":\"Invalid ID\"}"

        # Health check
        (Get, "/health") ->
            jsonResponse 200 "{\"status\":\"ok\",\"todos\":$(jsonNum (Num.toU64 (List.len model.todos)))}"

        # Catch-all
        _ ->
            jsonResponse 404 "{\"error\":\"Not found: $(req.url)\"}"

jsonResponse = \status, body -> Task.ok {
    status,
    headers: [
        { name: "Content-Type", value: Str.toUtf8 "application/json" },
        { name: "Access-Control-Allow-Origin", value: Str.toUtf8 "*" },
    ],
    body: Str.toUtf8 body,
}
```

```bash
# Create
curl -X POST http://localhost:8000/api/todos \
  -d '{"title":"Học Roc"}' | jq
# {"id":1,"title":"Học Roc","done":false}

# List
curl http://localhost:8000/api/todos | jq
# [{"id":1,"title":"Học Roc","done":false}]

# Toggle
curl -X PUT http://localhost:8000/api/todos/1/toggle
# {"message":"Toggled"}

# Delete
curl -X DELETE http://localhost:8000/api/todos/1
# {"message":"Deleted"}
```

---

## 25.5 — Error Handling Pattern

Response building: status code, headers, body. JSON responses cho API, HTML cho web pages. Helper functions giúp tạo response chuẩn — `jsonOk`, `notFound`, `badRequest`.

```roc
# Pattern: mọi error trả JSON consistent

errorResponse = \status, message ->
    jsonResponse status (jsonObject [("error", jsonStr message)])

# Sử dụng
# errorResponse 400 "Missing required field"
# → 400 {"error":"Missing required field"}

# errorResponse 404 "Resource not found"
# → 404 {"error":"Resource not found"}

# errorResponse 500 "Internal server error"
# → 500 {"error":"Internal server error"}
```

---

## 25.6 — Kiến trúc Web Service

Ví dụ hoàn chỉnh kết hợp mọi thứ: REST API cho task management với CRUD operations, middleware, error handling, và JSON serialization.

```
┌─────────────────────────────────────────────┐
│                 main.roc                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Router   │→│  Handler  │→│ Response  │  │
│  │(matching) │  │ (logic)  │  │(JSON fmt) │  │
│  └──────────┘  └────┬─────┘  └──────────┘  │
│                     │                        │
│  ┌──────────────────▼──────────────────┐    │
│  │          Domain (Pure Core)          │    │
│  │  addTodo, toggleTodo, deleteTodo     │    │
│  │  findTodo, validation, formatting    │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

- **Router** = pattern match `(method, url)` → dispatch
- **Handler** = parse request → call domain → format response
- **Domain** = pure functions, testable bằng `expect`
- **Response helpers** = `jsonResponse`, `errorResponse`

---


## ✅ Checkpoint 25

> Đến đây bạn phải hiểu:
> 1. `basic-webserver`: `Model` (state) + `server` (init + respond)
> 2. `respond : Request, Model -> Response` = function xử lý HTTP
> 3. Routing = pattern matching trên `request.url` + `request.method`
>
> **Test nhanh**: `server.respond` xử lý request bằng gì?
> <details><summary>Đáp án</summary>Pattern matching trên `request.method` và `request.url` — pure function!</details>

---

## 🏋️ Bài tập

**Bài 1** (15 phút): Bookmark API

Xây API quản lý bookmarks:

```
GET    /api/bookmarks         — list all
POST   /api/bookmarks         — add {url, title, tags}
GET    /api/bookmarks?tag=roc — filter by tag
DELETE /api/bookmarks/:id     — delete
```

<details><summary>💡 Gợi ý</summary>

- Model: `{ bookmarks: List { id, url, title, tags: List Str }, nextId }`
- Filter by tag: `List.keepIf bookmarks \b -> List.contains b.tags tag`
- Parse query params: `Str.splitFirst url "?tag="` 

</details>

---

**Bài 2** (20 phút): Note API với search

```
GET  /api/notes              — list all
POST /api/notes              — add {title, body}
GET  /api/notes/search?q=... — full-text search
GET  /api/notes/:id          — get one
PUT  /api/notes/:id          — update {title, body}
```

<details><summary>💡 Gợi ý</summary>

- Search: check if query appears in title OR body (case-insensitive)
- Update: find by id → replace record

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Port đã dùng | Server cũ chưa tắt | Kill process trên port 8000 |
| CORS error từ browser | Thiếu CORS headers | Thêm `Access-Control-Allow-Origin: *` |
| Request body rỗng | Client không gửi Content-Type | Kiểm tra curl có `-H "Content-Type: application/json"` |
| Model không thay đổi | Platform quản lý state | Kiểm tra respond function trả model mới |

---

## Tóm tắt

- ✅ **basic-webserver** = platform cho web services. `{ init, respond }` pattern.
- ✅ **Routing** = pattern match `(req.method, req.url)` — rõ ràng, exhaustive, type-safe.
- ✅ **JSON** = pure helpers: `jsonObject`, `jsonStr`, `jsonNum`, `jsonList`.
- ✅ **REST CRUD** = GET (list/get), POST (create), PUT (update), DELETE (remove).
- ✅ **Error handling** = consistent JSON errors: `{"error":"message"}`.
- ✅ **Architecture** = Router → Handler → Domain (pure) → Response.

## Tiếp theo

→ **Part VI: Capstone** — Chapter 26: **Capstone Part 1: Domain Model** ⭐ — xây hệ thống hoàn chỉnh: DDD model, pipelines, event sourcing, tests. Tổng hợp tất cả kiến thức!
