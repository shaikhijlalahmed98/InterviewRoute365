# 📝 Day 2 Quiz — User Login with JWT

**📖 Lesson**: [day-002-user-login-jwt.md](../lessons/day-002-user-login-jwt.md)
**🔑 Revision Keys**: [revision/day-002-user-login-jwt.md](../revision/day-002-user-login-jwt.md)
**📊 Total questions**: 30

---

## 🔧 Section A: Java / Spring Security (8 MCQs)

### Q1. JWT structure ke 3 hisse:
A) header.body.footer
B) header.payload.signature
C) algorithm.data.key
D) token.user.expiry

**Your Answer**: ___

### Q2. JWT signature kis liye hai:
A) Encryption
B) Compression
C) Tamper detection (prove token wasn't modified)
D) Faster lookup

**Your Answer**: ___

### Q3. JWT payload mein NEVER kya daalna chahiye:
A) userId
B) email
C) password ya secret keys (Base64 visible to anyone)
D) role

**Your Answer**: ___

### Q4. `BCrypt.matches(raw, hash)` kaise kaam karta hai bina salt diye:
A) Default salt use karta hai
B) Salt automatically embedded in hash string, extract karta hai
C) Salt randomly generate karta hai
D) Compile-time decided

**Your Answer**: ___

### Q5. `SecurityContextHolder` kya hai:
A) Database table
B) ThreadLocal storing authenticated user for current request
C) HTTP header
D) Spring annotation

**Your Answer**: ___

### Q6. `OncePerRequestFilter` ka matlab:
A) Filter only runs at startup
B) Filter runs ONCE per HTTP request — parses Bearer token, validates, sets context
C) Filter runs randomly
D) Filter runs in background thread

**Your Answer**: ___

### Q7. JWT secret key kahan rakhni chahiye:
A) Code mein hardcode
B) Git committed config file
C) Environment variable / secrets manager (NEVER in code)
D) Database

**Your Answer**: ___

### Q8. HS256 vs RS256:
A) HS256 faster, both symmetric
B) HS256 = symmetric (same key sign+verify); RS256 = asymmetric (private signs, public verifies) — better for distributed
C) RS256 deprecated
D) HS256 deprecated

**Your Answer**: ___

---

## 🌐 Section B: .NET / C# (5 MCQs)

### Q9. .NET mein JWT middleware register:
A) `app.UseJwt()`
B) `app.UseAuthentication()` registers `JwtBearerHandler`
C) `services.AddJwt()`
D) Auto-enabled

**Your Answer**: ___

### Q10. .NET equivalent of `SecurityContextHolder`:
A) `Session.User`
B) `HttpContext.User` (ClaimsPrincipal)
C) `ThreadStatic.User`
D) `Application.User`

**Your Answer**: ___

### Q11. `TokenValidationParameters` mein kya configure karte hain:
A) Database connection
B) Issuer, audience, lifetime, signing key validation rules
C) HTTP routes
D) Encryption algorithm

**Your Answer**: ___

### Q12. .NET mein BCrypt use karne ke liye:
A) Built-in `BCrypt` class
B) Install `BCrypt.Net-Next` NuGet package
C) Cannot use BCrypt in .NET
D) Use `IPasswordHasher` only

**Your Answer**: ___

### Q13. `JwtSecurityTokenHandler().WriteToken(token)` returns:
A) JSON object
B) Serialized JWT string (header.payload.signature)
C) Binary blob
D) URL

**Your Answer**: ___

---

## 🗄️ Section C: SQL (5 MCQs)

### Q14. Login query: `WHERE email = ?` — `email` pe `UNIQUE` constraint hai, to konsa scan hota hai:
A) Table Scan
B) Index Seek (O(log n))
C) Full Scan
D) Sequential Scan

**Your Answer**: ___

### Q15. `SELECT *` ka problem login mein:
A) Compiles slow
B) Wastes bandwidth + may break covering indexes
C) Returns wrong data
D) Locks all rows

**Your Answer**: ___

### Q16. Password DB comparison `WHERE password = ?` — galat kyun:
A) Cannot compare strings in SQL
B) DB mein hashed password hai, raw password se direct compare impossible; comparison MUST happen in app layer with BCrypt
C) Too slow
D) Causes SQL injection

**Your Answer**: ___

### Q17. Login (pure read) ke liye sufficient isolation level:
A) SERIALIZABLE
B) REPEATABLE READ
C) READ COMMITTED
D) READ UNCOMMITTED

**Your Answer**: ___

