# Chapter 30 — Application Security

> **Bạn sẽ học được**:
> - **OWASP Top 10** — 10 lỗ hổng phổ biến nhất
> - **SQL Injection** — tấn công và phòng chống
> - **XSS** (Cross-Site Scripting) — input escaping
> - **CSRF** (Cross-Site Request Forgery) — token protection
> - **Input validation** qua type system — "parse, don't validate"
> - Roc's security model vs traditional languages
>
> **Yêu cầu trước**: [Chapter 29 — Security Essentials](chapter_29_security_essentials.md)
> **Thời gian đọc**: ~40 phút | **Level**: Principal
> **Kết quả cuối cùng**: Nhận diện và phòng tránh lỗ hổng bảo mật phổ biến, áp dụng type-driven security.

---

OWASP Top 10 liệt kê 10 lỗ hổng phổ biến nhất trong web application. Injection, XSS, CSRF, broken authentication — mỗi lỗ hổng có cách phòng chống cụ thể. FP giúp giảm nhiều lỗ hổng nhờ immutability và explicit data flow.

## Application Security — OWASP Top 10

Bảo vệ application khỏi injection, XSS, CSRF. Roc's purity giúp ở mức function (no hidden side-effects), nhưng application-level validation vẫn cần: input sanitization, output encoding, security headers.


## 30.1 — OWASP Top 10 Overview

OWASP Top 10 cập nhật mỗi 3-4 năm. 2021 edition: #1 Broken Access Control, #2 Cryptographic Failures, #3 Injection. Biết top 10 = bảo vệ được 80% attack surface.

| # | Lỗ hổng | Roc mitigation |
|---|---------|----------------|
| 1 | **Injection** (SQL, Command) | Parameterized queries, opaque types |
| 2 | **Broken Authentication** | Type-safe JWT, opaque PasswordHash |
| 3 | **Sensitive Data Exposure** | Opaque types hide internals |
| 4 | **XML External Entities** | Roc không dùng XML |
| 5 | **Broken Access Control** | Tag union RBAC, compiler-checked |
| 6 | **Security Misconfiguration** | Platform handles config |
| 7 | **XSS** | Auto-escape outputs |
| 8 | **Insecure Deserialization** | `Decode` → `Result` (always validated) |
| 9 | **Known Vulnerabilities** | Minimal dependencies (platform-based) |
| 10 | **Insufficient Logging** | `Inspect` ability for structured logging |

---

## 30.2 — SQL Injection

Input validation — mọi data từ bên ngoài là không đáng tin. Parse, don't validate: chuyển raw input thành domain types (opaque types + smart constructors). Data qua parser = an toàn.

### Tấn công

```sql
-- User input: name = "'; DROP TABLE users; --"

-- ❌ String concatenation → DISASTER
-- "SELECT * FROM users WHERE name = '" + input + "'"
-- → SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
-- → XÓA TABLE!
```

### Phòng chống: Parameterized queries

```roc
# ═══════ ❌ DANGEROUS: String interpolation ═══════

dangerousQuery = \userInput ->
    "SELECT * FROM users WHERE name = '$(userInput)'"
    # Nếu userInput = "'; DROP TABLE users;--"
    # → SQL injection!

# ═══════ ✅ SAFE: Parameterized queries ═══════

# Tách SQL template và giá trị
SafeQuery : { sql : Str, params : List Str }

selectByName : Str -> SafeQuery
selectByName = \name ->
    { sql: "SELECT * FROM users WHERE name = ?", params: [name] }

selectByAge : U64, U64 -> SafeQuery
selectByAge = \minAge, maxAge ->
    { sql: "SELECT * FROM users WHERE age >= ? AND age <= ?", params: [Num.toStr minAge, Num.toStr maxAge] }

insertUser : Str, Str -> SafeQuery
insertUser = \name, email ->
    { sql: "INSERT INTO users (name, email) VALUES (?, ?)", params: [name, email] }

# Tests
expect
    q = selectByName "An"
    q.sql == "SELECT * FROM users WHERE name = ?"
    && q.params == ["An"]

expect
    q = selectByName "'; DROP TABLE users;--"
    # SQL vẫn an toàn — input là PARAMETER, không phải SQL code
    q.sql == "SELECT * FROM users WHERE name = ?"
    && q.params == ["'; DROP TABLE users;--"]

# Platform executes: bind params safely → no injection possible
```

### Type-safe query builder

