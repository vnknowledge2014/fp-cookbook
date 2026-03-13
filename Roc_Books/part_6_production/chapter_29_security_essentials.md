# Chapter 29 — Security Essentials

> **Bạn sẽ học được**:
> - **Password hashing** — tại sao không lưu plaintext, bcrypt/argon2
> - **JWT** (JSON Web Tokens) — encode/decode, claims, expiration
> - **OAuth 2.0** — authorization flow, tokens, scopes
> - **Roc security advantage** — purity = no state leaks, opaque types = encapsulation
> - Type-safe authentication patterns
>
> **Yêu cầu trước**: [Chapter 28 — Advanced Data Patterns](chapter_28_advanced_data_patterns.md)
> **Thời gian đọc**: ~40 phút | **Level**: Principal
> **Kết quả cuối cùng**: Hiểu security fundamentals, implement auth patterns type-safe trong Roc.

---

Năm 2012, LinkedIn bị hack và lộ 117 triệu passwords vì lưu bằng SHA1 không salt. Năm 2017, Equifax lộ dữ liệu 147 triệu người vì không patch Apache Struts. Security không phải tính năng phụ — nó là yêu cầu cơ bản.

## Security Essentials — Authentication & Authorization

Auth (ai đang dùng?), Authorization (họ được làm gì?), encryption, hashing — nền tảng security cho mọi application. Roc's type system giúp model permissions an toàn: Permission enum + exhaustive matching = compiler verify access control.


## 29.1 — Password Security

Không bao giờ lưu password dạng plain text. Hash bằng bcrypt/argon2 — thuật toán chậm có chủ đích để brute-force mất hàng nghìn năm. Salt mỗi password khác nhau để rainbow table vô dụng.

### ❌ KHÔNG BAO GIỜ lưu plaintext

```sql
-- ❌ TUYỆT ĐỐI KHÔNG
INSERT INTO users (email, password) VALUES ('an@mail.com', 'mypassword123');
-- Nếu DB bị hack → tất cả password lộ!

-- ✅ Lưu hash
INSERT INTO users (email, password_hash) VALUES ('an@mail.com', '$2b$12$...');
-- Nếu DB bị hack → attacker chỉ có hash, không thể reverse
```

### Hash functions

```
password ──→ hash function ──→ hash (fixed-length string)
"abc123"     SHA-256           "ba7816bf8f01cfea..."

Tính chất:
1. ONE-WAY: hash → password ← KHÔNG THỂ
2. DETERMINISTIC: cùng input → cùng hash
3. AVALANCHE: thay 1 ký tự → hash khác hoàn toàn
4. FIXED-LENGTH: mọi input → cùng độ dài output
```

### Password hashing trong Roc (concept)

```roc
# ═══════ PURE: Password validation logic ═══════

# Opaque type — không expose hash ra ngoài
PasswordHash := Str implements [Eq]

# Validate password strength (PURE — không cần crypto)
validatePasswordStrength : Str -> Result Str [TooShort, NoUppercase, NoDigit, NoSpecial]
validatePasswordStrength = \password ->
    bytes = Str.toUtf8 password
    len = List.len bytes
    if len < 8 then Err TooShort
    else
        hasUpper = List.any bytes \b -> b >= 65 && b <= 90
        hasDigit = List.any bytes \b -> b >= 48 && b <= 57
        hasSpecial = List.any bytes \b -> (b >= 33 && b <= 47) || (b >= 58 && b <= 64)
        if !hasUpper then Err NoUppercase
        else if !hasDigit then Err NoDigit
        else if !hasSpecial then Err NoSpecial
        else Ok password

# Tests
expect validatePasswordStrength "Secure1!" == Ok "Secure1!"
expect validatePasswordStrength "short" == Err TooShort
expect validatePasswordStrength "nouppercase1!" == Err NoUppercase
expect validatePasswordStrength "NoDigitHere!" == Err NoDigit
expect validatePasswordStrength "NoSpecial1" == Err NoSpecial

# ═══════ SHELL: Actual hashing (platform-provided) ═══════

# hashPassword : Str -> Task PasswordHash _
# hashPassword = \password ->
#     hashed = Crypto.bcrypt password 12    # platform provides
#     Task.ok (@PasswordHash hashed)

# verifyPassword : Str, PasswordHash -> Task Bool _
# verifyPassword = \password, @PasswordHash hash ->
#     Crypto.bcryptVerify password hash     # platform provides
```

---

## 29.2 — JWT (JSON Web Tokens)

Symmetric encryption: cùng key để encrypt và decrypt — AES-256. Asymmetric: public key encrypt, private key decrypt — RSA, Ed25519. HTTPS dùng cả hai: asymmetric cho handshake, symmetric cho data.

