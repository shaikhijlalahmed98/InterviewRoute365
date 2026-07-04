# 📝 Day 3 Quiz — User Profile Update

**📖 Lesson**: [day-003-user-profile-update.md](../lessons/day-003-user-profile-update.md)
**🔑 Revision Keys**: [revision/day-003-user-profile-update.md](../revision/day-003-user-profile-update.md)
**📊 Total questions**: 30

---

## 🔧 Section A: Java / Spring (8 MCQs)

### Q1. PATCH vs PUT semantics:
A) Both same
B) PUT = replace entire resource (full payload); PATCH = partial update (only changed fields)
C) PATCH faster than PUT
D) PUT is deprecated

**Your Answer**: ___

### Q2. `@Version` field on entity:
A) Tracks user version
B) Hibernate auto-increments on save + adds `WHERE version=N` to UPDATE for optimistic locking
C) JVM version
D) Manual increment by developer

**Your Answer**: ___

### Q3. `OptimisticLockingFailureException` kab throw hota hai:
A) Database down
B) Version mismatch detected (concurrent update happened)
C) Network timeout
D) Memory full

**Your Answer**: ___

### Q4. Optimistic lock exception ka client response:
A) 500 Internal Server Error
B) 409 Conflict (let client retry with fresh data)
C) 404 Not Found
D) 200 OK

**Your Answer**: ___

### Q5. DTO vs Entity ka main reason:
A) DTO faster
B) DTO controls allowed fields in API payload, prevents mass-assignment vulnerabilities
C) Entity is for frontend
D) No real difference

**Your Answer**: ___

### Q6. `@Valid` annotation ka kaam:
A) Encrypt request
B) Trigger Bean Validation on DTO (`@Email`, `@NotBlank` etc.)
C) Authenticate user
D) Log request

**Your Answer**: ___

### Q7. Mass-assignment vulnerability example:
A) Too many users registering
B) Request body directly bound to Entity → attacker sets `isAdmin=true` field
C) Memory leak
D) Database overflow

**Your Answer**: ___

### Q8. `@PatchMapping("/users/{id}")` vs `@PutMapping`:
A) Same
B) `@PatchMapping` for partial updates, `@PutMapping` for full replace
C) Patch is internal use
D) Both deprecated

**Your Answer**: ___

---

## 🌐 Section B: .NET / C# (5 MCQs)

### Q9. .NET mein PATCH endpoint:
A) `[HttpPatch("users/{id}")]`
B) `[HttpUpdate]`
C) `[HttpModify]`
D) `[HttpChange]`

**Your Answer**: ___

### Q10. JSON Patch standard ke liye class:
A) `JsonObject`
B) `JsonPatchDocument<T>` (RFC 6902 compliant)
C) `JsonValue`
D) `JsonNode`

**Your Answer**: ___

### Q11. `[Timestamp]` attribute:
A) Records current time
B) Optimistic concurrency token using SQL Server ROWVERSION
C) Schedules a job
D) Logs creation time

**Your Answer**: ___

### Q12. EF Core Change Tracker UPDATE optimization:
A) Updates all columns always
B) Only modified columns ka UPDATE generate karta hai (dirty checking)
C) Random columns
D) Primary key only

**Your Answer**: ___

### Q13. `DbUpdateConcurrencyException`:
A) Network exception
B) Thrown when ROWVERSION mismatch in WHERE clause (concurrent update detected)
C) Memory exception
D) Timeout exception

**Your Answer**: ___

---

## 🗄️ Section C: SQL (5 MCQs)

### Q14. Audit columns standard set:
A) `id`, `name`
B) `created_at`, `updated_at`, `created_by`, `version`
C) `password`, `email`
D) `start`, `end`

**Your Answer**: ___

### Q15. Soft delete column pattern:
A) `is_deleted BIT DEFAULT 0` + `deleted_at DATETIME2 NULL`
B) `DELETE FROM` immediately
C) Move to backup table
D) Encrypt the row

**Your Answer**: ___

### Q16. Active records filtered index:
A) `WHERE is_deleted = 1`
B) `WHERE is_deleted = 0` (only index non-deleted rows)
C) `WHERE active = NULL`
D) Cannot filter

**Your Answer**: ___

### Q17. Updated_at column update strategy:
A) Manual every UPDATE
B) Trigger or application code on every UPDATE
C) Database default
D) Random time

