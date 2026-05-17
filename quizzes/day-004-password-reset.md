# 📝 Day 4 Quiz — Password Reset Flow

**📖 Lesson**: [day-004-password-reset.md](../lessons/day-004-password-reset.md)
**🔑 Revision Keys**: [revision/day-004-password-reset.md](../revision/day-004-password-reset.md)
**⏱️ Suggested time**: 30-40 minutes
**📊 Total questions**: 50

---

## 📋 Instructions

1. Fill `**Your Answer**: <letter>` for each question
2. Don't peek at the answer key at the bottom until done
3. When complete, ask mentor: *"Day 4 quiz evaluate karo"*
4. Mentor will score + diagnose weak areas

---

## 🔧 Section A: Java / Spring Boot (10 MCQs)

### Q1. CSPRNG ka full form kya hai?
A) Common Standard PRNG
B) Cryptographically Secure Pseudo-Random Number Generator
C) Computer-Safe PRNG
D) Cyclic Secure PRNG

**Your Answer**: ___

### Q2. Java's `Random` class is NOT secure for tokens because it uses:
A) MD5 hashing
B) System time as only seed
C) Linear Congruential Generator (predictable from past outputs)
D) Random read from /dev/random

**Your Answer**: ___

### Q3. `/dev/urandom` provides random bytes from:
A) Hardcoded mathematical formula in kernel
B) Network packet contents
C) OS entropy pool (keyboard timings, mouse moves, disk latency, interrupts)
D) Internet-based randomness service

**Your Answer**: ___

### Q4. SHA-256 ki "deterministic" property ka matlab:
A) Output is always increasing
B) Same input always produces same hash
C) Output is sorted alphabetically
D) Hash includes timestamp

**Your Answer**: ___

### Q5. Tokens hash karke DB mein store karne ka main reason:
A) Faster lookup
B) Smaller storage size
C) DB breach pe raw tokens leak nahi hote (one-way property)
D) Required by SQL standard

**Your Answer**: ___

### Q6. Passwords ke liye SHA-256 ki bajaye BCrypt prefer karte hain kyunki:
A) BCrypt automatically encrypts data
B) BCrypt deliberately slow + has built-in salt + cost factor
C) SHA-256 is deprecated
D) BCrypt is shorter

**Your Answer**: ___

### Q7. "Salt" ka kaam password hashing mein:
A) Encrypt password before hashing
B) Random bytes added before hashing — same password → different hashes per user, defeats rainbow tables
C) Compress the password
D) Identify user identity

**Your Answer**: ___

### Q8. `@Transactional` annotation actually kaam karta hai through:
A) JVM bytecode modification
B) Spring runtime-generated dynamic proxy class wrapping your class
C) Database trigger
D) Magic global variable

**Your Answer**: ___

### Q9. Self-invocation bug ka matlab:
A) Method calling itself recursively
B) `@Transactional` method calling another `@Transactional` method in SAME class — proxy bypassed, no transaction
C) Service injecting itself
D) Constructor calling itself

**Your Answer**: ___

### Q10. `@Version` field optimistic locking enforce kaise karta hai:
A) Database-level mutex
B) Application-level semaphore
C) Hibernate auto-adds `WHERE version=N` to UPDATE; mismatch → 0 rows → exception
D) Polling every 100ms

**Your Answer**: ___

---

## 🌐 Section B: .NET / C# (8 MCQs)

### Q11. .NET ka `SecureRandom` equivalent:
A) `Random.Shared`
B) `RandomNumberGenerator.Fill()`
C) `Guid.NewGuid()`
D) `DateTime.Now.Ticks`

**Your Answer**: ___

### Q12. Windows pe `RandomNumberGenerator` internally uses:
A) /dev/urandom
B) CNG (Cryptography Next Generation)
C) Hardware TRNG only
D) Network-based service

**Your Answer**: ___

### Q13. .NET default `IPasswordHasher<TUser>` algorithm:
A) MD5
B) SHA-256
C) PBKDF2
D) Plain text

**Your Answer**: ___

### Q14. EF Core Change Tracker ka kaam:
A) Logs every query to disk
B) Compares loaded entity's snapshot with current state, generates UPDATE only for modified columns
C) Tracks user behavior for analytics
D) Backs up database every hour

**Your Answer**: ___