### Cấu trúc

```
JWT = Header.Payload.Signature

Header:  {"alg": "HS256", "typ": "JWT"}     → Base64
Payload: {"sub": "user-1", "exp": 1710547200} → Base64
Signature: HMAC-SHA256(header + payload, secret)

Kết quả: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyLTEifQ.signature
```

### JWT trong Roc

```roc
# ═══════ PURE: JWT claims & validation ═══════

# Claims = payload data
JWTClaims := {
    subject : Str,       # user ID
    role : [Admin, User, Guest],
    issuedAt : U64,      # timestamp
    expiresAt : U64,     # timestamp
} implements [Inspect]

createClaims : Str, [Admin, User, Guest], U64, U64 -> JWTClaims
createClaims = \subject, role, issuedAt, ttlSeconds ->
    @JWTClaims { subject, role, issuedAt, expiresAt: issuedAt + ttlSeconds }

# Validate claims (PURE — không cần crypto)
validateClaims : JWTClaims, U64 -> Result JWTClaims [Expired, InvalidSubject]
validateClaims = \@JWTClaims claims, currentTime ->
    if Str.isEmpty claims.subject then Err InvalidSubject
    else if currentTime > claims.expiresAt then Err Expired
    else Ok (@JWTClaims claims)

# Authorization check (PURE)
hasPermission : JWTClaims, [ReadUsers, WriteUsers, DeleteUsers, ReadOrders, WriteOrders] -> Bool
hasPermission = \@JWTClaims claims, permission ->
    when claims.role is
        Admin -> Bool.true    # admin can do everything
        User ->
            when permission is
                ReadUsers -> Bool.true
                ReadOrders -> Bool.true
                WriteOrders -> Bool.true
                _ -> Bool.false
        Guest ->
            when permission is
                ReadUsers -> Bool.true
                ReadOrders -> Bool.true
                _ -> Bool.false

# Tests
expect
    claims = createClaims "user-1" Admin 1000 3600    # expires at 4600
    validateClaims claims 2000 |> Result.isOk         # current = 2000 < 4600

expect
    claims = createClaims "user-1" User 1000 3600
    validateClaims claims 5000 == Err Expired          # current = 5000 > 4600

expect
    claims = createClaims "user-1" Admin 1000 3600
    hasPermission claims DeleteUsers == Bool.true

expect
    claims = createClaims "user-1" User 1000 3600
    hasPermission claims DeleteUsers == Bool.false

expect
    claims = createClaims "user-1" Guest 1000 3600
    hasPermission claims WriteOrders == Bool.false
```

### JWT Flow

```
1. Login: email + password → server validates → issue JWT
2. Client: store JWT (cookie/localStorage)
3. Requests: send JWT in header: Authorization: Bearer <token>
4. Server: decode JWT → validate claims → authorize
```

---

## 29.3 — OAuth 2.0

JWT (JSON Web Token) chứa thông tin user bên trong token — server không cần lookup database mỗi request. Trade-off: không thể revoke token đơn lẻ, phải đợi hết hạn.

### Authorization Code Flow

```
┌────────┐    1. Redirect    ┌──────────────┐
│  User  │ ───────────────→  │ Auth Provider │
│(browser)│                   │ (Google, etc.)│
│         │ ←───────────────  │              │
└────┬───┘  2. Auth Code     └──────┬───────┘
     │                               │
     │  3. Send code                 │
     ▼                               │
┌────────┐  4. Exchange code  ┌──────▼───────┐
│  Your  │ ───────────────→  │ Auth Provider │
│ Server │                   │              │
│        │ ←───────────────  │              │
└────────┘  5. Access Token   └──────────────┘
```

### OAuth trong Roc (concepts)

```roc
# ═══════ PURE: OAuth state machine ═══════

OAuthState : [
    NotStarted,
    AwaitingCode { state : Str, redirectUri : Str },
    AwaitingToken { code : Str },
    Authenticated { accessToken : Str, refreshToken : Str, expiresAt : U64 },
    Failed { error : Str },
]

# Build authorization URL (PURE)
buildAuthUrl = \config ->
    params = [
        "client_id=$(config.clientId)",
        "redirect_uri=$(config.redirectUri)",
        "response_type=code",
        "scope=$(Str.joinWith config.scopes " ")",
        "state=$(config.state)",
    ]
    "$(config.authEndpoint)?$(Str.joinWith params "&")"

# Validate OAuth callback (PURE)
validateCallback = \queryParams, expectedState ->
    state = Dict.get queryParams "state" |> Result.mapErr? \_ -> MissingState
    code = Dict.get queryParams "code" |> Result.mapErr? \_ -> MissingCode
    if state != expectedState then Err StateMismatch
    else Ok code

# Tests
expect
    config = {
        clientId: "my-app",
        redirectUri: "http://localhost:8000/callback",
        authEndpoint: "https://accounts.google.com/o/oauth2/auth",
        scopes: ["email", "profile"],
        state: "random123",
    }
    url = buildAuthUrl config
    Str.contains url "client_id=my-app"
    && Str.contains url "scope=email profile"

expect
    params = Dict.fromList [("state", "abc"), ("code", "xyz")]
    validateCallback params "abc" == Ok "xyz"

expect
    params = Dict.fromList [("state", "wrong"), ("code", "xyz")]
    validateCallback params "abc" == Err StateMismatch
```