```roc
# Dùng opaque types để enforce safety
SafeSQL := { sql : Str, params : List Str }

# Chỉ có thể tạo qua builder functions — KHÔNG string concat
select : Str, List Str -> SafeSQL
select = \table, columns ->
    cols = if List.isEmpty columns then "*" else Str.joinWith columns ", "
    @SafeSQL { sql: "SELECT $(cols) FROM $(table)", params: [] }

where : SafeSQL, Str, Str -> SafeSQL
where = \@SafeSQL query, column, value ->
    newSql = if Str.contains query.sql "WHERE" then
        "$(query.sql) AND $(column) = ?"
    else
        "$(query.sql) WHERE $(column) = ?"
    @SafeSQL { sql: newSql, params: List.append query.params value }

orderBy : SafeSQL, Str, [Asc, Desc] -> SafeSQL
orderBy = \@SafeSQL query, column, direction ->
    dir = when direction is
        Asc -> "ASC"
        Desc -> "DESC"
    @SafeSQL { query & sql: "$(query.sql) ORDER BY $(column) $(dir)" }

toSQL : SafeSQL -> { sql : Str, params : List Str }
toSQL = \@SafeSQL query -> query

# Usage — type-safe, no injection
expect
    q = select "orders" ["id", "total"]
        |> where "status" "completed"
        |> where "customer_id" "42"
        |> orderBy "total" Desc
        |> toSQL
    q.sql == "SELECT id, total FROM orders WHERE status = ? AND customer_id = ? ORDER BY total DESC"
    && q.params == ["completed", "42"]
```

---

## 30.3 — XSS (Cross-Site Scripting)

Access control: authentication (ai?) + authorization (được làm gì?). RBAC (role-based), ABAC (attribute-based). Enforce ở mọi endpoint, không chỉ UI.

### Tấn công

```html
<!-- User input: name = "<script>alert('hacked')</script>" -->

<!-- ❌ Raw output → XSS -->
<p>Hello, <script>alert('hacked')</script>!</p>
<!-- Browser executes JavaScript! Steal cookies, redirect, etc. -->
```

### Phòng chống: HTML escaping

```roc
# ═══════ PURE: HTML escape ═══════

escapeHtml : Str -> Str
escapeHtml = \raw ->
    raw
    |> Str.replaceEach "&" "&amp;"
    |> Str.replaceEach "<" "&lt;"
    |> Str.replaceEach ">" "&gt;"
    |> Str.replaceEach "\"" "&quot;"
    |> Str.replaceEach "'" "&#39;"

# Tests
expect escapeHtml "<script>alert('xss')</script>" == "&lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;"
expect escapeHtml "Hello, An" == "Hello, An"    # normal text unchanged
expect escapeHtml "a < b & c > d" == "a &lt; b &amp; c &gt; d"

# ═══════ Safe HTML builder ═══════

HtmlSafe := Str

# Chỉ tạo qua escape — KHÔNG raw string
fromUserInput : Str -> HtmlSafe
fromUserInput = \raw -> @HtmlSafe (escapeHtml raw)

# Trusted content (developer-controlled)
trusted : Str -> HtmlSafe
trusted = \html -> @HtmlSafe html

# Combine
concat : HtmlSafe, HtmlSafe -> HtmlSafe
concat = \@HtmlSafe a, @HtmlSafe b -> @HtmlSafe "$(a)$(b)"

# Render
render : HtmlSafe -> Str
render = \@HtmlSafe html -> html

# Usage
expect
    greeting = trusted "<h1>Hello, "
        |> concat (fromUserInput "<script>evil</script>")
        |> concat (trusted "!</h1>")
        |> render
    greeting == "<h1>Hello, &lt;script&gt;evil&lt;/script&gt;!</h1>"
```

---

## 30.4 — CSRF (Cross-Site Request Forgery)

Rate limiting ngăn abuse — giới hạn số requests/phút. Token bucket algorithm đơn giản: mỗi user có bucket N tokens, mỗi request dùng 1, bucket refill theo thời gian.

### Tấn công

```
1. User đăng nhập bank.com → có session cookie
2. User mở evil.com
3. evil.com tạo hidden form: POST bank.com/transfer (amount=1000000)
4. Browser tự gửi cookie → bank xử lý như request hợp lệ!
```

### Phòng chống: CSRF Token