### Q15. `[Timestamp]` attribute SQL Server mein konsa column type create karta hai:
A) DATETIME2
B) ROWVERSION (8-byte auto-incrementing counter)
C) BIGINT IDENTITY
D) UNIQUEIDENTIFIER

**Your Answer**: ___

### Q16. `DbUpdateConcurrencyException` kab throw hota hai:
A) Network failure during save
B) When rowversion in WHERE clause mismatches (concurrent update detected)
C) When transaction times out
D) When DbContext is disposed

**Your Answer**: ___

### Q17. .NET 8+ ka "keyed services" feature ka benefit:
A) Encrypts service registrations
B) `AddKeyedScoped<I, T>(key)` + `GetRequiredKeyedService<I>(key)` to register/resolve specific impl by key
C) Auto-generates DI graph diagrams
D) Reduces memory footprint

**Your Answer**: ___

### Q18. `ExecuteUpdateAsync` (EF Core 7+) over traditional UPDATE-then-Save:
A) Stronger transactions
B) Bulk UPDATE without loading entities into memory — much faster for batch operations
C) Required for async operations
D) Better security

**Your Answer**: ___

---

## 🗄️ Section C: SQL (12 MCQs)

### Q19. ACID ka "A" = ?
A) Asynchronous
B) Atomicity (all operations succeed or all rollback)
C) Acceleration
D) Authentication

**Your Answer**: ___

### Q20. ACID ka "I" = ?
A) Integration
B) Inheritance
C) Isolation (concurrent transactions don't see each other's uncommitted changes)
D) Indexing

**Your Answer**: ___

### Q21. Write-Ahead Log (WAL) ka purpose:
A) Performance monitoring
B) Sequential log file written BEFORE data file changes — enables crash recovery (durability guarantee)
C) Query result caching
D) Replication to read replicas

**Your Answer**: ___

### Q22. B-tree index pe 1 million rows mein lookup ka time complexity:
A) O(n)
B) O(1)
C) O(log n) — typically ~20 steps for 1M
D) O(n²)

**Your Answer**: ___

### Q23. Composite index `INDEX(country, city, age)` — konsi query index use karegi:
A) WHERE age = 30
B) WHERE city = 'Lahore'
C) WHERE country = 'Pakistan' AND city = 'Lahore'
D) WHERE age = 30 AND city = 'Lahore'

**Your Answer**: ___

### Q24. SQL Server / Oracle / Postgres ka default isolation level:
A) READ UNCOMMITTED
B) READ COMMITTED
C) REPEATABLE READ
D) SERIALIZABLE

**Your Answer**: ___

### Q25. "Dirty read" kis isolation level mein possible hai:
A) READ UNCOMMITTED only
B) READ COMMITTED
C) REPEATABLE READ
D) SERIALIZABLE

**Your Answer**: ___

### Q26. Multiple transactions same row pe ek saath konsa lock le sakte hain:
A) Exclusive (X) Lock
B) Update (U) Lock
C) Shared (S) Lock
D) None — locks are always single

**Your Answer**: ___

### Q27. `UPDATE WHERE` clause execute karte time DB engine pehle konsa lock leta hai:
A) S-Lock then X-Lock
B) U-Lock then converts to X-Lock if update happens
C) X-Lock directly
D) No lock needed

**Your Answer**: ___

### Q28. Atomic UPDATE pattern `UPDATE tokens SET used=1 WHERE used=0` TOCTOU bug se kaise bachata hai:
A) DB engine uses async retry
B) Single statement = single atomic row lock; only one concurrent request matches WHERE, others get rowcount=0
C) Triggers prevent it
D) Indexes are atomic

**Your Answer**: ___

### Q29. TOCTOU bug ka matlab:
A) Type Overflow Causing Tight Output Underflow
B) Time Of Check, Time Of Use — race condition where state changes between check and use
C) Transaction On Connection To Object Update
D) Tablespace Overload Caused Transaction Outage

**Your Answer**: ___

### Q30. Optimistic locking better choice hai jab:
A) High contention with many conflicting updates
B) Low contention — conflicts rare, want high throughput, no locking overhead
C) Banking transactions
D) Always — never use pessimistic

**Your Answer**: ___

---

## 🎨 Section D: Angular / RxJS (7 MCQs)

