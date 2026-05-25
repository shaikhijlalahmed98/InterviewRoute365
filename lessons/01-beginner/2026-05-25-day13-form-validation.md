# 🎯 🟢 Day 13 of Beginner (Level 1 of 7): Form Validation Full-Stack

**Overall Day**: Day 13 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 13 of 20
**Today's Theme**: User ek form bharta hai (signup / checkout address). Dikhne mein trivial — `<input>` + ek `required` attribute. Lekin asli production mein validation **ek single jagah ka kaam nahi**, balki **chaar layers** ka coordinated defense hai: (1) **Angular** (instant UX feedback, but security NAHI), (2) **Spring/.NET** (real security boundary — Bean Validation / Data Annotations), (3) **SQL** (last line of defense — `NOT NULL`, `UNIQUE`, `CHECK`), (4) **System Design** (defense-in-depth + single source of truth + async uniqueness ka TOCTOU race). Aaj ka OOP lesson: **Liskov Substitution Principle (L of SOLID)** — *NAYA KHILARI, WOHI RULES*. Validation engine khud LSP ka living example hai: har validator base contract follow karta hai, isliye polymorphic loop mein interchangeable hain.

---

## 📖 The Brainer Scenario (Real Problem)

Bhai, **Daraz** ya **FoodPanda** pe checkout karo. "Add Delivery Address" form khulta hai: Name, Phone, Email, City, Postal Code, Country. Tum galat email daalo — turant red border + "Invalid email". Phone khali chhodo — "Phone required". Sab sahi bharo → "Save". 2 second baad address save, order proceed.

User ko lagta hai *"itni si baat hai, ek `required` laga do"*. Lekin yeh form **char trust boundaries** cross karta hai:

```
T+0ms     User email field mein "ahmed@@gmail" type karta hai
T+5ms     Angular Reactive Form: Validators.email → field RED, "Save" disabled
          (yeh sirf UX hai — fast feedback, lekin SECURITY nahi)
T+2000ms  User sab sahi karke "Save" dabata hai → POST /api/addresses
T+2010ms  Spring Boot: @Valid AddressRequest → Bean Validation chalti hai
          (YEH asli security boundary hai — client bypass kar sakta hai)
T+2015ms  Service layer: business rule — "Saudi address mein postalCode 5 digit"
T+2020ms  SQL: INSERT → UNIQUE(email) constraint, CHECK(phone), NOT NULL
          (LAST line of defense — direct DB access / app bug se bachata hai)
T+2030ms  201 Created → Angular address list refresh
```

**The Real Challenge (The "Gotcha")**:
Junior dev sirf **Angular** pe validation lagata hai aur khush ho jata hai. Phir koi banda Postman / `curl` se direct API hit karta hai (Angular ko bypass karke) aur **garbage data** ghusa deta hai — `email = "'; DROP TABLE..."`, `phone = null`, `age = -5`. Frontend validation **cosmetic** hai; attacker browser ke bagair request bhej sakta hai. **Asli rule**: *client-side validation = UX; server-side validation = security; DB constraint = integrity guarantee.* Teeno chahiye.

Doosra gotcha: **email uniqueness**. Angular async validator "email already taken?" check karta hai (UX). Lekin check aur INSERT ke beech ek microsecond mein doosra user wahi email register kar sakta hai (**TOCTOU race** — Time Of Check to Time Of Use). Asli guarantee sirf **DB `UNIQUE` constraint** deti hai.

**Why this matters in production**:
- **Stripe** har API field ko server-side validate karta hai aur structured error return karta hai (`{ "error": { "param": "email", "code": "invalid_email" } }`) — frontend pe bharosa zero.
- **GitHub** signup mein username uniqueness async dikhata hai (UX) lekin DB unique index final arbiter hai — race pe ek user ko clean "username taken" error milta hai, crash nahi.

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: Bean Validation (Jakarta Validation / JSR-380), `@Valid`, built-in + custom `ConstraintValidator`, cross-field (class-level) constraints, aur `@ControllerAdvice` se clean error response.

**Under the Hood — Yeh Kaise Kaam Karta Hai**:
Pehle samjho **Bean Validation** kya hai. Yeh ek **spec** (rulebook) hai — JSR-380 / Jakarta Validation. **Hibernate Validator** uska sabse common **implementation** (engine) hai (Hibernate ORM se alag cheez, bas naam same company ka).

Jab tum controller method pe `@Valid AddressRequest req` likhte ho, yeh hota hai:
```
T+0: Request aata hai. Spring ka RequestResponseBodyMethodProcessor JSON ko AddressRequest object mein bind karta hai (Jackson).
T+1: Argument pe @Valid dikha → Spring LocalValidatorFactoryBean (jo Hibernate Validator wrap karta hai) ko call karta hai.
T+2: Validator REFLECTION se AddressRequest ki fields scan karta hai, har annotation (@NotBlank, @Email...) ke liye uska ConstraintValidator dhoondhta hai (metadata cache se).
T+3: Har validator ka isValid() chalta hai. Jo fail hote hain unke liye ConstraintViolation object banta hai (field name + message).
T+4: Agar koi violation hai → MethodArgumentNotValidException throw hoti hai, controller body chalta hi nahi.
T+5: @ControllerAdvice mein @ExceptionHandler us exception ko pakad ke 400 + structured JSON banata hai.
```
Method-level validation (`@Validated` class pe + `@NotNull` parameters pe) AOP **proxy** se chalti hai — Day 12 ke OCP/proxy concept se connected.

**Bhai, Simple Mein Samjho**:
`@Valid` ek **security guard** hai jo controller ke darwaze pe khara hai. JSON andar aaya, guard ne har field ka "ID card" (annotation rules) check kiya. Ek bhi galat → andar ghusne hi nahi diya, seedha 400 wapas. Tum manually `if (email == null) ... if (!email.contains("@")) ...` likhna **band** karte ho — declarative rules annotation pe likh dete ho, engine baaki sambhalta hai.

