# Chapter 33 — Capstone Part 2: Production Deployment ⭐

> **Đây là chương cuối cùng** — từ code đến production.
>
> **Bạn sẽ xây dựng**: Production-ready Task Management API
> - Roc webserver + SQLite persistence
> - JWT authentication + RBAC
> - Structured logging + health checks
> - Docker containerization
> - CI/CD pipeline
>
> **Thời gian**: ~60 phút | **Level**: Principal
> **Kết quả cuối cùng**: Hiểu full production deployment lifecycle cho Roc app.

---

Capstone cuối cùng — triển khai hệ thống café lên production. Docker, CI/CD, monitoring, logging — không phải lý thuyết nữa, mà là deploy thật. Đây là chapter kết thúc hành trình từ "Hello World" đến production-ready application.

## Capstone Production — Deploy và vận hành

Full deployment pipeline: containerization, CI/CD, monitoring, logging. Roc application từ development → staging → production. Đây là chapter cuối cùng — bạn hoàn thành hành trình từ "Hello World" đến production deployment.


## 33.1 — Project Structure

Kiến trúc tiêu chuẩn cho Roc app: Domain layer (pure) ở trung tâm, Auth và Storage (pure SQL builders) bao quanh, và main.roc làm shell kết nối tất cả với platform.

```
task-api/
├── main.roc              # Webserver entry point
├── Domain.roc             # Pure domain types & logic
├── Auth.roc               # JWT & RBAC (pure)
├── Storage.roc            # SQL builders (pure)
├── Logger.roc             # Structured logging (pure)
├── Dockerfile
├── .github/
│   └── workflows/
│       └── ci.yml
└── README.md
```

---

## 33.2 — Domain Layer (Pure)

Domain layer chứa toàn bộ business logic — không có IO, không có dấu `!`. Mọi thứ ở đây đều testable bằng `expect` mà không cần mock.

```roc
# filename: Domain.roc
# module [Task, TaskStatus, createTask, updateStatus, ...]

# ═══════ TYPES ═══════

TaskStatus : [Todo, InProgress, Done, Archived]

Task : {
    id : U64,
    title : Str,
    description : Str,
    status : TaskStatus,
    assignee : Str,
    priority : [Low, Medium, High, Critical],
    createdAt : Str,
    updatedAt : Str,
}

# ═══════ BUSINESS LOGIC ═══════

createTask = \id, title, description, assignee, priority, now ->
    { id, title, description, status: Todo, assignee, priority, createdAt: now, updatedAt: now }

updateStatus = \task, newStatus, now ->
    when (task.status, newStatus) is
        (Todo, InProgress) -> Ok { task & status: InProgress, updatedAt: now }
        (Todo, Archived) -> Ok { task & status: Archived, updatedAt: now }
        (InProgress, Done) -> Ok { task & status: Done, updatedAt: now }
        (InProgress, Todo) -> Ok { task & status: Todo, updatedAt: now }
        (Done, Archived) -> Ok { task & status: Archived, updatedAt: now }
        _ -> Err (InvalidTransition (statusToStr task.status) (statusToStr newStatus))

statusToStr = \status ->
    when status is
        Todo -> "todo"
        InProgress -> "in_progress"
        Done -> "done"
        Archived -> "archived"

priorityToStr = \priority ->
    when priority is
        Low -> "low"
        Medium -> "medium"
        High -> "high"
        Critical -> "critical"

# Filter & sort
filterByStatus = \tasks, status ->
    List.keepIf tasks \t -> t.status == status

filterByAssignee = \tasks, assignee ->
    List.keepIf tasks \t -> t.assignee == assignee

sortByPriority = \tasks ->
    List.sortWith tasks \a, b ->
        priorityRank = \p -> when p is
            Critical -> 0
            High -> 1
            Medium -> 2
            Low -> 3
        Num.compare (priorityRank a.priority) (priorityRank b.priority)

# Dashboard stats
dashboardStats = \tasks ->
    {
        total: List.len tasks,
        todo: List.len (filterByStatus tasks Todo),
        inProgress: List.len (filterByStatus tasks InProgress),
        done: List.len (filterByStatus tasks Done),
        archived: List.len (filterByStatus tasks Archived),
        critical: List.len (List.keepIf tasks \t -> t.priority == Critical && t.status != Done),
    }

# ═══════ TESTS ═══════

expect
    task = createTask 1 "Setup CI" "Configure pipeline" "An" High "2024-03-15"
    task.status == Todo && task.priority == High

expect
    task = createTask 1 "Test" "" "An" Low "now"
    when updateStatus task InProgress "now" is
        Ok t -> t.status == InProgress
        Err _ -> Bool.false

expect
    task = createTask 1 "Test" "" "An" Low "now"
    updateStatus task Done "now" |> Result.isErr    # Todo → Done invalid

expect
    tasks = [
        createTask 1 "A" "" "An" Low "now",
        createTask 2 "B" "" "An" Critical "now",
        createTask 3 "C" "" "An" Medium "now",
    ]
    sorted = sortByPriority tasks
    (List.first sorted |> Result.map .priority) == Ok Critical

expect
    tasks = [
        createTask 1 "A" "" "An" Low "now",
        { (createTask 2 "B" "" "An" High "now") & status: Done },
        { (createTask 3 "C" "" "An" Medium "now") & status: InProgress },
    ]
    stats = dashboardStats tasks
    stats.total == 3 && stats.todo == 1 && stats.done == 1
```

