# 🔑 Day 13 — Form Validation Full-Stack: Revision Keys

**📖 Full lesson**: [2026-05-25-day13-form-validation.md](../lessons/01-beginner/2026-05-25-day13-form-validation.md)
**⏱️ Reading time**: 5-7 minutes
**🎯 Use when**: Night before interview, weekly revision, concept refresh

---

## ⚡ If You Remember Only 10 Things

1. **CASE-D** = **C**lient (UX, trust=0) → **A**PI (security boundary) → **S**ervice (business rules) → **E**nforce in DB (integrity) → **D**efense-in-depth across all layers.
2. **Client-side validation = UX only, NOT security.** Attacker `curl`/Postman se Angular bypass karke garbage bhej sakta hai. Server MUST re-validate.
3. **Server = single source of truth.** Client mirrors. Stripe/GitHub: backend single arbiter, frontend just reflects.
4. **Email uniqueness ka asli guarantee = DB `UNIQUE` constraint**, app-level "exists?" check nahi. Check-then-insert = **TOCTOU race** (Time Of Check to Time Of Use).
5. **UNIQUE constraint = secret unique B-tree index.** Duplicate insert → 2627 (SQL Server) / 1062 (MySQL). Catch karke clean 409/400, 500 nahi. Isolation-independent.
6. **Spring**: `@Valid` on param → Hibernate Validator → reflection → `ConstraintValidator.isValid()` → `MethodArgumentNotValidException` → `@RestControllerAdvice` → structured 400.
7. **.NET**: `[ApiController]` auto-returns 400 on `!ModelState.IsValid` (registers an action filter). Without it, manual `if (!ModelState.IsValid)`.
8. **Cross-field rule** (Saudi/PK postal = 5 digit): Java class-level `@Constraint`, .NET `IValidatableObject.Validate()`, Angular FormGroup-level `ValidatorFn`.
9. **Async validator** (email taken) ko **debounce** karo (warna keystroke pe DDoS apne server pe) + **fail-open** (`catchError(() => of(null))`) — backend down ho to UX block na ho; DB constraint asli enforcement.
10. **LSP** ("NAYA KHILARI, WOHI RULES") = subtype base ki jagah substitutable. Validation engine living example: har validator same contract (`null`/`error`, never throw) → polymorphic loop safe.

---

## ☕ Java / Spring Boot — Quick Keys

- **Bean Validation** = spec (JSR-380/Jakarta). **Hibernate Validator** = implementation (Hibernate ORM se alag).
- `@Valid @RequestBody Dto` → validation chalti hai BEFORE controller body. Fail → `MethodArgumentNotValidException`.
- Built-ins: `@NotNull`, `@NotBlank` (string trim+empty), `@NotEmpty` (collection/string), `@Email`, `@Size(max=)`, `@Pattern(regexp=)`, `@Min/@Max`, `@Positive`.
- **`@NotNull` vs `@NotBlank` vs `@NotEmpty`**: NotNull=non-null; NotEmpty=non-null + size>0; NotBlank=non-null + trimmed length>0 (strings only).
- **Custom constraint**: `@interface` + `@Constraint(validatedBy=X.class)` + `ConstraintValidator<A,T>` with `isValid()`. Class-level (`@Target(TYPE)`) = cross-field.
- **Method-level validation**: `@Validated` on class + constraints on params → AOP proxy enforces.
- **Error handling**: `@RestControllerAdvice` + `@ExceptionHandler(MethodArgumentNotValidException.class)` → map `getBindingResult().getFieldErrors()` to `{field: message}`.
- **Groups**: `@Validated(OnCreate.class)` for different rules on create vs update.

## 🌐 .NET / C# — Quick Keys

