# 🔑 Day 2 — User Login with JWT: Revision Keys

**📖 Full lesson**: [day-002-user-login-jwt.md](../lessons/day-002-user-login-jwt.md)
**⏱️ Reading time**: 5 minutes
**🎯 Use when**: Pre-interview revision

---

## ⚡ If You Remember Only 7 Things

1. **VIBES** = Verify password (BCrypt) + Issue JWT + Bearer header + Expiry short + Signature stateless
2. **JWT structure**: `header.payload.signature` — first two Base64-encoded (NOT encrypted, payload visible!), third is HMAC-SHA256 signature
3. **NEVER put password/secrets in JWT payload** — visible to anyone
4. **`BCrypt.matches(raw, hash)`** auto-extracts salt from stored hash + re-hashes raw + compares
5. **Stateless** = server doesn't store session; signature alone proves validity. Horizontal scalable. Logout = client deletes token.
6. **Short-lived access token (15 min) + refresh token** = recoverable from theft. Worst case: hacker has 15 min.
7. **`localStorage` vs `HttpOnly cookie`** trade-off: XSS vs CSRF. Senior answer mentions both.

---

## ☕ Java / Spring Quick Keys

- `AuthenticationManager` + `DaoAuthenticationProvider` + `UserDetailsService` = Spring Security trio
- `BCryptPasswordEncoder.matches()` — salt embedded in hash, auto-extracted
- `JwtUtil.generateToken()` using jjwt: `Jwts.builder().setSubject().setExpiration().signWith(key)`
- `OncePerRequestFilter` = runs ONCE per request, parses Bearer token, validates, sets `SecurityContextHolder`
- `SecurityContextHolder` = `ThreadLocal` storing authentication for downstream code
- Secret key from `@Value("${app.jwt.secret}")` — environment variable, NEVER hardcoded
- HMAC-SHA256 = symmetric signature (same key signs + verifies)
- RS256 = asymmetric (private key signs, public key verifies) — better for distributed verification

---

## 🌐 .NET / C# Quick Keys

- `app.UseAuthentication()` registers `JwtBearerHandler` middleware
- `TokenValidationParameters` for issuer/audience/lifetime/signing key checks
- `HttpContext.User` = `ClaimsPrincipal` (vs Spring's `SecurityContextHolder`)
- `BCrypt.Net-Next` package for password verification (consistency with Java teams)
- `IConfiguration["Jwt:Secret"]` for secret from `appsettings.json` or env
- `JwtSecurityTokenHandler().WriteToken(token)` to serialize JWT

---

## 🗄️ SQL Quick Keys

- Login query: `SELECT id, email, password_hash, role FROM users WHERE email=?`
- Email `UNIQUE` constraint → auto B-tree index → **Index Seek** (O(log n))
- Without index = **Table Scan** (O(n)) — 1M users = seconds vs milliseconds
- Don't `SELECT *` — wastes bandwidth, breaks covering indexes
- Don't compare password in SQL — hash is irreversible, must use BCrypt in app layer
- Login is pure read → `READ COMMITTED` isolation enough

---

## 🎨 Angular Quick Keys

- `HttpClient.post()` returns **cold Observable** — only fires on `.subscribe()`
- Save token in `localStorage.setItem('token', res.token)` (with XSS trade-off awareness)
- **`HttpInterceptor`** = darbaan that auto-attaches `Authorization: Bearer <token>` header to every request
- Implement `intercept()` method, clone request, add header, pass to `next.handle()`
- Login error: generic user-friendly message ("Email ya password ghalat hai") — never reveal which field is wrong

---

## 🏗️ System Design Quick Keys

- **Stateful session** (server stores sessionId in Redis/DB) — instant revoke, but DB hit per request
- **Stateless JWT** (signature self-validates) — no DB lookup, scales horizontally, but cannot revoke before expiry
- **Solution for revoke**: short expiry (15 min) + refresh token, or Redis blocklist for compromised tokens
- **Microservices** love JWT — any service verifies with shared secret/public key, no central session store
- **Secret rotation** policy mandatory — leak = all tokens compromised

---

## 🏛️ OOP — Abstraction Quick Keys

- **Abstraction** = hide implementation complexity, expose only essential interface
- **Memory hook**: "NAQSHA AUR NAQSHANAVEES" — naqsha (interface/blueprint) dikhta hai, naqshanavees (impl) chhupa hota hai
- Example: `PasswordEncoder` interface — multiple impls (BCrypt, Argon2, PBKDF2). Caller doesn't care which.
- Example: `List<T>` interface — `ArrayList` vs `LinkedList` impls, same operations
- Java: `interface` or `abstract class`
- C#: same — `interface` or `abstract class`
- Benefit: swap implementation without changing consumer code

---

## 🎤 Interview One-Liners

- *"For login I use Spring Security's `BCrypt.matches()` — salt embedded in hash, auto-extracted. Then generate HS256-signed JWT with 15-min expiry."*
- *"`SecurityContextHolder` is ThreadLocal, per-request. `OncePerRequestFilter` parses Bearer token and sets it."*
- *"JWT is **signed, NOT encrypted**. Payload is Base64 — anyone can read. Never put password/secrets in claims."*
- *"For revoke, short access token + DB-backed refresh token. Or Redis blocklist for instant revocation."*

---

## ⚠️ Red Flags

- ❌ "JWT encrypted hota hai, password daal sakte hain" → JWT is signed, NOT encrypted
- ❌ "Login fail = 'email exist karta hai but password wrong' message" → user enumeration attack
- ❌ "Secret key code mein hardcode kar lia" → must be in env/secrets manager
- ❌ "Token kabhi expire na ho, convenience ke liye" → leak = lifetime access

---

**Self-test**: [Day 2 Quiz](../quizzes/day-002-user-login-jwt.md)