**Code Pattern**:
```java
// 1. DTO with declarative validation rules
public class AddressRequest {

    @NotBlank(message = "Name is required")
    @Size(max = 100, message = "Name too long")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    // Pakistan mobile: 03XXXXXXXXX (11 digits)
    @NotBlank
    @Pattern(regexp = "^03\\d{9}$", message = "Phone must be 03XXXXXXXXX")
    private String phone;

    @NotBlank
    private String city;

    @NotNull
    private CountryCode country; // enum: PK, SA, AE...

    private String postalCode;

    // getters/setters (Day 1 Encapsulation!)
}

// 2. Custom CLASS-LEVEL constraint for cross-field rule
@Documented
@Constraint(validatedBy = PostalCodeRuleValidator.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPostalCode {
    String message() default "Postal code invalid for selected country";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// The ConstraintValidator — note: ALL validators follow the SAME contract (LSP!)
public class PostalCodeRuleValidator
        implements ConstraintValidator<ValidPostalCode, AddressRequest> {

    @Override
    public boolean isValid(AddressRequest req, ConstraintValidatorContext ctx) {
        if (req == null || req.getCountry() == null) return true; // @NotNull handles null
        // Saudi = 5 digits, Pakistan = 5 digits, UAE = no postal code
        return switch (req.getCountry()) {
            case SA, PK -> req.getPostalCode() != null && req.getPostalCode().matches("\\d{5}");
            case AE     -> true; // UAE has no postal codes
        };
    }
}

// 3. Controller — @Valid triggers everything
@RestController
@RequestMapping("/api/addresses")
public class AddressController {

    private final AddressService service;
    AddressController(AddressService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<AddressResponse> create(@Valid @RequestBody AddressRequest req) {
        // Yahan aaye = validation PASS ho chuki. Body trustworthy hai.
        return ResponseEntity.status(201).body(service.save(req));
    }
}

// 4. Centralized error handling — clean structured response
@RestControllerAdvice
public class ValidationAdvice {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handle(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> fieldErrors.put(e.getField(), e.getDefaultMessage()));
        return Map.of("status", 400, "errors", fieldErrors);
    }
}
```

**Interview phrasing**:
"Iss scenario mein main client-side validation ko sirf UX maanunga. Server pe main Bean Validation (`@Valid` + Hibernate Validator) use karunga kyunki yeh declarative, reusable hai aur **security boundary** hai. Cross-field rules ke liye class-level custom constraint, aur `@RestControllerAdvice` se consistent error contract."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: Data Annotations + automatic `ModelState` validation (`[ApiController]`), `IValidatableObject` for cross-field, aur FluentValidation (industry-favorite alternative).

**Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai**:
Jab request aati hai, ASP.NET Core ka **model binding** JSON ko `AddressRequest` mein bind karta hai. Phir **ObjectModelValidator** reflection se har property ke `ValidationAttribute` (`[Required]`, `[EmailAddress]`...) ko run karta hai aur results `ModelState` (ek dictionary of errors) mein daalta hai.

`[ApiController]` attribute ka **magic**: yeh ek `ActionFilter` register karta hai jo automatically check karta hai `if (!ModelState.IsValid) return 400 ValidationProblemDetails;` — controller code chalta hi nahi. (Bilkul Spring ke `@Valid` jaisa.) Bina `[ApiController]` ke tumhe khud `if (!ModelState.IsValid)` likhna parta.

**Bhai, .NET Mein Yeh Kaise Hota Hai**:
.NET mein attributes Java annotations jaisi hi hain — square brackets mein. `[Required]` = `@NotBlank`, `[EmailAddress]` = `@Email`. `[ApiController]` lagao to validation **automatic**. Cross-field ke liye class `IValidatableObject` implement kare aur `Validate()` method likhe.

**Code Pattern**:
```csharp
public class AddressRequest : IValidatableObject
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100)]
    public string Name { get; set; }

    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string Email { get; set; }

    [Required]
    [RegularExpression(@"^03\d{9}$", ErrorMessage = "Phone must be 03XXXXXXXXX")]
    public string Phone { get; set; }

    [Required] public string City { get; set; }
    [Required] public CountryCode Country { get; set; }
    public string PostalCode { get; set; }

    // Cross-field rule (LSP-clean: returns results, never throws for "invalid")
    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if ((Country == CountryCode.SA || Country == CountryCode.PK)
            && (PostalCode == null || !Regex.IsMatch(PostalCode, @"^\d{5}$")))
        {
            yield return new ValidationResult(
                "Postal code must be 5 digits for this country",
                new[] { nameof(PostalCode) });
        }
    }
}

[ApiController] // <-- auto 400 on invalid ModelState
[Route("api/addresses")]
public class AddressController : ControllerBase
{
    private readonly IAddressService _service;
    public AddressController(IAddressService service) => _service = service; // ctor DI

    [HttpPost]
    public async Task<IActionResult> Create(AddressRequest req)
    {
        // Yahan aaye = ModelState valid. Body trustworthy.
        var saved = await _service.SaveAsync(req);
        return CreatedAtAction(nameof(Create), saved);
    }
}
```

FluentValidation alternative (testable, no attribute clutter):
```csharp
public class AddressValidator : AbstractValidator<AddressRequest>
{
    public AddressValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Phone).Matches(@"^03\d{9}$");
        RuleFor(x => x.PostalCode).Matches(@"^\d{5}$")
            .When(x => x.Country is CountryCode.SA or CountryCode.PK);
    }
}
```

**Java vs .NET Comparison Table**:
| Feature | Java/Spring | .NET/C# |
|---------|-------------|---------|
| Validation trigger | `@Valid` on param | `[ApiController]` auto / `ModelState.IsValid` |
| Required | `@NotBlank` / `@NotNull` | `[Required]` |
| Email | `@Email` | `[EmailAddress]` |
| Length | `@Size(max=100)` | `[StringLength(100)]` |
| Regex | `@Pattern(regexp=...)` | `[RegularExpression(...)]` |
| Cross-field | class-level `@Constraint` | `IValidatableObject.Validate()` |
| Fluent alternative | (custom) | FluentValidation library |
| Error container | `BindingResult` / `ConstraintViolation` | `ModelState` / `ValidationProblemDetails` |
| Engine | Hibernate Validator | built-in `ObjectModelValidator` |

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: DB-level constraints as the **last line of defense** — `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`, length caps — aur unique constraint ka race-condition role.