---

## 29.4 — Roc Security Advantages

CORS, CSRF, XSS — ba lỗ hổng web phổ biến nhất. CORS kiểm soát ai được gọi API, CSRF bắt request giả mạo, XSS ngăn chèn script vào trang. Defence in depth — bảo vệ nhiều tầng.

### 1. Purity = No state leaks

```roc
# ❌ Trong mutable languages: state leak
# var currentUser = null;  // global state → ai cũng đọc được
# function getUser() { return currentUser; }

# ✅ Roc: không có global mutable state
# Mọi data truyền explicit qua function parameters
# Không secret nào "lơ lửng" trong memory
```

### 2. Opaque types = Encapsulation

```roc
# PasswordHash := Str
# → Không thể Stdout.line! hash    (không implement Inspect)
# → Không thể so sánh 2 hashes     (không implement Eq)
# → Không thể trích xuất string    (opaque!)

# Token := Str
# → Chỉ có module Token mới unwrap được
# → Ngoài module: chỉ gọi Token.validate, Token.getSubject
```

### 3. Type system prevents bugs

```roc
# ❌ String typing → runtime errors
# authorize("admin", "delete_user")  ← string typos!

# ✅ Tag unions → compiler catches
authorize : [Admin, User, Guest], [ReadUsers, WriteUsers, DeleteUsers] -> Bool
# → Không typo. Compiler error nếu sai tag.
```

### 4. Platform isolation = Capability-based security

```roc
# App KHÔNG THỂ:
# - Đọc filesystem (trừ khi platform cho File capability)
# - Mở network (trừ khi platform cho Http capability)
# - Access env vars (trừ khi platform cho Env capability)
#
# → Malicious domain code KHÔNG THỂ exfiltrate data
# → Platform controls ALL IO capabilities
```

---

## 29.5 — Middleware: Auth trong Web Service

Security là quy trình liên tục, không phải checklist một lần. Dependencies cập nhật thường xuyên, audit logs cho mọi action quan trọng, principle of least privilege cho mọi component.

```roc
# ═══════ PURE: Auth middleware logic ═══════

# Extract token from header
extractBearerToken : List { name : Str, value : List U8 } -> Result Str [NoAuthHeader, InvalidFormat]
extractBearerToken = \headers ->
    authHeader = List.findFirst headers \h -> h.name == "Authorization"
        |> Result.mapErr? \_ -> NoAuthHeader
    value = Str.fromUtf8 authHeader.value |> Result.mapErr? \_ -> InvalidFormat
    when Str.splitFirst value "Bearer " is
        Ok { after, .. } -> Ok after
        Err _ -> Err InvalidFormat

# Auth middleware pipeline
authenticateRequest = \headers, currentTime ->
    token = extractBearerToken? headers
    claims = decodeAndValidateToken? token currentTime
    Ok claims

# Protected route
handleProtectedRoute = \req, currentTime ->
    when authenticateRequest req.headers currentTime is
        Ok claims ->
            if hasPermission claims ReadOrders then
                jsonResponse 200 "{\"orders\": []}"
            else
                jsonResponse 403 "{\"error\": \"Forbidden\"}"
        Err NoAuthHeader ->
            jsonResponse 401 "{\"error\": \"Missing auth token\"}"
        Err Expired ->
            jsonResponse 401 "{\"error\": \"Token expired\"}"
        Err _ ->
            jsonResponse 401 "{\"error\": \"Invalid token\"}"

# Helper (placeholder)
decodeAndValidateToken = \token, currentTime ->
    # Platform provides actual JWT decode
    if Str.isEmpty token then Err InvalidFormat
    else Ok (createClaims "user-1" User 0 999999999)

jsonResponse = \status, body ->
    { status, headers: [], body: Str.toUtf8 body }
```

---


## ✅ Checkpoint 29

