# 📚 Stack Coverage Index — Per Day, Per Stack

**Purpose**: Har din ke har stack mein kya kya cover hua, ek matrix mein dikhao. Quick cross-reference ke liye.

**Update cadence**: Mentor automatically updates this file every day after pushing the new lesson.

---

## 📊 Daily Stack Matrix

| Day | Topic | Java/Spring | .NET/C# | SQL | Angular | System Design | OOP/Pattern |
|-----|-------|-------------|---------|-----|---------|---------------|-------------|
| [1](./01-beginner/2026-05-13-day1-user-registration.md) | User Registration + Email Verify | `@Transactional` (AOP proxy), `BCryptPasswordEncoder`, `UUID.randomUUID()`, `JavaMailSender` | `BeginTransactionAsync`, `IPasswordHasher<T>`, `Guid.NewGuid()`, EF Core change tracker | `UNIQUE` constraint, filtered index on token, UTC `DATETIME2`, `@@ROWCOUNT` check | Reactive Forms, custom validators, `catchError`, `finalize` | Sync email vs Async queue, **Transactional Outbox Pattern** | **Encapsulation** (DABBA) |
| [2](./01-beginner/2026-05-14-day2-user-login-jwt.md) | User Login with JWT | Spring Security `AuthenticationManager`, `BCrypt.matches`, jjwt HS256, `OncePerRequestFilter`, `SecurityContextHolder` (ThreadLocal) | `JwtBearer` middleware, `JwtBearerHandler`, `ClaimsPrincipal`, `HttpContext.User`, BCrypt.Net-Next | `UNIQUE` = auto B-tree index, Index Seek vs Table Scan, READ COMMITTED for pure reads | `HttpClient` (cold Observable), `localStorage`, **`HttpInterceptor`** for auto-token | **Stateless JWT vs Stateful session**, short-lived access + refresh token | **Abstraction** (Naqsha aur Naqshanavees) |
| [3](./01-beginner/2026-05-15-day3-user-profile-update.md) | User Profile Update | PATCH vs PUT semantics, partial update DTO, `@Version` optimistic locking, audit fields | EF Core change tracker dirty-flag, `[ConcurrencyCheck]`, `JsonPatchDocument` | Optimistic locking with version column, `BaseAuditableEntity` pattern, `created_at`/`updated_at`/`version` | Form state mgmt, dirty checking, conflict resolution UX | Last-write-wins vs version-aware updates | **Inheritance** (Khandani Wirasat, BaseAuditableEntity) |
| [4](./01-beginner/2026-05-16-day4-password-reset.md) | Password Reset Flow | `SecureRandom` (CSPRNG), SHA-256 via `MessageDigest`, BCrypt salt+cost, **Spring AOP proxy** (`@Transactional` mechanism), DI with `List<Interface>`, polymorphic dispatch | `RandomNumberGenerator` (CNG), `IPasswordHasher` (PBKDF2), `[Timestamp]` ROWVERSION, **keyed services DI** (.NET 8+), `ExecuteUpdateAsync` (EF 7+) | **Atomic UPDATE WHERE used=0** pattern, lock types (S/X/U), full ACID, **WAL** (Write-Ahead Log), B-tree depth, composite index **leftmost-prefix rule**, all 4 isolation levels | **Reactive programming**, Observable vs Promise, RxJS `debounceTime`/`distinctUntilChanged`, **Zone.js** monkey-patching, change detection, cold Observables, `async` pipe | Sync vs Async architecture, **Kafka** (topic/producer/consumer/partition/offset), **DLQ**, **Token Bucket** rate limit on Redis, **Strategy Pattern** as architecture | **Polymorphism — Method Overriding** (BARTAN BADLO, vtable, JIT devirtualization) |
| [5](./01-beginner/2026-05-17-day5-product-listing.md) | Basic Product Listing | Spring Data JPA `Pageable`, `Specification` API, DTO projection, `@Transactional(readOnly=true)` | EF Core `IQueryable` composition, `AsNoTracking()`, `Skip().Take()`, deferred execution via Expression Tree | OFFSET vs **Keyset (cursor) pagination**, composite covering index with `INCLUDE`, filesort avoidance, READ COMMITTED for reads | RxJS `combineLatest`, `BehaviorSubject` for filter state, **virtual scrolling** (`cdk-virtual-scroll`), `switchMap` for race-condition fix | **Read-optimized CQRS-lite**: CDN + Redis cache + DB read replicas, cache-aside pattern | **Composition over Inheritance** (BIRYANI BANAO, KHAANDAN MAT BANAO; HAS-A over IS-A) |
| [6](./01-beginner/2026-05-18-day6-product-search-pagination.md) | Product Search with Pagination | `Pageable` + `Specification` for dynamic filters, `JpaRepository` + `JpaSpecificationExecutor`, DTO projection, `findAll(spec, pageable)` | `IQueryable` chain + `Skip/Take`, `EF.Functions.Like`, `Math.Clamp` defensive sizing, projection via `Select(new Dto)` | Composite covering index `(IsActive, CreatedAt DESC, Id DESC) INCLUDE (...)`, **keyset cursor SQL** with tiebreaker, `READ COMMITTED SNAPSHOT` | RxJS `debounceTime(300)`, `distinctUntilChanged`, `switchMap` (HTTP cancel via AbortController), `OnPush` change detection, `trackBy`, `IntersectionObserver` for infinite scroll | Read-optimized search service path → Redis cache (60s TTL) → **Elasticsearch evolution** (CDC/outbox sync) | **Coupling — Loose vs Tight** (INTERFACE PE PYAAR KARO, CLASS PE NAHI; SHADI-SHUDA vs DOSTI) |

