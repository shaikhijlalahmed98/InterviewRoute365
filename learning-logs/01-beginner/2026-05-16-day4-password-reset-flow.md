# 🎯 🟢 Day 4 of Beginner (Level 1 of 7): Password Reset Flow (Secure Token-Based)

**Overall Day**: Day 4 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 4 of 20 in this level
**Today's Theme**: Secure password reset using one-time tokens — handling expiry, single-use, timing attacks, and notifier abstraction via Polymorphism.

---

## 📖 The Brainer Scenario (Real Problem)

Bhai, scenario yeh hai: **Daraz** pe ek user ka account hai, kal raat tak login ho raha tha, aaj subah password bhool gaya. User "Forgot Password?" link click karta hai, apna email daalta hai, aur expect karta hai:

1. Ek **reset link** uske email pe aaye (kabhi SMS pe bhi — depending on user preference).
2. Link 15-30 minute mein expire ho jaye (security).
3. Link sirf **ek baar** use ho sake (replay attack se bachne ke liye).
4. Agar email exist nahi karta system mein — phir bhi same response aaye (taake hacker pata na lagaye ke kaunsa email registered hai — yeh **user enumeration** attack hota hai).
5. Naya password set hone ke baad, **purane sessions/JWTs invalid** ho jayein (taake agar hacker logged in tha, woh nikal jaye).

**The Real Challenge (The "Gotcha")**:
- **Timing attack**: Agar tum `if (user == null) return` quickly aur registered user pe slow response do, hacker timing measure karke pata laga sakta hai ke email registered hai ya nahi.
- **Token storage**: Raw token DB mein store karoge to DB breach = sab tokens leak. Hash karke store karna parta hai (BCrypt/SHA-256).
- **Single-use guarantee**: Race condition — do tabs mein simultaneously link click ho gaya, dono valid ho gaye? Atomic update chahiye.
- **Notification flexibility**: Aaj sirf email, kal SMS, parsoo WhatsApp via Twilio, ya push notification. Notifier ko **pluggable** banana parta hai — yahin **Polymorphism** chamakta hai.

**Why this matters in production**:
- **Stripe** ka password reset 1 hour expiry use karta hai with HMAC-signed tokens.
- **GitHub** sends reset email even for non-existent emails (same response) to prevent enumeration.
- **Google** invalidates all OAuth tokens + active sessions on password change.
- **AWS Cognito** uses one-time codes with hashed storage + automatic single-use marking.

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: Secure random token generation (`SecureRandom`), token hashing, atomic single-use enforcement via JPA optimistic locking, pluggable notifier via `@Qualifier` or strategy pattern.

**Under the Hood — Yeh Kaise Kaam Karta Hai**:

Spring Boot mein `SecureRandom` (CSPRNG — Cryptographically Secure Pseudo-Random Number Generator) JVM ke `java.security.SecureRandom` ko wrap karta hai. Linux pe yeh `/dev/urandom` se entropy uthata hai. Normal `Random` class **predictable** hai — usko **never** use karo tokens ke liye.

Spring's `@Async` annotation method ko **proxy** karta hai — call karte time CGLib/JDK proxy intercept karta hai aur task ko `TaskExecutor` thread pool pe dispatch karta hai. Isi liye email sending main request thread ko block nahi karti.

`PasswordEncoder` interface ka `BCryptPasswordEncoder` implementation uses adaptive hashing (cost factor 10-12), with built-in salt. Same plaintext → different hash every time (kyunki salt random hota hai).

**Bhai, Simple Mein Samjho**:

Token banao secure random se (32 bytes = 256 bits — practically uncrackable). Token ka **hash** DB mein save karo, raw token email mein bhejo. Jab user link click kare, raw token ka hash banake DB mein compare karo. Match hua + expired nahi + used nahi → naya password set karne do, phir token ko `used=true` mark karo **atomically** (taake double-click attack na ho).

**Code Pattern**:

```java
// ============ ENTITY ============
@Entity
@Table(name = "password_reset_tokens", indexes = {
    @Index(name = "idx_token_hash", columnList = "tokenHash", unique = true),
    @Index(name = "idx_user_expiry", columnList = "userId, expiresAt")
})
public class PasswordResetToken {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, length = 64)
    private String tokenHash;  // SHA-256 hex of raw token

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private Instant expiresAt;

    @Column(nullable = false)
    private boolean used = false;

    @Version
    private Long version;  // Optimistic locking — single-use ki guarantee
}

// ============ SERVICE ============
@Service
@RequiredArgsConstructor
public class PasswordResetService {
    private final UserRepository userRepo;
    private final PasswordResetTokenRepository tokenRepo;
    private final PasswordEncoder passwordEncoder;
    private final SessionService sessionService;
    private final NotificationSender notifier;  // Polymorphic!
    private final SecureRandom random = new SecureRandom();

    @Transactional
    public void requestReset(String email) {
        // Constant-time response — same path whether user exists or not
        Optional<User> userOpt = userRepo.findByEmail(email);
        
        if (userOpt.isPresent()) {
            User user = userOpt.get();
            
            // 1. Generate raw token (32 bytes → 43-char URL-safe Base64)
            byte[] tokenBytes = new byte[32];
            random.nextBytes(tokenBytes);
            String rawToken = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
            
            // 2. Hash before storing (DB breach → tokens still safe)
            String tokenHash = sha256Hex(rawToken);
            
            // 3. Invalidate previous unused tokens for this user
            tokenRepo.markAllUserTokensAsUsed(user.getId());
            
            // 4. Save new token (15-min expiry)
            PasswordResetToken token = new PasswordResetToken();
            token.setUserId(user.getId());
            token.setTokenHash(tokenHash);
            token.setExpiresAt(Instant.now().plus(15, ChronoUnit.MINUTES));
            tokenRepo.save(token);
            
            // 5. Send notification — polymorphic dispatch happens here!
            String resetLink = "https://daraz.pk/reset?token=" + rawToken;
            notifier.send(user, new PasswordResetMessage(resetLink));
        }
        
        // Always return success — no enumeration!
        // Even if user doesn't exist, sleep a tiny amount to equalize timing
    }

    @Transactional
    public void confirmReset(String rawToken, String newPassword) {
        String tokenHash = sha256Hex(rawToken);
        
        // Atomic: find unused, non-expired token AND mark as used
        PasswordResetToken token = tokenRepo
            .findByTokenHashAndUsedFalseAndExpiresAtAfter(tokenHash, Instant.now())
            .orElseThrow(() -> new InvalidTokenException("Token invalid or expired"));
        
        User user = userRepo.findById(token.getUserId())
            .orElseThrow(() -> new InvalidTokenException("User not found"));
        
        // Update password
        user.setPasswordHash(passwordEncoder.encode(newPassword));
        userRepo.save(user);
        
        // Mark token used — @Version protects against race
        token.setUsed(true);
        tokenRepo.save(token);  // Throws OptimisticLockException if concurrent click
        
        // Invalidate ALL active sessions/JWTs for this user
        sessionService.invalidateAllSessions(user.getId());
    }
    
    private String sha256Hex(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(md.digest(input.getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

**Interview phrasing**:
"Iss scenario mein main raw token sirf email mein bhejunga aur DB mein uska SHA-256 hash store karunga. Single-use guarantee `@Version` (optimistic locking) se enforce hoti hai, aur user enumeration roknay ke liye main success response always return karunga, chahe email registered ho ya na ho."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: `RandomNumberGenerator`, `IPasswordHasher<TUser>`, EF Core `[ConcurrencyCheck]` / `[Timestamp]`, DI container ke through `INotificationSender` ka polymorphic resolution.

**Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai**:

.NET mein `RandomNumberGenerator.Fill()` Windows pe CNG (Cryptography Next Generation) aur Linux pe `/dev/urandom` use karta hai — same CSPRNG quality jo Java mein hai. EF Core ka concurrency token (rowversion) SQL Server mein `ROWVERSION`/`TIMESTAMP` column use karta hai — SQL Server automatically increment karta hai jab row update hoti hai. Agar update query ka WHERE clause original rowversion match nahi karta, `DbUpdateConcurrencyException` throw hota hai.

.NET ke `Microsoft.Extensions.DependencyInjection` mein agar tum `IEnumerable<INotificationSender>` inject karo, woh **sab** registered implementations dega — perfect for multi-channel notification.

**Bhai, .NET Mein Yeh Kaise Hota Hai**:

Same logic — sirf syntax aur frameworks alag hain. EF Core `SaveChangesAsync()` mein agar `[Timestamp]` mismatch ho to exception throw karta hai, isse race condition catch hoti hai.

**Code Pattern**:

```csharp
// ============ ENTITY ============
public class PasswordResetToken
{
    public Guid Id { get; set; }
    public string TokenHash { get; set; } = default!;
    public long UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool Used { get; set; }
    
