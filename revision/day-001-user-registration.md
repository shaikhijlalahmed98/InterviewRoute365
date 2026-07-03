# 🔑 Day 1 — User Registration + Email Verification: Revision Keys

**📖 Full lesson**: [day-001-user-registration.md](../lessons/day-001-user-registration.md)
**⏱️ Reading time**: 5 minutes
**🎯 Use when**: Pre-interview revision

---

## ⚡ If You Remember Only 7 Things

1. **R.U.V.E.S** = Reactive Form + Unique constraint + Verification token + Email send (async) + Secure hash
2. **`@Transactional`** = AOP proxy wraps method; save user + send email atomic. Email fail → rollback user save.
3. **`BCryptPasswordEncoder`** with cost 10 (1024 iterations) — slow on purpose to defeat brute force
4. **UUID v4** = 122-bit random = collision practically impossible for verification tokens
5. **SQL `UNIQUE` constraint** auto-creates B-tree index. Atomic enforcement at DB level (vs app-level race).
6. **Filtered index** on `verification_token WHERE NOT NULL` = space efficient + fast lookup for unverified users
7. **Transactional Outbox Pattern** (production): save user + outbox row in SAME transaction, separate worker sends email with retries

---

## ☕ Java / Spring Quick Keys

- `@Transactional` ka actual mechanism = Spring runtime proxy class extends yours
- `BCryptPasswordEncoder.encode()` = auto-generates salt + embeds in hash
- `UUID.randomUUID()` = cryptographically secure random for tokens
- `JavaMailSender` = SMTP wrapper for sending emails
- Email failure inside `@Transactional` method → automatic rollback of user save
- **Self-invocation bug**: calling `@Transactional` method from same class = proxy bypass = no transaction

---

## 🌐 .NET / C# Quick Keys

- `_context.Database.BeginTransactionAsync()` = explicit transaction (vs Spring's declarative `@Transactional`)
- `_passwordHasher.HashPassword()` = default PBKDF2 (BCrypt-equivalent)
- `Guid.NewGuid()` = .NET ka UUID equivalent
- `using var transaction = ...` + try/catch → manual `CommitAsync` / `RollbackAsync`
- EF Core Change Tracker tracks entity state internally

---

## 🗄️ SQL Quick Keys

- `UNIQUE` constraint → DB-level race-condition-safe duplicate prevention
- Filtered index: `CREATE INDEX ... WHERE verification_token IS NOT NULL` (only unverified rows indexed)
- `DATETIME2` for UTC timestamps (better than DATETIME)
- `GETUTCDATE()` for current UTC time
- `@@ROWCOUNT` check after UPDATE to verify atomicity
- Atomic verify: `UPDATE WHERE token=? AND expiry>NOW() AND verified=0` then check rowcount

---

## 🎨 Angular Quick Keys

- **Reactive Forms** with `FormBuilder`, `FormGroup`, cross-field validators
- `Validators.required`, `Validators.email`, `Validators.minLength()`
- Custom validator: function returning `null` (valid) or `{ errorKey: true }` (invalid)
- `catchError` = handle errors; `finalize` = ALWAYS run (success OR error)
- `isSubmitting` flag → disable button to prevent duplicate requests

---

## 🏗️ System Design Quick Keys

- **Sync email** (simple): SMTP wait in request → slow API
- **Async queue** (good): publish event, worker sends, retries
- **Transactional Outbox** (best): save user + outbox row atomically, worker polls outbox + sends + marks done
- Resend rate limit (e.g., 3/hour/email) to prevent email bombing
- Email enumeration prevention: identical response whether email exists or not

---

## 🏛️ OOP — Encapsulation Quick Keys

- **Encapsulation** = bundle data + methods, hide internal state behind public interface
- **Memory hook**: "DABBA" = Data Access By Backed Access (ATM machine analogy)
- Bad: `public String password;` — anyone sets anything
- Good: `setPassword(raw)` method internally hashes + validates
- Java: `private` fields + `getX()/setX()` or Lombok `@Getter/@Setter`
- C#: properties (`public X { get; private set; }`)

---

## 🎤 Interview One-Liners

- *"Registration flow mein main `@Transactional` use karunga taake user DB save aur email send atomic ho. Email fail = rollback."*
- *"BCrypt cost factor 10 = 2^10 iterations = deliberately slow → brute force defeated."*
- *"UNIQUE constraint at DB level race-condition-safe — application-level duplicate check race kar sakta hai."*
- *"Production mein Transactional Outbox Pattern use karunga — guaranteed email delivery + fast API response."*

---

## ⚠️ Red Flags

- ❌ "Frontend pe validation kar lia, backend pe nahi karunga" → bypass possible
- ❌ "Password plain text store karo" → critical security failure
- ❌ "Email fail = user retry button" → user stuck if DB has user but email failed

---

**Self-test**: [Day 1 Quiz](../quizzes/day-001-user-registration.md)