- Data Annotations: `[Required]`, `[EmailAddress]`, `[StringLength(100)]`, `[RegularExpression(...)]`, `[Range(1,100)]`, `[Compare]`.
- `[ApiController]` → automatic 400 `ValidationProblemDetails` on invalid `ModelState`. Without → manual `if (!ModelState.IsValid) return BadRequest(ModelState);`.
- **Cross-field**: implement `IValidatableObject` → `IEnumerable<ValidationResult> Validate(ctx)` (yield results; never throw for "invalid").
- **FluentValidation** (industry favorite): `AbstractValidator<T>`, `RuleFor(x => x.Email).NotEmpty().EmailAddress()`, `.When(...)` conditional — testable, no attribute clutter.
- Under the hood: model binding → `ObjectModelValidator` → reflection over `ValidationAttribute`s → `ModelState` dictionary.

## 🗄️ SQL — Quick Keys

- **DB constraints = last line of defense**, enforced on EVERY insert/update regardless of source (app, script, DBA).
- `NOT NULL` (metadata flag), `UNIQUE` (unique B-tree index, race-proof), `CHECK` (boolean expr per row), `DEFAULT`, length caps.
- **Unique violation codes**: SQL Server **2627** (constraint) / 2601 (index), MySQL **1062**. Catch in `TRY/CATCH` → friendly error.
- **Cross-column CHECK**: `CHECK ((country IN ('PK','SA') AND postal LIKE '[0-9][0-9][0-9][0-9][0-9]') OR country='AE')`.
- **Don't rely on isolation level for uniqueness** — even READ COMMITTED has TOCTOU. Use the UNIQUE constraint.
- Normalize before unique-store (lowercase email) so `A@x` and `a@x` collide correctly.

## 🎨 Angular — Quick Keys

- **Reactive Forms** (testable) > template-driven for complex validation. `FormGroup`/`FormControl`/`FormBuilder`.
- **3-arg control**: `['', [syncValidators], [asyncValidators]]`.
- `ValidatorFn` contract: `(control) => ValidationErrors | null`. **null = valid**, object = invalid. **Never throw / never return undefined** (LSP!).
- Status: `VALID` / `INVALID` / `PENDING` (async running) / `DISABLED`. Sync runs first; async only if sync passes.
- **Async validator**: returns `Observable<ValidationErrors|null>`; use `timer(400)` + `switchMap` (debounce + cancel stale), `catchError(()=>of(null))` (fail-open).
- **Cross-field**: validator on the `FormGroup` (`{ validators: [fn] }`), read sibling controls via `group.get('x')`.
- `updateOn: 'blur'` to avoid red-on-every-keystroke. `markAllAsTouched()` before submit to surface errors.
- Map server 400 errors back to fields: `form.get(field)?.setErrors({ server: msg })`.

## 🏗️ System Design — Quick Keys

- **Defense in depth**: validate at every trust boundary, each with a purpose (UX / security / business / integrity).
- **Single source of truth**: generate FE rules from BE (OpenAPI), or shared JSON Schema/Zod → avoid FE/BE rule drift.
- **Edge/gateway**: schema + size validation (reject malformed/oversized early).
- **TOCTOU** for uniqueness → DB unique constraint is the only race-proof arbiter; client check is UX.
- Idempotency keys (Day 39) guard double-submit.
- Real companies: Stripe (structured `param`+`code` errors), GitHub (async username + unique index), Daraz/FoodPanda (FE feedback + BE re-validate + DB unique).

---

## 🏛️ OOP — Liskov Substitution Principle (LSP) Quick Keys