    [Timestamp]  // Optimistic concurrency
    public byte[] RowVersion { get; set; } = default!;
}

// ============ SERVICE ============
public class PasswordResetService
{
    private readonly AppDbContext _db;
    private readonly IPasswordHasher<User> _hasher;
    private readonly ISessionService _sessions;
    private readonly INotificationSender _notifier;  // Polymorphic

    public PasswordResetService(
        AppDbContext db,
        IPasswordHasher<User> hasher,
        ISessionService sessions,
        INotificationSender notifier)
    {
        _db = db; _hasher = hasher; _sessions = sessions; _notifier = notifier;
    }

    public async Task RequestResetAsync(string email)
    {
        var user = await _db.Users.FirstOrDefaultAsync(u => u.Email == email);

        if (user != null)
        {
            // Generate cryptographically secure token
            var tokenBytes = new byte[32];
            RandomNumberGenerator.Fill(tokenBytes);
            var rawToken = Base64UrlEncoder.Encode(tokenBytes);

            var tokenHash = Sha256Hex(rawToken);

            // Invalidate previous unused tokens
            await _db.PasswordResetTokens
                .Where(t => t.UserId == user.Id && !t.Used)
                .ExecuteUpdateAsync(s => s.SetProperty(t => t.Used, true));

            _db.PasswordResetTokens.Add(new PasswordResetToken
            {
                Id = Guid.NewGuid(),
                UserId = user.Id,
                TokenHash = tokenHash,
                ExpiresAt = DateTime.UtcNow.AddMinutes(15),
                Used = false
            });

            await _db.SaveChangesAsync();

            var resetLink = $"https://daraz.pk/reset?token={rawToken}";
            await _notifier.SendAsync(user, new PasswordResetMessage(resetLink));
        }
        // No enumeration leak — same response path
    }

    public async Task ConfirmResetAsync(string rawToken, string newPassword)
    {
        var tokenHash = Sha256Hex(rawToken);

        var token = await _db.PasswordResetTokens
            .FirstOrDefaultAsync(t =>
                t.TokenHash == tokenHash &&
                !t.Used &&
                t.ExpiresAt > DateTime.UtcNow);

        if (token == null)
            throw new InvalidTokenException("Token invalid or expired");

        var user = await _db.Users.FindAsync(token.UserId)
            ?? throw new InvalidTokenException("User not found");

        user.PasswordHash = _hasher.HashPassword(user, newPassword);
        token.Used = true;

        try
        {
            await _db.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            // Another click won the race — this one fails cleanly
            throw new InvalidTokenException("Token already used");
        }

        await _sessions.InvalidateAllSessionsAsync(user.Id);
    }