### Q31. Observable vs Promise ka main difference:
A) Observable is faster
B) Promise = single resolution + eager; Observable = stream of multiple values + lazy + cancellable
C) Observable requires TypeScript
D) Promise can only return strings

**Your Answer**: ___

### Q32. "Cold Observable" ka matlab:
A) Observable rate-limited hai
B) Subscribe karne tak kuch nahi hota; har subscribe alag execution trigger karta hai
C) Observable cached hai memory mein
D) Observable failed state mein hai

**Your Answer**: ___

### Q33. `debounceTime(300)` operator search box pe lagao to kya hota hai:
A) Sirf 300 milliseconds tak wait karta hai phir cancel
B) Last keystroke ke 300ms baad agar no new input → final value emit (intermediate cancel)
C) Sab requests 300ms slow ho jati hain
D) UI freeze ho jata hai

**Your Answer**: ___

### Q34. Zone.js Angular ke change detection mein kya kaam karti hai:
A) Component templates compile karti hai
B) JS async APIs (setTimeout, fetch, addEventListener) ko monkey-patch karke Angular ko notify karti hai "kuch async hua, check karo"
C) HTTP requests intercept karti hai
D) Form validation handle karti hai

**Your Answer**: ___

### Q35. "Monkey-patching" ka matlab software mein:
A) Code reviewing
B) Standard functions ko wrapped versions se replace karna runtime pe
C) Bug fixes ka informal naam
D) Code obfuscation technique

**Your Answer**: ___

### Q36. `async` pipe template mein use karne ka biggest benefit:
A) Faster rendering
B) Auto-subscribe + auto-unsubscribe on component destroy — memory leak prevention
C) Type safety
D) Better SEO

**Your Answer**: ___

### Q37. Reactive Forms vs Template-Driven Forms — Reactive better hai jab:
A) Simple 1-2 field forms
B) Complex validation, dynamic forms, custom validators, observables-driven logic
C) Static read-only displays
D) Forms with no validation

**Your Answer**: ___

---

## 🏗️ Section E: System Design (6 MCQs)

### Q38. Kafka mein "topic" ka matlab:
A) Database table
B) Category/channel jisme messages publish + subscribe hote hain
C) Configuration file
D) Network port

**Your Answer**: ___

### Q39. Producer-Consumer pattern mein producer aur consumer:
A) Same code mein run karte hain
B) Decoupled — producer publishes to broker, consumer subscribes independently. No direct connection.
C) Always run on same server
D) Share memory directly

**Your Answer**: ___

### Q40. Dead Letter Queue (DLQ) ka purpose:
A) Failed messages permanent delete
B) After N retry attempts exhausted, failed messages move to DLQ for ops review — prevents infinite retry + silent data loss
C) Storing old messages forever
D) Encrypted message queue

**Your Answer**: ___

### Q41. Token Bucket rate limiting algorithm mein:
A) Tokens hash karke compare hote hain
B) Bucket holds N tokens, refills R/sec, each request consumes 1, empty bucket = reject. Allows bursts + sustains long-term limit.
C) Each user gets one token forever
D) Bucket size = number of users

**Your Answer**: ___

### Q42. Async architecture (Kafka + worker) over sync (direct SMTP) ka main benefit:
A) Cheaper hardware
B) Fast API response (50ms vs 2-5s) + retryable + scalable + resilient to SMTP downtime
C) Better security
D) Easier debugging

**Your Answer**: ___

### Q43. Strategy Pattern as architecture ka core idea:
A) Most complex algorithms always win
B) Runtime-interchangeable algorithms via interface + multiple implementations — polymorphism + intent
C) Sequential strategies executed one-by-one
D) Centralized decision-making

**Your Answer**: ___

---

## 🏛️ Section F: OOP — Polymorphism (7 MCQs)

### Q44. Polymorphism ka literal Greek meaning:
A) "One shape"
B) "Many shapes"
C) "Changeable form"
D) "Same identity"

**Your Answer**: ___

### Q45. Method Overloading vs Overriding — decision kab hota hai:
A) Both compile time
B) Overloading = compile time (compiler picks by arg types) | Overriding = runtime (JVM/CLR picks by actual object type)
C) Both runtime
D) Decided by interface

**Your Answer**: ___

### Q46. Vtable ka structure:
A) HashMap of method names to code
B) Each class has array of function pointers; object holds class pointer; method call follows pointers to actual code
C) Database table for methods
D) JSON config for class hierarchy