---

## 🔍 Search By Stack — "Yeh Topic Kis Din Pe Cover Hua?"

### ☕ Java / Spring Boot — Concepts Covered So Far

| Concept | Day(s) |
|---------|--------|
| `@Transactional` (AOP proxy mechanism) | Day 1, Day 4 (deep dive) |
| Spring AOP / proxy generation | Day 4 |
| Dependency Injection (DI container) | Day 1, Day 4 (deep dive with interfaces) |
| `BCryptPasswordEncoder` | Day 1, Day 2, Day 4 |
| `UUID.randomUUID()` | Day 1 |
| `SecureRandom` (CSPRNG) | Day 4 |
| `MessageDigest` / SHA-256 | Day 4 |
| Spring Security | Day 2 |
| JWT (jjwt library) | Day 2 |
| `OncePerRequestFilter` | Day 2 |
| `SecurityContextHolder` | Day 2 |
| `@Version` optimistic locking | Day 3, Day 4 |
| `JavaMailSender` | Day 1 |
| Polymorphic `List<Interface>` injection | Day 4 |

### 🌐 .NET / C# — Concepts Covered So Far

| Concept | Day(s) |
|---------|--------|
| `BeginTransactionAsync` | Day 1 |
| `IPasswordHasher<TUser>` | Day 1, Day 4 |
| BCrypt.Net-Next | Day 2 |
| `RandomNumberGenerator` (CNG) | Day 4 |
| `Guid.NewGuid()` | Day 1 |
| EF Core change tracker | Day 1, Day 3, Day 4 (deep dive) |
| `[ConcurrencyCheck]` / `[Timestamp]` | Day 3, Day 4 |
| `JsonPatchDocument` | Day 3 |
| `JwtBearer` middleware | Day 2 |
| `ClaimsPrincipal` / `HttpContext.User` | Day 2 |
| Keyed services DI (.NET 8+) | Day 4 |
| `ExecuteUpdateAsync` (EF 7+) | Day 4 |
| DI scopes (Singleton/Scoped/Transient) | Day 4 |

### 🗄️ SQL — Concepts Covered So Far