**Under the Hood — SQL Engine Yeh Kaise Karta Hai**:
DB constraints **declarative rules** hain jo storage engine **har INSERT/UPDATE** pe enforce karta hai, chahe data kisi bhi raste se aaye (app, script, DBA, migration). Kaise:
- **`NOT NULL`**: column metadata mein flag. Row likhne se pehle engine check karta hai, null mila to error.
- **`UNIQUE(email)`**: engine secretly ek **unique B-tree index** banata hai. Naya row insert hote waqt engine index mein key dhoondhta hai — already present? → duplicate-key error (SQL Server 2627, MySQL 1062). **Yeh atomic hai** kyunki index insert pe engine row/key lock leta hai.
- **`CHECK`**: ek boolean expression jo engine row commit hone se pehle evaluate karta hai. False → reject.

**Race ka kamaal**: Do users ek saath `ahmed@x.com` register karte hain. Dono ki app-level "email exists?" SELECT ne *"nahi hai"* kaha (kyunki dono ne insert se pehle dekha — classic TOCTOU). Lekin INSERT pe unique index mein sirf **ek** ghusega; doosre ko duplicate-key error milega. **App ka SELECT check race-prone hai; DB ka UNIQUE constraint race-proof hai.**

**Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai**:
App validation chahe kitni perfect ho, DB ko **trust nahi karna chahiye** ke upar wali layer sahi hai. Ek background script, ek purana bug, ya direct SQL access — koi bhi garbage daal sakta hai. DB constraint **data integrity ki aakhri deewar** hai. Aur uniqueness ka asli faisla yahin hota hai, app mein nahi.

**SQL Example**:
```sql
CREATE TABLE addresses (
    id           BIGINT IDENTITY PRIMARY KEY,
    user_id      BIGINT      NOT NULL,
    name         NVARCHAR(100) NOT NULL,
    email        NVARCHAR(255) NOT NULL,
    phone        VARCHAR(11)  NOT NULL,
    city         NVARCHAR(80) NOT NULL,
    country      CHAR(2)      NOT NULL,
    postal_code  VARCHAR(10)  NULL,
    created_at   DATETIME2    NOT NULL DEFAULT SYSUTCDATETIME(),

    CONSTRAINT uq_user_email UNIQUE (user_id, email),                 -- race-proof uniqueness
    CONSTRAINT ck_phone      CHECK (phone LIKE '03%' AND LEN(phone) = 11),
    CONSTRAINT ck_country    CHECK (country IN ('PK','SA','AE')),
    CONSTRAINT ck_postal     CHECK (                                   -- cross-column rule
        (country IN ('PK','SA') AND postal_code LIKE '[0-9][0-9][0-9][0-9][0-9]')
        OR country = 'AE'
    )
);

-- Handling the duplicate gracefully (instead of letting it 500):
BEGIN TRY
    INSERT INTO addresses (user_id, name, email, phone, city, country, postal_code)
    VALUES (@userId, @name, @email, @phone, @city, @country, @postal);
END TRY
BEGIN CATCH
    IF ERROR_NUMBER() = 2627  -- unique violation
        THROW 50001, 'This email is already saved for your account', 1;
    ELSE THROW;
END CATCH;
```

**The Gotcha**:
Bina DB constraint ke, agar app-layer mein ek hi bug ho (ya koi naya consumer service validation skip kare), to corrupt data **permanently** table mein baith jata hai. Baad mein reports/queries galat, aur cleanup nightmare. Constraints "fail fast at the source" dete hain.

**Isolation Level Choice**:
Uniqueness ke liye isolation level pe rely **mat** karo — even READ COMMITTED mein TOCTOU race hota hai. Sahi answer: **unique constraint/index** (jo isolation-independent hai) + duplicate-key error ko gracefully handle karna. Isolation tab matter karta hai jab tum SELECT-then-INSERT pattern use karo — usse bachна better.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Reactive Forms (`FormGroup`/`FormControl`), built-in + custom `ValidatorFn`, `AsyncValidatorFn` (email uniqueness) with debounce, cross-field group validator, error display.

**Under the Hood — Angular Yeh Kaise Karta Hai**:
Reactive form mein har `FormControl` ke paas ek **value** aur ek **status** (`VALID`/`INVALID`/`PENDING`/`DISABLED`) hota hai. Jab value badalti hai:
```
T+0: User type karta hai → control.setValue() internally
T+1: Angular sab SYNC validators chalata hai. Har ValidatorFn (control) => ValidationErrors | null return karta hai.
     Koi error object diya? → status INVALID, errors merge ho jate hain.
T+2: Sab sync valid hue tabhi ASYNC validators chalte hain → status PENDING.
T+3: Async observable emit karta hai (null = valid, error obj = invalid) → final status set.
T+4: Angular change detection chalti hai → template mein red border / message update.
```
**Yahan LSP chhupa hai**: har validator — built-in ya custom, sync ya async — **same contract** follow karta hai (`null` for valid, `ValidationErrors` for invalid). Isi liye Angular unhe ek array mein interchangeably chala sakta hai. Agar koi custom validator contract tode (e.g., `throw` kare ya `undefined` return kare), poora form engine break ho jata hai — **LSP violation in action**.

**Bhai, Frontend Pe Yeh Kaise Handle Karein**:
Template-driven (`ngModel` + attributes) chhoti forms ke liye theek, lekin complex/testable validation ke liye **Reactive Forms** use karo — rules TypeScript mein, unit-testable. Async uniqueness pe **debounce** lagao warna har keystroke pe API call (DDoS apne hi server pe). Aur yaad rakho: yeh sab **UX** hai, security backend pe.