---

## 33.3 — Authentication Layer (Pure)

JWT authentication viết dưới dạng pure functions — tạo token, verify token, kiểm tra permissions. Không gọi database, không đọc environment variables. Tất cả dependencies được truyền vào qua tham số.

```roc
# filename: Auth.roc

# ═══════ JWT Claims ═══════

Role : [Admin, Manager, Developer, Viewer]

Claims : {
    userId : Str,
    role : Role,
    expiresAt : U64,
}

# Permission matrix
canPerform = \role, action ->
    when role is
        Admin -> Bool.true
        Manager ->
            when action is
                CreateTask -> Bool.true
                UpdateTask -> Bool.true
                DeleteTask -> Bool.true
                ViewTask -> Bool.true
                ManageUsers -> Bool.false
        Developer ->
            when action is
                CreateTask -> Bool.true
                UpdateTask -> Bool.true
                DeleteTask -> Bool.false
                ViewTask -> Bool.true
                ManageUsers -> Bool.false
        Viewer ->
            when action is
                ViewTask -> Bool.true
                _ -> Bool.false

# Validate token claims
validateAuth = \claims, currentTime, requiredAction ->
    if currentTime > claims.expiresAt then Err TokenExpired
    else if !(canPerform claims.role requiredAction) then Err Forbidden
    else Ok claims

# Tests
expect canPerform Admin DeleteTask == Bool.true
expect canPerform Developer DeleteTask == Bool.false
expect canPerform Viewer CreateTask == Bool.false

expect
    claims = { userId: "u1", role: Developer, expiresAt: 9999 }
    validateAuth claims 100 ViewTask |> Result.isOk

expect
    claims = { userId: "u1", role: Developer, expiresAt: 50 }
    validateAuth claims 100 ViewTask == Err TokenExpired
```

---

## 33.4 — Storage Layer (Pure SQL Builders)

Storage layer không chạy SQL — nó chỉ **tạo câu SQL** dưới dạng strings. Việc thực thi SQL là trách nhiệm của shell (main.roc). Pattern này giữ cho storage layer hoàn toàn pure và testable.

