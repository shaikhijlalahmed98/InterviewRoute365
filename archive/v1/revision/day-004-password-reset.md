# 🔑 Day 4 — Password Reset Flow: Revision Keys

**📖 Full lesson**: [day-004-password-reset.md](../lessons/day-004-password-reset.md)
**⏱️ Reading time**: 5-7 minutes
**🎯 Use when**: Night before interview, weekly revision, concept refresh

---

## ⚡ If You Remember Only 10 Things

1. **TRACE** = Token (SecureRandom, 32 bytes, hashed) + Rate-limit (3/hr) + Atomic single-use + Consistent response (no enumeration) + Expiry (15 min) + session invalidation
2. **Tokens**: raw in email, **SHA-256 hash in DB**. DB breach = tokens still safe (one-way hash).
3. **`SecureRandom` (CSPRNG)** NEVER `Random`. `Random` = predictable Linear Congruential Generator. Attacker can predict future after 3-4 outputs.
4. **SHA-256 for tokens** (fast 1μs) | **BCrypt for passwords** (slow 200ms on purpose, prevents brute force on low-entropy human passwords)
5. **Atomic UPDATE pattern**: `UPDATE tokens SET used=1 WHERE token_hash=? AND used=0` — single statement prevents TOCTOU race
6. **@Transactional = Spring AOP proxy** — runtime-generated subclass wraps method with `begin/commit/rollback`
7. **@Version optimistic locking** — auto WHERE `version=N` added to UPDATE; mismatch = `OptimisticLockException`
8. **Polymorphism** = "BARTAN BADLO, KHAANA WOHI" — runtime decides actual impl via **vtable** (array of function pointers per class)
9. **Anti-enumeration**: same response for "email exists" and "doesn't exist" — frontend AND backend AND timing
10. **Session invalidation strategies** (4): DB sessions DELETE | Refresh tokens delete | Token version increment | Redis blocklist

---

## ☕ Java / Spring Boot — Quick Keys

- **`SecureRandom`** wraps OS-level CSPRNG. Linux: `/dev/urandom` (entropy from keyboard timings, mouse moves, disk I/O, network)
- **"Wrap"** = layer over OS-specific API. SecureRandom hides "Linux vs Windows vs Mac" complexity behind simple method.
- **32 bytes = 256 bits = 2^256 = 10^77 possibilities** = brute force mathematically impossible (universe has 10^80 atoms)
- **SHA-256 properties**: deterministic | one-way (irreversible) | avalanche (1 bit change → 50% bits change)
- **BCrypt** = slow + auto-salt + cost factor (default 10 = 1024 iterations). 100k× slower than SHA-256 by design.
- **Salt** = random bytes added BEFORE hashing. Same password → different hashes per user. Defeats rainbow tables.
- **`@Transactional` mechanism**: Spring generates dynamic proxy class extending yours via CGLib. Proxy wraps method with `txManager.begin() → super.method() → commit/rollback`.
- **Self-invocation bug**: calling `@Transactional` method from SAME class bypasses proxy = no transaction. Common interview gotcha!
- **`@Version`**: Hibernate auto-adds `WHERE version=N` to UPDATE. Increments on save. 0 rows affected → `OptimisticLockException`.
- **DI with interfaces**: `List<NotificationSender>` injected = ALL implementations. Map by getChannel() for dispatch. New channel = new class, zero existing code change.

---

## 🌐 .NET / C# — Quick Keys

