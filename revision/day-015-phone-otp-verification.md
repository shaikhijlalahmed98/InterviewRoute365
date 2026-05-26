# 🔑 Day 15 — Phone OTP Verification: Revision Keys

**📖 Full lesson**: [2026-05-27-day15-phone-otp-verification.md](../lessons/01-beginner/2026-05-27-day15-phone-otp-verification.md)
**⏱️ Reading time**: 5-7 minutes
**🎯 Use when**: Night before interview, weekly revision, concept refresh

---

## ⚡ If You Remember Only 10 Things

1. **OTP-SECURE** = **O**TP hashed + **T**hrottled requests + **P**roviders chained + **S**ingle-use + **E**xpiry enforced + **C**onstant-time compare + **U**pserted atomically + **R**edis rate-limited + **E**rrors clear in UI
2. **OTP store as SHA-256 hash**, NEVER plain. DB leak = millions of OTPs exposed otherwise. Saudi STC Pay follows this strictly.
3. **6 digits = 1 million combos**. Without rate limit + attempt throttling = brute-forceable in 3 hours by bot at 100 req/sec.
4. **Constant-time compare**: `MessageDigest.isEqual()` (Java) / `CryptographicOperations.FixedTimeEquals()` (.NET). NEVER `.equals()` — timing attack leaks OTP.
5. **Redis INCR atomic** = rate limit primitive. Single-threaded Redis architecture = no race conditions. 3 requests/hour per phone typical.
6. **DIP (Dependency Inversion)** = "INTERFACE PE BHAROSA, IMPLEMENTATION PE NAHI". `OtpService` depends on `ISmsSender`, not `TwilioClient`. Swap provider = zero business code change.
7. **MERGE upsert** (SQL Server) / `ON CONFLICT` (PostgreSQL) = atomic insert-or-update. Without it = race window between SELECT and INSERT/UPDATE = crash on unique constraint.
8. **Attempts counter atomic**: `UPDATE otp SET attempts=attempts+1 WHERE phone=? OUTPUT inserted.attempts`. Without atomicity = parallel brute force bypasses counter.
9. **Single-use OTP**: on successful verify, DELETE the row. Otherwise replay attacks possible.
10. **Provider chain pattern**: Twilio (primary) → Veevo (fallback) → Jazz (backup). Circuit breaker per provider. Resilient to regional outages (Uber, Telegram model).

---

## ☕ Java / Spring Boot — Quick Keys

- **`SecureRandom`** for OTP digit generation. `new SecureRandom().nextInt(10)` per digit. NEVER `Math.random()` — PRNG, predictable.
- **`MessageDigest.isEqual(byte[], byte[])`** = constant-time. Loops over ALL bytes regardless of mismatch position. Defeats timing attacks.
- **`RedisTemplate.opsForValue().increment(key)`** = atomic INCR. Returns new value. Pair with `expire(key, Duration)` for TTL.
- **`@Qualifier("twilio")`** picks specific bean when multiple `ISmsSender` impls registered. Or use `@Primary` for default.
- **`@Transactional`** ensures attempts increment + OTP record state changes commit atomically. Self-invocation gotcha applies.
- **`@RequiredArgsConstructor`** (Lombok) auto-generates constructor for `final` fields = clean constructor injection without boilerplate.
- **Spring profiles**: `@Profile("test")` for `MockSmsSender` (logs to console), `@Profile("prod")` for `TwilioSmsSender`. Same code, different impls per env.
- **Rate-limit exception → 429 Too Many Requests**: `@ResponseStatus(HttpStatus.TOO_MANY_REQUESTS)` on exception class. Set `Retry-After` header.

---

## 🌐 .NET / C# — Quick Keys

- **`RandomNumberGenerator.Fill(Span<byte>)`** = CSPRNG. Uses stack-allocated `Span<byte>` for zero heap allocation.
- **`CryptographicOperations.FixedTimeEquals()`** = .NET's constant-time comparison helper (since .NET Core 2.1).
- **`IDistributedCache`** = Redis abstraction. `GetStringAsync` / `SetStringAsync` with `AbsoluteExpirationRelativeToNow`.
- **Built-in DI**: `services.AddScoped<ISmsSender, TwilioSmsSender>()`. Switch provider in `Program.cs` = zero service code change. **Pure DIP**.
- **`IOptions<TwilioConfig>`** = strongly-typed config binding from `appsettings.json`. Better than scattered `IConfiguration.GetValue`.
- **Named/keyed services** (.NET 8+): `services.AddKeyedScoped<ISmsSender, TwilioSmsSender>("twilio")`. Resolve via `GetRequiredKeyedService<ISmsSender>("twilio")`.
- **`MERGE` via raw SQL** in EF Core: `_db.Database.ExecuteSqlInterpolated($"MERGE ...")`. Or use `ON CONFLICT` for PostgreSQL via Npgsql.
- **Async patterns**: every IO method `Async`. `await` doesn't block thread — releases to thread pool, resumes on continuation.