```roc
# filename: Storage.roc

# ═══════ MIGRATIONS ═══════

migrations = [
    {
        version: 1,
        sql:
            """
            CREATE TABLE tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                description TEXT DEFAULT '',
                status TEXT NOT NULL DEFAULT 'todo',
                assignee TEXT NOT NULL,
                priority TEXT NOT NULL DEFAULT 'medium',
                created_at TEXT NOT NULL,
                updated_at TEXT NOT NULL
            );
            CREATE INDEX idx_tasks_status ON tasks(status);
            CREATE INDEX idx_tasks_assignee ON tasks(assignee);
            """,
    },
    {
        version: 2,
        sql:
            """
            CREATE TABLE users (
                id TEXT PRIMARY KEY,
                name TEXT NOT NULL,
                email TEXT UNIQUE NOT NULL,
                role TEXT NOT NULL DEFAULT 'viewer',
                password_hash TEXT NOT NULL
            );
            """,
    },
    {
        version: 3,
        sql:
            """
            CREATE TABLE audit_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id TEXT NOT NULL,
                action TEXT NOT NULL,
                resource TEXT NOT NULL,
                resource_id TEXT,
                timestamp TEXT NOT NULL
            );
            CREATE INDEX idx_audit_user ON audit_log(user_id);
            """,
    },
]

# ═══════ QUERY BUILDERS ═══════

insertTask = \title, description, assignee, priority, now ->
    {
        sql: "INSERT INTO tasks (title, description, assignee, priority, created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?)",
        params: [title, description, assignee, priority, now, now],
    }

selectTasks = \filters ->
    base = "SELECT * FROM tasks"
    conditions = List.keepIf filters \f -> !(Str.isEmpty f.value)
    if List.isEmpty conditions then
        { sql: "$(base) ORDER BY id DESC", params: [] }
    else
        clauses = List.map conditions \f -> "$(f.column) = ?"
        params = List.map conditions \f -> f.value
        where = Str.joinWith clauses " AND "
        { sql: "$(base) WHERE $(where) ORDER BY id DESC", params }

updateTaskStatus = \id, status, now ->
    { sql: "UPDATE tasks SET status = ?, updated_at = ? WHERE id = ?", params: [status, now, Num.toStr id] }

insertAuditLog = \userId, action, resource, resourceId, now ->
    { sql: "INSERT INTO audit_log (user_id, action, resource, resource_id, timestamp) VALUES (?, ?, ?, ?, ?)",
      params: [userId, action, resource, resourceId, now] }

# Tests
expect
    q = insertTask "Setup CI" "Pipeline" "An" "high" "2024-03-15"
    Str.contains q.sql "INSERT INTO tasks" && List.len q.params == 6

expect
    q = selectTasks [{ column: "status", value: "todo" }, { column: "assignee", value: "An" }]
    Str.contains q.sql "WHERE" && List.len q.params == 2

expect
    q = selectTasks []
    !(Str.contains q.sql "WHERE")
```

---

## 33.5 — Structured Logger (Pure)

Structured logging tạo JSON log entries thay vì plain text. Mỗi log entry là một record với timestamp, level, message, và context — dễ parse, dễ search, dễ aggregate trên production.

```roc
# filename: Logger.roc

logRequest = \method, path, statusCode, durationMs, userId ->
    fields = [
        ("method", method),
        ("path", path),
        ("status", Num.toStr statusCode),
        ("duration_ms", Num.toStr durationMs),
        ("user_id", userId),
    ]
    formatJsonLog "INFO" "request" fields

logError = \message, context ->
    formatJsonLog "ERROR" message context

logAudit = \userId, action, resource, resourceId ->
    formatJsonLog "AUDIT" "$(action) $(resource)" [
        ("user_id", userId),
        ("action", action),
        ("resource", resource),
        ("resource_id", resourceId),
    ]

formatJsonLog = \level, message, fields ->
    pairs = [("level", level), ("msg", message)]
        |> List.concat fields
        |> List.map \(k, v) -> "\"$(k)\":\"$(v)\""
    "{$(Str.joinWith pairs ",")}"

# Tests
expect
    log = logRequest "GET" "/api/tasks" 200 12 "user-1"
    Str.contains log "\"status\":\"200\"" && Str.contains log "\"duration_ms\":\"12\""

expect
    log = logAudit "user-1" "create" "task" "42"
    Str.contains log "\"level\":\"AUDIT\""
```

---

## 33.6 — API Server (Shell)

Đây là **shell** — nơi duy nhất có side effects. main.roc nhận HTTP requests, gọi Domain/Auth/Storage (pure) để xử lý logic, rồi thực thi SQL và trả response. Pattern pure core / effectful shell ở quy mô production.

