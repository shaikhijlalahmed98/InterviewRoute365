# 📝 Day 13 Quiz — Form Validation Full-Stack (LSP)

**📖 Lesson**: [2026-05-25-day13-form-validation.md](../lessons/01-beginner/2026-05-25-day13-form-validation.md)
**🔑 Revision**: [day-013-form-validation.md](../revision/day-013-form-validation.md)
**Total**: 50 MCQs | **Pass mark**: 40/50 (80%)
**Sections**: A (Output) · B (Bug Spot) · C (Dry-Run) · D (Scenarios) · E (Concept)

> Pehle saare 50 attempt karo, phir niche answer key kholo. Likhne ke liye `**Your Answer**: ___` use karo.

---

## Section A — Code Output Prediction (Q1-Q10)

### Q1. ⭐⭐ Spring DTO validation — kya hota hai jab `name` blank ho?
```java
public class Req {
    @NotBlank private String name;   // value = ""
}
// Controller: public X create(@Valid @RequestBody Req r) { return svc.save(r); }
```
A) Controller chalता hai, `save()` blank name save kar deta hai
B) `MethodArgumentNotValidException` throw hoti hai, controller body chalta hi nahi
C) Compile error
D) `name` automatically `null` ho jata hai

**Your Answer**: ___

---

### Q2. ⭐⭐ `@NotBlank` vs `@NotEmpty` vs `@NotNull` — `value = "   "` (spaces) ke liye kaun fail hota hai?
A) Teeno fail
B) Sirf `@NotBlank` fail (trim ke baad empty)
C) Sirf `@NotNull` fail
D) Koi fail nahi

**Your Answer**: ___

---

### Q3. ⭐⭐ Angular sync validator output:
```typescript
const c = new FormControl('ahmed@@gmail', [Validators.required, Validators.email]);
console.log(c.valid, c.errors);
```
A) `true null`
B) `false { email: true }`
C) `false { required: true }`
D) `true { email: true }`

**Your Answer**: ___

---

### Q4. ⭐⭐⭐ Custom validator return value:
```typescript
function strong(c: AbstractControl): ValidationErrors | null {
  return c.value?.length >= 8 ? null : { weak: true };
}
const c = new FormControl('abc', [strong]);
console.log(c.valid);
```
A) `true`
B) `false`
C) `undefined`
D) throws

**Your Answer**: ___

---

### Q5. ⭐⭐ SQL duplicate insert:
```sql
-- uq_user_email UNIQUE(user_id, email) exists; row (1,'a@x.com') already present
INSERT INTO addresses(user_id,email,...) VALUES (1,'a@x.com',...);
```
A) Row insert ho jata hai (overwrite)
B) Unique violation error (SQL Server 2627 / MySQL 1062)
C) Silently ignored
D) `NULL` insert hota hai

**Your Answer**: ___

---

### Q6. ⭐⭐ .NET `[ApiController]` behaviour:
```csharp
[ApiController] public class C : ControllerBase {
  [HttpPost] public IActionResult Create(Req r) { return Ok(); } // r.Email invalid
}
```
A) `Ok()` return hota hai, invalid data accept
B) Automatic `400 ValidationProblemDetails`, action body chalta hi nahi
C) `500` error
D) Compile error

**Your Answer**: ___

---

### Q7. ⭐⭐⭐ Angular async validator status — submit pe (abhi async pending):
```typescript
email: ['', [Validators.required], [emailTakenAsync]]  // value typed, async running
console.log(form.get('email').status);
```
A) `VALID`
B) `INVALID`
C) `PENDING`
D) `DISABLED`

**Your Answer**: ___

---

### Q8. ⭐⭐ LSP loop output (BAD validator throws):
```java
// EmailValidator throws on empty; others return result
for (Validator v : validators) results.add(v.validate(""));
System.out.println(results.size());  // 4 validators, EmailValidator is 2nd
```
A) `4`
B) `2`
C) Exception propagates, loop dies (size never printed)
D) `0`

**Your Answer**: ___

---