**Your Answer**: ___

### Q18. Optimistic locking column in SQL Server:
A) `version INT` ya `rowversion ROWVERSION`
B) `lock_id GUID`
C) `mutex BIT`
D) `semaphore INT`

**Your Answer**: ___

---

## 🎨 Section D: Angular (5 MCQs)

### Q19. `patchValue()` vs `setValue()` on form:
A) Same
B) `patchValue` = partial update (some fields); `setValue` = full replace (all fields required)
C) Both deprecated
D) Both full replace

**Your Answer**: ___

### Q20. Form `dirty` flag tracks:
A) Form has bugs
B) User has modified at least one field (helps decide what to send in PATCH)
C) Form requires validation
D) Form is invalid

**Your Answer**: ___

### Q21. Optimistic UI update:
A) Wait for server response
B) Change UI immediately + rollback on server error (perceived speed)
C) Disable UI during save
D) Hide UI completely

**Your Answer**: ___

### Q22. 409 Conflict handling in UI:
A) Show generic error
B) "Someone else modified this — reload?" dialog for conflict resolution
C) Auto-retry forever
D) Logout user

**Your Answer**: ___

### Q23. Version field in PATCH payload:
A) Not needed
B) Include original version for server-side optimistic lock check
C) Random version
D) Always 1

**Your Answer**: ___

---

## 🏗️🏛️ Section E: System Design + OOP (7 MCQs)

### Q24. Last-write-wins problem:
A) Always correct
B) Concurrent updates silently overwrite each other's changes — data loss
C) Cannot happen
D) Only in NoSQL

**Your Answer**: ___

### Q25. Version-aware update fixes:
A) Performance
B) Concurrent update conflict detection — client must send original version
C) Memory issues
D) Encryption

**Your Answer**: ___

### Q26. Sensitive field changes (email, password):
A) Allow without extra checks
B) Require extra verification (OTP for email, old password for password) even if user is authenticated
C) Skip altogether
D) Always require admin approval

**Your Answer**: ___

### Q27. Inheritance OOP principle:
A) Same method, different params
B) Child class inherits properties + methods from parent ("IS-A")
C) Hide implementation
D) Multiple interfaces

**Your Answer**: ___

### Q28. Inheritance Roman Urdu memory hook:
A) DABBA
B) BARTAN BADLO
C) KHANDANI WIRASAT (baap ke gun bachhon ko)
D) AZAAN PATTERN

**Your Answer**: ___

### Q29. `BaseAuditableEntity` pattern example:
A) `class User extends BaseAuditableEntity` — User free mein createdAt, updatedAt, version fields milte hain
B) Encryption helper
C) Random utility
D) Form validator

**Your Answer**: ___

### Q30. Liskov Substitution Principle rule:
A) Use any class anywhere
B) Subclass must honor parent's contract (no stricter exceptions, no narrower types) — child should substitute parent without breaking
C) Always extend
D) Never extend

**Your Answer**: ___

---

## 🔒 Answer Key

<details>
<summary>⚠️ Click to reveal — complete first!</summary>

1. **B** — PUT full, PATCH partial
2. **B** — Auto-increment + WHERE clause
3. **B** — Version mismatch
4. **B** — 409 Conflict for retry
5. **B** — Prevents mass-assignment
6. **B** — Trigger Bean Validation
7. **B** — Body→Entity = attacker sets isAdmin
8. **B** — PATCH partial, PUT replace
9. **A** — `[HttpPatch]`
10. **B** — `JsonPatchDocument<T>`
11. **B** — Optimistic concurrency via ROWVERSION
12. **B** — Only modified columns
13. **B** — ROWVERSION mismatch
14. **B** — created_at, updated_at, created_by, version
15. **A** — is_deleted BIT + deleted_at
16. **B** — WHERE is_deleted = 0
17. **B** — Trigger or app code
18. **A** — version INT or ROWVERSION
19. **B** — patchValue partial, setValue full
20. **B** — User modified fields
21. **B** — UI immediate + rollback on error
22. **B** — Conflict resolution dialog
23. **B** — Include original version
24. **B** — Silent data loss
25. **B** — Concurrent conflict detection
26. **B** — Extra verification required
27. **B** — Inheritance IS-A
28. **C** — Khandani Wirasat
29. **A** — Free audit fields via parent class
30. **B** — Honor parent contract

</details>