### Q18. 10 lakh users pe login query slow if:
A) Server is fast
B) `email` column pe index nahi hai (Table Scan)
C) Network is good
D) Browser is old

**Your Answer**: ___

---

## 🎨 Section D: Angular (5 MCQs)

### Q19. Cold Observable from `HttpClient.post()` — 2 baar `.subscribe()` karne pe:
A) Cached response milta hai
B) 2 separate HTTP requests fire hote hain (each subscribe = new execution)
C) Error throw karta hai
D) Same Observable share hota hai

**Your Answer**: ___

### Q20. JWT token kahan store kar sakte ho frontend pe:
A) Only sessionStorage
B) localStorage (XSS risk) ya HttpOnly cookie (CSRF risk) — trade-off
C) Only IndexedDB
D) Only memory

**Your Answer**: ___

### Q21. `HttpInterceptor` ka kaam:
A) Block network requests
B) Auto-attach `Authorization: Bearer <token>` header to every outgoing request
C) Compress requests
D) Encrypt requests

**Your Answer**: ___

### Q22. `intercept()` method must:
A) Return void
B) Return `Observable<HttpEvent<any>>` via `next.handle(modifiedReq)`
C) Modify response directly
D) Throw exception

**Your Answer**: ___

### Q23. Logout JWT app mein simple way:
A) Server pe API call mandatory
B) Client `localStorage.removeItem('token')` — stateless, token deleted from client
C) Reload page
D) Cookie clear

**Your Answer**: ___

---

## 🏗️🏛️ Section E: System Design + OOP (7 MCQs)

### Q24. Stateless JWT ka biggest disadvantage:
A) Slow
B) Insecure
C) Cannot revoke before expiry (server doesn't store sessions)
D) Big payload

**Your Answer**: ___

### Q25. Stateless JWT scale karta hai because:
A) Smaller token
B) No central session store needed — each server verifies independently with signature
C) Better encryption
D) Auto-load-balanced

**Your Answer**: ___

### Q26. Revoke support add karne ke liye stateless JWT mein:
A) Long expiry
B) Short expiry (15 min) + refresh token, OR Redis blocklist of revoked jti
C) Restart server
D) Cannot revoke ever

**Your Answer**: ___

### Q27. User enumeration prevention login mein:
A) Show "email exists but password wrong" message
B) Always generic message "Invalid email or password" regardless of which field
C) Show CAPTCHA always
D) Block all login attempts

**Your Answer**: ___

### Q28. Abstraction OOP principle:
A) Hide all variables
B) Show only essential interface, hide implementation complexity
C) Inherit from parent
D) Multiple inheritance

**Your Answer**: ___

### Q29. Abstraction Roman Urdu memory hook:
A) DABBA
B) BARTAN BADLO
C) NAQSHA AUR NAQSHANAVEES (naqsha=interface dikhta, naqshanavees=impl chhupa)
D) AZAAN PATTERN

**Your Answer**: ___

### Q30. `List<T>` interface ke 2 implementations example:
A) String aur Integer
B) ArrayList aur LinkedList — same operations interface, different internal storage
C) Singleton aur Factory
D) Public aur Private

**Your Answer**: ___

---

## 🔒 Answer Key

<details>
<summary>⚠️ Click to reveal — complete first!</summary>

1. **B** — header.payload.signature
2. **C** — Tamper detection
3. **C** — Password/secrets visible in Base64
4. **B** — Salt embedded in hash
5. **B** — ThreadLocal authenticated user
6. **B** — Once per HTTP request
7. **C** — Environment variable
8. **B** — HS256 symmetric, RS256 asymmetric
9. **B** — `app.UseAuthentication()`
10. **B** — `HttpContext.User`
11. **B** — Issuer/audience/lifetime/key rules
12. **B** — `BCrypt.Net-Next` package
13. **B** — Serialized JWT string
14. **B** — Index Seek
15. **B** — Bandwidth + covering index break
16. **B** — DB has hash, app-layer BCrypt comparison
17. **C** — READ COMMITTED
18. **B** — No index = table scan
19. **B** — 2 separate HTTP requests
20. **B** — localStorage vs HttpOnly cookie trade-off
21. **B** — Auto-attach Bearer header
22. **B** — Return `Observable<HttpEvent>` via next.handle
23. **B** — Client localStorage.removeItem
24. **C** — Cannot revoke before expiry
25. **B** — No central session store
26. **B** — Short expiry + refresh token OR Redis blocklist
27. **B** — Always generic message
28. **B** — Essential interface, hide complexity
29. **C** — Naqsha aur Naqshanavees
30. **B** — ArrayList, LinkedList

</details>