### Q9. ⭐⭐ `@Pattern` for Pakistan phone:
```java
@Pattern(regexp = "^03\\d{9}$") private String phone;  // phone = "0300123456" (10 digits)
```
A) Valid
B) Invalid (needs 11 digits total: 03 + 9 digits)
C) Valid only if starts with 03
D) Compile error

**Your Answer**: ___

---

### Q10. ⭐⭐⭐ Base64-free — `@RestControllerAdvice` response shape:
```java
fieldErrors.put(e.getField(), e.getDefaultMessage());
return Map.of("status", 400, "errors", fieldErrors);
```
Agar `email` aur `phone` dono invalid hain, response `errors` mein kitni entries?
A) 1 (first error only)
B) 2 (email + phone)
C) 0
D) 4

**Your Answer**: ___

---

## Section B — Bug Spotting (Q11-Q18)

### Q11. ⭐⭐ Is code mein kya galat hai?
```typescript
// Email uniqueness check
emailTaken(): AsyncValidatorFn {
  return (ctrl) => this.http.get<boolean>(`/api/email-exists?e=${ctrl.value}`)
                       .pipe(map(exists => exists ? { taken: true } : null));
}
```
A) `null` galat hai
B) Debounce nahi hai — har keystroke pe API call (self-DDoS); `timer(400)+switchMap` chahiye
C) `map` galat operator hai
D) Kuch galat nahi

**Your Answer**: ___

---

### Q12. ⭐⭐⭐ Production bug — uniqueness:
```java
if (repo.existsByEmail(email)) throw new DuplicateException();
repo.save(new User(email));   // gap between check and save
```
A) `existsByEmail` slow hai
B) TOCTOU race — do concurrent requests dono check pass kar lenge, duplicate save; DB UNIQUE constraint chahiye
C) `save` galat hai
D) Exception type galat

**Your Answer**: ___

---

### Q13. ⭐⭐⭐ LSP violation spot karo:
```java
abstract class Validator { abstract Result validate(String v); } // contract: never throw for invalid
class AgeValidator extends Validator {
    Result validate(String v) {
        int age = Integer.parseInt(v);   // <-- ?
        return age >= 18 ? Result.ok() : Result.error("Under 18");
    }
}
```
A) `age` field missing
B) `Integer.parseInt("")` `NumberFormatException` phenkta hai — base contract (never throw) toota, LSP violation
C) `>= 18` galat
D) Kuch galat nahi

**Your Answer**: ___

---

### Q14. ⭐⭐ Security bug:
```java
@PostMapping public X create(@RequestBody Req r) { return svc.save(r); }  // note: no @Valid
```
A) `@RequestBody` galat
B) `@Valid` missing — validation chalti hi nahi, garbage save ho jayega
C) Return type galat
D) Kuch galat nahi

**Your Answer**: ___

---

### Q15. ⭐⭐ Angular bug:
```typescript
function postalRule(g: AbstractControl): ValidationErrors | null {
  const postal = g.get('postalCode').value;
  if (!/^\d{5}$/.test(postal)) return { bad: true };
  return null;  // applies even when country = AE (no postal codes!)
}
```
A) Regex galat
B) Country check missing — UAE (no postal) ke liye bhi reject karega; conditional rule chahiye
C) `null` galat
D) Kuch galat nahi

**Your Answer**: ___

---

### Q16. ⭐⭐⭐ .NET cross-field bug:
```csharp
public IEnumerable<ValidationResult> Validate(ValidationContext ctx) {
    if (PostalCode == null) throw new ArgumentNullException();  // <-- ?
    yield return new ValidationResult("bad");
}
```
A) `yield` galat
B) `throw` from `Validate()` — should `yield return` a ValidationResult, not throw (breaks validation pipeline)
C) `ValidationContext` galat
D) Kuch galat nahi

**Your Answer**: ___

---

### Q17. ⭐⭐ SQL bug:
```sql
CREATE TABLE users (
  email NVARCHAR(255) NOT NULL   -- no UNIQUE constraint
);
-- App relies only on existsByEmail() check before insert
```
A) `NVARCHAR` galat
B) No UNIQUE constraint — concurrent duplicates ghus jayenge (app check race-prone)
C) `NOT NULL` galat
D) Kuch galat nahi