```roc
# ═══════ PURE: CSRF token logic ═══════

CSRFToken := Str implements [Eq]

# Generate random token (shell provides randomness)
# validateCSRF: compare token in form vs token in session
validateCSRF : CSRFToken, CSRFToken -> Result {} [CSRFMismatch]
validateCSRF = \formToken, sessionToken ->
    if formToken == sessionToken then Ok {} else Err CSRFMismatch

# Embed in form (HTML)
csrfField : CSRFToken -> Str
csrfField = \@CSRFToken token ->
    "<input type=\"hidden\" name=\"csrf_token\" value=\"$(token)\">"

# Tests
expect
    token = @CSRFToken "abc123"
    validateCSRF token token == Ok {}

expect
    formToken = @CSRFToken "abc123"
    sessionToken = @CSRFToken "xyz789"
    validateCSRF formToken sessionToken == Err CSRFMismatch

# Middleware
protectCSRF = \req, sessionToken ->
    formToken = extractFormField req.body "csrf_token"
    when formToken is
        Ok token ->
            validateCSRF (@CSRFToken token) sessionToken
        Err _ ->
            Err CSRFMismatch

extractFormField = \body, fieldName ->
    bodyStr = Str.fromUtf8 body |> Result.withDefault ""
    pattern = "$(fieldName)="
    when Str.splitFirst bodyStr pattern is
        Ok { after, .. } ->
            when Str.splitFirst after "&" is
                Ok { before, .. } -> Ok before
                Err _ -> Ok after
        Err _ -> Err FieldNotFound
```

---

## 30.5 — Input Validation: Type-Driven Security

Logging cho security: ghi lại mọi authentication attempt, authorization failure, và data access. Structured logs (JSON) dễ search và alert tự động.

### "Parse, don't validate" = Security by construction

```roc
# ═══════ DANGEROUS: String everywhere ═══════

# processOrder : Str, Str, Str -> ...
# processOrder "user-1" "100" "pending"
# → Ai cũng gọi được với bất kỳ string → bugs, injection

# ═══════ SAFE: Types enforce correctness ═══════

# processOrder : UserId, Money, OrderStatus -> ...
# → Compiler FORCES correct types
# → Giá trị đã validate khi construct

# Opaque types = smart constructors = validation at boundary
UserId := U64 implements [Eq, Hash]

userId : U64 -> Result UserId [InvalidUserId]
userId = \n -> if n > 0 then Ok (@UserId n) else Err InvalidUserId

Email := Str implements [Eq, Hash]

email : Str -> Result Email [InvalidEmail]
email = \raw ->
    trimmed = Str.trim raw
    if Str.contains trimmed "@" && Str.contains trimmed "." && Str.countUtf8Bytes trimmed >= 5
    then Ok (@Email (Str.toUppercase trimmed |> \_ -> trimmed))  # store original
    else Err InvalidEmail

# Boundary: parse duy nhất 1 lần
parseRequest = \rawBody ->
    userIdStr = extractFormField? rawBody "user_id"
    emailStr = extractFormField? rawBody "email"

    uid = Str.toU64 userIdStr |> Result.mapErr? \_ -> InvalidUserId
    id = userId? uid
    mail = email? emailStr

    Ok { userId: id, email: mail }
    # Từ đây: code CHỈ dùng UserId và Email — type-safe!

extractFormField = \body, field ->
    bodyStr = Str.fromUtf8 body |> Result.withDefault ""
    when Str.splitFirst bodyStr "$(field)=" is
        Ok { after, .. } ->
            when Str.splitFirst after "&" is
                Ok { before, .. } -> Ok before
                Err _ -> Ok after
        Err _ -> Err (MissingField field)

# Tests
expect userId 42 |> Result.isOk
expect userId 0 == Err InvalidUserId
expect email "an@mail.com" |> Result.isOk
expect email "invalid" == Err InvalidEmail
```

---

## 30.6 — Security Checklist

Security headers: Content-Security-Policy, X-Frame-Options, Strict-Transport-Security. Mỗi header ngăn một loại attack cụ thể. Set once trên web server — chi phí thấp, lợi ích cao.

| Category | Checks |
|----------|--------|
| **Input** | ✅ Validate tại boundary, ✅ Opaque types cho domain values, ✅ Parameterized SQL |
| **Output** | ✅ Escape HTML, ✅ JSON Content-Type, ✅ Security headers |
| **Auth** | ✅ Hash passwords, ✅ JWT expiry, ✅ RBAC via tag unions |
| **Session** | ✅ CSRF tokens, ✅ HttpOnly cookies, ✅ HTTPS |
| **Data** | ✅ No sensitive data in logs, ✅ Opaque types hide internals |
| **Roc** | ✅ Platform isolation, ✅ Pure functions = no side-channel, ✅ Immutable = no race conditions |

