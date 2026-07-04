# 📝 Day 1 Quiz — User Registration + Email Verification

**📖 Lesson**: [day-001-user-registration.md](../lessons/day-001-user-registration.md)
**🔑 Revision Keys**: [revision/day-001-user-registration.md](../revision/day-001-user-registration.md)
**⏱️ Suggested time**: 25-30 minutes
**📊 Total questions**: 30

---

## 📋 Instructions

Fill `**Your Answer**: <letter>` for each question. When done, ask mentor: *"Day 1 quiz evaluate karo"*.

---

## 🔧 Section A: Java / Spring (8 MCQs)

### Q1. `@Transactional` annotation kaam karta hai through:
A) JVM bytecode rewriting
B) Spring runtime-generated proxy class
C) Database trigger
D) Annotation processor

**Your Answer**: ___

### Q2. `BCryptPasswordEncoder` ka default cost factor:
A) 5
B) 10
C) 15
D) 20

**Your Answer**: ___

### Q3. Same password ko BCrypt 2 baar hash karne pe:
A) Same hash milta hai (deterministic)
B) Different hash milta hai (random salt embedded)
C) Compile error
D) Exception throw karta hai

**Your Answer**: ___

### Q4. `UUID.randomUUID()` ka size:
A) 64-bit
B) 128-bit (122-bit random)
C) 256-bit
D) 512-bit

**Your Answer**: ___

### Q5. Self-invocation bug `@Transactional` mein:
A) Recursive call exception
B) Same class ke andar `@Transactional` method call karne se proxy bypass = no transaction
C) Deadlock
D) Memory leak

**Your Answer**: ___

### Q6. Email send fail ho `@Transactional` method ke andar to:
A) Transaction continues, user saved
B) Automatic rollback (user save also reverted)
C) Email retried automatically
D) Server crashes

**Your Answer**: ___

### Q7. `JavaMailSender` kis protocol use karta hai:
A) HTTP
B) FTP
C) SMTP
D) WebSocket

**Your Answer**: ___

### Q8. Spring's DI container responsibility:
A) Compile Java code
B) Manage object lifecycle + automatic wiring of dependencies
C) Database connection
D) HTTP routing

**Your Answer**: ___

---

## 🌐 Section B: .NET / C# (5 MCQs)

### Q9. .NET mein transaction explicit start:
A) `@Transactional` attribute
B) `_context.Database.BeginTransactionAsync()`
C) `using (var tx)` only
D) Automatic always

**Your Answer**: ___

### Q10. .NET default password hasher algorithm:
A) MD5
B) SHA-256
C) PBKDF2
D) BCrypt

**Your Answer**: ___

### Q11. `Guid.NewGuid()` ka Java equivalent:
A) `UUID.fromString()`
B) `UUID.randomUUID()`
C) `Random.nextLong()`
D) `Math.random()`

**Your Answer**: ___

### Q12. EF Core Change Tracker tracks:
A) User clicks on UI
B) Network requests
C) Entity state changes (Added/Modified/Unchanged) for efficient UPDATE generation
D) Application errors

**Your Answer**: ___

### Q13. `try/catch` mein `await transaction.RollbackAsync()` kab call karte hain:
A) Always
B) Exception aaye toh
C) Never
D) Only on warnings

**Your Answer**: ___

---

## 🗄️ Section C: SQL (7 MCQs)

### Q14. `UNIQUE` constraint automatically banata hai:
A) Foreign key
B) B-tree index
C) Trigger
D) View

**Your Answer**: ___

### Q15. Filtered index ka benefit:
A) Compresses data
B) Sirf matching rows index karta hai (space efficient + fast for that subset)
C) Encrypts column
D) Auto-deletes old rows

**Your Answer**: ___

### Q16. `@@ROWCOUNT = 0` after UPDATE matlab:
A) Database connection lost
B) WHERE clause matched 0 rows — no update happened
C) Query syntax error
D) Transaction rolled back

**Your Answer**: ___

### Q17. `DATETIME2` over `DATETIME` ka benefit:
A) Smaller storage
B) Better precision + larger date range
C) Automatic timezone conversion
D) Faster queries

**Your Answer**: ___

### Q18. Application-level duplicate check vs DB-level UNIQUE constraint — race-condition-safe:
A) Application-level (faster)
B) Both same
C) DB-level UNIQUE constraint (atomic enforcement)
D) Neither