**Code Pattern**:
```typescript
@Component({ selector: 'app-address-form', templateUrl: './address-form.html' })
export class AddressFormComponent {
  form = this.fb.group({
    name:  ['', [Validators.required, Validators.maxLength(100)]],
    email: ['',
      [Validators.required, Validators.email],
      [this.emailTaken()]           // async validator (3rd arg)
    ],
    phone: ['', [Validators.required, Validators.pattern(/^03\d{9}$/)]],
    country: ['PK', Validators.required],
    postalCode: ['']
  }, { validators: [this.postalRule] });   // cross-field group validator

  constructor(private fb: FormBuilder, private http: HttpClient) {}

  // Cross-field validator — SAME ValidatorFn contract (LSP)
  private postalRule(group: AbstractControl): ValidationErrors | null {
    const country = group.get('country')?.value;
    const postal = group.get('postalCode')?.value;
    if ((country === 'PK' || country === 'SA') && !/^\d{5}$/.test(postal)) {
      return { postalInvalid: true };
    }
    return null;
  }

  // Async validator with debounce — returns null (valid) or error object
  private emailTaken(): AsyncValidatorFn {
    return (ctrl: AbstractControl): Observable<ValidationErrors | null> =>
      timer(400).pipe(   // debounce: wait 400ms after last keystroke
        switchMap(() => this.http.get<boolean>(`/api/addresses/email-exists?e=${ctrl.value}`)),
        map(exists => (exists ? { emailTaken: true } : null)),
        catchError(() => of(null))   // network fail = don't block UX; server re-checks anyway
      );
  }

  submit() {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    this.http.post('/api/addresses', this.form.value).subscribe({
      next: () => { /* refresh list */ },
      error: (err) => {
        // Server-side errors map back to fields (source of truth)
        if (err.status === 400 && err.error?.errors) {
          Object.entries(err.error.errors).forEach(([field, msg]) =>
            this.form.get(field)?.setErrors({ server: msg }));
        }
      }
    });
  }
}
```

**UX Concern**:
Bina proper frontend handling ke: user poora form bharta hai, Save dabata hai, phir page reload ke baad pata chalta hai email galat tha — **frustrating**. Inline + on-blur validation se user ko turant pata chalta hai. Lekin over-eager validation (har keystroke pe red) bhi bura — isliye `updateOn: 'blur'` ya debounce.

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: **Defense in Depth** (multi-layer validation) + **Single Source of Truth** for rules + async uniqueness ke TOCTOU race ko DB constraint se settle karna.

**Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai**:
Validation ek hi jagah nahi, **har trust boundary** pe hoti hai, lekin alag maqsad se:
- **Client (Angular)** → fast UX feedback. **Trust level: ZERO.**
- **Edge / API Gateway** → schema/shape validation (JSON Schema, OpenAPI). Malformed/oversized payload yahin reject.
- **Service layer (Spring/.NET)** → field + business rules (asli security boundary).
- **Database** → integrity constraints (last line, race-proof uniqueness).

**Single source of truth ka dard**: agar Angular mein "phone 11 digit" aur backend mein "10 digit" likha ho, to user confuse. Solution: rules ek jagah define karo aur share — e.g., backend OpenAPI spec se frontend types generate karo, ya shared validation schema (JSON Schema / Zod). Stripe/GitHub backend ko **single arbiter** rakhte hain; frontend bas mirror.

**Bhai, Architecture Level Pe Yeh Kaise Design Karein**:
Socho ek **mehman ka ghar mein dakhil hona**: (1) gate pe chowkidar shakal dekh ke andaza lagata hai (Angular — fast, but fool ho sakta hai), (2) reception pe ID card check (Backend — asli verification), (3) andar locker room mein biometric (DB — final, koi bypass nahi). Har layer ka apna kaam; ek pe bharosa karke baaki hata do to security toot jati hai.

**Architecture Diagram**:
```
┌─────────────┐   POST    ┌──────────────┐  INSERT  ┌──────────────┐
│   Angular   │──────────▶│ Spring/.NET  │─────────▶│   SQL DB     │
│ ReactiveForm│  (bypassable!)│  @Valid    │          │ UNIQUE/CHECK │
└─────────────┘           └──────────────┘          └──────────────┘
   UX only                  SECURITY boundary          INTEGRITY guarantee
   - sync validators        - Bean Validation          - NOT NULL
   - async email (debounce) - cross-field constraint   - UNIQUE (race-proof)
   - trust: 0               - @ControllerAdvice 400    - CHECK constraints
        │                          │                          │
        └────────── attacker can curl past Angular ──────────┘
                    => server + DB MUST re-validate
```

**Trade-offs**:
| Approach | Pros | Cons |
|----------|------|------|
| Validate ONLY on client | Fast, no server load | Zero security — trivially bypassed (curl/Postman) |
| Validate ONLY on server | Secure | Bad UX — round-trip for every error, no instant feedback |
| Validate ONLY in DB | Integrity guaranteed | Ugly errors, late failure, business rules awkward in SQL |
| **Defense in depth (all layers)** | UX + security + integrity | Rule duplication risk → needs single source of truth |

**Real Companies Using This**:
- **Stripe**: every field server-validated, structured `param`+`code` errors; client mirrors.
- **GitHub**: async username availability (UX) + unique DB index (truth) → clean race handling.
- **Daraz/FoodPanda**: client instant feedback + backend re-validation + DB unique on phone/email.

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Cross-Questioning Proof)

**Today's OOP/Pattern Focus**: **Liskov Substitution Principle (L of SOLID)**

### Principle/Pattern Definition

**Concept**: LSP (Barbara Liskov, 1987) kehta hai — *agar `S` type `T` ka subtype hai, to program mein `T` ke objects ko `S` ke objects se replace karne par program ka **correctness toot nahi sakta**.* Yani: **subtype apne base ki jagah, bina kisi surprise ke, kaam kar sake.**

Formal rules (yeh interview mein bolne layak):
- **Preconditions strengthen nahi kar sakte** — subtype base se *zyada* demand na kare.
- **Postconditions weaken nahi kar sakte** — subtype base se *kam* guarantee na de.
- **Invariants preserve hone chahiye** — base ke rules subtype bhi maane.
- **No new exceptions** — subtype aisi exception na phenke jo base ke contract mein nahi.