**Your Answer**: ___

---

### Q18. ⭐⭐⭐ Angular async stale-response bug:
```typescript
return this.http.get(`/check?e=${ctrl.value}`).pipe(map(...));
// user types fast: "a","ab","abc" → 3 requests; "a" response arrives LAST
```
A) `map` galat
B) No `switchMap` — purani (stale) response baad mein aa ke result overwrite kar sakti hai; `switchMap` cancels stale
C) `http.get` galat
D) Kuch galat nahi

**Your Answer**: ___

---

## Section C — Dry-Run Tracing (Q19-Q28)

### Q19. ⭐⭐⭐ TOCTOU trace:
```
Initial: users table empty. UNIQUE(email) exists.
T+0ms  Req A: existsByEmail('x') → false
T+1ms  Req B: existsByEmail('x') → false
T+2ms  Req A: INSERT x → OK
T+3ms  Req B: INSERT x → ?
```
A) Req B INSERT OK (2 rows)
B) Req B INSERT fails with unique violation (only 1 row); app must catch
C) Both rolled back
D) Req A fails

**Your Answer**: ___

---

### Q20. ⭐⭐⭐ Angular validator order:
```
FormControl('', [required, email], [asyncTaken])
T+0  value set to ''
T+1  sync validators run → required fails
T+2  ?
```
A) async validator chalta hai
B) async **nahi** chalta (sync fail hone par async skip); status INVALID
C) dono fail
D) status PENDING

**Your Answer**: ___

---

### Q21. ⭐⭐ Spring validation chain:
```
T+0  JSON arrives
T+1  Jackson binds to Req
T+2  @Valid → Hibernate Validator runs
T+3  3 fields invalid
T+4  ?
```
A) Controller runs, ignores errors
B) MethodArgumentNotValidException with 3 field errors → @ControllerAdvice → 400
C) Only first error returned
D) 500 error

**Your Answer**: ___

---

### Q22. ⭐⭐⭐ LSP substitution trace (Square/Rectangle):
```
Rectangle r = new Square();   // Square overrides setWidth to also set height
r.setWidth(5);
r.setHeight(10);
print(r.area());
```
A) 50
B) 100 (setHeight(10) also set width=10 → 10×10)
C) 15
D) Exception

**Your Answer**: ___

---

### Q23. ⭐⭐ Debounce trace:
```
Async validator: timer(400) + switchMap(http.get)
T+0    user types 'a'  → timer starts
T+100  user types 'b'  → ?
```
A) 'a' ka request fire ho chuka
B) timer(400) abhi pending tha; naya emission timer reset karta (no request fired yet for 'a')
C) 2 requests fire
D) Error

**Your Answer**: ___

---

### Q24. ⭐⭐⭐ DB CHECK constraint trace:
```sql
-- CHECK ((country IN ('PK','SA') AND postal LIKE '[0-9][0-9][0-9][0-9][0-9]') OR country='AE')
INSERT ... country='AE', postal=NULL;
```
A) Rejected (postal NULL)
B) Accepted (AE branch true regardless of postal)
C) Rejected (country invalid)
D) Error

**Your Answer**: ___

---

### Q25. ⭐⭐⭐ Async fail-open trace (backend down):
```
emailTaken: ...pipe(switchMap(http.get), catchError(() => of(null)))
T+0  user finishes typing
T+400 http.get → 503 Service Unavailable
T+401 ?
```
A) Field marked INVALID, submit blocked
B) catchError returns null → field treated VALID (fail-open); DB constraint enforces at submit
C) Form crashes
D) Retry forever

**Your Answer**: ___

---

### Q26. ⭐⭐ Multi-layer trace — attacker bypasses Angular:
```
T+0  attacker curl POST /api/addresses  body: { email: null, phone: 'DROP TABLE' }
T+1  (Angular skipped entirely)
T+2  Spring @Valid runs → ?
```
A) Saved (no frontend = no validation)
B) @Valid rejects (email @NotBlank, phone @Pattern) → 400; backend is the security boundary
C) 500
D) SQL injection succeeds

**Your Answer**: ___

---