- **`RandomNumberGenerator.Fill()`** — equivalent to SecureRandom. Windows: CNG (Cryptography Next Generation). Linux: `/dev/urandom`.
- **CNG** replaced legacy CryptoAPI in early 2000s. Modern crypto interface for Windows.
- **`IPasswordHasher<TUser>`** default = PBKDF2 (BCrypt's cousin, same purpose: slow + salted)
- **EF Core Change Tracker**: snapshot-based. Tracks entity original state, generates UPDATE only for modified columns. `.AsNoTracking()` skips tracking for read-only queries (perf gain).
- **`[Timestamp]`** maps to SQL Server `ROWVERSION` — 8-byte counter auto-incremented per row update DB-wide. EF Core adds `WHERE rowversion=@orig` automatically.
- **`DbUpdateConcurrencyException`** thrown when rowversion mismatch (concurrent update detected)
- **`ExecuteUpdateAsync`** (EF 7+) = bulk UPDATE without loading entities. Much faster for cleanup operations.
- **Keyed Services DI** (.NET 8+): `AddKeyedScoped<I, T>(key)` + `GetRequiredKeyedService<I>(key)` — resolve specific impl by key. Cleaner than Java's `@Qualifier`.

---

## 🗄️ SQL — Quick Keys

- **Lock types**: 🔵 S-Lock (shared, read, multiple) | 🔴 X-Lock (exclusive, write, single) | 🟡 U-Lock (intermediate, prevents deadlock)
- **ACID**: **A**tomicity (all or nothing) | **C**onsistency (constraints maintained) | **I**solation (concurrent txns isolated) | **D**urability (committed = persisted)
- **WAL (Write-Ahead Log)** = sequential log file written BEFORE data file. Crash recovery replays log. Foundation of durability.
- **B-tree**: balanced tree, log(n) lookup, disk-page-sized nodes (8KB). 1M rows = ~3-4 disk reads.
- **Composite index leftmost-prefix rule**: `INDEX(a, b, c)` used only if WHERE contains `a` first. Order matters!
- **Atomic UPDATE pattern**: `UPDATE x SET y=1 WHERE id=? AND y=0`. Check rowcount. Eliminates TOCTOU race.
- **TOCTOU bug**: Time Of Check, Time Of Use. SELECT then UPDATE = race. Atomic single-statement = safe.
- **Isolation levels** (weak → strong):
  1. READ UNCOMMITTED (dirty reads possible)
  2. **READ COMMITTED** (default for SQL Server/Oracle/Postgres) — only committed data
  3. REPEATABLE READ (MySQL default) — same row stays frozen for your txn
  4. SERIALIZABLE — transactions appear sequential
- **For password reset**: READ COMMITTED enough — atomic UPDATE takes row X-Lock automatically
- **Optimistic vs Pessimistic**: Optimistic = check at commit (better for low contention). Pessimistic = lock at read (better for high contention).
- **Unique index** auto-created by UNIQUE constraint. Enforces uniqueness AND fast lookup (race-condition safe).

---

## 🎨 Angular — Quick Keys

- **Observable** = stream of future values. Multiple emissions over time. Cancellable.
- **vs Promise**: Promise = single resolution, eager (starts immediately). Observable = stream, lazy (cold — needs subscribe to start).
- **Cold Observable**: HTTP requests don't fire until `.subscribe()`. 2 subscribes = 2 separate HTTP calls.
- **Hot Observable**: shared execution. Use `shareReplay()` to convert.
- **`debounceTime(300)`**: setTimeout-based. Cancel previous on new emit. After 300ms idle → emit final value. Spam prevention.
- **`distinctUntilChanged`**: skip emit if value unchanged from last
- **Zone.js** = monkey-patches all JS async APIs (setTimeout, fetch, addEventListener). After async completes → notifies Angular → change detection runs → templates re-render. This is HOW `count++` magically updates UI.
- **Monkey-patching** = replacing standard functions with wrapped versions
- **Reactive Forms** (TypeScript-managed) vs Template-Driven (HTML-managed) — Reactive for anything complex.
- **`valueChanges` Observable**: emits on every form change. Combine with `debounceTime` + `distinctUntilChanged` + `map` for real-time features.
- **`async` pipe**: auto-subscribe + auto-unsubscribe on component destroy. **Always prefer over manual subscribe** to prevent memory leaks.
- **Anti-enumeration on frontend**: same UI message for success AND error (else DevTools = enumeration leak)

---

## 🏗️ System Design — Quick Keys

- **Sync vs Async**: Sync = wait for slow ops (SMTP 2-5s) → bad UX. Async = publish event → return fast → background worker processes.
- **Message Broker** = middleware between producer and consumer. Like post office.
- **Kafka** = distributed, durable, persisted-to-disk, replayable, multi-server cluster
- **Kafka concepts**: Topic (channel) | Producer (publisher) | Consumer (subscriber) | Partition (subdivision for parallelism) | Offset (sequential message ID)
- **Dead Letter Queue (DLQ)**: After N retries, failed messages go here. Ops team reviews. Prevents infinite retry loops and silent data loss.
- **Token Bucket rate limit**: Bucket holds N tokens, refills R/sec, each request consumes 1. Empty = reject. Allows bursts + sustains long-term limit.
- **Redis** = in-memory key-value store, microsecond latency, perfect for rate limiting counters
- **Strategy Pattern** (as architecture): runtime-interchangeable algorithms via interface + multiple implementations. Polymorphism + intent.
- **Real companies**: Stripe (HMAC tokens, 1hr), GitHub (3hr, anti-enumeration), Slack (random+SHA-256+SendGrid), AWS Cognito (6-digit OTP, attempt-limited)

---

## 🏛️ OOP — Polymorphism (Method Overriding) Quick Keys

- **Polymorphism** = "poly" (many) + "morph" (shape). Same method call, different actual behavior based on object's runtime type.
- **3 types**: Compile-time (overloading) | Runtime (overriding) | Parametric (generics). Today's focus: #2.
- **Overloading vs Overriding**:
  | Overloading | Overriding |
  |-------------|------------|
  | Same class | Parent + child |
  | Different parameters | Same signature |
  | Compile-time decided | Runtime decided |
  | No inheritance | Inheritance required |
- **Vtable** = each class's array of function pointers. Object has hidden class pointer → vtable → method address.
- **Virtual dispatch cost** = ~1-5ns per call (vtable lookup). JIT can devirtualize if monomorphic (single impl seen).
- **`@Override`** in Java = optional but RECOMMENDED (catches signature mismatch). C# `override` = MANDATORY.
- **Java methods virtual by default**; C# methods sealed unless `virtual` declared.
- **Polymorphism in Day 4 code**: `NotificationSender` interface → Email/SMS/Push implementations. Service uses interface reference. New channel = new class, zero existing change. Open/Closed Principle achieved.
- **Common mistakes**:
  1. Skip `@Override` → typo creates silent new method instead of override
  2. Call overridable method in constructor → child fields not yet initialized → NPE
  3. Override `equals()` but forget `hashCode()` → HashMap/HashSet break

---

## 🎤 Interview One-Liners (Bolne Layak Confidently)

- *"For password reset, I'd use a 32-byte token from `SecureRandom`, stored as SHA-256 hash. Raw token goes only in email. DB breach doesn't compromise tokens because hashing is one-way."*
- *"Single-use guarantee comes from atomic UPDATE — `UPDATE tokens SET used=1 WHERE used=0`. SQL engine row-locks, only one request wins, others get rowcount=0."*
- *"Same response whether email exists or not — anti-enumeration. Otherwise attackers harvest valid emails for credential stuffing."*
- *"`@Transactional` works via Spring AOP — runtime-generated proxy class wraps your method with begin/commit/rollback. Self-invocation bypasses proxy though, common gotcha."*
- *"For multi-channel notifications I use polymorphism — `NotificationSender` interface with Email/SMS/Push implementations, dispatched at runtime via DI. Adding WhatsApp = new class, zero existing code change. That's the Open/Closed Principle in action."*
- *"After password reset I invalidate all sessions — DB sessions DELETE for stateful, refresh token DELETE for stateless JWT, or token version increment to invalidate all old JWTs at once."*

---

## ⚠️ Red Flags — Don't Say These In Interview

- ❌ "Token plaintext store kar denge" → DB breach = mass account takeover
- ❌ "Email not found error dikhao" → enumeration attack
- ❌ "Token kabhi expire na ho" → screenshot exploitable months later
- ❌ "Math.random() / Random() se token banao" → predictable
- ❌ "SELECT then UPDATE in separate steps" → TOCTOU race
- ❌ "Sessions are separate concern" → incomplete security (hacker stays logged in)
- ❌ "Frontend pe alag message for UX" → DevTools enumeration

---

## 🧠 Memory Hooks Recap

- **TRACE** — Token, Rate-limit, Atomic, Consistent, Expiry
- **"BARTAN BADLO, KHAANA WOHI"** — Polymorphism
- **"Magic shredder machine"** — Hash function
- **"Pandit ka register"** — Optimistic locking version check
- **"Tum mere liye bana ke do"** — Dependency Injection
- **"Careem ride types"** — Polymorphic dispatch (Go/Plus/Bike)

---

## 🔗 Cross-References

- For **SQL deep**: see [cheatsheets/sql-locking-isolation.md](../cheatsheets/sql-locking-isolation.md)
- For **OOP deep**: see [cheatsheets/oop-principles.md](../cheatsheets/oop-principles.md)
- For **Java↔.NET**: see [cheatsheets/spring-vs-dotnet.md](../cheatsheets/spring-vs-dotnet.md)
- For **all memory hooks**: see [cheatsheets/memory-hooks.md](../cheatsheets/memory-hooks.md)

**Self-test**: Take the [Day 4 quiz](../quizzes/day-004-password-reset.md) — 50 MCQs to verify mastery.
