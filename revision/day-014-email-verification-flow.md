# 🔑 Day 14 — Email Verification Flow: Revision Keys

**📖 Full lesson**: [2026-05-26-day14-email-verification-flow.md](../lessons/01-beginner/2026-05-26-day14-email-verification-flow.md)
**⏱️ Reading time**: 5-7 minutes
**🎯 Use when**: Night before interview, weekly revision, concept refresh

---

## ⚡ If You Remember Only 10 Things

1. **VERIFY** = **V**alidate token + **E**xpiry check + **R**ace-condition safe + **I**dempotent + **F**ire async email + **Y**ield clear UI feedback
2. **Token = `SecureRandom` (32 bytes = 256 bits)**, NEVER `userId + timestamp` (predictable = account takeover). Booking.com 2017 = 50,000 accounts hijacked.
3. **Store SHA-256 hash of token in DB, raw token only in email link.** DB leak = tokens still safe (one-way hash).
4. **Atomic single-use UPDATE**: `UPDATE tokens SET used=1 WHERE token=? AND used=0 AND expires_at > NOW()`. `@@ROWCOUNT = 0` → token invalid/expired/used. Single statement = race-safe.
5. **Idempotent verify**: already-verified user clicks link 2nd time → return 200 OK (success state), NOT error. Otherwise UX panic.
6. **Outbox Pattern** solves dual-write problem — same DB transaction inserts user + outbox row; background worker dispatches SMTP with retry. SMTP down ≠ user creation fail.
7. **ISP (Interface Segregation)** = "BHARI INTERFACE MAT BHEJO". Don't make `EmailVerificationService` depend on fat `INotificationService` with sendEmail/sendSms/sendPush. Inject only `IEmailSender`.
8. **Unique index on `token` column** = required. Duplicate prevention + O(log n) lookup. Without it = O(n) full scan + collision risk.
9. **Anti-enumeration on duplicate signup**: return generic "check email for next steps" — never reveal "email exists". Stripe/GitHub follow this.
10. **HTTP 410 Gone (not 400)** for expired link with `Retry-After` header. Frontend shows "Resend link" with rate-limit (3/hour).

---

## ☕ Java / Spring Boot — Quick Keys

- **`SecureRandom`** = OS-level CSPRNG (cryptographically secure pseudo-random number generator). Linux: `/dev/urandom`. Windows: `CryptGenRandom`. Hides OS differences behind one API.
- **`Random` (java.util)** = LCG (Linear Congruential Generator) — predictable after observing 3-4 outputs. NEVER use for security.
- **32 bytes Base64URL-encoded** = 43 characters (no padding). 2^256 possibilities = mathematically unbreakable.
- **`@Transactional` mechanism**: Spring CGLib/JDK proxy wraps method → `txManager.begin() → method() → commit/rollback`. Self-invocation bypasses proxy!
- **`@Async`**: separate thread pool, HTTP response returns immediately, email send happens in background. Requires `@EnableAsync` on `@Configuration` class.
- **`JavaMailSender`** interface = Spring Boot abstraction; auto-configured from `spring.mail.*` properties. Tests can swap with mock.
- **Idempotency check pattern**: `if (vt.isUsed()) return;` — silent success, not throw. User clicked link 2 times = same outcome.
- **`@Scheduled(cron = "...")`**: cleanup expired tokens daily/hourly. Requires `@EnableScheduling`.

---

## 🌐 .NET / C# — Quick Keys

- **`RandomNumberGenerator.Fill(bytes)`** = .NET equivalent of `SecureRandom`. Uses Windows CNG or `/dev/urandom`. Static method, thread-safe.
- **`Convert.ToBase64String()`** + replace `+→-`, `/→_`, trim `=` for URL-safe Base64 (RFC 4648 §5).
- **EF Core**: `_db.Entities.Add(vt) + SaveChangesAsync()`. Change tracker dirty-flags entities, generates SQL on save. `.AsNoTracking()` for read-only.
- **Transactions**: `await using var tx = await _db.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted)`. Dispose pattern auto-rollback on exception.
- **`BackgroundService`** + `Channel<T>` = Spring's `@Async` equivalent. Long-running hosted service, queue-based work distribution.
- **`Include(v => v.User)`** = EF Core eager loading via SQL JOIN. Without it = N+1 problem.
- **Mailing libraries**: `MailKit` (recommended, full SMTP), `SendGrid SDK`, `Azure.Communication.Email`. Inject as `IEmailSender`.
- **HTTP status code helpers**: `return StatusCode(410)` for expired, `return Ok()` for success, `return BadRequest()` for invalid format.