    private static string Sha256Hex(string input)
    {
        using var sha = SHA256.Create();
        var bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(bytes).ToLowerInvariant();
    }
}
```

**Java vs .NET Comparison Table**:

| Feature | Java/Spring | .NET/C# |
|---------|-------------|---------|
| Secure random | `SecureRandom.nextBytes()` | `RandomNumberGenerator.Fill()` |
| Password hashing | `BCryptPasswordEncoder` | `IPasswordHasher<TUser>` (PBKDF2 by default) |
| SHA-256 | `MessageDigest.getInstance("SHA-256")` | `SHA256.Create()` |
| Optimistic lock | `@Version` (JPA) | `[Timestamp]` / `[ConcurrencyCheck]` (EF Core) |
| Async dispatch | `@Async` + `TaskExecutor` | `async/await` + `Task` |
| Polymorphic DI | `@Qualifier` or `List<INotifier>` | `IEnumerable<INotifier>` |
| URL-safe Base64 | `Base64.getUrlEncoder()` | `Base64UrlEncoder.Encode` (IdentityModel) |

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: Atomic `UPDATE ... WHERE used = 0` for single-use enforcement, indexes on `token_hash` (unique) and `(user_id, expires_at)`, optionally row-versioning column.

**Under the Hood — SQL Engine Yeh Kaise Karta Hai**:

SQL Server mein `UPDATE` statement implicitly **U-locks** (Update locks) leta hai pehle, phir **X-locks** (Exclusive locks) mein convert karta hai actual modify ke time. Agar do simultaneous updates same row pe aayein, ek wait karega lock release ke liye, doosra serially execute hoga. Yeh **isolation level READ COMMITTED** mein bhi guarantee hai.

`UNIQUE INDEX` ke piché B+ tree hota hai. Insert ke time engine check karta hai ke same key already exist to nahi — agar hai to constraint violation throw karta hai. Composite index `(user_id, expires_at)` leftmost-prefix rule follow karta hai — query `WHERE user_id = ? AND expires_at > NOW()` super fast hoti hai kyunki index seek + range scan hota hai.

**Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai**:

Two flows:
1. **Insert token**: Pehle purane unused tokens ko `used=1` mark karo (taake user "Forgot Password" 5 baar click kare to sirf latest link valid ho), phir naya insert karo.
2. **Consume token**: **Atomic update** karo `WHERE used=0 AND expires_at > NOW()` — agar `@@ROWCOUNT = 1` to safe, `0` matlab token invalid/used/expired hai.

**SQL Example**:

```sql
-- Table definition
CREATE TABLE password_reset_tokens (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    user_id         BIGINT NOT NULL,
    token_hash      CHAR(64) NOT NULL,           -- SHA-256 hex
    expires_at      DATETIME2 NOT NULL,
    used            BIT NOT NULL DEFAULT 0,
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    row_version     ROWVERSION,                  -- Auto-managed for optimistic lock
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Unique index prevents duplicate hashes (collision check)
CREATE UNIQUE INDEX idx_token_hash ON password_reset_tokens(token_hash);

-- Composite index for "active token for user" queries
CREATE INDEX idx_user_active 
    ON password_reset_tokens(user_id, used, expires_at);

-- ============ ATOMIC SINGLE-USE CONSUMPTION ============
BEGIN TRANSACTION;

-- This UPDATE is atomic — either succeeds (rowcount=1) or fails (rowcount=0)
UPDATE password_reset_tokens
SET used = 1, used_at = SYSUTCDATETIME()
WHERE token_hash = @tokenHash
  AND used = 0
  AND expires_at > SYSUTCDATETIME();

IF @@ROWCOUNT = 0
BEGIN
    ROLLBACK TRANSACTION;
    THROW 51000, 'Token invalid, expired, or already used', 1;
END

-- Now safely update password
UPDATE users
SET password_hash = @newPasswordHash,
    password_changed_at = SYSUTCDATETIME()
WHERE id = (SELECT user_id FROM password_reset_tokens WHERE token_hash = @tokenHash);

COMMIT TRANSACTION;
```

**The Gotcha**:

Agar tum SELECT pehle karo, code mein check karo, phir UPDATE — yeh **TOCTOU** (Time Of Check, Time Of Use) bug hai. Do simultaneous requests dono ka SELECT pass ho jayega, dono UPDATE chala denge, password reset dohra ho jayega. Atomic `UPDATE ... WHERE used=0` se yeh issue eliminate hota hai.

**Isolation Level Choice**:

**READ COMMITTED** (SQL Server default) sufficient hai kyunki single-statement atomic UPDATE row-level X-lock leta hai. SERIALIZABLE overkill hai aur performance kharab karega. Agar app-level transaction long hai (multiple reads + update), tab REPEATABLE READ consider karo.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Reactive Forms with custom validators, RxJS `debounceTime` for password strength check, route param extraction for token, loading states, generic error message (no enumeration).

**Under the Hood — Angular Yeh Kaise Karta Hai**:

Angular Reactive Forms ke piché `FormControl` ek `Observable<value>` (`valueChanges`) expose karta hai. Jab tum `debounceTime(300)` pipe karte ho, RxJS internally `setTimeout` use karta hai aur previous emission cancel kar deta hai. Yeh **change detection** ko bhi optimize karta hai — Zone.js detect karta hai ke kuch change hua, lekin sirf affected components ko mark karta hai dirty.

`HttpClient` Observables **cold** hain — har subscriber pe naya request fire hota hai. `take(1)` ya `async` pipe automatically unsubscribe karta hai response ke baad.

**Bhai, Frontend Pe Yeh Kaise Handle Karein**:

User enumeration roknay ke liye **frontend pe bhi same message** dikhao chahe email valid ho ya invalid: "Agar yeh email registered hai, link bhej diya gaya hai." Token URL se uthao, naya password form dikhao, password strength real-time check karo (debounced).

**Code Pattern**:

```typescript
// ============ REQUEST RESET COMPONENT ============
@Component({
  selector: 'app-forgot-password',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" type="email" placeholder="Email" />
      <button [disabled]="form.invalid || loading">
        {{ loading ? 'Bhej raha hai...' : 'Reset Link Bhejo' }}
      </button>
    </form>
    <div *ngIf="submitted" class="success">
      Agar yeh email registered hai, reset link bhej diya gaya hai. Inbox check karein.
    </div>
  `
})
export class ForgotPasswordComponent {
  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]]
  });
  loading = false;
  submitted = false;

  constructor(private fb: FormBuilder, private auth: AuthService) {}

  submit() {
    this.loading = true;
    this.auth.requestPasswordReset(this.form.value.email!).pipe(
      finalize(() => this.loading = false)
    ).subscribe({
      next: () => this.submitted = true,
      // SAME message on error — never reveal whether email exists
      error: () => this.submitted = true
    });
  }
}

// ============ CONFIRM RESET COMPONENT ============
@Component({
  selector: 'app-reset-password',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="password" type="password" placeholder="Naya Password" />
      <div class="strength" [class]="strength$ | async"></div>
      <input formControlName="confirm" type="password" placeholder="Confirm" />
      <div *ngIf="form.errors?.['mismatch']" class="error">
        Passwords match nahi karte
      </div>
      <button [disabled]="form.invalid || loading">Set Password</button>
    </form>
  `
})
export class ResetPasswordComponent implements OnInit {
  form = this.fb.group({
    password: ['', [Validators.required, Validators.minLength(8)]],
    confirm: ['', Validators.required]
  }, { validators: this.matchValidator });

  loading = false;
  token!: string;
  strength$!: Observable<'weak' | 'medium' | 'strong'>;

  constructor(
    private fb: FormBuilder,
    private route: ActivatedRoute,
    private router: Router,
    private auth: AuthService
  ) {}

  ngOnInit() {
    this.token = this.route.snapshot.queryParamMap.get('token') ?? '';
    if (!this.token) {
      this.router.navigate(['/login']);
      return;
    }

    // Real-time strength meter — debounced to avoid spam
    this.strength$ = this.form.get('password')!.valueChanges.pipe(
      debounceTime(250),
      distinctUntilChanged(),
      map(pw => this.calculateStrength(pw ?? ''))
    );
  }

  submit() {
    this.loading = true;
    this.auth.confirmReset(this.token, this.form.value.password!).pipe(
      finalize(() => this.loading = false)
    ).subscribe({
      next: () => this.router.navigate(['/login'], { 
        queryParams: { resetSuccess: 'true' } 
      }),
      error: err => alert(err.error?.message ?? 'Link expired ho gaya. Dobara try karein.')
    });
  }

  private matchValidator(group: AbstractControl): ValidationErrors | null {
    const pw = group.get('password')?.value;
    const cf = group.get('confirm')?.value;
    return pw === cf ? null : { mismatch: true };
  }

  private calculateStrength(pw: string): 'weak' | 'medium' | 'strong' {
    if (pw.length < 8) return 'weak';
    const hasUpper = /[A-Z]/.test(pw);
    const hasLower = /[a-z]/.test(pw);
    const hasNum = /\d/.test(pw);
    const hasSym = /[^A-Za-z0-9]/.test(pw);
    const score = [hasUpper, hasLower, hasNum, hasSym].filter(Boolean).length;
    return score >= 3 && pw.length >= 12 ? 'strong' 
         : score >= 2 ? 'medium' : 'weak';
  }
}
```

**UX Concern**:

Agar tum frontend pe bhi error message different dikhao ("Email not found" vs "Reset sent"), to attacker DevTools open karke email enumeration kar lega. **Identical response** zaroori hai — UX team ko convince karna parta hai ke security UX se zyada important hai is case mein.

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: **Strategy Pattern** for notification dispatch + **Event-Driven** for async sending + **Token-Bucket Rate Limiter** for "Forgot Password" endpoint abuse.

**Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai**:

Strategy Pattern Polymorphism ka practical application hai — runtime pe algorithm interchange ho. Spring/​.NET DI containers strategies ko `Map<String, Strategy>` ya `IEnumerable<IStrategy>` ke through inject karte hain. Naya channel add karna ho — bas naya implementation likho aur container register kar deta hai, **zero existing code change**.

Rate limiting **token bucket algorithm** use karta hai: har user ke liye Redis mein ek counter, har request 1 token consume karta hai, har minute X tokens refill hote hain. Reset endpoint pe 3 attempts/hour cap reasonable hai.

**Bhai, Architecture Level Pe Yeh Kaise Design Karein**:

Reset request aaye to **synchronously** token banao + DB save karo, lekin email/SMS sending **async** ho event/queue ke through. Isse user response 50ms mein milta hai (SMTP slow ho sakta hai 2-5 seconds). Notification service crash ho jaye to **DLQ** (dead letter queue) mein chala jaye for retry.

**Architecture Diagram**:

```
┌────────────────────┐    ┌──────────────────────┐
│  Angular Frontend  │    │   API Gateway        │
│  /forgot-password  │───▶│   + Rate Limiter     │
│  /reset?token=...  │    │   (3 req/hour/IP)    │
└────────────────────┘    └──────────┬───────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │  Auth Service        │
                          │  (Spring / .NET)     │
                          │  - Generate token    │
                          │  - Hash + Save       │
                          │  - Publish event     │
                          └──┬──────────────┬────┘
                             │              │
                             ▼              ▼
                   ┌──────────────┐  ┌──────────────────┐
                   │  SQL DB      │  │  Kafka / SNS     │
                   │  - tokens    │  │  topic:          │
                   │  - users     │  │  password.reset  │
                   │  - sessions  │  └────────┬─────────┘
                   └──────────────┘           │
                                              ▼
                              ┌──────────────────────────────┐
                              │  Notification Service        │
                              │  ┌────────────────────────┐  │
                              │  │ INotificationSender    │  │
                              │  │  ├ EmailSender (SES)   │  │
                              │  │  ├ SmsSender (Twilio)  │  │
                              │  │  └ PushSender (FCM)    │  │
                              │  └────────────────────────┘  │
                              │  Strategy chosen by user pref│
                              └────────────┬─────────────────┘
                                           │
                                           ▼ (on failure)
                                  ┌─────────────────┐
                                  │  Dead Letter Q  │
                                  └─────────────────┘
```

**Trade-offs**:

| Approach | Pros | Cons |
|----------|------|------|
| **Sync email send** | Simple, immediate confirmation | API slow (SMTP delays), failure = user sees error |
| **Async via queue** | Fast response, retryable, scalable | Eventual delivery, debugging complex, infra cost |
| **JWT-based token (stateless)** | No DB lookup, scalable | Cannot revoke before expiry, replay-able if leaked |
| **Random + DB token** | Revocable, single-use enforced | DB hit per validation |

**Real Companies Using This**:
- **Stripe**: Async via internal Kafka, supports email + SMS, 1-hour expiry, HMAC tokens.
- **GitHub**: Sync send for email reset, 3-hour expiry, identical response for enumeration prevention.
- **Slack**: Magic-link login uses same pattern — random token, hashed in DB, single-use, async send via SendGrid.

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Cross-Questioning Proof)

**Today's OOP/Pattern Focus**: **Polymorphism (Method Overriding)** — Day 4 of OOP Fundamentals

### Principle/Pattern Definition

**Concept**: Polymorphism (literally "many shapes") means ek hi method call **different behavior** dikha sake based on the actual object type at runtime. **Method overriding** is the specific mechanism — subclass redefines a method declared in parent class/interface, aur compiler ko parwaah nahi, runtime pe **virtual dispatch** decide karta hai konsi implementation chalegi.

**Bhai, Simple Mein Samjho**:

Tumhare paas ek `NotificationSender` interface hai with method `send(user, message)`. Email, SMS, WhatsApp — sab classes implement karti hain. Tumhara service code sirf `sender.send(...)` call karta hai — usse pata bhi nahi konsa actual class chal raha hai. **Yeh hai Polymorphism** — same call, different behavior, decided at runtime.

**Real-Life Analogy (Pakistani Context)**:

Bhai, polymorphism aisa hai jaise **Careem app**. Tum "Ride book" button click karte ho — app ko parwaah nahi tum **Careem Go**, **Careem Plus**, ya **Rickshaw** book kar rahe ho. Sab ka method same hai: `bookRide()`. Lekin har vehicle ke driver ki, fare calculation, aur arrival time **alag** hota hai. Tumhara phone wahi button dabata hai (`bookRide`), backend mein vehicle type ke hisaab se **alag implementation** run hoti hai. **Bas yahi Polymorphism hai** — ek interface, kayi implementations, runtime pe decide.

Ya phir socho: tumhara cricket team mein 11 players hain. Coach bolta hai "Apni position pe khelo." Hare khilari ko `play()` ke instructions same hain, lekin batsman bat karta hai, bowler bowling, fielder catch. Same call, different behavior.

---

### Aaj Ke Brainer Mein Yeh Pattern/Principle Kahan Hai?

Aaj ke Password Reset scenario mein **Polymorphism literally har jagah** hai — yeh ek "live wire" example hai:

1. **`NotificationSender` interface** — `EmailNotificationSender`, `SmsNotificationSender`, `PushNotificationSender` sab ne `send()` method override kiya. `PasswordResetService` ko sirf interface ka reference hai. Future mein WhatsApp add karna ho? Naya class likho — service code change nahi hoga.

2. **`PasswordEncoder` (Spring) / `IPasswordHasher<T>` (.NET)** — `BCryptPasswordEncoder`, `Argon2PasswordEncoder`, `Pbkdf2PasswordEncoder` sab ne `encode()` aur `matches()` override kiye. Tum kal Argon2 pe migrate karna chahte ho, bas DI configuration badlo — service code untouched.

3. **`MessageDigest.getInstance("SHA-256")` vs `.getInstance("SHA-512")`** — JVM internally polymorphic dispatch karta hai, tum sirf algorithm name pass karte ho.

Polymorphism is the secret sauce that makes our `PasswordResetService` **open for extension, closed for modification** (foreshadowing Day 12 — Open/Closed Principle!).

---

### Code Mein Yeh Principle Kaise Apply Hota Hai?

**❌ BAD (Polymorphism Not Used — `if/else` Hell)**:

```java
@Service
public class PasswordResetService {
    
    public void sendResetNotification(User user, String link, String channel) {
        // 😱 Tight coupling, hard to extend, violates Open/Closed
        if (channel.equals("EMAIL")) {
            JavaMailSender mailSender = ...;
            MimeMessage msg = mailSender.createMimeMessage();
            // ... 20 lines of email-specific code
            mailSender.send(msg);
        } else if (channel.equals("SMS")) {
            TwilioRestClient twilio = ...;
            // ... 15 lines of SMS-specific code
            twilio.messages().create(...);
        } else if (channel.equals("PUSH")) {
            FirebaseMessaging fcm = ...;
            // ... 25 lines of push-specific code
            fcm.send(...);
        }
        // Naya channel add karna hai? Yahin if/else chain mein add karo — har baar service file modify!
    }
}
```

**Problems with this approach**:
- Naya channel = is class ko modify karna parega (Open/Closed violated).
- Unit testing impossible — saari dependencies hard-coded.
- Single Responsibility violated — ek class email + SMS + push sab kar rahi hai.
- Adding WhatsApp = touching 50+ lines of unrelated code.

**✅ GOOD (Polymorphism Used Properly)**:

```java
// ============ INTERFACE / ABSTRACT CONTRACT ============
public interface NotificationSender {
    void send(User user, NotificationMessage message);
    NotificationChannel getChannel();
}

// ============ IMPLEMENTATIONS — har one method override karta hai ============

@Component
@RequiredArgsConstructor
public class EmailNotificationSender implements NotificationSender {
    private final JavaMailSender mailSender;
    
    @Override
    public void send(User user, NotificationMessage message) {
        // Email-specific logic — encapsulated yahin
        MimeMessage mime = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(mime, true);
        try {
            helper.setTo(user.getEmail());
            helper.setSubject(message.getSubject());
            helper.setText(message.getHtmlBody(), true);
            mailSender.send(mime);
        } catch (MessagingException e) {
            throw new NotificationException("Email send failed", e);
        }
    }
    
    @Override
    public NotificationChannel getChannel() { return NotificationChannel.EMAIL; }
}

@Component
@RequiredArgsConstructor
public class SmsNotificationSender implements NotificationSender {
    private final TwilioClient twilio;
    
    @Override
    public void send(User user, NotificationMessage message) {
        // SMS-specific logic
        twilio.messages().create(
            new PhoneNumber(user.getPhone()),
            new PhoneNumber("+923001234567"),
            message.getSmsBody()
        );
    }
    
    @Override
    public NotificationChannel getChannel() { return NotificationChannel.SMS; }
}

// ============ CONSUMER — no knowledge of concrete types ============

@Service
@RequiredArgsConstructor
public class PasswordResetService {
    private final Map<NotificationChannel, NotificationSender> senders;
    
    // Spring magic: auto-injects ALL implementations as a map keyed by channel
    public PasswordResetService(List<NotificationSender> allSenders, ...) {
        this.senders = allSenders.stream()
            .collect(Collectors.toMap(NotificationSender::getChannel, s -> s));
    }
    
    public void sendResetNotification(User user, String link) {
        NotificationSender sender = senders.get(user.getPreferredChannel());
        // ⭐ POLYMORPHIC CALL — runtime decides which send() runs
        sender.send(user, new PasswordResetMessage(link));
    }
}
```

**Now adding WhatsApp = just write one new class. Zero modifications to existing code.** That's the power.

---

### Java vs .NET Mein Yeh Principle Kaise Implement Hota Hai?

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Override keyword | `@Override` annotation (compile hint, optional but recommended) | `override` keyword (**mandatory**) |
| Base method requirement | Methods are virtual by default | Base must be `virtual` or `abstract` |
| Hide vs override | Always overrides (no hiding) | `new` keyword to hide (different semantics) |
| Interface methods | Implicitly abstract | Implicit, or `default` (C# 8+) |
| Sealed override | `final` keyword on method | `sealed override` keyword |
| Polymorphic DI | `List<INotifier>` or `Map<Key, INotifier>` | `IEnumerable<INotifier>` or keyed services (.NET 8+) |

```csharp
// C# equivalent
public interface INotificationSender
{
    Task SendAsync(User user, NotificationMessage message);
    NotificationChannel Channel { get; }
}

public class EmailNotificationSender : INotificationSender
{
    private readonly IEmailClient _email;
    public NotificationChannel Channel => NotificationChannel.Email;

    public EmailNotificationSender(IEmailClient email) => _email = email;

    public async Task SendAsync(User user, NotificationMessage message)
    {
        await _email.SendAsync(user.Email, message.Subject, message.HtmlBody);
    }
}

public class SmsNotificationSender : INotificationSender
{
    private readonly ITwilioClient _twilio;
    public NotificationChannel Channel => NotificationChannel.Sms;
    public SmsNotificationSender(ITwilioClient twilio) => _twilio = twilio;

    public async Task SendAsync(User user, NotificationMessage message)
    {
        await _twilio.SendSmsAsync(user.Phone, message.SmsBody);
    }
}

// Registration in Program.cs (.NET 8+ keyed services)
builder.Services.AddKeyedScoped<INotificationSender, EmailNotificationSender>(NotificationChannel.Email);
builder.Services.AddKeyedScoped<INotificationSender, SmsNotificationSender>(NotificationChannel.Sms);

// Consumer
public class PasswordResetService
{
    private readonly IServiceProvider _sp;
    public PasswordResetService(IServiceProvider sp) => _sp = sp;

    public async Task NotifyAsync(User user, string link)
    {
        var sender = _sp.GetRequiredKeyedService<INotificationSender>(user.PreferredChannel);
        await sender.SendAsync(user, new PasswordResetMessage(link));  // Polymorphic!
    }
}
```

---

### 🎯 Cross-Questioning Drill (Interview Defense)

**Cross Q1**: "Polymorphism aur Method Overloading mein kya difference hai? Dono mein same method name use hota hai."

**Confident Answer**:
"Bhai, yeh classic confusion hai. **Overloading (compile-time polymorphism)** — same class mein same method name with **different parameters**. Compiler dekh kar decide karta hai which one to call, parameters ki list dekh ke. Like `print(int)`, `print(String)`. **Overriding (runtime polymorphism)** — child class parent ki same signature wali method **redefine** karti hai. JVM runtime pe object ka actual class dekh ke decide karta hai which version run karega. **Virtual dispatch table** (vtable) ke through. Aaj ke scenario mein `NotificationSender.send()` ko `EmailNotificationSender` aur `SmsNotificationSender` dono override karte hain — same signature, different body. Compiler nahi jaanta runtime pe konsa call hoga, JVM jaanta hai."

---

**Cross Q2**: "Tumne strategy pattern use kiya. Kya alternative tha if/else without polymorphism?"

**Confident Answer**:
"Haan, alternative tha — `switch` ya `if/else` chain. But three problems:
1. **Open/Closed violation** — naya channel add karne ke liye existing service file modify karni parti.
2. **Single Responsibility violation** — ek class email/SMS/push sab ka knowledge rakhti.
3. **Testability** — `EmailService` mock karna painful, sab dependencies ek hi class mein.

Strategy + Polymorphism approach DI container ko leverage karta hai — naya implementation register karo, container automatically inject kar dega. **Functional alternative** ho sakta tha — `Map<Channel, Function<User, Message, Void>>` — Java 8+ mein chalega, but interfaces zyada testable + IDE-friendly hain enterprise codebases mein."

---

**Cross Q3**: "Polymorphism violate kab kar sakte ho jaan-boojh ke?"

**Confident Answer**:
"Performance-critical hot paths mein virtual dispatch ka cost hota hai (extra pointer indirection, prevents inlining sometimes). High-frequency trading mein, JIT compiler aggressive optimization karta hai, lekin generally polymorphism cheap hai. **Real reason to skip**: jab sirf ek implementation hi possible ho aur future mein bhi nahi badlegi — to YAGNI principle apply karo (Day 18!), seedha concrete class use karo. Premature abstraction equally bad hai jaise no abstraction. Aaj ka case clearly polymorphic — multiple channels are real requirement, hypothetical nahi."

---

**Cross Q4**: "Yeh principle Spring/JPA mein automatically follow hota hai ya manual?"

**Confident Answer**:
"Spring **heavily** polymorphism pe based hai — pura DI container hi `BeanFactory` interface ke around bana hai. Tum `@Autowired NotificationSender notifier` likho, Spring dekhta hai konse implementations registered hain, qualifier/primary decide karta hai. **JPA** mein `EntityManager` interface hai, Hibernate ka `SessionImpl` implementation hai — tum interface se kaam karte ho, switch karna chaho to EclipseLink pe ja sakte ho without code change. **Spring AOP** itself polymorphism use karta hai — proxy class create karta hai jo target class extend/implement karti hai aur `@Transactional` jaise concerns inject karti hai. So Spring polymorphism manually enforce nahi karta, lekin uska entire framework polymorphism PE built hai."

---

**Cross Q5**: "Production mein Polymorphism scale kaise karta hai?"

**Confident Answer**:
"Microservices architecture mein polymorphism service-level pe extend hoti hai. **Payment Gateway**: Stripe, JazzCash, EasyPaisa sab `PaymentProvider` interface implement karte hain. Naya provider add = new microservice + register in service registry. **Notification service** Netflix scale pe: 100M+ users, har user ki preference different — email, push, SMS, in-app. Strategy pattern + polymorphism allow karta hai 50+ channels to coexist without single God class.

Catch: polymorphism **runtime cost** rakhta hai (~5-10ns per call due to virtual table lookup). For 1 trillion calls/day systems, this adds up. Solutions: JIT compiler optimization, monomorphic call sites (same implementation always), or `sealed`/`final` hints to compiler."

---

### 🧩 Pattern Combination (How Today's Pattern Pairs With Others)

Polymorphism akele nahi chalti — yeh dusre patterns ko **enable** karti hai.

**Pairs Well With**:
- **Strategy Pattern (Day 62)** — Strategy literally polymorphism use karta hai. Strategy = polymorphism + intent.
- **Factory Method (Day 32)** — Factory returns polymorphic type. `NotifierFactory.create(channel)` returns `NotificationSender`.
- **Template Method (Day 66)** — Base class skeleton fix, subclass steps override karte hain.
- **Open/Closed Principle (Day 12)** — Polymorphism is the **primary mechanism** to achieve OCP.
- **Dependency Inversion (Day 15)** — Depend on abstractions = depend on polymorphic interfaces.

**Tension With**:
- **YAGNI (Day 18)** — Don't create interfaces for single implementations. Polymorphism has cognitive cost.
- **KISS (Day 17)** — Sometimes a direct switch statement is clearer than 5 classes for 5 channels.
- **Performance-critical code** — Virtual dispatch prevents some JIT optimizations.

---

### 🎓 Real Production Code Where This Matters

**Spring's `PasswordEncoder` is textbook polymorphism**:

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}

// Implementations: BCryptPasswordEncoder, Argon2PasswordEncoder, 
// Pbkdf2PasswordEncoder, SCryptPasswordEncoder, NoOpPasswordEncoder (testing only)

// Spring Security itself:
@Bean
public PasswordEncoder passwordEncoder() {
    return new DelegatingPasswordEncoder("bcrypt", Map.of(
        "bcrypt", new BCryptPasswordEncoder(),
        "argon2", new Argon2PasswordEncoder(),
        "pbkdf2", new Pbkdf2PasswordEncoder("")
    ));
}
```

**DelegatingPasswordEncoder** ka kamaal: stored hash mein prefix hota hai `{bcrypt}$2a$10$...` — runtime pe prefix dekh ke correct encoder ko delegate karta hai. **Polymorphism + Strategy + Factory all in one**. Yeh hi reason hai Spring Security smooth migration support karta hai BCrypt se Argon2 tak — purane hashes bhi work karte hain, naye Argon2 mein hash hote hain.

**Stripe's API SDK** uses polymorphism heavily — `PaymentMethod` is base type, `Card`, `BankAccount`, `Wallet`, `BNPL` all extend it. One API call returns polymorphic type, your code handles each subtype.

---

### 💡 Memory Hook for This Principle/Pattern

**POLYMORPHISM = "BARTAN BADLO, KHAANA WOHI"**

Bhai, polymorphism aisi cheez hai: **bartan** (concrete class) badal jaye, **khaana** (interface contract) wohi rahe. Pateele mein biryani, handi mein biryani, degh mein biryani — bartan alag, function (biryani pakana) same.

Ya phir: **"EK CALL, KAYI CHEHRE"** — same method call, different faces showing up at runtime.

Yaad rakhne ka tareeqa:
- **Overloading** = "Sub apne pyaalon mein chai" (same chai, alag pyaale — compile-time, same class)
- **Overriding** = "Beta baap ki recipe mein twist" (parent method ko redefine, runtime decide)

---

### ⚠️ Common Mistakes Jo Aaj Bachne Hain

**❌ Mistake 1**: `@Override` annotation skip karna in Java
**Why it's wrong**: Agar tumne method signature galat likhi (typo, wrong parameter type), compiler chup rahega — silently new method ban jayegi, override nahi. Production mein parent ka method chalega, tumhari child wali kabhi nahi. Bug debug karna nightmare.
**Correct approach**: Hamesha `@Override` lagao. Compiler error de dega agar signature mismatch ho.

```java
// BAD — typo, no @Override → silent bug
public class EmailNotifier implements NotificationSender {
    public void Send(User user, Message msg) { ... }  // capital S — different method!
}

// GOOD — compiler catches it
@Override
public void send(User user, Message msg) { ... }
```

---

**❌ Mistake 2**: Calling overridable methods from constructor
**Why it's wrong**: Constructor mein `this.someMethod()` call karo aur woh method child mein override hai — child class ka constructor abhi run nahi hua, child ki fields uninitialized hain. NullPointerException ya weird behavior.
**Correct approach**: Constructor mein sirf `private` ya `final` methods call karo. Initialization complete hone ke baad virtual calls karo.

```java
// BAD
public class Parent {
    public Parent() { initialize(); }       // calls overridable
    public void initialize() { ... }
}
public class Child extends Parent {
    private String name = "default";
    @Override
    public void initialize() {
        System.out.println(name.toUpperCase());  // NPE! name is null when parent constructor runs
    }
}
```

---

**❌ Mistake 3**: `equals()` override karte time `hashCode()` bhulna
**Why it's wrong**: HashMap/HashSet contract toot jata hai. Do equal objects different hashCode → set mein duplicate, map mein wrong bucket. Polymorphism ke contract violations dangerous hain.
**Correct approach**: `equals()` aur `hashCode()` **always together** override karo. Lombok `@EqualsAndHashCode`, IDE auto-generate, ya Java records use karo.

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom):