### Q27. ⭐⭐⭐ Group validator timing:
```
form = group({country, postalCode}, { validators: [postalRule] })
T+0  country='SA', postalCode='12'
T+1  postalRule runs at GROUP level → ?
```
A) Runs per-control only, never sees both
B) Group-level validator sees both controls → returns { postalInvalid: true } (SA needs 5 digits)
C) Throws
D) null

**Your Answer**: ___

---

### Q28. ⭐⭐ markAllAsTouched trace:
```
T+0  form has hidden errors, user clicks Submit
T+1  if(form.invalid){ form.markAllAsTouched(); return; }
T+2  ?
```
A) Form submits anyway
B) Error messages ab dikhne lagte hain (touched=true triggers display), submit aborted
C) Form resets
D) Crash

**Your Answer**: ___

---

## Section D — Scenario-Based Decisions (Q29-Q38)

### Q29. ⭐⭐ Daraz checkout: user complain karta hai "maine valid email dala but reject hua".
Setup: FE regex `^\S+@\S+$`, BE `@Email` (stricter). Email = `a@b` (no TLD).
A) FE bug — make FE stricter
B) Rule mismatch — FE/BE rules diverge; need single source of truth (generate FE from BE)
C) BE bug — remove @Email
D) DB bug

**Your Answer**: ___

---

### Q30. ⭐⭐⭐ 10,000 users same promo pe register. Email uniqueness?
A) `existsByEmail()` check before insert
B) DB UNIQUE constraint + catch duplicate-key → clean error; app check is race-prone
C) SERIALIZABLE isolation on everything
D) Lock the whole table

**Your Answer**: ___

---

### Q31. ⭐⭐ Async email validator network fail. Best behaviour?
A) Block submit (fail-closed)
B) Fail-open (`catchError(()=>of(null))`) — don't block UX; DB constraint is real enforcement
C) Retry 100 times
D) Show 500 to user

**Your Answer**: ___

---

### Q32. ⭐⭐⭐ Mobile app + web both hit same API. Where MUST validation live?
A) Each client validates; backend trusts clients
B) Backend (single source of truth) — clients can't be trusted; each adds its own UX validation
C) Only mobile
D) Only DB

**Your Answer**: ___

---

### Q33. ⭐⭐ Legacy code: `Square extends Rectangle`, callers breaking. Fix?
A) Add more if-checks in callers
B) Replace inheritance with composition / common `Shape` interface (LSP fix)
C) Make Rectangle final
D) Delete Square

**Your Answer**: ___

---

### Q34. ⭐⭐⭐ Validator that can't handle base inputs (throws on null). LSP-correct fix?
A) Add try/catch at every callsite
B) Make subtype honor base contract — accept null, return error-result (don't throw)
C) Remove the validator
D) Catch and ignore

**Your Answer**: ___

---

### Q35. ⭐⭐ FoodPanda: address form har keystroke pe red flash karta, users annoyed.
A) Remove validation
B) Use `updateOn: 'blur'` / debounce — validate on blur, not every keystroke
C) Only validate on backend
D) Disable the form

**Your Answer**: ___

---

### Q36. ⭐⭐⭐ Duplicate email INSERT throws SQL 2627, API returns 500 to user. Better?
A) Leave it (500 is fine)
B) Catch unique violation → return 409/400 with friendly "email already used" message
C) Remove the constraint
D) Retry the insert

**Your Answer**: ___

---

### Q37. ⭐⭐ Two services write to `users`. One skips validation. Result?
A) Fine, other service validates
B) Corrupt data possible — DB constraints (NOT NULL/CHECK/UNIQUE) are the shared last line of defense
C) Faster writes
D) Nothing

**Your Answer**: ___

---

### Q38. ⭐⭐⭐ `PaymentServiceV2` replacing V1. V2 requires a new mandatory field. Problem?
A) None
B) Strengthened precondition → LSP/backward-compat violation; old clients break. New field must be optional/defaulted
C) V2 is faster, fine
D) Delete V1 clients

**Your Answer**: ___

---

## Section E — Concept Mastery (Q39-Q50)

