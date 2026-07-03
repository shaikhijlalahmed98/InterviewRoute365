# 🔑 Day 3 — User Profile Update: Revision Keys

**📖 Full lesson**: [day-003-user-profile-update.md](../lessons/day-003-user-profile-update.md)
**⏱️ Reading time**: 5 minutes
**🎯 Use when**: Pre-interview revision

---

## ⚡ If You Remember Only 7 Things

1. **PCAVR** = PATCH semantics + Concurrency token + Audit trail + Validation + Retry on conflict
2. **PATCH vs PUT**: PUT = replace entire resource (need full payload). PATCH = partial update (send only changed fields).
3. **Concurrent updates** → last-write-wins WIPES other user's changes. Fix: optimistic locking with `@Version`.
4. **`@Version`** field auto-managed by Hibernate — increments per save, WHERE clause check, mismatch → `OptimisticLockException`.
5. **DTO ≠ Entity** — DTO for API payload (only allowed fields), Entity for DB. Prevents mass-assignment vulnerabilities.
6. **BaseAuditableEntity** = abstract parent class with `createdAt`, `updatedAt`, `createdBy`, `version` fields. Inherited by all entities.
7. **Sensitive field changes** need extra verification: email change → OTP, password change → old password.

---

## ☕ Java / Spring Quick Keys

- `@PatchMapping("/users/{id}")` vs `@PutMapping`
- Partial update DTO: only updateable fields, nullable so null = "don't change"
- `@Version private Long version;` on entity for optimistic locking
- Catch `OptimisticLockingFailureException` → return 409 Conflict to client
- Validate DTO with `@Valid` + Bean Validation (`@Email`, `@NotBlank`, custom)
- Don't directly accept Entity from request body (mass assignment risk)

---

## 🌐 .NET / C# Quick Keys

- `[HttpPatch("users/{id}")]` for PATCH endpoint
- `JsonPatchDocument<T>` for JSON Patch standard (RFC 6902)
- `[Timestamp]` attribute for optimistic concurrency (RowVersion column)
- EF Core Change Tracker only updates dirty (modified) columns
- `DbUpdateConcurrencyException` for version mismatches
- DataAnnotations or FluentValidation for DTO validation

---

## 🗄️ SQL Quick Keys

- Optimistic locking column: `version INT NOT NULL DEFAULT 1` or `rowversion ROWVERSION` (SQL Server)
- Audit columns: `created_at DATETIME2`, `updated_at DATETIME2`, `created_by BIGINT`, `version INT`
- Trigger or app code to update `updated_at` on every UPDATE
- Soft delete: `is_deleted BIT DEFAULT 0` + `deleted_at DATETIME2 NULL`
- Filtered index on `WHERE is_deleted = 0` for active records only

---

## 🎨 Angular Quick Keys

- Reactive Form with `patchValue()` to load existing user data
- `dirty` flag tracking — only send modified fields in PATCH request
- Optimistic UI update: change UI immediately, rollback on server 409
- Conflict resolution UX: show "Someone else modified this — reload?" dialog
- Version field included in PATCH payload for server-side check

---

## 🏗️ System Design Quick Keys

- **Last-write-wins** (naive) = data loss on concurrent updates
- **Version-aware updates** = optimistic locking, client must send original version
- **CRDT** (Conflict-free Replicated Data Type) = for distributed collaborative editing (advanced, Day 280+)
- **Event sourcing** = store events not state, replay to derive state (advanced)

---

## 🏛️ OOP — Inheritance Quick Keys

- **Inheritance** = child class inherits properties + methods from parent. "IS-A" relationship.
- **Memory hook**: "KHANDANI WIRASAT" — baap ke gun (fields/methods) bachhon ko milte hain
- Example: `BaseAuditableEntity` (parent) → `User`, `Order`, `Product` (children) all get audit fields free
- Java: `class Dog extends Animal`
- C#: `class Dog : Animal`
- **⚠️ Liskov rule**: child must honor parent's contract. Don't throw stricter exceptions, don't return narrower types.
- **⚠️ Prefer composition over inheritance** when relationship isn't true IS-A. Deep hierarchies = maintenance nightmare.

---

## 🎤 Interview One-Liners

- *"For profile updates I use PATCH with partial DTO + `@Version` optimistic locking. Concurrent updates detected via version mismatch."*
- *"All entities extend `BaseAuditableEntity` for audit columns — DRY principle via inheritance."*
- *"DTO ≠ Entity to prevent mass-assignment vulnerabilities. Only allowed fields in request payload."*
- *"Sensitive field changes (email, password) need extra verification — OTP or old password — even for authenticated users."*

---

## ⚠️ Red Flags

- ❌ "PUT mein partial fields bhejo, baaki null kar do" → wipes existing data
- ❌ "Last write wins is fine" → silent data loss on concurrent edits
- ❌ "Request body directly Entity mein bind kar lo" → mass assignment vulnerability (user sets `isAdmin=true`)
- ❌ "Email change directly allow karo" → account takeover risk

---

**Self-test**: [Day 3 Quiz](../quizzes/day-003-user-profile-update.md)