**Your Answer**: ___

### Q19. `GETUTCDATE()` returns:
A) Local server time
B) UTC current time
C) User's timezone
D) Database creation time

**Your Answer**: ___

### Q20. Filtered index syntax:
A) `CREATE INDEX ... WHERE condition`
B) `CREATE INDEX ... FILTER (condition)`
C) `CREATE FILTERED INDEX ...`
D) `CREATE INDEX condition`

**Your Answer**: ___

---

## 🎨 Section D: Angular (5 MCQs)

### Q21. Reactive Form vs Template-Driven Form — Reactive better hai when:
A) Simple 1-field forms
B) Complex validation + dynamic forms + custom validators
C) Read-only data display
D) No forms at all

**Your Answer**: ___

### Q22. `finalize` operator kab chalta hai:
A) Only on success
B) Only on error
C) Always — success OR error (perfect for cleanup like loading=false)
D) Never

**Your Answer**: ___

### Q23. Custom validator function return karta hai:
A) `boolean` (true/false)
B) `null` (valid) or `{ errorKey: true }` (invalid)
C) `string` error message
D) `Promise<boolean>`

**Your Answer**: ___

### Q24. `isSubmitting = true` button pe lagane ka purpose:
A) Form styling
B) Disable button to prevent duplicate requests during submission
C) Auto-clear form
D) Validation trigger

**Your Answer**: ___

### Q25. `HttpClient.post()` returns:
A) Promise
B) Cold Observable (request fires on `.subscribe()`)
C) Synchronous response
D) Callback

**Your Answer**: ___

---

## 🏗️🏛️ Section E: System Design + OOP (5 MCQs)

### Q26. Transactional Outbox Pattern ka core idea:
A) Encrypt email content
B) Save user + outbox row in SAME transaction; separate worker polls outbox + sends + marks done = guaranteed delivery
C) Outsource email to third party
D) Skip transactions for speed

**Your Answer**: ___

### Q27. Sync email send ka downside:
A) Cheap
B) Easy to implement
C) Slow API response (SMTP wait) + SMTP failure rolls back user save
D) Better security

**Your Answer**: ___

### Q28. Encapsulation OOP principle:
A) Inherit from parent class
B) Bundle data + methods, hide internal state behind controlled public interface
C) Same method, different behavior
D) Hide complexity

**Your Answer**: ___

### Q29. Encapsulation ka Roman Urdu memory hook:
A) Bartan Badlo
B) DABBA = Data Access By Backed Access (ATM machine)
C) Naqsha aur Naqshanavees
D) Khandani Wirasat

**Your Answer**: ___

### Q30. Bad encapsulation example:
A) `private String password; public void setPassword(String) { ... validates + hashes }`
B) `public String password;` (anyone sets anything, no validation)
C) Lombok `@Getter @Setter`
D) Auto-properties in C#

**Your Answer**: ___

---

## 🔒 Answer Key (SCROLL ONLY WHEN DONE!)

<details>
<summary>⚠️ Click to reveal answers — DO NOT peek before completing</summary>

1. **B** — Spring runtime-generated proxy
2. **B** — 10 (2^10 = 1024 iterations)
3. **B** — Different hash (random salt embedded)
4. **B** — 128-bit (122 random + 6 reserved)
5. **B** — Same-class call bypasses proxy
6. **B** — Automatic rollback
7. **C** — SMTP
8. **B** — Manage object lifecycle + wiring
9. **B** — `BeginTransactionAsync()`
10. **C** — PBKDF2
11. **B** — `UUID.randomUUID()`
12. **C** — Entity state changes for efficient UPDATE
13. **B** — On exception
14. **B** — B-tree index
15. **B** — Subset index, space efficient
16. **B** — WHERE matched 0 rows
17. **B** — Better precision + larger range
18. **C** — DB-level UNIQUE (atomic)
19. **B** — UTC current time
20. **A** — `CREATE INDEX ... WHERE condition`
21. **B** — Complex validation + dynamic
22. **C** — Always (success or error)
23. **B** — null or error object
24. **B** — Prevent duplicate submissions
25. **B** — Cold Observable
26. **B** — Same-txn save + outbox + worker
27. **C** — Slow + cascading failure
28. **B** — Bundle data + methods, hide state
29. **B** — DABBA (ATM)
30. **B** — Public fields, no validation

</details>
