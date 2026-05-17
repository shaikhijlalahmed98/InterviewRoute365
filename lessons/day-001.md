# 🎯 🟢 Day 1 of Beginner (Level 1 of 7): User Registration with Email Verification

**Overall Day**: Day 1 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 1 of 20 in this level
**Today's Theme**: Naya user kaise register hota hai, email kaise verify hoti hai — full stack, ek saath!

---

## 📖 The Brainer Scenario (Real Problem)

Daraz pe naya seller signup kar raha hai. Usne email aur password dala. Ab system ko yeh sab karna hai:

1. User ko database mein save karna hai
2. Verification email bhejni hai ek unique link ke saath
3. Jab tak email verify na ho — user login **nahi** kar sakta
4. Verification link sirf **24 ghante** valid rahe

**The Real Challenge (The "Gotcha")**:
Agar email bhejne mein fail ho jaye — kya user database mein already save ho chuka hai? Kya woh dobara register kar sakta hai ya system "email already exists" bolta hai? Token expire ho jaye toh? Yeh teeno scenarios production mein din mein kaafi baar hote hain.

**Why this matters in production**:
GitHub ka production system mein email verification idempotent hai — same email pe multiple registrations gracefully handled hoti hain. GitHub ke verification emails exactly 24 hours mein expire hote hain, aur unka resend flow rate-limited hai (spam prevention). Stripe bhi exactly yahi pattern follow karta hai apni merchant onboarding mein.

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: `@Transactional`, `JavaMailSender`, `BCryptPasswordEncoder`, UUID token generation

**Under the Hood — Yeh Kaise Kaam Karta Hai**:
Spring Boot ka `@Transactional` annotation AOP (Aspect-Oriented Programming) proxy create karta hai. Jab tum `registerUser()` call karte ho, Spring actually ek **proxy object** call karta hai jo real method se pehle transaction start karta hai aur baad mein commit ya rollback decide karta hai.

`BCryptPasswordEncoder` deliberately slow hai (cost factor = 10 by default, yaani 2^10 = 1024 iterations). Yeh brute-force attacks slow karta hai. Salt automatically generate hota hai — isliye same password ka har baar **alag hash** aata hai.

`UUID.randomUUID()` cryptographically secure random 122-bit number generate karta hai. Collision probability itni low hai ki practically impossible — isliye verification token ke liye perfect hai.

**Bhai, Simple Mein Samjho**:
Socho tum ek daftar ke manager ho. Naya employee aaya — tum pehle uska record banao (DB save), phir usse ID card courier karo (email bhejo). Agar courier fail ho jaye (SMTP down), record delete kar do (rollback). Yahi `@Transactional` karta hai — ya **sab hoga** ya **kuch nahi**!

**Code Pattern**:
```java
@Service
public class UserRegistrationService {

    @Autowired private UserRepository userRepository;
    @Autowired private EmailService emailService;
    @Autowired private PasswordEncoder passwordEncoder;

    @Transactional  // AOP proxy wraps entire method — all or nothing
    public RegistrationResponse registerUser(RegistrationRequest request) {
        // Step 1: Duplicate check (DB UNIQUE constraint also guards this)
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new EmailAlreadyExistsException("Email already registered");
        }

        // Step 2: Build unverified user entity
        User user = new User();
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword())); // BCrypt
        user.setVerified(false);

        // Step 3: Generate 24-hour token
        String token = UUID.randomUUID().toString();
        user.setVerificationToken(token);
        user.setTokenExpiry(LocalDateTime.now().plusHours(24));

        // Step 4: Save FIRST (within transaction)
        userRepository.save(user);

        // Step 5: Send email — if this throws, @Transactional rolls back the save above
        emailService.sendVerificationEmail(user.getEmail(), token);

        return new RegistrationResponse("Registration successful! Check your email.");
    }

    @Transactional
    public void verifyEmail(String token) {
        User user = userRepository.findByVerificationToken(token)
            .orElseThrow(() -> new InvalidTokenException("Invalid or already used token"));

        if (user.getTokenExpiry().isBefore(LocalDateTime.now())) {
            throw new TokenExpiredException("Token expired. Request a new verification email.");
        }

        user.setVerified(true);
        user.setVerificationToken(null); // Single-use: clear after verify
        user.setTokenExpiry(null);
        userRepository.save(user);
    }
}
```