- **Definition** (Barbara Liskov, 1987): subtype `S` ko base `T` ki jagah rakho → program correctness toot na sake. **"is-substitutable-for"**, not just "is-a".
- **Mnemonic**: **"NAYA KHILARI, WOHI RULES"** + **"BETA BAAP KI KURSI PE — BINA GADBAD"**.
- **4 rules**: (1) preconditions **strengthen** na ho, (2) postconditions **weaken** na ho, (3) invariants preserve, (4) no new exceptions outside base contract.
- **In today's brainer**: validator hierarchy — har subtype `ValidationResult` return kare (never throw for invalid). Empty input pe exception = LSP violation → polymorphic loop crash.
- **Classic violation**: `Square extends Rectangle` — `setWidth(5)` pe height bhi badle → caller assumption tooti. Fix: `Shape` interface + composition, inheritance nahi.
- **Java case study**: `Collections.unmodifiableList().add()` → `UnsupportedOperationException` = documented LSP violation (signal hierarchy ko split karo).
- **Pairs with**: OCP (Day 12 — LSP makes OCP *correct*), Strategy (Day 62), Composition over Inheritance (Day 5).
- **Tension with**: deep inheritance, premature abstraction (YAGNI, Day 18).
- **Common mistakes**:
  1. Subtype mein extra precondition (strengthening) → substitute pe break.
  2. Subtype mein nayi exception → polymorphic caller crash.
  3. "IS-A" sochke inherit karna bina behavioural substitutability check.

---

## 🎤 Interview One-Liners (Bolne Layak Confidently)

- *"Client-side validation main sirf UX ke liye use karta hun — security backend pe, kyunki attacker curl se Angular bypass kar sakta hai. Backend = single source of truth, client bas mirror."*
- *"Email uniqueness app-level SELECT se guarantee nahi hoti — woh TOCTOU race hai. Asli guarantee DB UNIQUE constraint deti hai, aur duplicate-key error ko main gracefully handle karta hun, 500 nahi deta."*
- *"Spring mein `@Valid` Hibernate Validator ko trigger karta hai, jo reflection se annotations scan karke ConstraintValidator chalata hai; fail pe MethodArgumentNotValidException, jise main @RestControllerAdvice se clean 400 mein convert karta hun."*
- *"Async email-check ko debounce + fail-open rakhta hun — backend down ho to UX block na ho, kyunki asli enforcement DB karta hai."*
- *"LSP ka matlab subtype base ki jagah bina surprise fit ho — Square extends Rectangle classic violation hai, kyunki setWidth pe height badal jata hai; sahi fix composition hai. Mera validation engine LSP-clean hai: har validator same contract follow karta hai, isliye polymorphic loop mein interchangeable."*

---

## ⚠️ Red Flags — Don't Say These In Interview

- ❌ "Frontend pe validation laga di, kaafi hai" → zero security, bypassable
- ❌ "Email exists check karke insert kar dunga" → TOCTOU race, concurrent duplicate
- ❌ "DB constraints ki zaroorat nahi, app validate kar raha" → ek bug = permanent corrupt data
- ❌ "Square is-a Rectangle, inheritance theek hai" → behavioural substitutability fail (LSP)
- ❌ "Validator empty input pe exception phenkega" → LSP violation, loop crash
- ❌ "Har keystroke pe email API call kar dunga" → self-DDoS, no debounce
- ❌ "Uniqueness ke liye SERIALIZABLE isolation laga dunga" → overkill; unique constraint hi kaafi

---

## 🧠 Memory Hooks Recap

- **CASE-D** — Client → API → Service → Enforce-in-DB → Defense-in-depth
- **"NAYA KHILARI, WOHI RULES"** + **"BETA BAAP KI KURSI PE — BINA GADBAD"** — Liskov Substitution
- **"TOCTOU"** — check aur use ke beech gap = race; DB unique constraint hi race-proof
- **"VIP shaadi mein dakhil"** — gate (Angular) → reception ID (backend) → biometric (DB)
- **"null = valid"** — Angular/JSR validator contract (never throw)

---

## 🔗 Cross-References

- For **OOP deep**: see [cheatsheets/oop-principles.md](../cheatsheets/oop-principles.md)
- For **Java↔.NET**: see [cheatsheets/spring-vs-dotnet.md](../cheatsheets/spring-vs-dotnet.md)
- For **SQL deep**: see [cheatsheets/sql-locking-isolation.md](../cheatsheets/sql-locking-isolation.md)
- For **all memory hooks**: see [cheatsheets/memory-hooks.md](../cheatsheets/memory-hooks.md)

**Self-test**: Take the [Day 13 quiz](../quizzes/day-013-form-validation.md) — 50 MCQs to verify mastery.