```
1. USER ACTION (Angular)
   └─▶ Click "Forgot Password"
       └─▶ Reactive Form submit, loading spinner

2. API REQUEST (Spring Boot / .NET)
   └─▶ POST /api/auth/forgot-password
       └─▶ Rate limiter check (3/hour/IP)

3. BUSINESS LOGIC (Java / C#)
   └─▶ Generate SecureRandom token (32 bytes)
       └─▶ SHA-256 hash → save to DB
           └─▶ Polymorphic notifier.send(user, msg)

4. DATABASE (SQL)
   └─▶ INSERT password_reset_tokens (hash, expires_at)
       └─▶ Index on token_hash for fast lookup

5. EVENT PUBLISHING (System Design)
   └─▶ Publish to Kafka topic
       └─▶ NotificationService consumes async
           └─▶ EmailSender OR SmsSender (polymorphism)

6. USER CLICKS LINK
   └─▶ GET /reset?token=xyz → Angular form
       └─▶ POST /api/auth/reset {token, newPassword}
           └─▶ Atomic UPDATE tokens SET used=1 WHERE used=0 AND not expired
               └─▶ Update password_hash, invalidate all sessions
                   └─▶ 200 OK → Angular redirects to login
```

**What Breaks If You Skip ANY Layer**:

- **Skip Angular generic response** → User enumeration leak via UI, attackers harvest emails.
- **Skip Backend rate limit** → DDoS via "Forgot password" spam, mailbox flooding attacks.
- **Skip token hashing** → DB breach = all active reset tokens leaked = mass account takeover.
- **Skip atomic SQL update** → Race condition, token used twice, double password reset, account compromise.
- **Skip async queue** → SMTP slowdown blocks API, 504 timeouts, poor UX.
- **Skip session invalidation** → Hacker who already had access stays logged in even after password change.