| Concept | Day(s) |
|---------|--------|
| `UNIQUE` constraint | Day 1, Day 2 |
| B-tree index | Day 1, Day 2, Day 4 (deep dive) |
| Filtered / partial index | Day 1 |
| Composite index (leftmost-prefix rule) | Day 4 |
| Index Seek vs Table Scan | Day 2, Day 4 |
| `@@ROWCOUNT` check | Day 1, Day 4 |
| Atomic UPDATE pattern (TOCTOU fix) | Day 4 |
| Optimistic locking via version | Day 3, Day 4 |
| `ROWVERSION` (SQL Server) | Day 4 |
| ACID properties (full) | Day 4 |
| Write-Ahead Log (WAL) | Day 4 |
| Isolation levels (READ UNCOMMITTED → SERIALIZABLE) | Day 4 |
| Lock types (Shared / Exclusive / Update) | Day 4 |
| UTC `DATETIME2` | Day 1 |
| `BaseAuditableEntity` pattern | Day 3 |

### 🎨 Angular — Concepts Covered So Far

| Concept | Day(s) |
|---------|--------|
| Reactive Forms + `FormGroup` | Day 1, Day 4 |
| Custom validators (sync + cross-field) | Day 1, Day 4 |
| Template-Driven vs Reactive | Day 4 |
| `HttpClient` (cold Observable) | Day 2, Day 4 |
| `HttpInterceptor` | Day 2 |
| `localStorage` | Day 2 |
| RxJS `catchError`, `finalize`, `tap` | Day 1, Day 2 |
| RxJS `debounceTime`, `distinctUntilChanged`, `map` | Day 4 |
| Observable vs Promise | Day 4 |
| Reactive programming concept | Day 4 |
| Zone.js + monkey-patching | Day 4 |
| Change detection internals | Day 4 |
| `async` pipe (auto-subscribe/unsubscribe) | Day 4 |
| `ActivatedRoute.queryParamMap` | Day 4 |

### 🏗️ System Design — Concepts Covered So Far

| Concept | Day(s) |
|---------|--------|
| Sync vs Async architecture | Day 1, Day 4 |
| Transactional Outbox Pattern | Day 1 |
| Stateless vs Stateful auth | Day 2 |
| Short-lived access + refresh token | Day 2 |
| Last-write-wins vs version-aware updates | Day 3 |
| Message broker (Kafka basics) | Day 4 |
| Topics, producers, consumers, partitions, offsets | Day 4 |
| Dead Letter Queue (DLQ) | Day 4 |
| Token Bucket rate limit algorithm | Day 4 |
| Redis for rate limiting (atomic INCR + EXPIRE) | Day 4 |
| Strategy Pattern (architecture-level) | Day 4 |
| Anti-enumeration / timing attacks | Day 4 |
| HMAC-signed tokens | Day 4 |

### 🏛️ OOP / Design Patterns — Covered So Far

| Concept | Day(s) | Memory Hook |
|---------|--------|-------------|
| Encapsulation | Day 1 | DABBA (ATM machine) |
| Abstraction | Day 2 | Naqsha aur Naqshanavees |
| Inheritance | Day 3 | Khandani Wirasat |
| Polymorphism — Method Overriding | Day 4 | BARTAN BADLO, KHAANA WOHI |
| Method Overloading vs Overriding | Day 4 | Compile-time vs Runtime |
| Vtable / virtual dispatch | Day 4 | — |
| JIT devirtualization | Day 4 | — |
| Strategy Pattern | Day 4 (preview, formal Day 62) | — |
| Open/Closed Principle (preview) | Day 4 (formal Day 12) | EXTEND HAAN, MODIFY NAA |

---

## 🎯 How To Use This Index

**Use Case 1 — Pre-Interview Quick Scan**:
"Mera interview Java/Spring pe hai" → scroll to **Java section**, see all days covered, focus revision.

**Use Case 2 — Concept Lookup**:
"vtable kya tha?" → search this file for "vtable" → find Day 4 → open lesson.

**Use Case 3 — Coverage Gap Check**:
"Kya kabhi Kafka cover hua?" → search "Kafka" → Day 4 → open.

**Use Case 4 — Daily Planning**:
"Aaj Java revision karni hai" → see all Java concepts + their days → plan revision.

---

## 📝 Conventions

- **Day cell**: links to actual lesson file
- **Stack cells**: comma-separated concepts/techniques, with `**bold**` for new-this-day deep dives
- **Search sections**: alphabetical-ish by concept name
- **Empty cells (`—`)**: means future days haven't pushed yet

**Auto-updated by mentor** every day after lesson push. Manual edits welcome (notes, corrections).