**Bhai, Simple Mein Samjho**:
LSP ka matlab: *beta baap ki kursi pe baith sakta hai bina logon ko pareshaan kiye.* Agar tumne `Validator` interface banaya jisme `validate()` "valid ya error return karta hai, kabhi crash nahi", to har subtype validator wahi vaada nibhaye. Agar koi subtype `validate()` mein exception phenk de jab base "error return" karta tha — usne base ki jagah lene se inkaar kar diya. Yeh LSP violation hai, aur polymorphic code (jo sab validators ko ek loop mein chalata hai) **toot jata hai**.

**Real-Life Analogy (Pakistani Context)**:
"Bhai, cricket ka **substitute (12th man)** socho. Rule: substitute fielding kar sakta hai — catch le, throw kare, run bachaye. Ab agar substitute aa ke **batting** shuru kar de ya **bowling** karne lage, to woh '12th man' ke contract ko tod raha hai — umpire match rok dega. LSP yahi kehta hai: subtype ko base ki **poori jagah** leni hai, **usi rules** ke saath — na zyada haq jatana (precondition strengthen), na apna kaam adhoora chhodna (postcondition weaken). Agar substitute base ki jagah fit nahi baith raha, to woh sahi subtype hai hi nahi."

Classic example: **Square extends Rectangle** — math mein square ek rectangle hai, lekin code mein `rect.setWidth(10)` karne pe Square chupke se height bhi 10 kar deta hai. Jo caller `Rectangle` samajh ke `setWidth` aur `setHeight` alag-alag set karta tha, uska assumption toot gaya. **Square, Rectangle ki jagah substitute nahi ho sakta** → LSP violation.

---

### Aaj Ke Brainer Mein Yeh Pattern/Principle Kahan Hai?

Form validation engine **LSP ka living example** hai. Socho ek `FieldValidator` hierarchy:

```
Validator (base contract: validate(value) -> ValidationResult, NEVER throws for "invalid")
   ├── RequiredValidator
   ├── EmailValidator
   ├── PhoneValidator
   └── PostalCodeValidator
```

Controller / form engine in sab ko **polymorphically** ek list mein chalata hai:
```java
List<Validator> validators = List.of(new RequiredValidator(), new EmailValidator(), ...);
for (Validator v : validators) results.add(v.validate(value)); // base ki jagah koi bhi subtype
```

Yeh loop sirf tab kaam karta hai jab **har subtype base ka contract nibhaye** — sab `ValidationResult` return karein (kabhi `null`, kabhi `throw`). Agar `EmailValidator` empty input pe `NullPointerException` phenk de jabki base "return invalid-result" karta tha → loop crash, baaki validators chalte hi nahi. **Yeh exactly woh LSP violation hai jo validation systems ko todta hai.** Angular ka `ValidatorFn` contract (`null | ValidationErrors`) bhi yahi LSP enforce karta hai.

---

### Code Mein Yeh Principle Kaise Apply Hota Hai?

**❌ BAD (Principle Violated)**:
```java
abstract class Validator {
    // Contract: return result; empty errors = valid. NEVER throw for invalid input.
    abstract ValidationResult validate(String value);
}

class EmailValidator extends Validator {
    @Override
    ValidationResult validate(String value) {
        // VIOLATION 1: strengthens precondition — base accepts any String incl. null/empty
        if (value == null || value.isBlank())
            throw new IllegalArgumentException("value required"); // VIOLATION 2: new exception
        if (!value.contains("@"))
            return ValidationResult.error("Invalid email");
        return ValidationResult.ok();
    }
}

// Caller written against base — assumes "validate never throws, returns result":
for (Validator v : validators) {
    ValidationResult r = v.validate(input); // 💥 input="" → EmailValidator THROWS
    collect(r);                              // never reached; loop dies; other fields unvalidated
}
```
Kyun bura: caller ne base contract pe code likha ("har validator result deta hai"). `EmailValidator` ne base ki jagah lene se **inkaar** kar diya — extra precondition (non-empty) lagayi aur naya exception phenka. Result: poora validation loop crash, ya har caller ko `try/catch` daalna parta (defeats polymorphism).

**✅ GOOD (Principle Followed)**:
```java
abstract class Validator {
    abstract ValidationResult validate(String value); // contract: always returns, never throws
}

class EmailValidator extends Validator {
    @Override
    ValidationResult validate(String value) {
        // Honors base contract: accepts ANYTHING base accepts (incl. null/empty)
        if (value == null || value.isBlank())
            return ValidationResult.error("Email is required");  // returns, doesn't throw
        if (!value.matches("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"))
            return ValidationResult.error("Invalid email format");
        return ValidationResult.ok();
    }
}
// Now ANY subtype is safely substitutable in the loop. No try/catch needed. No surprises.
```

---

### Java vs .NET Mein Yeh Principle Kaise Implement Hota Hai?

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Substitutable contract | `interface Validator { ValidationResult validate(T v); }` | `interface IValidator<T> { ValidationResult Validate(T v); }` |
| Base method | `abstract` / interface method | `abstract` / interface method |
| Override keyword | `@Override` (optional, compiler-checked) | `override` (required, explicit) |
| Strengthen-precondition guard | discipline + contract tests | discipline + contract tests |
| No-new-checked-exceptions | enforced for checked exceptions by compiler | C# has no checked exceptions (more discipline needed) |
| Variance | invariant generics; use-site wildcards | declaration-site `in`/`out` variance |

```csharp
public interface IValidator<T> {
    // Contract: returns result; never throws for "invalid"
    ValidationResult Validate(T value);
}

public class EmailValidator : IValidator<string> {
    public ValidationResult Validate(string value) {
        if (string.IsNullOrWhiteSpace(value))   // accepts what base accepts
            return ValidationResult.Error("Email is required");
        if (!value.Contains('@'))
            return ValidationResult.Error("Invalid email");
        return ValidationResult.Ok();
    }
}
// All IValidator<string> impls interchangeable in: foreach (var v in validators) ...
```

---

### 🎯 Cross-Questioning Drill (Interview Defense)