**Interview phrasing**:
"Iss scenario mein main Spring ka `@Transactional` use karunga kyunki user DB save aur email send ek **atomic operation** honi chahiye — agar email fail ho, user DB mein persist nahi hona chahiye, warna user stuck ho jaata hai (already exists but unverified)."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: `IEmailSender`, `BeginTransactionAsync()`, Entity Framework Core, `IPasswordHasher<T>`, `Guid`

**Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai**:
.NET mein tum explicitly `_context.Database.BeginTransactionAsync()` use karo. EF Core ka **Change Tracker** entities ke states track karta hai (`Added`, `Modified`, `Unchanged`). Jab `SaveChangesAsync()` call hoti hai, EF Core Change Tracker se sab changes read karke SQL generate karta hai — ek batch mein.

ASP.NET Core's DI container `DbContext` ko **Scoped** lifetime mein manage karta hai — ek HTTP request = ek DbContext instance. Yeh thread safety ensure karta hai.

**Bhai, .NET Mein Yeh Kaise Hota Hai**:
Java mein tum annotation se transaction declare karo (`@Transactional`), .NET mein tum explicitly `BeginTransactionAsync()` use karo. Dono ka result same — **atomicity**! Bas syntax alag hai.

**Code Pattern**:
```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IUserService _userService;

    public AuthController(IUserService userService) => _userService = userService;

    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegistrationRequest request)
    {
        try
        {
            var result = await _userService.RegisterUserAsync(request);
            return Ok(result);
        }
        catch (EmailAlreadyExistsException)
        {
            return Conflict(new { message = "Email already registered" });
        }
    }

    [HttpGet("verify")]
    public async Task<IActionResult> VerifyEmail([FromQuery] string token)
    {
        await _userService.VerifyEmailAsync(token);
        return Ok(new { message = "Email verified successfully!" });
    }
}

public class UserService : IUserService
{
    private readonly AppDbContext _context;
    private readonly IEmailService _emailService;
    private readonly IPasswordHasher<User> _passwordHasher;

    public UserService(AppDbContext context, IEmailService emailService,
        IPasswordHasher<User> passwordHasher)
    {
        _context = context;
        _emailService = emailService;
        _passwordHasher = passwordHasher;
    }

    public async Task<RegistrationResponse> RegisterUserAsync(RegistrationRequest request)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();
        try
        {
            if (await _context.Users.AnyAsync(u => u.Email == request.Email))
                throw new EmailAlreadyExistsException();

            var user = new User
            {
                Email = request.Email,
                PasswordHash = _passwordHasher.HashPassword(null!, request.Password),
                IsVerified = false,
                VerificationToken = Guid.NewGuid().ToString(),
                TokenExpiry = DateTime.UtcNow.AddHours(24)
            };

            _context.Users.Add(user);
            await _context.SaveChangesAsync();

            await _emailService.SendVerificationEmailAsync(user.Email, user.VerificationToken);

            await transaction.CommitAsync();
            return new RegistrationResponse("Check your email!");
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

**Java vs .NET Comparison Table**:

| Feature | Java / Spring Boot | .NET / C# |
|---|---|---|
| Transaction | `@Transactional` (AOP, declarative) | `BeginTransactionAsync()` (explicit) |
| DI | `@Autowired` / Constructor | Constructor injection (preferred) |
| ORM | JPA / Hibernate | Entity Framework Core |
| Password Hash | `BCryptPasswordEncoder` | `IPasswordHasher<T>` |
| UUID / GUID | `UUID.randomUUID()` | `Guid.NewGuid()` |
| Mail | `JavaMailSender` | `IEmailSender` / MailKit |
| Async | `CompletableFuture` | `async / await` |
| HTTP Status | `ResponseEntity.status(409)` | `return Conflict(...)` |

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: `UNIQUE` constraint, filtered index on token, UTC datetime, `@@ROWCOUNT` check

**Under the Hood — SQL Engine Yeh Kaise Karta Hai**:
SQL Server mein `UNIQUE` constraint pe **B-Tree index** automatically banta hai. Jab `INSERT` hoti hai, SQL engine pehle index traverse karta hai — O(log n) — duplicate check ke liye. Agar mila, constraint violation throw hoti hai **before the row is written**. Yeh application-level check se zyada reliable hai.

Filtered index (`WHERE verification_token IS NOT NULL`) sirf unverified users ke tokens index karta hai — space efficient aur fast lookup.

**Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai**:
Socho database ek sarkari register ki tarah hai. Ek CNIC (email) ek baar hi likha ja sakta hai — UNIQUE constraint yahi karta hai. Token ek temporary visitor pass ki tarah hai jo 24 ghante ke baad invalid ho jata hai.

**SQL Example**:
```sql
-- Users table
CREATE TABLE users (
    id                 BIGINT IDENTITY(1,1) PRIMARY KEY,
    email              NVARCHAR(255) NOT NULL,
    password_hash      NVARCHAR(500) NOT NULL,
    is_verified        BIT NOT NULL DEFAULT 0,
    verification_token NVARCHAR(36) NULL,
    token_expiry       DATETIME2 NULL,
    created_at         DATETIME2 NOT NULL DEFAULT GETUTCDATE(),

    CONSTRAINT UQ_users_email UNIQUE (email)
);