---

## 🧭 MENTAL MAP — How to Memorize This

```
                    [PASSWORD RESET BRAINER]
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   [FRONTEND]            [BACKEND]              [DATABASE]
        │                     │                     │
    Angular            Spring / .NET              SQL
        │                     │                     │
   ReactiveForms       SecureRandom token      token_hash CHAR(64)
   Same response       SHA-256 hash             UNIQUE index
   Token from URL      @Transactional           Atomic UPDATE
   Strength meter      Polymorphic Notifier     used = 1 WHERE used=0
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                     [SYSTEM DESIGN]
                              │
              Strategy Pattern + Event-Driven
              ┌──────────┬──────────┬──────────┐
              │  Email   │   SMS    │  Push    │
              │  Sender  │  Sender  │  Sender  │
              └──────────┴──────────┴──────────┘
                  (all implement INotifier)
```

**Mental Story to Remember (Roman Urdu)**:

"Bhai, password reset ko shaadi ke invitation card jaisa socho:

- **Angular** = Card design (kaisa dikhega user ko)
- **Spring/.NET** = Card printer (banata hai card, unique number deta hai)
- **Java/C#** = Tahara number generator (random, unique, secure)
- **SQL** = Guest list (kis ko bheja, kab expire hoga, RSVP done ya nahi)
- **System Design** = Postman selection — DHL (Email), TCS (SMS), khud jaa ke dena (Push). **Sender alag, kaam wohi — guest ko invite milna**.