**Cross Q1**: "Yeh LSP kyun zaroori? Bina iske kya hota?"
**Confident Answer**:
"Bhai, LSP **polymorphism ko safe** banata hai. Validation engine sab validators ko ek list mein chalata hai (`for (Validator v : list) v.validate(x)`). Agar koi subtype base ka contract tode — jaise empty input pe exception phenke jabki base 'invalid-result return' karta hai — to caller ka assumption galat ho jata hai aur loop crash ho jata hai. Bina LSP, har callsite pe `instanceof` check ya `try/catch` daalna parta, jo polymorphism ka pura faida khatam kar deta. LSP ke saath, naya validator add karna **zero risk** hai — woh base ki jagah seamlessly fit ho jata hai (yeh OCP — Day 12 — se bhi connected hai)."

---

**Cross Q2**: "Tumne LSP follow kiya — kya alternative tha?"
**Confident Answer**:
"Alternative: inheritance hata ke har validator ko alag-alag concrete class rakho aur callsite pe explicit `if/else` se decide karo kaunsa chalana hai. Lekin yeh OCP tod deta hai aur duplicate ho jata. Doosra alternative: **composition over inheritance** (Day 5) — base class ke bajaye chhote interfaces compose karo. Aksar LSP violation ka asli ilaaj inheritance ko **interface/composition** se replace karna hota hai (e.g., Square-Rectangle problem: dono ko `Shape` interface se compose karo, ek doosre se inherit mat karao). Main inheritance tabhi rakhta hun jab subtype **sach mein** base ke har behaviour ko honor kar sake."

---

**Cross Q3**: "LSP violate kab kar sakte ho jaan-boojh ke?"
**Confident Answer**:
"Honestly, LSP ko jaan-boojh ke todna almost hamesha bad design ka signal hai — yeh OCP/DRY jaisa 'pragmatic trade-off' principle nahi. Lekin agar legacy hierarchy mein ek subtype genuinely base ke ek operation ko support nahi karta (classic `UnsupportedOperationException` jaise Java ke immutable `List` mein `add()`), to woh **conscious LSP violation** hai jise documentation aur `@throws` se clearly batana parta hai. Sahi fix: hierarchy ko split karna (`ReadOnlyList` vs `MutableList`) — jo ISP (Day 14) se connect karta hai. To main 'violate' nahi karta, main **hierarchy redesign** karta hun."

---

**Cross Q4**: "Yeh principle Spring/Angular mein automatically follow hota hai ya manual?"
**Confident Answer**:
"Frameworks LSP ko **expect** karte hain lekin enforce poori tarah nahi karte. Spring ka `ConstraintValidator` aur Angular ka `ValidatorFn` ek **fixed contract** dete hain — agar tum us contract ko honor karo (return result/null, don't throw randomly), sab interchangeable hain. Lekin agar tum custom validator mein contract tod do (e.g., Angular validator `undefined` return kare bajaye `null`, ya exception phenke), framework ka engine misbehave karega — Spring 500 de dega, Angular ka form status stuck ho jayega. To framework contract **deta** hai, LSP **honor karna hamari zimmedari** hai."

---

**Cross Q5**: "Production mein LSP scale kaise karta hai?"
**Confident Answer**:
"Microservices mein LSP **API contract level** pe apply hota hai. Agar `PaymentServiceV2` `PaymentServiceV1` ko replace kar raha hai, to V2 ko V1 ke saare requests handle karne chahiye bina naye mandatory fields maange (precondition strengthen na ho) aur same response guarantees deni chahiye (postcondition weaken na ho) — warna purane clients toot jayenge. Yeh **backward compatibility** ka core hai. CRDTs/consensus tak nahi jaate — bas: 'naya version purane ki jagah, bina surprise'. Contract tests (Pact) yeh LSP-at-scale verify karte hain."

---

### 🧩 Pattern Combination (How Today's Pattern Pairs With Others)

**Pairs Well With**:
- **OCP (Day 12)** — kyunki naya validator add karna tabhi safe hai jab subtypes substitutable hon. LSP, OCP ko *correct* banata hai.
- **Strategy Pattern (Day 62)** — har validator ek strategy hai; LSP guarantee karta hai strategies interchangeable hain.
- **Composition over Inheritance (Day 5)** — LSP violations ka common ilaaj inheritance ko composition se replace karna.

**Conflicts With / Tension With**:
- **Deep inheritance hierarchies** — jitni gehri inheritance, utna LSP todna aasan. Flat hierarchies + interfaces behtar.
- **Premature abstraction (YAGNI, Day 18)** — ek hi implementation ke liye base type banana over-engineering; LSP tab matter karta hai jab 2+ subtypes hon.

---

### 🎓 Real Production Code Where This Matters

Java Collections Framework LSP ka classic case study hai (achha + bura dono):
- **Achha**: `List<T>` ke saare implementations (`ArrayList`, `LinkedList`) base contract honor karte hain — `add`, `get`, `size` sab predictable. Tum `List<T>` likho, koi bhi impl daal do, code chalta hai.
- **Bura (jaan-boojh ke documented)**: `Collections.unmodifiableList()` aur `Arrays.asList()` ke `add()` pe `UnsupportedOperationException` aati hai. Yeh **technically LSP violation** hai (subtype base ka operation support nahi karta) — isliye Java docs explicitly warn karti hain. Lesson: agar subtype base ka koi operation nahi de sakta, to shayad hierarchy galat hai.

Spring/Angular validator engines isi liye **narrow, throw-free contracts** rakhte hain — taa-ke har validator sach mein substitutable rahe.

---

### 💡 Memory Hook for This Principle/Pattern

- **LSP**: **"NAYA KHILARI, WOHI RULES"** — substitute player base ki jagah aaye to usi rulebook pe khele; na zyada demand, na kam delivery.
- Aur: **"BETA BAAP KI KURSI PE — BINA GADBAD"** — child class parent ki jagah baithe, koi caller pareshaan na ho.

---

### ⚠️ Common Mistakes Jo Aaj Bachne Hain

**❌ Mistake 1**: Subtype mein extra precondition lagana (e.g., subtype validator empty input reject karta hai jabki base accept karta tha).
**Why it's wrong**: Caller base contract pe likhta hai; subtype zyada demand kare to substitute karne pe code toot jata hai (precondition strengthening).
**Correct approach**: Subtype kam-se-kam utna accept kare jitna base; rules sirf andar ki logic mein.