---

## 🗄️ SQL — Quick Keys

- **`MERGE` (SQL Server)** = atomic upsert. Single statement, single lock, no race window. Syntax: `MERGE target USING src ON cond WHEN MATCHED THEN UPDATE WHEN NOT MATCHED THEN INSERT`.
- **`ON CONFLICT` (PostgreSQL)**: `INSERT ... ON CONFLICT (phone) DO UPDATE SET ...`. Same semantics, cleaner syntax.
- **`OUTPUT inserted.*`** (SQL Server) / `RETURNING *` (PostgreSQL) = get back row after DML without separate SELECT.
- **Primary key on `phone`** = one OTP per phone (enforced). Implicit unique index = fast lookup.
- **Expiry index**: `CREATE INDEX idx_expiry ON otp_records(expires_at)`. Enables cleanup `DELETE WHERE expires_at < NOW()` without table scan.
- **Attempts counter as `INT NOT NULL DEFAULT 0`**: atomic increment via `UPDATE ... SET attempts = attempts + 1`.
- **VARCHAR(128) for SHA-256 hex**: 64 hex chars + buffer. Could use BINARY(32) for raw bytes (more compact).
- **Cleanup schedule**: every 10 min DELETE expired rows older than 30 min buffer. Prevents table bloat.

---

## 🎨 Angular — Quick Keys

- **`timer(0, 1000)`** = RxJS countdown emitter. Combine with `map(t => 300 - t)` + `takeWhile(t => t >= 0)` for 5-minute countdown.
- **`Validators.pattern(/^\d{6}$/)`** = exactly 6 digits. Pair with `maxlength="6"` on input + `inputmode="numeric"` for mobile keyboard.
- **`(input)` handler** auto-submits on 6 digits filled: `if (e.target.value.length === 6 && form.valid) verify()`. UX delight on mobile.
- **`BehaviorSubject<number>(30)`** for resend cooldown. `.next(value)` updates. `async` pipe in template auto-subscribes.
- **`finalize(() => verifying = false)`** runs on Observable completion or error. Like try-finally for streams.
- **`takeUntil(destroy$)`** pattern: unsubscribe on component destroy. Or use `async` pipe for auto-unsubscribe.
- **`disabled` binding on button**: `[disabled]="form.invalid || verifying"`. Prevents double-submit during in-flight request.
- **Error mapping**: 429 → "Too many requests, wait X minutes". 410 → "OTP expired, resend". 400 → "Invalid OTP, retry".

---

## 🏗️ System Design — Quick Keys

- **Token Bucket algorithm**: bucket holds N tokens, refills R/sec, each request consumes 1. Empty = reject. Allows bursts + sustains long-term limit. Implemented via Redis INCR + EXPIRE.
- **Leaky Bucket** (alternative): fixed-rate drain, queue requests above rate. Smoother but no burst capacity.
- **Provider Chain Pattern**: list of providers tried in order. First success returns. All fail → throw `AllProvidersFailedException`. Logs every fallback.
- **Circuit Breaker**: 3 failures in 10s → OPEN circuit for 60s (reject without trying). HALF-OPEN after 60s, single test request, success → CLOSED, fail → re-OPEN. Resilience4j / Polly libraries.
- **Hexagonal Architecture / Ports & Adapters**: business core "inside", external dependencies as adapters implementing port interfaces. DIP at architectural level.
- **SMS bombing attack**: attacker triggers 1000s of OTPs to victim's phone → harassment + your $$ loss. Rate limit per phone (not per IP) is the defense.
- **Real companies**: Uber (MessageBird → Twilio → Plivo), Telegram (4+ providers, country-specific routing), Easypaisa (Jazz + Telenor + Ufone direct).
- **Cost model**: each SMS ~$0.05. 1000 fake OTPs = $50 loss. Rate limit = cost control + security.

---

## 🏛️ OOP — Dependency Inversion Principle (DIP) Quick Keys