---

## 🗄️ SQL — Quick Keys

- **`UNIQUE` constraint on `token`** = creates implicit unique B-tree index. Duplicate INSERT → 2627 (SQL Server) / 1062 (MySQL). Catch → return 409.
- **B-tree on token** = O(log n) lookup. 1M rows = ~3-4 disk page reads. Without index = O(n) full table scan = production killer at scale.
- **Atomic check-and-set**: `UPDATE ... WHERE token=? AND is_used=0 AND expires_at > NOW()`. Single statement = no TOCTOU race.
- **`@@ROWCOUNT` / `OUTPUT inserted.user_id`**: check 0 means token invalid/used/expired without separate SELECT.
- **Partial/filtered index**: `CREATE INDEX idx_token ON verification_tokens(token) WHERE is_used=0`. Smaller index → faster lookup → less memory.
- **Cleanup job**: `DELETE WHERE expires_at < NOW() AND created_at < NOW() - INTERVAL 7 DAY`. Scheduled hourly/daily.
- **READ COMMITTED** sufficient. Single-statement UPDATE takes row-level X-lock automatically. Higher isolation overkill.
- **Don't store raw token**: store `SHA-256(token)` hash. DB dump leak → attacker still can't use tokens.

---

## 🎨 Angular — Quick Keys

- **`ActivatedRoute.snapshot.paramMap.get('token')`** = read URL param at component init. Snapshot = one-time read; use `paramMap` Observable for reactive.
- **`HttpClient` Observable** = cold; doesn't fire until `.subscribe()`. Multiple subscribes = multiple HTTP calls.
- **`catchError(err => { ...; return EMPTY })`** = swallow error, emit nothing. Pair with state machine for UX.
- **State machine pattern**: `'loading' | 'success' | 'expired' | 'invalid'`. `[ngSwitch]` renders right view per state.
- **HTTP status mapping**: `err.status === 410` → 'expired'; `404` → 'invalid'; `200` → 'success'. Map to user-friendly UI.
- **Loading spinner during HTTP**: prevents user thinking site is broken on slow networks. Critical for verify flows (email click → page load).
- **`routerLink="/login"`** = declarative navigation; preserves browser history. Better than `window.location.href`.
- **Resend flow**: button with rate-limit countdown (`30s` cooldown via RxJS `interval + take(30)`). Prevents spam clicks → SMS bombing on backend.

---

## 🏗️ System Design — Quick Keys

- **Dual-write problem**: same operation writes to DB + external system (SMTP). Network partition → DB committed, email not sent → inconsistent.
- **Outbox Pattern** = solution. Same DB transaction inserts business data + `outbox_events` row. Background worker polls outbox → dispatches to external (SMTP/Kafka).
- **Outbox guarantees**: at-least-once delivery, eventual consistency, replayable, retryable. Workers must be **idempotent** (same event 2x = same outcome).
- **TTL pattern**: token table grows unbounded without cleanup. Scheduled DELETE keeps it tight. Alternative: PostgreSQL `pg_partman` or Redis with native TTL.
- **Provider chain**: primary SMTP fails → fallback to secondary (Mailgun → SendGrid → AWS SES). Circuit breaker prevents hammering down service.
- **Anti-enumeration**: signup with existing email → return generic "check email" response. Same response for verified and unverified existing accounts. Defeats user-list scraping.
- **Real companies**: Shopify (Outbox for order confirmations), GitHub (background workers via Resque), Auth0 (Kafka-based auth events pipeline).
- **Idempotency at every layer**: API (idempotency key), service (already-verified check), DB (atomic UPDATE WHERE is_used=0). Defense-in-depth.

---

## 🏛️ OOP — Interface Segregation Principle (ISP) Quick Keys