Polymorphism = chahe DHL bheje ya TCS, **method same**: `deliver(invitation, guest)`. Tum bas bolo 'deliver', sender khud decide karega kaise.

Agar koi step missing — shaadi mein guest nahi aayega, ya galat guest aa jayega (hacker)."

**Acronym/Mnemonic**: **"TRACE"** for Password Reset Security:

- **T** — Token (SecureRandom, 32 bytes)
- **R** — Rate limit (3 attempts/hour)
- **A** — Atomic single-use (SQL UPDATE WHERE used=0)
- **C** — Consistent response (no enumeration)
- **E** — Expiry + session invalidation (15-min + logout all)

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Design karo Daraz ke liye complete password reset flow. Security considerations bhi explain karo aur batao notification multi-channel kaise pluggable banaoge?"

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Iss scenario ko handle karne ke liye, hum 5 layers ko coordinate karenge with security at every step — frontend pe enumeration prevention, backend pe secure token generation, DB pe atomic single-use, async notification dispatch, aur polymorphic sender for channel flexibility."

**Body** (60 sec):
1. **Frontend** (Angular): Reactive Form for email submit, **identical success response** chahe email exist kare ya na, taake user enumeration na ho. Reset page token ko URL se uthaye, password strength real-time check kare with `debounceTime` for performance.