### Q39. ⭐ Client-side validation ka primary purpose?
A) Security
B) UX (instant feedback, fewer round-trips) — NOT security
C) Data integrity
D) Compression

**Your Answer**: ___

---

### Q40. ⭐ Email uniqueness ka asli (race-proof) guarantee kahan?
A) Angular async validator
B) DB UNIQUE constraint
C) Backend `existsByEmail()`
D) API gateway

**Your Answer**: ___

---

### Q41. ⭐⭐ TOCTOU ka matlab?
A) Type Of Check Or Use
B) Time Of Check to Time Of Use — check aur use ke beech state badal sakti hai (race)
C) Total Order Concurrency
D) Transaction Of Commit

**Your Answer**: ___

---

### Q42. ⭐ LSP ek line mein?
A) Ek class, ek kaam
B) Subtype base ki jagah substitutable — bina program correctness tode
C) Extend haan, modify naa
D) Depend on abstractions

**Your Answer**: ___

---

### Q43. ⭐⭐ LSP ke 4 rules mein se kaunsa galat hai?
A) Preconditions strengthen na ho
B) Postconditions weaken na ho
C) Subtype nayi exceptions phenk sakta hai (outside base contract)
D) Invariants preserve hon

**Your Answer**: ___

---

### Q44. ⭐⭐ `@Valid` Spring mein internally kis engine ko trigger karta hai?
A) Jackson
B) Hibernate Validator (Bean Validation impl)
C) JPA/Hibernate ORM
D) Tomcat

**Your Answer**: ___

---

### Q45. ⭐⭐ Angular `ValidatorFn` valid input ke liye kya return karta hai?
A) `true`
B) `null`
C) `{}`
D) `undefined`

**Your Answer**: ___

---

### Q46. ⭐ `@NotBlank` kis type pe lagta hai?
A) Any object
B) String (non-null + trimmed length > 0)
C) Collections only
D) Numbers

**Your Answer**: ___

---

### Q47. ⭐⭐ Square/Rectangle LSP violation ka root cause?
A) Square slow hai
B) `setWidth` setting both sides breaks the Rectangle contract caller relied on
C) Inheritance allowed nahi
D) Square ka area galat

**Your Answer**: ___

---

### Q48. ⭐⭐ LSP kis principle ko *correct* banata hai?
A) SRP
B) OCP — substitutable subtypes ke bina "open for extension" safe nahi
C) DRY
D) KISS

**Your Answer**: ___

---

### Q49. ⭐⭐ Defense in depth ka matlab validation mein?
A) Sirf DB pe validate
B) Har trust boundary pe validate, har layer ka apna purpose (UX/security/integrity)
C) Sirf ek layer chuno
D) Client pe sab kuch

**Your Answer**: ___

---

### Q50. ⭐⭐⭐ Java `Collections.unmodifiableList().add()` `UnsupportedOperationException` phenkta hai. Yeh kya darshata hai?
A) Bug in Java
B) Documented LSP violation — subtype base ka operation honor nahi karta (signal: hierarchy split karo / ISP)
C) Performance optimization
D) Thread safety

**Your Answer**: ___

---

<details>
<summary>⚠️ Click to reveal answers — complete all 50 first!</summary>

### Section A: Code Output Prediction
1. **B** — `@Valid` fail → `MethodArgumentNotValidException` before controller body runs.
2. **B** — `@NotBlank` trims then checks length; spaces fail it. `@NotNull`/`@NotEmpty` pass for "   " (non-null, length>0).
3. **B** — `required` passes (non-empty), `Validators.email` fails → `{ email: true }`, valid=false.
4. **B** — 'abc' length 3 < 8 → returns `{ weak: true }` → control invalid.
5. **B** — UNIQUE constraint violation: SQL Server 2627, MySQL 1062.
6. **B** — `[ApiController]` auto-returns 400 ValidationProblemDetails on invalid ModelState.
7. **C** — async validator running → status `PENDING` until it resolves.
8. **C** — EmailValidator throws (LSP violation); exception propagates, loop dies, size never printed.
9. **B** — `^03\d{9}$` = 03 + 9 digits = 11 total; "0300123456" is 10 → invalid.
10. **B** — all field errors collected into the map → 2 entries (email + phone).