- **ISP** = Clients should not be forced to depend on interfaces (methods) they don't use. Break fat interfaces into role-specific small interfaces.
- **Memory hook**: **"BHARI INTERFACE MAT BHEJO"** / **"CHOTA INTERFACE, BARA SUKOON"**
- **Real-life analogy**: shaadi vendors — har vendor ka apna interface (`IPhotographer`, `IDecorator`, `ICaterer`), na ke ek fat `IWeddingVendor`. Photographer ko mehndi method implement nahi karna parta.
- **Bad**: One `INotificationService` with sendEmail/sendSms/sendPush/sendWhatsApp → forced dependencies on unused methods.
- **Good**: Separate `IEmailSender`, `ISmsSender`, `IPushNotifier`. Each consumer injects only what it needs.
- **Java**: multiple interfaces on one class allowed (no diamond issue for pure interfaces). Spring `@Qualifier` picks specific impl.
- **C#**: same pattern. `services.AddScoped<IEmailSender, SendGridEmailSender>()` in `Program.cs`.
- **Test benefit**: mock 1 method vs mock 4 methods. Faster, cleaner, less false confidence.
- **Conflicts with**: Facade pattern (intentional fat interface), generic repository (`IRepository<T>` with all CRUD).
- **Pairs with**: SRP (small interface = single responsibility), DIP (depend on abstraction = easier when abstraction is small), Strategy Pattern.
- **Common mistake**: Adding methods to existing interface → breaks all implementers. Create new interface or extension interface instead.

---

## 🎤 Interview One-Liners (Bolne Layak Confidently)

- *"Token generate karne ke liye `SecureRandom` use karunga `Random` ke bajaye — kyunki `Random` ka LCG predictable hai. 32 bytes Base64URL = 43 chars, 2^256 possibilities, mathematically unbreakable."*
- *"DB mein raw token nahi, SHA-256 hash store karunga. DB leak ho jaye to bhi tokens compromised nahi honge — one-way hash."*
- *"Single-use guarantee atomic UPDATE se: `UPDATE tokens SET used=1 WHERE token=? AND used=0 AND expires_at > NOW()`. `@@ROWCOUNT = 0` matlab invalid/expired/used. No TOCTOU race."*
- *"Email sending `@Async` rakhunga — HTTP response 200ms mein wapis aaye, SMTP latency user ko nahi feel ho. Production mein Outbox pattern use karunga for delivery guarantee."*
- *"Service ko `IEmailSender` interface inject karunga, na ke concrete `SendGridClient`. Interface Segregation Principle — kal Mailgun pe shift karna ho to OtpService ka code touch nahi karna parega."*
- *"Already-verified user dobara link click kare to 200 OK return karunga, error nahi. Idempotency = better UX + safer."*
- *"Duplicate signup pe 'email already exists' nahi return karunga — user enumeration attack se bachne ke liye. Generic 'check your email' response, Stripe/GitHub follow karte hain."*

---

## ⚠️ Red Flags — Don't Say These In Interview

- ❌ "Token `userId + timestamp` se bana lenge" → predictable, account takeover risk
- ❌ "Token plaintext DB mein save karenge" → DB leak = mass account takeover
- ❌ "`@Transactional` ke andar `emailSender.send()` directly call kar denge" → dual-write problem, SMTP failure rolls back user creation
- ❌ "Already-verified pe error throw karenge" → bad UX, user panic
- ❌ "Concrete `SendGridClient` directly inject kar denge — easy hai" → vendor lock-in, testing impossible
- ❌ "Email exists error throw karenge" → enumeration attack
- ❌ "Token kabhi expire nahi karega" → GDPR/PDPL violation, security disaster

---

## 🧠 Memory Hooks Recap

- **VERIFY** — Validate, Expiry, Race-safe, Idempotent, Fire async, Yield UI feedback
- **"BHARI INTERFACE MAT BHEJO"** — ISP
- **"CHOTA INTERFACE, BARA SUKOON"** — ISP (alt)
- **"OS ki lottery ticket machine"** — SecureRandom (OS entropy pool)
- **"DABBA-WALA jo email deliver karta hai"** — Outbox pattern
- **"Ghar ka chowkidar — bahar wala andar nahi"** — Anti-enumeration

---

## 🔗 Cross-References

- For **OOP deep**: see [cheatsheets/oop-principles.md](../cheatsheets/oop-principles.md) (ISP section)
- For **Java↔.NET**: see [cheatsheets/spring-vs-dotnet.md](../cheatsheets/spring-vs-dotnet.md)
- For **all memory hooks**: see [cheatsheets/memory-hooks.md](../cheatsheets/memory-hooks.md)
- **Previous lesson**: [Day 13 — Form Validation](./day-013-form-validation.md) (LSP — base contract honor karo)
- **Next lesson**: [Day 15 — Phone OTP Verification](./day-015-phone-otp-verification.md) (DIP — abstraction pe depend karo)

**Self-test**: Take the [Day 14 quiz](../quizzes/day-014-email-verification-flow.md) — 50 MCQs to verify mastery.