> Đến đây bạn phải hiểu:
> 1. Password hashing: Argon2/bcrypt (SLOW) > SHA/MD5 (FAST = BAD)
> 2. JWT = stateless token (header.payload.signature)
> 3. Roc advantage: purity = no state leaks, opaque types = encapsulation
>
> **Test nhanh**: Tại sao SHA-256 KHÔNG dùng cho passwords?
> <details><summary>Đáp án</summary>SHA-256 quá nhanh! Attacker brute-force dễ. Password hash phải CHẬM (Argon2, bcrypt).</details>

---

## 🏋️ Bài tập

**Bài 1** (10 phút): Role-Based Access Control

Implement RBAC với 4 roles và 6 permissions:

```roc
# Roles: SuperAdmin, Admin, Editor, Viewer
# Permissions: ManageUsers, ManageRoles, EditContent, PublishContent, ViewContent, ViewAnalytics
# SuperAdmin: all | Admin: all except ManageRoles | Editor: Edit+Publish+View | Viewer: View only
```

<details><summary>✅ Lời giải</summary>

```roc
checkAccess = \role, permission ->
    when role is
        SuperAdmin -> Bool.true
        Admin ->
            when permission is
                ManageRoles -> Bool.false
                _ -> Bool.true
        Editor ->
            when permission is
                EditContent -> Bool.true
                PublishContent -> Bool.true
                ViewContent -> Bool.true
                ViewAnalytics -> Bool.true
                _ -> Bool.false
        Viewer ->
            when permission is
                ViewContent -> Bool.true
                ViewAnalytics -> Bool.true
                _ -> Bool.false

expect checkAccess SuperAdmin ManageRoles == Bool.true
expect checkAccess Admin ManageRoles == Bool.false
expect checkAccess Editor EditContent == Bool.true
expect checkAccess Viewer EditContent == Bool.false
```

</details>

---

**Bài 2** (15 phút): Token refresh flow

Implement token refresh logic:

```roc
# Access token: short-lived (15 phút)
# Refresh token: long-lived (7 ngày)
# Khi access expired → dùng refresh để lấy access mới
# Khi refresh expired → phải login lại
```

<details><summary>✅ Lời giải</summary>

```roc
TokenPair : { accessToken : JWTClaims, refreshToken : { expiresAt : U64 } }

createTokenPair = \userId, role, currentTime ->
    accessClaims = createClaims userId role currentTime 900       # 15 min
    refreshExpiry = currentTime + 604800                          # 7 days
    { accessToken: accessClaims, refreshToken: { expiresAt: refreshExpiry } }

refreshAccess = \tokenPair, currentTime ->
    if currentTime > tokenPair.refreshToken.expiresAt then
        Err RefreshExpired    # phải login lại
    else
        # Issue new access token
        @JWTClaims oldClaims = tokenPair.accessToken
        newAccess = createClaims oldClaims.subject oldClaims.role currentTime 900
        Ok { tokenPair & accessToken: newAccess }

expect
    pair = createTokenPair "user-1" User 1000
    refreshAccess pair 2000 |> Result.isOk    # refresh chưa expired

expect
    pair = createTokenPair "user-1" User 1000
    refreshAccess pair 700000 == Err RefreshExpired  # 7 ngày đã qua
```

</details>

---

## 🔧 Troubleshooting

| Vấn đề | Nguyên nhân | Cách sửa |
|---|---|---|
| Password hash chậm | Đó là FEATURE, không phải bug | bcrypt/argon2 intentionally slow (chống brute-force) |
| JWT quá lớn | Nhét quá nhiều claims | Chỉ lưu essentials: sub, role, exp. Chi tiết query DB |
| Token leaked | Lưu token insecure | HttpOnly cookie (not localStorage), HTTPS only |
| OAuth state mismatch | CSRF attack hoặc expired state | Random state parameter, TTL 10 phút |

---

## Tóm tắt

- ✅ **Password hashing** = KHÔNG plaintext. bcrypt/argon2 (platform provides). Pure validation logic testable.
- ✅ **JWT** = Header.Payload.Signature. Claims: subject, role, expiry. Pure validation + permission checks.
- ✅ **OAuth 2.0** = Authorization Code flow. Pure state machine + URL builder + callback validator.
- ✅ **Roc advantages** = purity (no state leaks), opaque types (encapsulation), tag unions (no typos), platform isolation (capability-based).
- ✅ **Auth middleware** = `extractToken? → validateClaims? → checkPermission` pipeline.

## Tiếp theo

→ Chapter 30: **Application Security** — OWASP Top 10, injection prevention, XSS, CSRF, input validation qua type system.