```roc
# filename: main.roc
app [Model, server] {
    webserver: platform "https://github.com/roc-lang/basic-webserver/releases/download/0.10.0/lWg35nBCAzFBqEhIM0eBI_6hWM3e7h4iGjM6bBCwals.tar.br"
}

import webserver.Webserver exposing [Request, Response]

# ═══════ Modules (inline for demo) ═══════

# Import pure modules: Domain, Auth, Storage, Logger
# All pure logic lives there — tested via `expect`

# ═══════ SERVER ═══════

Model : {
    tasks : List Task,
    nextId : U64,
}

Task : { id : U64, title : Str, description : Str, status : Str, assignee : Str, priority : Str, createdAt : Str, updatedAt : Str }

server = {
    init: Task.ok { tasks: [], nextId: 1 },
    respond,
}

respond : Request, Model -> Task Response [ServerErr Str]_
respond = \req, model ->
    when (req.method, req.url) is
        # Health check
        (Get, "/health") ->
            json 200 "{\"status\":\"ok\",\"tasks\":$(Num.toStr (List.len model.tasks))}"

        # List tasks
        (Get, "/api/v1/tasks") ->
            tasksJson = List.map model.tasks taskToJson |> \items -> "[$(Str.joinWith items ",")]"
            json 200 "{\"data\":$(tasksJson),\"total\":$(Num.toStr (List.len model.tasks))}"

        # Dashboard
        (Get, "/api/v1/dashboard") ->
            total = List.len model.tasks
            todo = List.len (List.keepIf model.tasks \t -> t.status == "todo")
            done = List.len (List.keepIf model.tasks \t -> t.status == "done")
            json 200 "{\"total\":$(Num.toStr total),\"todo\":$(Num.toStr todo),\"done\":$(Num.toStr done)}"

        # Create task
        (Post, "/api/v1/tasks") ->
            bodyStr = req.body |> Str.fromUtf8 |> Result.withDefault ""
            title = parseField bodyStr "title" |> Result.withDefault "Untitled"
            assignee = parseField bodyStr "assignee" |> Result.withDefault "unassigned"
            priority = parseField bodyStr "priority" |> Result.withDefault "medium"
            task = { id: model.nextId, title, description: "", status: "todo", assignee, priority, createdAt: "now", updatedAt: "now" }
            json 201 (taskToJson task)

        # 404
        _ ->
            json 404 "{\"error\":\"Not found: $(req.url)\"}"

taskToJson = \task ->
    "{\"id\":$(Num.toStr task.id),\"title\":\"$(task.title)\",\"status\":\"$(task.status)\",\"assignee\":\"$(task.assignee)\",\"priority\":\"$(task.priority)\"}"

parseField = \body, field ->
    when Str.splitFirst body "\"$(field)\":\"" is
        Ok { after, .. } ->
            when Str.splitFirst after "\"" is
                Ok { before, .. } -> Ok before
                Err _ -> Err FieldNotFound
        Err _ -> Err FieldNotFound

json = \status, body -> Task.ok {
    status,
    headers: [
        { name: "Content-Type", value: Str.toUtf8 "application/json" },
        { name: "X-Request-Id", value: Str.toUtf8 "req-001" },
    ],
    body: Str.toUtf8 body,
}
```

---

## 33.7 — Docker

Docker đóng gói app thành container — chạy được trên mọi server mà không lo dependencies. Dockerfile cho Roc app thường đơn giản vì Roc compile ra binary tĩnh.

```dockerfile
# Dockerfile
# Multi-stage build — minimal final image

# Stage 1: Build
FROM rocklang/roc:latest AS builder
WORKDIR /app
COPY . .
RUN roc build --optimize main.roc

# Stage 2: Runtime (minimal)
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/main /app/main
WORKDIR /app
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["./main"]
```

```bash
# Build & Run
docker build -t task-api .
docker run -p 8000:8000 task-api

# Test
curl http://localhost:8000/health
# {"status":"ok","tasks":0}
```

### Docker Compose (sản phẩm hoàn chỉnh)

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DB_PATH=/data/tasks.db
      - LOG_LEVEL=info
    volumes:
      - task-data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

volumes:
  task-data:
```

---

## 33.8 — CI/CD Pipeline

GitHub Actions tự động: mỗi push → chạy tests → build Docker image → deploy. Không ai deploy tay trên production — mọi thay đổi đều qua pipeline.

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Roc
        run: |
          curl -fsSL https://github.com/roc-lang/roc/releases/latest/download/roc-linux-x86_64.tar.gz \
            | tar xz
          sudo mv roc /usr/local/bin/

      - name: Run tests
        run: roc test main.roc

      - name: Build
        run: roc build --optimize main.roc

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: task-api
          path: main

  docker:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build & Push Docker image
        run: |
          docker build -t task-api:${{ github.sha }} .
          docker tag task-api:${{ github.sha }} task-api:latest
          # docker push registry/task-api:latest
```

---

## 33.9 — Production Checklist

Trước khi deploy lên production, kiểm tra danh sách này. Mỗi item là một bài học từ những sự cố production thật — không phải lý thuyết.

```
PRE-DEPLOY
  ✅ roc test — all tests pass
  ✅ roc build --optimize — production binary
  ✅ Docker build succeeds
  ✅ Health check responds OK
  ✅ Security headers present (Content-Type, X-Request-Id)
  ✅ CORS configured

MONITORING
  ✅ Structured JSON logs (grep-able, parsable)
  ✅ Health endpoint with service stats
  ✅ Audit log for mutations
  ✅ Request logging (method, path, status, duration)

SECURITY
  ✅ JWT auth on protected routes
  ✅ RBAC permission checks
  ✅ Input validation — parse, don't validate
  ✅ Parameterized SQL queries
  ✅ No secrets in code or logs

RELIABILITY
  ✅ Graceful error responses (JSON, correct status codes)
  ✅ Circuit breaker for external services
  ✅ Rate limiting for public endpoints
  ✅ Database migrations versioned
```

---

## 🎉🎉🎉 CUỐN SÁCH HOÀN THÀNH! 🎉🎉🎉

### Toàn bộ hành trình — 34 chapters

```
Part 0: CS Foundations (Ch 0-3)
├─ Algorithms, Data Structures, FP Thinking, Functional Data Structures

Part I: Roc Fundamentals (Ch 4-10)
├─ Syntax, Types, Control Flow, Records, Pattern Matching
├─ Lists, Modules, Opaque Types

Part II: Thinking Functionally (Ch 11-14)
├─ Purity, Tasks & Effects, Result & Error Handling, Abilities

Part III: Design Patterns (Ch 15-16)
├─ FP Patterns (Strategy, Command, Builder, Interpreter)
├─ Event Sourcing

Part IV: Domain-Driven Design (Ch 17-21)
├─ DDD Introduction, Domain Modeling, Workflows as Pipelines
├─ Platform Separation, Serialization

Part V: Testing & Apps (Ch 22-26)
├─ Testing (expect, TDD), PBT (properties, round-trip)
├─ CLI Applications, Web Services, Capstone 1 (Café System)

Part VI: Production Engineering (Ch 27-33)
├─ Database (SQL, normalization, ACID)
├─ Advanced Data (CQRS, event store, caching, repository)
├─ Security (passwords, JWT, OAuth, OWASP, XSS, CSRF)
├─ Distributed Systems (CAP, Saga, Circuit Breaker)
├─ System Design (capacity, API, architecture, URL shortener, rate limiter)
└─ Capstone 2 (Production deployment) ← BẠN Ở ĐÂY! ✅
```

### Kiến thức bạn đã master

| Skill | Chapters |
|-------|----------|
| Roc syntax & type system | 0-10 |
| Functional programming | 11-16 |
| Domain-Driven Design | 17-21 |
| Testing & quality | 22-23 |
| Application development | 24-26 |
| Production engineering | 27-33 |

### Bước tiếp theo

1. 🏗️ **Xây project riêng** — pick a domain, model it, build it, deploy it
2. 📖 **Đọc Roc official docs** — [roc-lang.org](https://www.roc-lang.org)
3. 💬 **Tham gia cộng đồng** — Roc Zulip chat
4. 🔀 **So sánh** — cùng project bằng Rust, OCaml, Haskell, Zig
5. 🎯 **Contribute** — contribute platforms, packages, hoặc documentation

> **"The purpose of abstraction is not to be vague, but to create a new semantic level in which one can be absolutely precise."** — Edsger Dijkstra

🚀 **Happy coding with Roc! Chúc bạn thành công trên hành trình functional programming!**