- **DIP** = (1) High-level modules don't depend on low-level. Both depend on **abstractions**. (2) Abstractions don't depend on details; details depend on abstractions.
- **Memory hook**: **"INTERFACE PE BHAROSA, IMPLEMENTATION PE NAHI"** / **"SOCKET PATTERN"** (kettle uses socket, not specific power source)
- **Real-life analogy**: electric kettle works with any wall socket. Tomorrow switch from WAPDA to solar → kettle unchanged. Socket = abstraction; provider = detail.
- **Bad**: `new TwilioClient(...)` inside `OtpService`. Tightly coupled, untestable, vendor lock-in.
- **Good**: Constructor injects `ISmsSender`. Composition root (Spring `@Configuration` / `Program.cs`) decides concrete impl.
- **DI containers ENABLE DIP**: Spring's `@Autowired`, .NET's `IServiceCollection`. They don't FORCE DIP — you can still `new` concrete inside service.
- **Common code where DIP needed**: third-party SDKs (Twilio, SendGrid, AWS), databases, file system, time (`Clock` abstraction for testable time).
- **NOT needed for**: stable framework types (`String`, `List`, `LocalDateTime`), pure functions (`Math.sqrt`), no-boundary internal utilities.
- **Hexagonal Architecture** = DIP applied to architecture. Core hexagon (business logic) + adapters (DB, API, queue) implementing ports (interfaces).
- **Combines with**: Strategy Pattern (interchangeable algorithms), Factory (create concrete from string), Decorator (wrap with cross-cutting concerns).

---

## 🎤 Interview One-Liners (Bolne Layak Confidently)

- *"OTP plaintext kabhi store nahi karunga — SHA-256 hash DB mein. DB leak ho jaye to bhi OTPs safe rahein."*
- *"6-digit OTP industry standard hai (WhatsApp, Google, Stripe) — entropy vs UX balance. Rate limit + attempt throttling brute force ko 60+ hours mein le jate hain, jo unrealistic risk hai."*
- *"OTP compare karne ke liye `MessageDigest.isEqual()` / `FixedTimeEquals()` use karunga — timing attack se bachne ke liye. `.equals()` ka early-exit OTP characters leak kar sakta hai."*
- *"Rate limiting Redis INCR se — atomic operation, single-thread Redis = no race condition. 3 requests/hour per phone, key expiry 1 hour."*
- *"`OtpService` ko `ISmsSender` inject karunga, na ke concrete `TwilioClient` — Dependency Inversion. Kal Veevo, Jazz, ya MockSender for tests — `Program.cs` mein swap, zero service change."*
- *"Multi-provider fallback: Twilio primary, Veevo secondary, Jazz tertiary. Circuit breaker per provider — failing provider ko 60s cooldown."*
- *"Per-phone OTP record via `MERGE` upsert — atomic, race-safe. Naya OTP request purana overwrite kar deta hai automatically."*

---

## ⚠️ Red Flags — Don't Say These In Interview

- ❌ "OTP plaintext store kar denge convenience ke liye" → DB leak disaster
- ❌ "`.equals()` se OTP compare kar lenge, kya difference parta hai" → timing attack vulnerable
- ❌ "Twilio directly use karenge, abstraction over-engineering hai" → vendor lock-in, untestable
- ❌ "Rate limiting baad mein add karenge, MVP mein nahi" → SMS bombing financial attack
- ❌ "Attempts unlimited rakho, UX friction nahi" → brute force trivial (1M combos in hours)
- ❌ "Same phone se 2 users register nahi kar sakte ever" → business decision, depends on context (family phones legit)
- ❌ "`Math.random() % 10`" for OTP digits → predictable PRNG, breaks security

---

## 🧠 Memory Hooks Recap

- **OTP-SECURE** — OTP hashed, Throttled, Providers chained, Single-use, Expiry, Constant-time, Upsert atomic, Redis limited, Errors clear
- **"INTERFACE PE BHAROSA, IMPLEMENTATION PE NAHI"** — DIP
- **"SOCKET PATTERN"** — DIP (kettle ↔ wall socket ↔ any provider)
- **"ATM ke 3 attempts"** — Attempt throttling
- **"Jaisa register milta tha bank mein"** — Per-phone upsert
- **"Lottery ticket ka secret box"** — Hashed OTP storage

---

## 🔗 Cross-References

- For **OOP deep**: see [cheatsheets/oop-principles.md](../cheatsheets/oop-principles.md) (DIP section)
- For **design patterns**: see [cheatsheets/design-patterns.md](../cheatsheets/design-patterns.md) (Strategy, Circuit Breaker)
- For **all memory hooks**: see [cheatsheets/memory-hooks.md](../cheatsheets/memory-hooks.md)
- **Previous lesson**: [Day 14 — Email Verification](./day-014-email-verification-flow.md) (ISP — chote interfaces)
- **Next lesson**: Day 16 — Basic Admin Panel (DRY — Don't Repeat Yourself)

**Self-test**: Take the [Day 15 quiz](../quizzes/day-015-phone-otp-verification.md) — 50 MCQs to verify mastery.