-- Filtered index: only index unverified users' tokens (space efficient)
CREATE INDEX IX_users_verification_token
ON users (verification_token)
WHERE verification_token IS NOT NULL;

-- Registration: Insert new unverified user
INSERT INTO users (email, password_hash, is_verified, verification_token, token_expiry)
VALUES (@email, @passwordHash, 0, @token, DATEADD(HOUR, 24, GETUTCDATE()));

-- Verify: atomic check + update
UPDATE users
SET    is_verified        = 1,
       verification_token = NULL,
       token_expiry       = NULL
WHERE  verification_token = @token
  AND  token_expiry       > GETUTCDATE()
  AND  is_verified        = 0;

-- 0 rows = invalid, expired, or already verified
IF @@ROWCOUNT = 0
    RAISERROR('Token invalid, expired, or already used', 16, 1);

-- Resend: generate fresh token
UPDATE users
SET    verification_token = @newToken,
       token_expiry       = DATEADD(HOUR, 24, GETUTCDATE())
WHERE  email       = @email
  AND  is_verified = 0;
```

**The Gotcha**:
Agar `verification_token` pe index na ho aur 500K unverified users ho — har verification request pe **full table scan** hogi. 500K rows = slow query = bad UX under load. Filtered index is the elegant fix.

**Isolation Level Choice**:
`READ COMMITTED` (default) yahan kaafi hai. UNIQUE constraint at DB level duplicate prevention handle karta hai — application-level lock ki zaroorat nahi.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Reactive Forms, `FormGroup`, custom validators, `catchError`, `finalize`, route redirect

**Under the Hood — Angular Yeh Kaise Karta Hai**:
Angular Reactive Forms ek `FormGroup` mein `FormControl`s ka tree banata hai. Validators **pure functions** hain — control value le ke `null` (valid) ya error object return karte hain.

`HttpClient.post()` ek **cold Observable** return karta hai — request tab shuru hogi jab `subscribe()` karoge. `finalize` **hamesha** chalti hai — success ya error dono cases mein. Isliye `isSubmitting = false` ko `finalize` mein rakhna sahi hai.

**Bhai, Frontend Pe Yeh Kaise Handle Karein**:
Frontend ka kaam: form valid hai? Submit button disable karo jab submitting. Error aaye toh user-friendly message dikhao. Success pe redirect karo.

**Code Pattern**:
```typescript
// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private http: HttpClient) {}

  register(data: RegistrationRequest): Observable<RegistrationResponse> {
    return this.http.post<RegistrationResponse>('/api/auth/register', data);
  }

  verifyEmail(token: string): Observable<void> {
    return this.http.get<void>(`/api/auth/verify?token=${token}`);
  }
}

// registration.component.ts
@Component({
  selector: 'app-registration',
  templateUrl: './registration.component.html'
})
export class RegistrationComponent {
  form: FormGroup;
  isSubmitting = false;
  successMessage = '';
  errorMessage = '';