**Your Answer**: ___

### Q47. JIT compiler ka "devirtualization" optimization:
A) Converts all methods to static
B) Agar runtime mein sirf ek implementation always chal raha (monomorphic), JIT skip karta hai vtable lookup → direct call (perf boost)
C) Removes virtual keyword
D) Deletes unused methods

**Your Answer**: ___

### Q48. Java vs C# — `override` keyword:
A) Both languages: optional
B) Java: optional (`@Override` annotation recommended but not required) | C#: **mandatory** `override` keyword
C) Both: mandatory
D) Only Java has it

**Your Answer**: ___

### Q49. Aaj ke code mein `NotificationSender` interface + Email/SMS/Push implementations + DI = konsa principle achieve hota hai:
A) Liskov Substitution
B) Open/Closed Principle (open for extension via new classes, closed for modification of existing service)
C) Single Responsibility
D) Don't Repeat Yourself

**Your Answer**: ___

### Q50. Common mistake: constructor mein overridable method call karna kyun bug hai:
A) Compiler error
B) Parent constructor calls method → JVM uses child's vtable → child's override runs → BUT child's fields not yet initialized → NullPointerException
C) Memory leak
D) Stack overflow

**Your Answer**: ___

---

## 🔒 Answer Key (SCROLL ONLY WHEN DONE!)

<details>
<summary>⚠️ Click to reveal answers — DO NOT peek before completing</summary>

### Java / Spring Boot
1. **B** — Cryptographically Secure Pseudo-Random Number Generator
2. **C** — Linear Congruential Generator
3. **C** — OS entropy pool
4. **B** — Same input always produces same hash
5. **C** — One-way property protects against DB breach
6. **B** — Slow + auto-salt + cost factor
7. **B** — Random bytes added before hashing
8. **B** — Runtime-generated dynamic proxy class
9. **B** — Same-class call bypasses proxy
10. **C** — Auto WHERE version=N

### .NET / C#
11. **B** — RandomNumberGenerator.Fill()
12. **B** — CNG
13. **C** — PBKDF2
14. **B** — Snapshot diff for UPDATE generation
15. **B** — ROWVERSION
16. **B** — Rowversion mismatch
17. **B** — Keyed services
18. **B** — Bulk UPDATE without loading entities

### SQL
19. **B** — Atomicity
20. **C** — Isolation
21. **B** — Log before data file, crash recovery
22. **C** — O(log n)
23. **C** — WHERE country = ? (leftmost prefix used)
24. **B** — READ COMMITTED
25. **A** — READ UNCOMMITTED only
26. **C** — Shared (S) Locks
27. **B** — U-Lock → X-Lock conversion
28. **B** — Single statement atomic row lock
29. **B** — Time Of Check, Time Of Use
30. **B** — Low contention scenarios

### Angular / RxJS
31. **B** — Promise = single+eager, Observable = stream+lazy+cancellable
32. **B** — Subscribe triggers execution; each subscribe = separate run
33. **B** — Last keystroke + 300ms idle = final emit
34. **B** — Monkey-patches async APIs to notify Angular
35. **B** — Replacing standard functions with wrapped versions
36. **B** — Auto subscribe/unsubscribe = no memory leak
37. **B** — Complex validation + dynamic + observables

### System Design
38. **B** — Category/channel for pub-sub
39. **B** — Decoupled via broker
40. **B** — Failed messages → ops review after retries
41. **B** — Bucket + refill rate, burst-friendly
42. **B** — Fast response + retryable + scalable
43. **B** — Runtime-interchangeable algorithms

### OOP — Polymorphism
44. **B** — "Many shapes"
45. **B** — Compile vs runtime decision
46. **B** — Array of function pointers + class pointer
47. **B** — Monomorphic call site optimization
48. **B** — Java optional, C# mandatory
49. **B** — Open/Closed Principle
50. **B** — Child fields not yet initialized → NPE

</details>

---

## 📤 Submit for Evaluation

When done, send mentor:
```
Day 4 quiz evaluate karo. Answers:
1.B 2.C 3.A 4.B 5.D ... (all 50)
```

Mentor will respond with:
- ✅ Correct: X/50
- ❌ Wrong question numbers + correct answers + brief explanation
- 🎯 Weak area diagnosis
- 📚 Recommended re-read sections