---


## ✅ Checkpoint 30

> Đến đây bạn phải hiểu:
> 1. OWASP Top 10: injection, XSS, CSRF — biết để phòng
> 2. Input validation: whitelist > blacklist, type-driven (parse via `Decode`)
> 3. Roc: purity + opaque types + platform isolation = triple defense
>
> **Test nhanh**: SQL injection xảy ra khi nào?
> <details><summary>Đáp án</summary>Khi user input ghép trực tiếp vào SQL string. Phòng: parameterized queries (`?` placeholder).</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Input sanitizer

Viết hàm sanitize cho các loại input:

```roc
# sanitizeFilename: loại bỏ path traversal (../, ..\)
# sanitizeSearch: loại bỏ SQL wildcard characters (%, _)
# sanitizeUrl: chỉ cho phép http:// và https://
```

<details><summary>✅ Lời giải</summary>

```roc
sanitizeFilename = \raw ->
    raw
    |> Str.replaceEach "../" ""
    |> Str.replaceEach "..\\" ""
    |> Str.replaceEach "/" ""
    |> Str.replaceEach "\\" ""

sanitizeSearch = \raw ->
    raw
    |> Str.replaceEach "%" ""
    |> Str.replaceEach "_" ""
    |> Str.replaceEach "'" ""
    |> Str.replaceEach "\"" ""

sanitizeUrl = \raw ->
    if Str.startsWith raw "https://" then Ok raw
    else if Str.startsWith raw "http://" then Ok raw
    else Err UnsafeUrl

expect sanitizeFilename "../etc/passwd" == "etcpasswd"
expect sanitizeSearch "an%admin" == "anadmin"
expect sanitizeUrl "https://safe.com" == Ok "https://safe.com"
expect sanitizeUrl "javascript:alert(1)" == Err UnsafeUrl
```

</details>

---

**Bài 2** (15 phút): Secure API endpoint

Viết endpoint có đầy đủ security:

```roc
# POST /api/transfer
# 1. CSRF check
# 2. JWT auth + permission check
# 3. Input validation (amount > 0, toAccount exists)
# 4. Parameterized SQL
# 5. Audit log
```

<details><summary>✅ Lời giải</summary>

```roc
handleTransfer = \req, session, currentTime ->
    # 1. CSRF
    _ = protectCSRF? req session.csrfToken
    # 2. Auth
    claims = authenticateRequest? req.headers currentTime
    _ = if hasPermission claims WriteOrders then Ok {} else Err? Forbidden
    # 3. Validate
    transfer = parseTransferInput? req.body
    # 4. SQL (parameterized)
    sql = {
        sql: "UPDATE accounts SET balance = balance - ? WHERE id = ?; UPDATE accounts SET balance = balance + ? WHERE id = ?",
        params: [Num.toStr transfer.amount, Num.toStr transfer.fromId, Num.toStr transfer.amount, Num.toStr transfer.toId],
    }
    # 5. Audit
    auditLog = { action: "transfer", userId: claims.subject, amount: transfer.amount, timestamp: currentTime }
    Ok { sql, auditLog }
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| SQL Injection vẫn xảy ra | Dùng string interpolation | LUÔN dùng `?` placeholders |
| XSS bypass | Escape không đủ characters | Dùng `escapeHtml` cover 5 characters: `& < > " '` |
| CSRF token mismatch | Token expired hoặc sai session | Regenerate token mỗi session, check đúng session |
| Validation bypass | Validate ở client, không server | Server-side validation ALWAYS (Roc: tại boundary) |

---

## Tóm tắt

- ✅ **OWASP Top 10** — 10 lỗ hổng phổ biến. Roc mitigates nhiều by design.
- ✅ **SQL Injection** — parameterized queries + opaque SafeSQL type.
- ✅ **XSS** — `escapeHtml` + opaque `HtmlSafe` type. User input luôn escaped.
- ✅ **CSRF** — token-based protection. Compare form vs session token.
- ✅ **Type-driven security** — "Parse, don't validate". Opaque types = validated at boundary.
- ✅ **Roc advantages** — platform isolation, immutability, purity, no global state.

## Tiếp theo

→ Chapter 31: **Distributed Systems Fundamentals** — CAP theorem, consistency models, Saga pattern, Circuit Breaker, structured logging.