**❌ Mistake 2**: Subtype mein nayi exception phenkna jo base ke contract mein nahi (jaise validator jo `throw` kare jab base "return error-result" karta hai).
**Why it's wrong**: Polymorphic caller un exceptions ke liye taiyar nahi → crash ya har callsite pe `try/catch`.
**Correct approach**: Base ke declared behaviour ke andar raho — invalid input pe result return karo, exception sirf truly exceptional (programmer error) ke liye.

**❌ Mistake 3**: "IS-A" sochke inheritance laga dena (Square IS-A Rectangle) bina behaviour check kiye.
**Why it's wrong**: Real-world "is-a" hamesha behavioural substitutability guarantee nahi karta. Square `setWidth` pe height bhi badal deta — caller ka assumption toot.
**Correct approach**: "IS-SUBSTITUTABLE-FOR" test karo, sirf "IS-A" nahi. Doubt ho to inheritance ke bajaye composition/interface use karo.

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom):

```
1. USER ACTION (Angular)
   └─▶ Form bharta hai → sync validators (red/instant) + async email check (debounce)
       └─▶ UX only, trust = 0

2. API REQUEST (Spring Boot / .NET)
   └─▶ @Valid / [ApiController] → Bean Validation / ModelState
       └─▶ Security boundary: garbage/curl-attack yahin ruka

3. BUSINESS LOGIC (Java / C#)
   └─▶ Cross-field rule (Saudi postal = 5 digit) via class-level constraint
       └─▶ Validators polymorphic, LSP-clean (substitutable, throw-free)

4. DATABASE (SQL)
   └─▶ NOT NULL / CHECK / UNIQUE → integrity + race-proof uniqueness
       └─▶ Duplicate-key handled gracefully (not a 500)

5. ARCHITECTURE (System Design)
   └─▶ Defense in depth + single source of truth for rules
       └─▶ Server = single arbiter; client mirrors

6. RESPONSE (All layers)
   └─▶ Valid → 201 Created → Angular list refresh
   └─▶ Invalid → 400 + field errors → Angular maps to fields
```

**What Breaks If You Skip ANY Layer**:
- Skip **Angular**: kaam karega but UX terrible (har error ke liye round-trip).
- Skip **Backend**: SECURITY HOLE — attacker `curl` se garbage daal dega; SQLi/XSS ka rasta.
- Skip **Business rule layer**: structurally valid but business-invalid data (UAE address with fake postal).
- Skip **SQL constraints**: ek app bug ya direct DB access permanent corrupt data; email duplicate race.
- Skip **System design**: rules diverge (FE 10 digit, BE 11) → user confusion, support tickets.

---

## 🧭 MENTAL MAP — How to Memorize This

```
                [FORM VALIDATION BRAINER]
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   [Frontend]         [Backend]         [Database]
        │                 │                 │
    Angular          Spring/.NET           SQL
        │                 │                 │
  ReactiveForm        @Valid /         NOT NULL
  sync+async         [ApiController]    UNIQUE (race-proof)
  debounce           cross-field        CHECK
  (UX, trust=0)      (SECURITY)         (INTEGRITY)
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                  [SYSTEM DESIGN]
            Defense in depth + Single source of truth
                          │
                  [OOP: LSP]
        Validators substitutable → polymorphic loop safe
```

**Mental Story to Remember (Roman Urdu)**:
"Imagine karo ek **VIP shaadi** mein dakhil hona:
- Angular = **Gate pe chowkidar** — shakal dekh ke andaza (fast, lekin fool ho sakta hai).
- Spring/.NET = **Reception pe ID check** — asli verification, yahan se fake nikal jata hai.
- SQL = **Andar biometric lock** — koi bypass nahi, aakhri deewar.
- System Design = **Pura security plan** — har layer ka apna kaam, ek pe bharosa karke baaki hata do to system toot jaye.
- LSP = **Substitute guard** — agar ek guard ki jagah doosra aaye, usay **wahi rules** follow karne honge, warna security chaos.

Agar koi bhi layer missing, shaadi mein gatecrasher ghus jayega."

**Acronym/Mnemonic**:
**"CASE-D"** = **C**lient (UX) → **A**PI (security) → **S**ervice (business rule) → **E**nforce in DB (integrity) → **D**efense-in-depth. Aur LSP ka hook: **"NAYA KHILARI, WOHI RULES"**.

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Tum ek signup/checkout form bana rahe ho. Validation kahan-kahan karoge aur kyun? Aur uniqueness (email already exists) kaise guarantee karoge?"

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Validation ek single jagah ka kaam nahi — main ise **defense in depth** ke taur pe har trust boundary pe karunga, har layer ka alag maqsad."