  constructor(private fb: FormBuilder, private auth: AuthService, private router: Router) {
    this.form = this.fb.group({
      email:           ['', [Validators.required, Validators.email]],
      password:        ['', [Validators.required, Validators.minLength(8)]],
      confirmPassword: ['', Validators.required]
    }, { validators: this.passwordMatchValidator });
  }

  passwordMatchValidator(group: FormGroup) {
    const pass    = group.get('password')?.value;
    const confirm = group.get('confirmPassword')?.value;
    return pass === confirm ? null : { passwordMismatch: true };
  }

  onSubmit(): void {
    if (this.form.invalid) return;

    this.isSubmitting = true;
    this.errorMessage = '';

    this.auth.register(this.form.value).pipe(
      catchError(err => {
        this.errorMessage = err.status === 409
          ? 'Yeh email pehle se registered hai!'
          : 'Kuch masla ho gaya, thodi der baad try karo.';
        return EMPTY;
      }),
      finalize(() => this.isSubmitting = false) // Always re-enable button
    ).subscribe(() => {
      this.successMessage = 'Registration hogaya! Email check karo.';
      setTimeout(() => this.router.navigate(['/login']), 3000);
    });
  }
}
```

```html
<!-- registration.component.html -->
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <input formControlName="email" type="email" placeholder="Email address">
  <span *ngIf="form.get('email')?.invalid && form.get('email')?.touched">
    Valid email address dalo!
  </span>

  <input formControlName="password" type="password" placeholder="Password (min 8 chars)">
  <input formControlName="confirmPassword" type="password" placeholder="Confirm Password">

  <span *ngIf="form.errors?.['passwordMismatch'] && form.get('confirmPassword')?.touched">
    Passwords match nahi kar rahe!
  </span>

  <button type="submit" [disabled]="form.invalid || isSubmitting">
    {{ isSubmitting ? 'Registering...' : 'Create Account' }}
  </button>

  <p class="success" *ngIf="successMessage">{{ successMessage }}</p>
  <p class="error"   *ngIf="errorMessage">{{ errorMessage }}</p>
</form>
```

**UX Concern**:
Agar submit button disable na karo during submission — user bar bar click karega, server pe **duplicate requests** jayengi. Always `[disabled]="isSubmitting"` lagao!

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: Synchronous registration + async email queue (Transactional Outbox for production)

**Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai**:
Seedha approach: user save karo, email bhejo — sab ek HTTP request mein. Problem: SMTP server slow ho toh registration slow lagti hai. Better: **Transactional Outbox Pattern** — user save karo aur SAME transaction mein `email_outbox` table mein record insert karo. Alag worker emails bhejta hai with retry.

**Architecture Diagram**:
```
Simple (Day 1):
Angular → API → DB → SMTP → Response

Production (Outbox Pattern):
┌──────────────────┐         ┌─────────────────────────┐
│   Angular App    │──POST──▶│   Auth Service          │
│  (Reactive Form) │◀──200───│   (Spring Boot / .NET)  │
└──────────────────┘         └────────────┬────────────┘
                                          │
                              ┌───────────▼────────────┐
                              │     SQL Database       │
                              │  users table           │
                              │  email_outbox table ←──┼── Same transaction
                              └───────────┬────────────┘
                                          │
                              ┌───────────▼────────────┐
                              │   Email Worker         │
                              │   Polls outbox table   │
                              │   Sends via SendGrid   │
                              │   Retries on failure   │
                              └────────────────────────┘
```

**Trade-offs**:

| Approach | Pros | Cons |
|---|---|---|
| Sync Email (Simple) | Simple, easy to implement | Slow if SMTP laggy; SMTP failure = rollback |
| Async Queue (Good) | Fast response, email retries independently | User saved but email might delay |
| Transactional Outbox (Best) | Guaranteed delivery, no lost emails, fast | Most complex: needs outbox table + worker |

**Real Companies Using This**:
- **GitHub**: Async email with dedicated email worker service, rate-limited resend
- **Stripe**: Verification emails via SendGrid asynchronously, Transactional Outbox internally
- **Shopify**: Sidekiq (job queue) for all transactional emails — never blocks HTTP response

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom):

```
1. USER ACTION (Angular)
   └─▶ Registration form fill karta hai
       └─▶ Reactive Form real-time validation (email format, password match)
           └─▶ Submit click → button disabled (isSubmitting = true)
               └─▶ HTTP POST /api/auth/register