### Section B: Bug Spotting
11. **B** — no debounce → API call per keystroke (self-DDoS); use `timer(400)+switchMap`.
12. **B** — TOCTOU race; only DB UNIQUE constraint is race-proof.
13. **B** — `Integer.parseInt("")` throws `NumberFormatException`, violating base "never throw for invalid" → LSP violation. Parse defensively, return error-result.
14. **B** — no `@Valid` → validation never runs; garbage saved.
15. **B** — no country check; UAE (no postal) wrongly rejected; make rule conditional.
16. **B** — `Validate()` should `yield return` ValidationResult, not throw; throwing breaks the pipeline.
17. **B** — no UNIQUE constraint → concurrent duplicates slip past app-level check.
18. **B** — no `switchMap`; stale response can overwrite latest; `switchMap` cancels in-flight.

### Section C: Dry-Run Tracing
19. **B** — both checks see empty, but only one INSERT succeeds; the other hits unique violation → app must catch.
20. **B** — sync fails (required) → async skipped; status INVALID.
21. **B** — all violations collected → MethodArgumentNotValidException (3 errors) → advice → 400.
22. **B** — Square's setHeight also sets width → 10×10 = 100; caller expected 50 (LSP violation).
23. **B** — `timer(400)` still pending when 'b' typed; new emission resets timer; no request fired yet.
24. **B** — CHECK passes via `country='AE'` branch (OR short-circuits true); postal NULL irrelevant.
25. **B** — `catchError(()=>of(null))` → field treated valid (fail-open); DB enforces on submit.
26. **B** — Angular bypassed, but `@Valid` on backend rejects → 400; backend is the security boundary.
27. **B** — group-level validator sees both controls; SA + 2-digit postal → `{ postalInvalid: true }`.
28. **B** — `markAllAsTouched()` makes hidden errors visible; submit aborted (return).

### Section D: Scenario-Based Decisions
29. **B** — FE/BE rules diverged; fix with single source of truth (generate FE rules from BE schema).
30. **B** — DB UNIQUE constraint + catch duplicate-key; app-level check is race-prone.
31. **B** — fail-open so UX isn't blocked by a transient outage; DB constraint is real enforcement.
32. **B** — backend = single source of truth (clients untrusted); each client adds UX validation.
33. **B** — replace inheritance with composition / `Shape` interface (canonical LSP fix).
34. **B** — make the subtype honor base contract (accept base inputs, return error-result, don't throw).
35. **B** — `updateOn: 'blur'`/debounce so it validates on blur, not every keystroke.
36. **B** — catch unique violation → 409/400 friendly message, not a 500.
37. **B** — DB constraints are the shared last line of defense across services; without them, corruption.
38. **B** — strengthened precondition (new mandatory field) breaks old clients (LSP/backward-compat); make it optional/defaulted.

### Section E: Concept Mastery
39. **B** — client validation = UX, not security.
40. **B** — DB UNIQUE constraint (race-proof, isolation-independent).
41. **B** — Time Of Check to Time Of Use.
42. **B** — subtype substitutable for base without breaking correctness. (C is OCP, D is DIP.)
43. **C** — wrong; subtypes must NOT throw new exceptions outside the base contract.
44. **B** — Hibernate Validator (the Bean Validation implementation); distinct from Hibernate ORM.
45. **B** — `null` for valid; an error object for invalid.
46. **B** — `@NotBlank` is for strings: non-null + trimmed length > 0.
47. **B** — setWidth setting both sides breaks the contract the Rectangle caller relied on.
48. **B** — OCP; extension is only safe if subtypes are substitutable.
49. **B** — validate at every trust boundary, each with its own purpose.
50. **B** — documented LSP violation; signal the hierarchy should be split (relates to ISP, Day 14).

**Scoring**: 45-50 🏆 Senior-ready | 40-44 ✅ Solid | 30-39 📚 Revise lesson | <30 🔁 Re-read lesson + revision keys.

</details>

---

**Next**: Day 14 — Email Verification Flow (OOP: Interface Segregation Principle).