**Body** (60 sec):
1. **Frontend** (Angular): Reactive Forms se instant UX feedback, async email-check with debounce — **lekin yeh sirf UX hai, trust zero.**
2. **Backend API** (Spring/.NET): `@Valid` / `[ApiController]` se Bean Validation / ModelState — **yeh asli security boundary hai**, kyunki attacker `curl` se Angular bypass kar sakta hai.
3. **Business Logic** (Java/C#): cross-field rules (country-specific postal code) class-level constraint se.
4. **Database** (SQL): `NOT NULL`/`CHECK`/`UNIQUE` — last line of defense, integrity guarantee.
5. **Uniqueness**: client check sirf UX; asli guarantee **DB `UNIQUE` constraint** kyunki check-then-insert mein TOCTOU race hai. Duplicate-key error ko gracefully handle karunga."

**Closing** (10 sec):
"Server ko **single source of truth** rakhunga, client bas mirror. Isse UX + security + integrity teeno milte hain bina kisi single point of failure ke."

### Under-the-Hood Concepts You MUST Know

1. **Bean Validation flow**: `@Valid` → Hibernate Validator → reflection → `ConstraintValidator.isValid()` → `MethodArgumentNotValidException` → `@ControllerAdvice`.
2. **TOCTOU race**: SELECT-exists then INSERT mein gap; do concurrent users same email; sirf DB unique index race-proof.
3. **Unique constraint = unique B-tree index**: insert pe key lock; duplicate → 2627 (SQL Server)/1062 (MySQL).
4. **Angular validator pipeline**: sync first, async only if sync pass; status `VALID/INVALID/PENDING`; contract `null | ValidationErrors`.
5. **LSP**: subtype substitutability — no strengthened preconditions, no weakened postconditions, no new exceptions.

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

**Counter Q1 (Trade-off focused)**: "Agar backend already validate kar raha, to frontend validation ki zaroorat hi kya hai? Duplication nahi?"
**Your Answer**:
"Duplication ka risk sach hai, lekin dono ka maqsad alag hai. Frontend validation **UX** ke liye — user ko turant feedback, round-trip bachana, server load kam. Backend **security** ke liye — non-negotiable. Duplication ko main **single source of truth** se manage karunga: backend OpenAPI/JSON Schema se frontend validation rules generate karna, ya shared schema (Zod/JSON Schema). To rule ek jagah, dono jagah enforce. Bina frontend ke app secure to hoga lekin UX bekaar; bina backend ke fast lekin insecure."

---

**Counter Q2 (Scale focused)**: "10,000 users ek saath same promo pe register kar rahe hain. Email uniqueness kaise handle karoge?"
**Your Answer**:
"App-level 'email exists?' SELECT at scale **useless** hai — TOCTOU race har jagah. Main DB **UNIQUE constraint** pe rely karunga jo unique index se atomic uniqueness deta hai, isolation level se independent. INSERT pe duplicate-key error (2627/1062) ko catch karke clean 409/400 return karunga, 500 nahi. Async email-check sirf UX. Agar throughput aur barhe, to email ko normalize karke (lowercase, gmail dots strip) store karunga taa-ke `a.b@gmail` aur `ab@gmail` duplicate na banein. Idempotency keys (Day 39) bhi double-submit se bachate hain."

---

**Counter Q3 (Failure mode focused)**: "Async email validator ke time backend down ho jaye to kya hoga?"
**Your Answer**:
"Async validator ko **fail-open** design karunga UX ke liye — network error pe `catchError(() => of(null))`, yani field ko block nahi karta, kyunki yeh sirf UX check hai. Asli enforcement DB unique constraint karega submit pe. Agar main fail-closed karta (error pe field invalid mark), to backend hiccup pe user form submit hi nahi kar pata — bad UX. Submit pe agar backend down hai to retry-with-backoff (Day 65 concept) + user ko clear error. Important: async check ka result kabhi 'final truth' nahi maanunga."

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "Tumne `Square extends Rectangle` likha. Kya yeh sahi design hai?"
**Junior Answer**: "Haan, square ek rectangle hi hota hai (is-a), to inheritance sahi hai."
**Senior Answer**: "Math mein is-a sahi hai, lekin **behavioural substitutability** nahi — yeh classic LSP violation hai. `Rectangle r = new Square(); r.setWidth(5); r.setHeight(10);` — caller expect karta hai width 5, height 10, area 50. Lekin Square `setHeight(10)` pe width bhi 10 kar deta (invariant: all sides equal), area 100 nikal aata. Jo code Rectangle ke contract pe bharosa karke likha tha, woh silently toot gaya. Sahi design: dono ko ek `Shape` interface se compose karo, ek doosre se inherit mat karao. Yeh batata hai ke 'is-a' relationship inheritance ka sufficient test nahi — 'is-substitutable-for' asli test hai."

### Red Flag Signals (Don't Say These!)

- ❌ "Frontend pe validation laga di, kaafi hai" — Why: zero security, curl/Postman se bypass.
- ❌ "Email exists check karke phir insert kar dunga" — Why: TOCTOU race; concurrent duplicate ghus jayega.
- ❌ "DB constraints ki zaroorat nahi, app validate kar raha" — Why: ek bug/direct access = permanent corrupt data.
- ❌ "Square is-a Rectangle to inheritance theek hai" — Why: behavioural substitutability fail (LSP).
- ❌ "Validator empty input pe exception phenk dega" — Why: LSP violation, polymorphic loop crash.

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ Validation har layer pe kyun (client=UX, server=security, DB=integrity)?
2. ✅ `@Valid`/`[ApiController]` internally kaise kaam karta hai (reflection, ConstraintValidator, advice)?
3. ✅ Email uniqueness ka TOCTOU race aur DB UNIQUE constraint kyun asli guarantee hai?
4. ✅ LSP kya hai, aur validation engine usse kaise demonstrate karta hai?
5. ✅ Square-Rectangle LSP violation aur uska sahi fix (composition/interface)?

---

## 🔗 Tomorrow's Connection Hint

Tomorrow we'll cover: **Day 14 — Email Verification Flow** (OOP: **Interface Segregation Principle, I of SOLID**).

Aaj ke concept se kaise connected hai:
Aaj hum ne LSP dekha — subtypes base ki jagah fit hon. Kal ISP: **fat interfaces** mat banao; clients ko sirf woh methods milein jo woh use karte hain. Notice karo: aaj jab maine kaha "agar subtype base ka koi operation support nahi kar sakta to hierarchy split karo" — woh exactly ISP ka rasta hai. LSP aur ISP behno jaisi hain. Aur email verification flow validation se naturally aage badhta hai — user ne valid email diya, ab verify karo woh sach mein uska hai.

---

## 📚 Progress Tracker

```
🟢 Beginner     ██████████████░░░░░░  Day 13/20   ← CURRENT
🟡 Intermediate ░░░░░░░░░░░░░░░░░░░░  Day 0/30
🟠 Advanced     ░░░░░░░░░░░░░░░░░░░░  Day 0/40
🔴 Expert       ░░░░░░░░░░░░░░░░░░░░  Day 0/50
⚫ Master       ░░░░░░░░░░░░░░░░░░░░  Day 0/60
🟣 Architect    ░░░░░░░░░░░░░░░░░░░░  Day 0/70
💎 Principal    ░░░░░░░░░░░░░░░░░░░░  Day 0/95
```

**Overall**: Day 13 of 365 (3.6% complete) — 🔥 13-day streak