2. **Backend API** (Spring/.NET): Rate-limited endpoint (3 req/hour/IP), `SecureRandom` se 256-bit token generate, raw token email mein, **SHA-256 hash DB mein**. `@Transactional` enforce, async notification dispatch via event.

3. **Business Logic** (Java/C#): **Polymorphic `NotificationSender` interface** — Email, SMS, Push implementations. User preference ke hisaab se DI container correct sender resolve karta hai. Naya channel = bas naya class.

4. **Database** (SQL): `password_reset_tokens` table with unique index on hash, composite index on `(user_id, expires_at)`. Token consumption **atomic**: `UPDATE WHERE used=0 AND expires_at > NOW()` — `@@ROWCOUNT=1` matlab safe. `@Version` for optimistic locking.

5. **Architecture**: Strategy Pattern + Event-driven via Kafka. API publishes `PasswordResetRequested`, Notification Service consumes, polymorphic sender dispatches. DLQ for failed sends.

**Closing** (10 sec):
"By combining these layers with polymorphic notification dispatch, hum guarantee karte hain ke yeh flow secure, scalable, aur infinitely extensible hai — kal WhatsApp add karna ho, zero existing code change chahiye."

### Under-the-Hood Concepts You MUST Know

1. **`SecureRandom` vs `Random`**: `Random` is predictable (linear congruential generator), `SecureRandom` uses OS-level entropy (`/dev/urandom`). Token mein **kabhi** `Random` use mat karo.

2. **SHA-256 vs BCrypt for tokens**: SHA-256 fast, suitable for token hashing (random already high-entropy). BCrypt slow, suitable for **password** hashing (low entropy input, needs cost factor).

3. **Optimistic vs Pessimistic Locking**: Optimistic (`@Version`) = no lock during read, conflict detected at commit. Pessimistic (`SELECT FOR UPDATE`) = lock during read. Optimistic better for low-contention (this case).

4. **Virtual dispatch (vtable)**: Polymorphism ka cost — runtime mein method pointer lookup. JIT can devirtualize if only one implementation exists at runtime.

5. **Token enumeration attacks**: Timing differences, response sizes, error messages — sab leak vector hain. Constant-time comparison + identical responses chahiye.

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

**Counter Q1 (Trade-off focused)**: "JWT-based reset token (stateless) ya DB-stored random token (stateful) — kya choose karoge aur kyun?"

**Your Answer**:
"Bhai, dono ke trade-offs hain:

- **JWT (stateless)**: No DB hit per validation, scalable. **Lekin** — single-use enforce karna mushkil hai (kyunki state DB mein nahi). Revoke karna impossible before expiry. Replay attack vulnerable.

- **DB-stored (stateful)**: Single-use atomic UPDATE se enforce, revocable, audit trail. **Lekin** — DB hit per validation, requires schema.

Password reset jaisi **security-critical, single-use** operation ke liye **DB-stored definitely better**. JWT ke convenience ke chakkar mein security compromise nahi karta. Reset throughput low hai (few requests/sec even at scale), DB cost minimal. Magic links (Slack-style) bhi same approach use karte hain."

---

**Counter Q2 (Scale focused)**: "10 million users hain, 1% per day password reset karte hain — 100K requests/day. SMTP slow hai 2-3 seconds per send. Architecture kya hogi?"

**Your Answer**:
"100K/day = ~1.2 req/sec average, peak 10x = 12 req/sec. Without queue, agar SMTP synchronous hai 3 sec, tum 12 concurrent SMTP connections chahiye **constantly**. SMTP providers (SES) limit 14/sec — manageable but fragile.

**Async architecture**:
- API saves token + publishes Kafka event in <50ms (fast).
- Notification consumer pool 20 workers parallel send karte hain.
- Failed sends → DLQ → manual retry or alert.
- 99th percentile API latency stays <100ms, user dekhta hai instant success.

10x scale (1M users) tak yeh comfortable hai. 100M scale pe regional sharding — har region apna notification service + SMTP."

---

**Counter Q3 (Failure mode focused)**: "User ne reset link click kiya, password type kiya, submit click kiya — server crash. User dobara try karta hai. Kya hoga?"

**Your Answer**:
"Bhai, depends on which step pe crash hua:

1. **Before UPDATE token used=1**: Token still valid, user retry karta hai, **succeeds**. Idempotent.

2. **After token UPDATE, before password UPDATE**: Token consumed, password unchanged. User retry karta hai → 'token invalid' error. **Bad UX** — user dobara 'Forgot Password' click karega.

3. **After both UPDATE, before response**: Both done, user dekhta hai 'failed', dobara try karta hai → token invalid. Lekin **password already updated** — user can login with new password. Should redirect to login with message.

**Solution**: Both UPDATEs **same transaction** mein, **atomic commit**. Either both succeed or both fail. PostgreSQL/SQL Server ACID guarantee deta hai. Crash recovery via WAL (write-ahead log) ensures consistency.

**Better UX**: Frontend pe show 'Reset already completed, please login' if token marked used but password matches — but yeh leak vector ho sakta hai, careful design needed."

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "Password reset ke baad active sessions kaise invalidate karoge across multiple servers/devices?"

**Junior Answer**: "Database mein sessions table hai, sab delete kar denge user_id ke liye. JWT case mein... uhh... kuch nahi kar sakte?"

**Senior Answer**: "Multi-pronged approach:

1. **Stateful sessions (DB-stored)**: Simple — `DELETE FROM sessions WHERE user_id = ?`. Next request 401.

2. **Stateless JWT**: Token can't be revoked directly. Solutions:
   - **Short expiry (15 min) + refresh tokens**: Refresh token stored in DB, deleted on password change. Access token expires soon.
   - **Token version in JWT claims**: User entity has `token_version` integer. JWT includes it. Validation checks `jwt.tokenVersion == user.tokenVersion`. Password change → increment version → all old JWTs invalid.
   - **Redis blocklist**: Add JWT jti to Redis blocklist with TTL = remaining expiry. Validation checks blocklist. Scales to millions.

3. **Real-time push to clients**: WebSocket/SSE notify connected clients to re-authenticate immediately (don't wait for next request).

4. **Audit logging**: Log all session terminations for compliance/forensics.

Choose based on scale aur architecture — Slack uses Redis blocklist, Google uses token versioning, banking uses sessions DB."

### Red Flag Signals (Don't Say These!)

- ❌ **"Email mein plaintext token bhej do, DB mein bhi plaintext save kar do"** — Why: DB breach instantly leaks all active reset tokens, complete account takeover.

- ❌ **"Agar email registered nahi to 'Email not found' error dikhao"** — Why: User enumeration attack, attacker harvests valid emails for credential stuffing.

- ❌ **"Token expire hi nahi karega, user jab chahe use kare"** — Why: Leaked email/screenshot 6 months baad bhi exploit ho sakta hai. Industry standard 15min-1hour.

- ❌ **"Math.random() se token bana lo"** — Why: Predictable PRNG. Attacker guess kar sakta hai future tokens. ALWAYS `SecureRandom` / `RandomNumberGenerator`.

- ❌ **"Token validate karne ke baad alag transaction mein use mark karenge"** — Why: TOCTOU race condition, double-use possible. Atomic UPDATE essential.

- ❌ **"Sessions kya hoti hain reset ke baad, woh to alag concern hai"** — Why: Senior engineers think holistically. Password reset bina session invalidation = incomplete.

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ Secure random token kaise generate karte hain aur **kyun** `SecureRandom` use karna mandatory hai?
2. ✅ Tokens DB mein **hashed** kyun store karte hain raw kyun nahi?
3. ✅ User enumeration attack kya hai aur kaise prevent karte hain frontend + backend dono pe?
4. ✅ **Atomic UPDATE** pattern kya hai aur kaise single-use guarantee deta hai?
5. ✅ Polymorphism real-world mein **NotificationSender** jaisi situations mein kaise apply hoti hai? Aur **Method Overloading vs Overriding** ka difference?
6. ✅ Password reset ke baad active sessions kaise invalidate karte hain — stateful vs stateless?

---

## 🔗 Tomorrow's Connection Hint

Tomorrow we'll cover: **Day 5 — Basic Product Listing + Composition over Inheritance**

Aaj ke concept se kaise connected hai:
Aaj humne polymorphism dekha (inheritance/interface-based). Kal **composition over inheritance** principle dekhenge — jab inheritance/polymorphism nuksaan kar sakti hai aur **composition** better choice hoti hai. Product listing scenario mein category/filter/sort features compose karke build karenge, deep inheritance hierarchy avoid karke. Yeh OOP design ka next critical step hai — kab inherit karna hai, kab compose. Aur SQL pe pagination + indexing strategies deep dive.

---

## 📚 Progress Tracker

```
🟢 Beginner     ████░░░░░░░░░░░░░░░░  Day 4/20  ← YOU ARE HERE
🟡 Intermediate ░░░░░░░░░░░░░░░░░░░░  Day 0/30
🟠 Advanced     ░░░░░░░░░░░░░░░░░░░░  Day 0/40
🔴 Expert       ░░░░░░░░░░░░░░░░░░░░  Day 0/50
⚫ Master       ░░░░░░░░░░░░░░░░░░░░  Day 0/60
🟣 Architect    ░░░░░░░░░░░░░░░░░░░░  Day 0/70
💎 Principal    ░░░░░░░░░░░░░░░░░░░░  Day 0/95

OOP Foundation: Day 4/30 (Polymorphism — Method Overriding) ✅
Overall Journey: 4/365 days (1.1% complete)
```

**Streak**: 🔥 4 days strong, bhai! Keep showing up — consistency hi senior banayegi.