2. API REQUEST (Spring Boot / .NET)
   └─▶ Controller @Valid DTO validate karta hai
       └─▶ 400 Bad Request if DTO invalid
           └─▶ Passes to UserRegistrationService

3. BUSINESS LOGIC (Java / C#)
   └─▶ @Transactional / BeginTransactionAsync() opens transaction
       └─▶ Duplicate email check → 409 Conflict if exists
           └─▶ BCrypt password hash
               └─▶ UUID/GUID token generate (24hr expiry)
                   └─▶ Save user to DB
                       └─▶ Send email (or push to outbox)

4. DATABASE (SQL)
   └─▶ INSERT hits UNIQUE constraint on email
       └─▶ Filtered index on verification_token → fast lookup
           └─▶ token_expiry stored as UTC DATETIME2

5. EMAIL (System Design)
   └─▶ Email service sends link: https://app.com/verify?token=UUID
       └─▶ Worker retries if SMTP fails (outbox pattern)
           └─▶ 24-hour expiry enforced in DB at verify time

6. RESPONSE (All layers)
   └─▶ DB commit → Service returns → API 200 OK
       └─▶ Angular shows "Check your email!" (isSubmitting = false)
           └─▶ After 3 seconds, router.navigate(['/login'])
```

**What Breaks If You Skip ANY Layer**:

| Skipped Layer | What Breaks |
|---|---|
| Angular validation | Server flooded with invalid requests, UX terrible |
| `@Transactional` | User saved in DB but email failed → stuck forever |
| UNIQUE constraint | Duplicate users in DB, data corruption |
| Token expiry | Old links work forever → security vulnerability |
| Async email queue | Registration slow, poor UX during SMTP issues |

---

## 🧭 MENTAL MAP — How to Memorize This

```
              [USER REGISTRATION + EMAIL VERIFY]
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
     [Frontend]       [Backend]          [Database]
         │                 │                 │
     Angular          Java/.NET            SQL
         │                 │                 │
   Reactive Form    @Transactional      UNIQUE email
   Password Match   UUID Token          Filtered Index
   Error Handling   BCrypt Hash         UTC Expiry
   Submit Disable   Email Send          @@ROWCOUNT Check
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │
                   [SYSTEM DESIGN]
              Sync Email vs Async Queue
              Transactional Outbox Pattern
              Rate-Limited Resend
```

**Mental Story (Roman Urdu)**:
Imagine karo tum Daraz pe naya seller account bana rahe ho:

- **Angular** = Daraz ka registration page (form bhar, galat email type karo — red border aayegi!)
- **Spring/.NET** = Customer care officer (info check karta hai, duplicate dhundta hai)
- **Java/C#** = Officer ka kaam: password lock mein bandh karo (BCrypt), temporary entry pass banao (UUID token)
- **SQL** = Sarkari register (ek email = ek account, UNIQUE constraint, token 24 ghante ka)
- **System Design** = TCS courier jo confirmation letter bhejti hai (fast ya queue mein?)

Agar officer ka kaam adhoora raha (email fail), sarkari register se entry delete ho jati hai (@Transactional rollback). Phir tum dobara apply kar sakte ho!

**Acronym/Mnemonic — R.U.V.E.S**:
- **R**eactive Form (Angular — validate first)
- **U**nique Constraint (SQL — no duplicates)
- **V**erification Token (UUID, 24hr expiry)
- **E**mail Send (async/queue for production)
- **S**ecure Hash (BCrypt — never plain text)

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Ek secure user registration system design karo jisme email verification bhi ho. Production challenges aur architecture discuss karo."

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Is problem mein hum 5 layers coordinate karenge — frontend validation se lekar database constraints aur async email delivery tak, sab milke ek reliable aur secure flow banana hai..."

**Body** (60 sec):
1. **Frontend** (Angular): "Reactive Forms se real-time validation — email format, password match, min length. Submit button disable during submission."
2. **Backend API** (Spring/.NET): "`@Valid` DTO validation, phir `@Transactional` annotation jo ensure karta hai user save aur email send ek atomic unit hai."
3. **Business Logic** (Java/C#): "BCrypt se password hash, `UUID.randomUUID()` se 36-char token 24-hour expiry ke saath."
4. **Database** (SQL): "UNIQUE constraint on email, filtered index on verification_token, `@@ROWCOUNT` check verify step mein."
5. **Architecture**: "Production mein Transactional Outbox pattern — user save aur outbox table update ek transaction mein. Worker async email bhejta hai with retries."

**Closing** (10 sec):
"Is approach se registration reliable hai under high load, email delivery guaranteed hai even if SMTP temporarily down ho, aur user experience fast aur smooth rahe."

---

### Under-the-Hood Concepts You MUST Know

1. **`@Transactional` AOP Proxy**: Spring ek proxy class generate karta hai jo actual service wrap karta hai. Isliye `@Transactional` **same class ke andar** call karne pe kaam nahi karta — self-invocation proxy bypass kar deta hai. Common interview gotcha!

2. **BCrypt Internals**: Deliberately slow. Salt auto-generated + embedded in hash string — `matches()` method verify karte waqt salt extract kar leta hai. Same password ka har baar alag hash, lekin `matches()` phir bhi `true`.

3. **UUID v4 Collision Safety**: 122-bit random. 1 trillion tokens/second for 1 billion years — collision probability negligible. Perfect for tokens.

4. **SQL UNIQUE Constraint vs Application Check**: Application check race condition ka shikar ho sakta hai — two concurrent requests both check (no duplicate), both insert — duplicate! DB UNIQUE constraint atomic hai — sirf ek winner.

5. **Angular `finalize` vs `catchError`**: `catchError` sirf errors pe. `finalize` **hamesha** — success ya error. `isSubmitting = false` `finalize` mein rakhna sahi hai — button hamesha re-enable hoga.

---

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

**Counter Q1 (Trade-off focused)**: "Agar email send fail ho jaye transaction ke andar aur rollback ho, toh user phir dobara register karna chahe — experience smooth hoga?"

**Your Answer**:
"Pure rollback ka masla: user register karta hai, email fail hoti hai, sab rollback, user ko generic 'Server Error' milta hai — confused.

Better: **Transactional Outbox Pattern**. User save karo DB mein, aur SAME transaction mein `email_outbox` table mein row insert karo. Transaction commit — dono DB writes guaranteed. Background worker `email_outbox` monitor karta hai, emails bhejta hai with exponential backoff retry.

Benefits: Registration fast (no SMTP wait), user confirmed registration milti hai, email delivery independently retry hoti hai even agar SMTP 30 min down rahe. GitHub aur Stripe yahi karte hain."

---

**Counter Q2 (Scale focused)**: "Daraz pe Black Friday pe 50,000 registrations per hour ho rahi hain. Kya changes karoge?"

**Your Answer**:
"50K/hour = ~14/second average, peak pe 5-10x zyada.

**Database**: UNIQUE constraint aur filtered index already optimized. HikariCP connection pool tune — `maximumPoolSize = 20-30` per instance.

**Email**: Async queue mandatory. 14 emails/second — dedicated email worker instances. SendGrid ya AWS SES (managed, scalable). Batch processing from outbox table.

**Application**: Multiple Spring Boot instances (Kubernetes HPA auto-scale on CPU). Stateless — tokens DB ya Redis mein.

**Rate Limiting**: Same IP 10+ registrations per 5 min? Block + CAPTCHA. Bot prevention zaruri.

**Redis for Tokens**: 50K tokens/hour, 24hr TTL — Redis mein store karo instead of SQL. Faster lookup, automatic expiry, DB pressure kam."

---

**Counter Q3 (Failure mode focused)**: "User ka verification email 24 ghante baad expire ho gaya. Kya scenarios handle karne hain?"

**Your Answer**:
"Teen scenarios aur security implications:

**Resend Verification**: `/resend-verification` endpoint. Check: email exists AND `is_verified = 0`. Haan: new UUID, DB update, fresh email.

**Security — Rate Limit Resend**: 3 resends per hour per email max. Warna email bombing attack. Redis pe `resend_count:{email}` TTL ke saath track karo.

**Security — Token Rotation**: Resend pe PURANA token immediately NULL. Agar purana valid rahe aur attacker ne intercept kiya — still usable. New token = old token automatically invalid.

**Cleanup Job**: Unverified accounts 7+ days purane — daily cron soft-delete kare. `deleted_at = now()`, not hard DELETE.

**Email Enumeration Prevention**: Response SAME dikhao chahe email registered ho ya na ho. Always: 'If this email is registered and unverified, we sent a new link.'"

---

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "Token-based link verification aur OTP verification mein se kya choose karoge email registration ke liye?"

**Junior Answer**: "Token better hai kyunki link mein sirf click karna hota hai."

**Senior Answer**:
"Dono ke alag use cases hain:

**Token (Link-based)**:
- Pros: One-click UX, mobile-friendly, 24hr validity handles timezone differences
- Cons: Phishing-like appearance, link tampering without HMAC signing
- When: Email verification, password reset, low-stakes flows

**OTP (6-digit code)**:
- Pros: Phishing-resistant, works great in apps, short validity
- Cons: Poor UX (typing), brute-force risk (rate-limit needed), expires too fast
- When: Login 2FA, payment confirmation, high-security actions

**My choice for email registration**: Token-based link — UX better, low-stakes flow, 24hr handles timezones.

**Production hardening**:
- HMAC-sign token: `token = UUID + HMAC(UUID, secret)` — prevent guessing attacks
- HTTPS only
- Single-use: clear from DB immediately after verification
- Don't put userId in URL — token only — prevent user enumeration

GitHub exact yahi karta hai: link for email verify, TOTP for login 2FA."

---

### Red Flag Signals (Don't Say These!)

- ❌ "Main sirf frontend pe validation karunga" — Why: Frontend bypass ho sakta hai (Postman direct API call). Server-side validation **always mandatory**.
- ❌ "Password plain text store karo, baad mein encrypt kar lenge" — Why: Critical security failure. BCrypt **registration ke moment pe**. Plain text = GDPR violation + catastrophic breach.
- ❌ "Email fail ho toh user ko retry button dikhao" — Why: User stuck agar DB mein save hai but email nahi gayi. Better: rollback ya outbox pattern.

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ `@Transactional` Spring mein kaise kaam karta hai, AOP proxy ka role, aur self-invocation bug kya hai?
2. ✅ BCrypt hashing kyun use karte hain, salt kya hota hai, aur plain text storage kyun never acceptable?
3. ✅ SQL `UNIQUE` constraint vs application-level check — kyun DB constraint zyada reliable hai (race condition)?
4. ✅ Angular Reactive Forms mein custom validators kaise likhte hain aur `finalize` vs `catchError` ka fark?
5. ✅ Sync email vs Async Queue vs Transactional Outbox — kab kya choose karein?

---

## 🔗 Tomorrow's Connection Hint

**Tomorrow**: Day 2 — **User Login with JWT (JSON Web Tokens)**

Aaj ke concept se connection:
Aaj humne user register kiya, email verify ki, aur `is_verified = 1` set kiya. Kal dekhenge ki verified user kaise securely login karta hai. JWT token ka internal structure (header.payload.signature), Spring Security ka `UsernamePasswordAuthenticationFilter` internals, aur .NET mein `JwtBearerAuthentication` middleware. Aaj ka `is_verified` field kal directly use hoga — unverified users ko JWT generate nahi hoga, `403 Forbidden` milega!

---

## 📚 Progress Tracker

```
🟢 Beginner     [█░░░░░░░░░░░░░░░░░░░] Day  1/20  ← YOU ARE HERE
🟡 Intermediate [░░░░░░░░░░░░░░░░░░░░] Day  0/30  (Locked - starts Day 21)
🟠 Advanced     [░░░░░░░░░░░░░░░░░░░░] Day  0/40  (Locked - starts Day 51)
🔴 Expert       [░░░░░░░░░░░░░░░░░░░░] Day  0/50  (Locked - starts Day 91)
⚫ Master       [░░░░░░░░░░░░░░░░░░░░] Day  0/60  (Locked - starts Day 141)
🟣 Architect    [░░░░░░░░░░░░░░░░░░░░] Day  0/70  (Locked - starts Day 201)
💎 Principal    [░░░░░░░░░░░░░░░░░░░░] Day  0/95  (Locked - starts Day 271)

Overall: Day 1 of 365 (0.3% complete — journey shuru hogaya, bhai! 💪)
```
