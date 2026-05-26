# 🎯 🟢 Day 14 of Beginner (Level 1 of 7): Email Verification Flow

**Overall Day**: Day 14 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 14 of 20
**Today's Theme**: Token-based email verification — secure link generation, expiry, single-use, full-stack flow

---

## 📖 The Brainer Scenario (Real Problem)

Bhai, socho — tum **Daraz.pk** ke liye signup flow bana rahe ho. User ne email + password daala. Ab tum kaise verify karoge ke yeh email **woh user ka actual email hai**, koi fake nahi?

Real flow yeh hai:
1. User signup karta hai → account banta hai but `email_verified = false`
2. Backend ek **secure unique token** generate karta hai (cryptographic random, NOT predictable)
3. Token ko DB mein save karta hai with **expiry (24 hours)** aur **status (used/unused)**
4. User ko email jata hai: `https://daraz.pk/verify?token=a8f3...xyz`
5. User link click karta hai → Angular page load hota hai → token backend ko bhejta hai
6. Backend token validate karta hai: exists? expired? already used?
7. Agar sab sahi hai → `email_verified = true`, token ko `used = true` mark karta hai
8. Without verified email, user discount coupons claim nahi kar sakta, address checkout pe nahi daal sakta

**The Real Challenge (The "Gotcha")**: 

Junior dev kya karte hain? Token jaise `userId + timestamp` se banate hain — **PREDICTABLE**. Attacker easily guess karke kisi aur ka email verify kar sakta hai. Ya phir token DB mein plain text save karte hain — agar DB leak hua, sare unused tokens compromised. Ya phir expiry handle nahi karte — saal purane links bhi kaam karte hain. Ya phir same token 100 baar use ho sakta hai — replay attack!

Aur sabse bara gotcha: **email service kya hai?** SendGrid? AWS SES? Mailgun? In-house SMTP? Agar tumne directly `SendGridClient` inject kar diya, kal change karna ho to **pura codebase touch karna parega**. Yahin **Interface Segregation** kaam aata hai.

**Why this matters in production**: 
- **Stripe** sends email verification with tokens that expire in 1 hour, single-use, HMAC-signed
- **GitHub** uses 32-byte random tokens stored as bcrypt hashes (DB leak → tokens still safe)
- **Shopify** rate-limits verification email sending per IP (5 per hour) to prevent abuse
- **Booking.com** ka case: 2017 mein unke verification tokens predictable the, attacker ne 50,000 accounts hijack kiye

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: `SecureRandom` for token generation, `@Transactional` for atomicity, `JavaMailSender` abstraction, scheduled cleanup of expired tokens

**Under the Hood — Yeh Kaise Kaam Karta Hai**: 

`java.util.Random` ek **Linear Congruential Generator (LCG)** use karta hai — predictable hai. Agar tumne seed dekh liya ya kuch outputs dekh liye, next values calculate kar sakte ho. `SecureRandom` instead uses OS-level entropy source (`/dev/urandom` on Linux, `CryptGenRandom` on Windows) — cryptographically secure.

Spring ka `JavaMailSender` ek **abstraction** hai — actual implementation `JavaMailSenderImpl` hai, but tum interface pe code karte ho. Yeh **Dependency Inversion** + **Interface Segregation** ka classic example hai. Spring Boot auto-configures it based on `spring.mail.*` properties.

`@Transactional` Spring AOP proxy banata hai — runtime pe tumhare service ka subclass create karta hai jo method call ko intercept karke transaction start/commit/rollback karta hai.

**Bhai, Simple Mein Samjho**: 

Token generate karna = ek **lottery ticket** banane jaisa hai. Lottery numbers agar predictable hon, koi bhi jeet jaye. Isliye `SecureRandom` use karte hain — yeh OS ke "entropy pool" (mouse movements, network noise, etc.) se random bytes leta hai. Phir hum usko `Base64URL` mein encode karke link mein daalte hain.

Email bhejne ke liye `EmailSender` interface chahiye — implementation kal SendGrid ho ya SMTP, **service ko fark nahi parna chahiye**.

**Code Pattern**:
```java
// Interface — sirf email bhejne ka kaam (ISP)
public interface EmailSender {
    void send(String to, String subject, String htmlBody);
}

// SendGrid implementation
@Service
@ConditionalOnProperty(name = "email.provider", havingValue = "sendgrid")
public class SendGridEmailSender implements EmailSender {
    private final SendGrid sendGrid;
    
    @Override
    public void send(String to, String subject, String htmlBody) {
        Mail mail = new Mail(new Email("noreply@daraz.pk"), subject, 
                             new Email(to), new Content("text/html", htmlBody));
        sendGrid.api(new Request().method(Method.POST).endpoint("mail/send")
                                   .body(mail.build()));
    }
}

@Service
@RequiredArgsConstructor
public class EmailVerificationService {
    private final UserRepository userRepo;
    private final VerificationTokenRepository tokenRepo;
    private final EmailSender emailSender;  // Interface, not concrete!
    private static final SecureRandom RANDOM = new SecureRandom();
    
    @Transactional
    public void sendVerificationEmail(Long userId) {
        User user = userRepo.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        if (user.isEmailVerified()) {
            throw new AlreadyVerifiedException();
        }
        
        // 32 bytes = 256 bits of entropy — uncrackable
        byte[] tokenBytes = new byte[32];
        RANDOM.nextBytes(tokenBytes);
        String rawToken = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
        
        // Store HASH of token in DB — agar DB leak ho, tokens safe rahein
        String tokenHash = BCrypt.hashpw(rawToken, BCrypt.gensalt(10));
        
        VerificationToken token = VerificationToken.builder()
            .userId(userId)
            .tokenHash(tokenHash)
            .expiresAt(Instant.now().plus(24, ChronoUnit.HOURS))
            .used(false)
            .build();
        tokenRepo.save(token);
        
        String link = "https://daraz.pk/verify?token=" + rawToken + "&uid=" + userId;
        emailSender.send(user.getEmail(), "Verify your email",
            "<a href='" + link + "'>Click to verify</a> (expires in 24 hours)");
    }
    
    @Transactional
    public void verifyToken(Long userId, String rawToken) {
        VerificationToken token = tokenRepo.findActiveByUserId(userId)
            .orElseThrow(() -> new InvalidTokenException("Token not found"));
        
        if (token.isUsed()) throw new InvalidTokenException("Already used");
        if (token.getExpiresAt().isBefore(Instant.now())) 
            throw new InvalidTokenException("Expired");
        if (!BCrypt.checkpw(rawToken, token.getTokenHash()))
            throw new InvalidTokenException("Invalid token");
        
        token.setUsed(true);
        User user = userRepo.findById(userId).orElseThrow();
        user.setEmailVerified(true);
        // @Transactional ensures both updates commit together
    }
}
```

**Interview phrasing**: 
"Iss scenario mein main `SecureRandom` (32 bytes) use karunga predictability avoid karne ke liye, `BCrypt` se hash karke store karunga DB leak protection ke liye, aur `EmailSender` interface inject karunga taake provider switch karna trivial ho. `@Transactional` ensure karta hai ke token-used flag aur email-verified flag dono ek saath commit hon — partial state nahi rahegi."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: `RandomNumberGenerator` (cryptographic), `IEmailSender` interface, EF Core transactions, `BCrypt.Net` for hashing

**Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai**: 

C# mein `System.Random` Java ke `Random` jaisa hi insecure hai. `System.Security.Cryptography.RandomNumberGenerator` use karna parta hai — yeh Windows pe `BCryptGenRandom` (Bcrypt API, not the hash) aur Linux pe `/dev/urandom` use karta hai.

EF Core ka `DbContext` change tracker rakhta hai — jab tum entity modify karte ho, woh dirty flag set karta hai. `SaveChangesAsync()` pe woh ek transaction mein sare changes flush karta hai. Manual transaction ke liye `BeginTransactionAsync()` use hota hai.

ASP.NET Core mein `IEmailSender` Identity package mein already define hai — Microsoft ne jaan-boojh ke chhota interface rakha hai (Interface Segregation).

**Bhai, .NET Mein Yeh Kaise Hota Hai**: 

.NET mein interface bana ke `Program.cs` mein `services.AddScoped<IEmailSender, SendGridEmailSender>()` se register karte ho. Constructor injection automatic. Token generate karne ka logic same, bas API names alag hain.

**Code Pattern**:
```csharp
// Small focused interface — ISP follow
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string htmlBody);
}

public class SendGridEmailSender : IEmailSender
{
    private readonly ISendGridClient _client;
    public SendGridEmailSender(ISendGridClient client) => _client = client;
    
    public async Task SendAsync(string to, string subject, string htmlBody)
    {
        var msg = MailHelper.CreateSingleEmail(
            new EmailAddress("noreply@daraz.pk"),
            new EmailAddress(to), subject, "", htmlBody);
        await _client.SendEmailAsync(msg);
    }
}

[ApiController]
[Route("api/auth")]
public class EmailVerificationController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IEmailSender _emailSender;
    
    public EmailVerificationController(AppDbContext db, IEmailSender emailSender)
    {
        _db = db;
        _emailSender = emailSender;
    }
    
    [HttpPost("send-verification/{userId}")]
    public async Task<IActionResult> SendVerification(long userId)
    {
        await using var tx = await _db.Database.BeginTransactionAsync(
            IsolationLevel.ReadCommitted);
        
        var user = await _db.Users.FindAsync(userId);
        if (user is null) return NotFound();
        if (user.EmailVerified) return BadRequest("Already verified");
        
        // Cryptographically secure random
        var tokenBytes = RandomNumberGenerator.GetBytes(32);
        var rawToken = Base64UrlEncoder.Encode(tokenBytes);
        var tokenHash = BCrypt.Net.BCrypt.HashPassword(rawToken, workFactor: 10);
        
        _db.VerificationTokens.Add(new VerificationToken
        {
            UserId = userId,
            TokenHash = tokenHash,
            ExpiresAt = DateTime.UtcNow.AddHours(24),
            Used = false
        });
        await _db.SaveChangesAsync();
        await tx.CommitAsync();
        
        var link = $"https://daraz.pk/verify?token={rawToken}&uid={userId}";
        await _emailSender.SendAsync(user.Email, "Verify your email",
            $"<a href='{link}'>Click to verify</a>");
        
        return Ok();
    }
    
    [HttpPost("verify")]
    public async Task<IActionResult> Verify([FromBody] VerifyRequest req)
    {
        await using var tx = await _db.Database.BeginTransactionAsync();
        
        var token = await _db.VerificationTokens
            .Where(t => t.UserId == req.UserId && !t.Used)
            .OrderByDescending(t => t.ExpiresAt)
            .FirstOrDefaultAsync();
        
        if (token is null) return BadRequest("Invalid");
        if (token.ExpiresAt < DateTime.UtcNow) return BadRequest("Expired");
        if (!BCrypt.Net.BCrypt.Verify(req.Token, token.TokenHash))
            return BadRequest("Invalid token");
        
        token.Used = true;
        var user = await _db.Users.FindAsync(req.UserId);
        user!.EmailVerified = true;
        
        await _db.SaveChangesAsync();
        await tx.CommitAsync();
        return Ok();
    }
}
```

**Java vs .NET Comparison Table**:
| Feature | Java/Spring | .NET/C# |
|---------|-------------|---------|
| Secure Random | `SecureRandom` | `RandomNumberGenerator.GetBytes()` |
| Email Abstraction | `JavaMailSender` / custom interface | `IEmailSender` (built-in Identity) |
| Transaction | `@Transactional` | `BeginTransactionAsync()` / `TransactionScope` |
| DI Registration | `@Service` auto-scan | `services.AddScoped<I, Impl>()` in `Program.cs` |
| Hashing | `BCrypt.hashpw()` (jBCrypt) | `BCrypt.Net.BCrypt.HashPassword()` |
| Base64 URL | `Base64.getUrlEncoder()` | `Base64UrlEncoder.Encode()` |

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: Schema design for tokens, unique index on token hash, expiry cleanup job, composite index for fast lookup

**Under the Hood — SQL Engine Yeh Kaise Karta Hai**: 

Jab tum `WHERE user_id = ? AND used = false` query chalate ho, SQL Server pehle **statistics** dekhta hai. Agar `verification_tokens` table mein millions of rows hain aur 95% rows `used = true` hain, then query optimizer **filtered index** prefer karega — index sirf `used = false` rows pe banta hai. Yeh disk space bhi bachata hai aur query speed bhi badhata hai.

`SCHEDULED JOB` (SQL Server Agent ya .NET Hangfire / Spring `@Scheduled`) daily expired tokens delete karta hai. Without cleanup, table infinitely grow karega aur queries slow ho jayengi.

**Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai**: 

Token table alag rakhna chahiye `users` se. Why? Kyunki ek user multiple times verification request kar sakta hai (link kho gaya, resend kiya). Plus, tokens **transient data** hain — expired hone ke baad delete kar sakte ho. Users table mein mix karne se schema bloat hota hai.

**SQL Example**:
```sql
-- Token table design
CREATE TABLE verification_tokens (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    token_hash VARCHAR(60) NOT NULL,  -- BCrypt hash is always 60 chars
    expires_at DATETIME2 NOT NULL,
    used BIT NOT NULL DEFAULT 0,
    created_at DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT fk_token_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Composite filtered index — fast lookup for active tokens
CREATE INDEX ix_tokens_user_active 
ON verification_tokens(user_id, expires_at) 
WHERE used = 0;

-- Verify flow query
BEGIN TRANSACTION;

DECLARE @tokenId BIGINT, @hash VARCHAR(60), @expires DATETIME2;

SELECT TOP 1 @tokenId = id, @hash = token_hash, @expires = expires_at
FROM verification_tokens WITH (UPDLOCK, ROWLOCK)
WHERE user_id = @userId 
  AND used = 0
  AND expires_at > SYSUTCDATETIME()
ORDER BY created_at DESC;

IF @tokenId IS NULL
BEGIN
    ROLLBACK;
    THROW 51000, 'No valid token', 1;
END

-- Application layer verifies BCrypt(rawToken, @hash) === true
-- Then:
UPDATE verification_tokens SET used = 1 WHERE id = @tokenId;
UPDATE users SET email_verified = 1 WHERE id = @userId;

COMMIT TRANSACTION;

-- Daily cleanup job
DELETE FROM verification_tokens 
WHERE expires_at < DATEADD(DAY, -7, SYSUTCDATETIME())
   OR (used = 1 AND created_at < DATEADD(DAY, -30, SYSUTCDATETIME()));
```

**The Gotcha**: 

Agar tumne `used` flag check nahi kiya, ek hi link 100 baar use ho sakti hai — **replay attack**. Agar tumne `UPDLOCK` nahi liya, **race condition** hoga: two concurrent requests same token use kar sakte hain. Agar tumne `expires_at` check nahi kiya database query mein (sirf app layer mein), **TOCTOU bug** (time-of-check-to-time-of-use) ho sakta hai.

**Isolation Level Choice**: 

`READ_COMMITTED` enough hai because hum `UPDLOCK` se explicit row lock le rahe hain. Higher isolation (SERIALIZABLE) overkill hai — phantom reads ka koi risk nahi hai single-token operation mein.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Route guards, `ActivatedRoute` for query params, RxJS error handling, user-friendly states (loading/success/error)

**Under the Hood — Angular Yeh Kaise Karta Hai**: 

Angular routing zone.js ke through trigger hota hai. Jab user `/verify?token=xyz&uid=123` pe land hota hai, `ActivatedRoute` ka `queryParamMap` observable emit karta hai. Component constructor mein inject karke `pipe(switchMap(...))` se HTTP call chain kar sakte ho.

Change detection by default zone.js har async operation ke baad trigger karta hai. `OnPush` strategy mein sirf input change ya observable emit pe trigger hota hai — faster.

**Bhai, Frontend Pe Yeh Kaise Handle Karein**: 

User experience critical hai — link click pe blank page nahi dikhana. **3 states** chahiye:
1. **Loading**: spinner + "Verifying your email..."
2. **Success**: green check + "Email verified! Redirecting to dashboard..."
3. **Error**: red icon + reason ("Link expired" / "Already used" / "Invalid")

Resend button bhi chahiye for expired case.

**Code Pattern**:
```typescript
// Small focused service — ISP applied here too
export interface VerificationApi {
  verify(userId: number, token: string): Observable<void>;
  resend(userId: number): Observable<void>;
}

@Injectable({ providedIn: 'root' })
export class EmailVerificationService implements VerificationApi {
  constructor(private http: HttpClient) {}
  
  verify(userId: number, token: string): Observable<void> {
    return this.http.post<void>('/api/auth/verify', { userId, token });
  }
  
  resend(userId: number): Observable<void> {
    return this.http.post<void>(`/api/auth/send-verification/${userId}`, {});
  }
}

@Component({
  selector: 'app-verify-email',
  template: `
    <div class="verify-container">
      <ng-container [ngSwitch]="state">
        <div *ngSwitchCase="'loading'">
          <mat-spinner></mat-spinner>
          <p>Verifying your email, bhai... ek second</p>
        </div>
        <div *ngSwitchCase="'success'" class="success">
          <mat-icon>check_circle</mat-icon>
          <h2>Email Verified!</h2>
          <p>Redirecting to dashboard in {{countdown}}s...</p>
        </div>
        <div *ngSwitchCase="'error'" class="error">
          <mat-icon>error</mat-icon>
          <h2>Verification Failed</h2>
          <p>{{errorMsg}}</p>
          <button *ngIf="canResend" (click)="resend()">Resend Link</button>
        </div>
      </ng-container>
    </div>
  `
})
export class VerifyEmailComponent implements OnInit {
  state: 'loading' | 'success' | 'error' = 'loading';
  errorMsg = '';
  canResend = false;
  countdown = 3;
  private userId!: number;
  
  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private verifyService: EmailVerificationService
  ) {}
  
  ngOnInit() {
    this.route.queryParamMap.pipe(
      switchMap(params => {
        const token = params.get('token');
        this.userId = Number(params.get('uid'));
        if (!token || !this.userId) {
          return throwError(() => new Error('Invalid link'));
        }
        return this.verifyService.verify(this.userId, token);
      }),
      catchError((err: HttpErrorResponse) => {
        this.state = 'error';
        if (err.error?.message?.includes('Expired')) {
          this.errorMsg = 'Link expired ho gayi. Naya bhej dein?';
          this.canResend = true;
        } else if (err.error?.message?.includes('used')) {
          this.errorMsg = 'Link already use ho chuki hai.';
        } else {
          this.errorMsg = 'Link invalid hai. Please try again.';
        }
        return EMPTY;
      })
    ).subscribe(() => {
      this.state = 'success';
      interval(1000).pipe(take(3)).subscribe(i => {
        this.countdown = 2 - i;
        if (i === 2) this.router.navigate(['/dashboard']);
      });
    });
  }
  
  resend() {
    this.verifyService.resend(this.userId).subscribe({
      next: () => this.errorMsg = 'Naya link bhej diya — apna email check karein',
      error: () => this.errorMsg = 'Resend failed, thori der baad try karein'
    });
  }
}
```

**UX Concern**: 

Without proper state handling, user blank white screen dekhega — confusion. Without resend button on expired, user frustrated hoga aur support pe email karega. Without countdown redirect, user manually navigate karega (extra friction).

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: Asynchronous email sending via **Message Queue** (decouple email send from API response)

**Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai**: 

Synchronous email sending mein API request 2-5 seconds wait karta hai (SMTP slow hai). Yeh do problems banata hai: (1) User ko slow signup experience, (2) Agar SendGrid down hai, signup fail ho jata hai.

Solution: Signup hote hi event publish karo (`UserRegisteredEvent`) to Kafka/RabbitMQ. Email service async consume karta hai aur email bhejta hai. Agar fail ho, **retry queue** pe dalta hai (exponential backoff). 5 retries ke baad **dead letter queue** mein jata hai — manual investigation ke liye.

**Bhai, Architecture Level Pe Yeh Kaise Design Karein**: 

Email bhejna **side concern** hai — main signup flow ko block nahi karna chahiye. User signup karein, instantly response milein "Email bhej diya gaya hai" (eventually consistent). Email queue mein process ho raha hai background mein.

**Architecture Diagram**:
```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│   Angular   │───▶│  Spring Boot │───▶│   SQL DB     │
│  /verify    │    │  /.NET API   │    │  users +     │
└─────────────┘    └──────┬───────┘    │  tokens      │
       ▲                  │            └──────────────┘
       │ Click link       │ Publish event
       │                  ▼
       │           ┌──────────────┐
       │           │ Kafka topic  │
       │           │ user.signup  │
       │           └──────┬───────┘
       │                  │
       │                  ▼
       │           ┌──────────────┐    ┌──────────────┐
       └───────────│ Email Worker │───▶│  SendGrid /  │
                   │ (consumer)   │    │  AWS SES     │
                   └──────┬───────┘    └──────────────┘
                          │
                          ▼ (on failure)
                   ┌──────────────┐
                   │ Retry Queue  │
                   │ + DLQ        │
                   └──────────────┘
```

**Trade-offs**:
| Approach | Pros | Cons |
|----------|------|------|
| Sync email send | Simple, immediate feedback | Slow API, fails if SMTP down |
| Async via queue | Fast API, resilient, scalable | Eventual consistency, queue infrastructure cost |
| Inline + retry | No queue needed | Still blocks for first attempt |

**Real Companies Using This**: 
- **Netflix** uses Kafka for all email/notification triggers — signup, password reset, recommendations
- **Airbnb** uses SQS + Lambda for transactional emails
- **Uber** ka **Ringpop** internal email service async hai with retry + DLQ

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Cross-Questioning Proof)

**Today's OOP/Pattern Focus**: SOLID — Interface Segregation Principle (I) — Day 14

### Principle/Pattern Definition

**Concept**: **Interface Segregation Principle (ISP)** — "Clients should not be forced to depend on interfaces they do not use." Bare interfaces ko todo chhote, focused interfaces mein. Ek class ko sirf woh methods chahiye jo woh actually use karta hai.

**Bhai, Simple Mein Samjho**: 

Socho ek **universal remote** banaya gaya jo TV, AC, geyser, microwave, fan — sab control karta hai. 200 buttons hain. Tumne sirf TV ke liye liya — par 200 buttons ke saath aata hai. Confusing! Behtar yeh ke alag TV remote, alag AC remote — har ek apne use case ke liye focused.

Code mein same — agar ek interface mein 20 methods hain, aur tumhari class ko sirf 2 chahiye, baki 18 ka `UnsupportedOperationException` throw karna parega. Yeh ISP violation hai.

**Real-Life Analogy (Pakistani Context)**:

"Bhai, ISP aisa hai jaise **darzi (tailor)** ka kaam. Tum kameez silwane gaye ho. Darzi pucche: 'Suit silwana hai? Sherwani? Pant? Coat?' — agar har customer ko poori menu padhni pare, time waste. Behtar specialist darzi: koi sirf kameez-shalwar specialist, koi suit specialist, koi shaadi ke joray ka specialist. Tum apne use case ke specialist ke paas jate ho. Yeh ISP hai — har interface ek specialty."

Ya phir: "**FoodPanda** rider app dekho. Rider ko sirf chahiye: order accept karo, pickup karo, deliver karo. Restaurant owner ko chahiye: menu update karo, orders dekho. Customer ko chahiye: order place karo. Agar Foodpanda ek hi `IFoodPandaUser` interface bana de jisme 50 methods hain (`acceptOrder`, `updateMenu`, `placeOrder`, `viewRiderEarnings`, ...) — disaster! Har role apna interface deserves."

---

### Aaj Ke Brainer Mein Yeh Pattern/Principle Kahan Hai?

Hum ne `EmailSender` interface banaya — **sirf ek method**: `send(to, subject, body)`. Yahi ISP ka direct application hai.

**Galat approach (ISP violation)** hota agar hum ek bara `INotificationService` banate:
```java
interface INotificationService {
    void sendEmail(...);
    void sendSMS(...);
    void sendPushNotification(...);
    void sendWhatsAppMessage(...);
    void sendSlackMessage(...);
    void scheduleNotification(...);
    void cancelScheduledNotification(...);
    void getNotificationHistory(...);
}
```

Phir `EmailVerificationService` ko inject karte to **8 methods milte jabki sirf 1 chahiye**. Agar koi method add hota interface mein, **sare implementations break** hote — even those who don't care. **ISP says: small focused interfaces.**

Hamare design mein:
- `EmailSender` — sirf email
- `SmsSender` — sirf SMS (separate)
- `PushSender` — sirf push (separate)

`EmailVerificationService` sirf `EmailSender` depend karta hai. Clean!

---

### Code Mein Yeh Principle Kaise Apply Hota Hai?

**❌ BAD (Principle Violated)**:
```java
// Fat interface — sab kuch ek jagah
public interface INotificationService {
    void sendEmail(String to, String subject, String body);
    void sendSMS(String phone, String message);
    void sendPushNotification(String deviceToken, String message);
    void sendWhatsApp(String phone, String message);
    List<NotificationLog> getHistory(Long userId);
    void scheduleEmail(String to, String subject, String body, Instant when);
}

// Implementation forced to handle ALL methods
public class SendGridProvider implements INotificationService {
    public void sendEmail(...) { /* actual */ }
    public void sendSMS(...) { 
        throw new UnsupportedOperationException("SendGrid doesn't do SMS"); 
    }
    public void sendPushNotification(...) { 
        throw new UnsupportedOperationException(); 
    }
    public void sendWhatsApp(...) { 
        throw new UnsupportedOperationException(); 
    }
    // ... etc — ugly and dangerous
}

// Caller forced to know about 6 methods when it needs 1
public class EmailVerificationService {
    private final INotificationService notif;  // BAD: bloated dependency
    
    public void send(String email) {
        notif.sendEmail(email, "Verify", "...");  // sirf yeh chahiye!
    }
}
```

**✅ GOOD (Principle Followed)**:
```java
// Small focused interfaces — Interface Segregation
public interface EmailSender {
    void send(String to, String subject, String htmlBody);
}

public interface SmsSender {
    void send(String phoneNumber, String message);
}

public interface PushNotificationSender {
    void send(String deviceToken, String title, String body);
}

// Each provider implements only what it can
public class SendGridEmailSender implements EmailSender {
    public void send(String to, String subject, String htmlBody) {
        // Real implementation, no UnsupportedOperationException nonsense
    }
}

public class TwilioSmsSender implements SmsSender {
    public void send(String phone, String message) {
        // Twilio's actual SMS logic
    }
}

// Caller depends ONLY on what it needs
public class EmailVerificationService {
    private final EmailSender emailSender;  // GOOD: minimal dependency
    
    public EmailVerificationService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
    
    public void send(String email) {
        emailSender.send(email, "Verify", "...");
    }
}

// SMS-based 2FA service — different dependency
public class TwoFactorService {
    private final SmsSender smsSender;  // Only SMS needed
}
```

---

### Java vs .NET Mein Yeh Principle Kaise Implement Hota Hai?

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Interface keyword | `interface` | `interface` |
| Multiple interface impl | `implements I1, I2` | `: I1, I2` |
| Default methods | Java 8+ `default` methods | C# 8+ default interface methods |
| DI registration | `@Bean` / `@Component` per interface | `services.AddScoped<I, Impl>()` per interface |
| Composing many | Spring `@Qualifier` | Named registration / keyed services (C# 8+) |

```csharp
// C# equivalent — ISP applied
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string htmlBody);
}

public interface ISmsSender
{
    Task SendAsync(string phoneNumber, string message);
}

public class SendGridEmailSender : IEmailSender
{
    public Task SendAsync(string to, string subject, string htmlBody)
    {
        // Real implementation only
    }
}

public class EmailVerificationService
{
    private readonly IEmailSender _email;
    public EmailVerificationService(IEmailSender email) => _email = email;
    
    public Task SendVerificationAsync(string emailAddr)
        => _email.SendAsync(emailAddr, "Verify", "...");
}

// In Program.cs
builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();
builder.Services.AddScoped<ISmsSender, TwilioSmsSender>();
```

---

### 🎯 Cross-Questioning Drill (Interview Defense)

**Cross Q1**: "ISP kyun zaroori hai? Bina iske kya hota?"
**Confident Answer**:

"Bhai, bina ISP ke teen bari problems hain:

(1) **Forced implementations**: Agar `INotificationService` mein 10 methods hain aur SendGrid sirf email karta hai, baki 9 methods mein `throw new UnsupportedOperationException()` likhna parega. Yeh **Liskov Substitution** bhi violate karta hai — kyunki implementation behave nahi karta jaisa interface promise karta hai.

(2) **Ripple effect on changes**: Interface mein naya method add karo, **sari** implementations break. 50 providers hain to 50 jagah change. ISP follow karte, alag interface bana lete naye feature ke liye — purana code untouched.

(3) **Misleading API for consumers**: `EmailVerificationService` ko constructor mein `INotificationService` mila jisme 10 methods hain — usse pata nahi kya use karna safe hai. Plus testing mein har method mock karna parega even if unused. Small interface = clear contract = easy mocking."

---

**Cross Q2**: "Tumne ISP follow kiya, kya alternative tha?"
**Confident Answer**:

"Alternatives the:

(1) **Fat interface with default methods** (Java 8+, C# 8+): Bara interface but methods ki default implementation interface mein de do. Problem: consumers ko abhi bhi pata nahi kaunsa method 'real' hai vs default. Hidden complexity.

(2) **Single class with multiple methods** (no interface): Direct concrete class inject karo. Problem: testing mushkil, mocking mushkil, provider switch mushkil (SendGrid → SES). Dependency Inversion bhi violate.

(3) **Service Locator pattern**: Ek `NotificationLocator.getEmailSender()` se grab karo. Anti-pattern hai — hidden dependencies, hard to test.

ISP best balance hai: explicit small interfaces + DI = easy to test, easy to swap, clear contracts."

---

**Cross Q3**: "ISP jaan-boojh ke violate kab kar sakte ho?"
**Confident Answer**:

"Maturity yeh hai ke principles **dogmatic** nahi banayein. Violate karna chalega jab:

(1) **Tightly cohesive operations**: `Repository<T>` mein `save/findById/delete/update` together rakho — yeh CRUD ek cohesive unit hai. Inhe alag interface mein todna over-engineering hai.

(2) **Truly universal contracts**: `Comparable<T>` mein sirf `compareTo` hai — focused already. `Iterable<T>` mein sirf `iterator()`. Inhe further todna possible nahi.

(3) **Small team, prototype code**: Agar 2-week MVP bana rahe ho, hyper-segregated interfaces overhead hain. Ship first, refactor when grows.

But pakka rule yeh hai: **agar koi implementation `UnsupportedOperationException` throw karta hai, ISP violate ho raha hai. Refactor karo.**"

---

**Cross Q4**: "Yeh principle Spring/EF Core mein automatically follow hota hai ya manual?"
**Confident Answer**:

"Frameworks ne ISP mostly **promote** karte hain but enforce nahi karte:

**Spring**: `JdbcTemplate`, `RestTemplate` jaise classes bare hain — yeh ISP violation hai technically. But naye Spring APIs (e.g., `WebClient`, reactive `R2dbcEntityTemplate`) chhote focused interfaces use karte hain. Spring Data ka `JpaRepository` bara hai — but `CrudRepository`, `PagingAndSortingRepository`, `QueryByExampleExecutor` mein toda gaya hai — you pick what you need. Yeh ISP ka classic example hai!

**EF Core**: `DbContext` bara hai (Save, Find, Update, etc.) — convenience ke liye. But agar tum Repository pattern wrap karte ho top pe, tum apne interfaces choose karte ho — yahin ISP apply karo.

**ASP.NET Core Identity**: `IEmailSender` deliberately ek method hai — Microsoft ne ISP follow kiya. Inspire ho sakte ho."

---

**Cross Q5**: "Production mein ISP scale kaise karta hai?"
**Confident Answer**:

"Microservices architecture mein ISP service level pe extend hota hai:

(1) **Service contracts (gRPC/REST)**: Har service ka API focused hona chahiye. `UserService` mein `getUser`, `updateUser` rakho — `processPayment` nahi. Agar mix karoge, **service boundaries** loose ho jayenge — Conway's Law issues.

(2) **Event consumers (Kafka)**: Har consumer sirf woh topics subscribe kare jo woh actually process karta hai — `EmailWorker` sirf `user.registered`, `password.reset` topics. Sab topics ek consumer mein dalna = ISP violation at architecture level.

(3) **Backwards compatibility**: ISP follow karne se naye methods alag interface mein add hote — purane clients impact nahi hote. Multi-version API support easy.

Real example: **Stripe** ka API breakdown — `payments`, `subscriptions`, `connect`, `terminal` — sab alag focused APIs. Ek bara `IStripeAPI` nahi hai."

---

### 🧩 Pattern Combination (How Today's Pattern Pairs With Others)

**Pairs Well With**: 
- **Dependency Inversion Principle (DIP)** — small interface + inject abstraction = clean code
- **Single Responsibility Principle (SRP)** — small interface naturally serves one responsibility
- **Strategy Pattern** — small interface = perfect strategy contract (e.g., `EmailSender` strategies: SendGrid, SES, SMTP)
- **Decorator Pattern** — focused interface easy to decorate (e.g., `LoggingEmailSender` wraps `EmailSender`)
- **Adapter Pattern** — small interface = small adapter, easy to write

**Conflicts With**: 
- **Convenience APIs** — chhote interfaces zyada files matlab zyada navigation
- **Active Record Pattern** — Active Record entity me sab methods rakhta hai (fat by design)
- **God Object anti-pattern** — directly opposite philosophy

---

### 🎓 Real Production Code Where This Matters

**Spring Data Repository Hierarchy** — perfect ISP example:
```java
// Spring breaks repository capabilities into focused interfaces
public interface Repository<T, ID> {}  // marker only

public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);
    Optional<T> findById(ID id);
    // ... basic CRUD
}

public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}

public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID> {
    void flush();
    <S extends T> List<S> saveAll(Iterable<S> entities);
}
```

You choose what you need:
- Just CRUD? Extend `CrudRepository`
- Need paging? Extend `PagingAndSortingRepository`  
- Need JPA-specific? Extend `JpaRepository`

Nobody forces you to take everything. **Pure ISP application.**

---

### 💡 Memory Hook for This Principle/Pattern

**ISP**: "**CHOTI PLATE, EK DISH**" — har plate (interface) pe ek hi dish (responsibility). Buffet ki bari thali mein 20 dishes mix mat karo — small plates, focused dishes.

Alternative: "**REMOTE PER REMOTE, RIMOTE NA BANAO**" — har device ka apna remote, ek bara universal mat banao.

---

### ⚠️ Common Mistakes Jo Aaj Bachne Hain

**❌ Mistake 1**: "Ek bari `IService` interface bana di jisme 15 methods hain — DRY ke naam pe"
**Why it's wrong**: DRY (Don't Repeat Yourself) data/logic ke liye hai, **interfaces ke liye nahi**. Interfaces ko **client perspective** se design karo, not implementation perspective.
**Correct approach**: Har client ki needs identify karo. Agar 5 clients hain aur har ek 2 methods use karta hai, 5 chhote interfaces banao (even agar overlap ho).

---

**❌ Mistake 2**: "Interface tab banao jab unit test likhna ho — warna concrete class theek hai"
**Why it's wrong**: Yeh **YAGNI ka misuse** hai. Interface sirf testing ke liye nahi — **future flexibility** ke liye hai. Production mein email provider switch karna pare to **abhi pachta paoge**.
**Correct approach**: Boundary classes (anything talking to external systems: DB, HTTP, email, file system) ke liye **hamesha** interface banao. Internal pure logic ke liye optional.

---

**❌ Mistake 3**: "Mein ne `IRepository<T>` generic bana diya — yeh ISP follow karta hai kyunki sirf CRUD hai"
**Why it's wrong**: Generic interface mein bhi unnecessary methods ho sakte hain. `IReadOnlyRepository<T>` vs `IWriteRepository<T>` zyada precise ISP follow hai — query handlers ko sirf read chahiye, command handlers ko sirf write.
**Correct approach**: **CQRS pattern** sath combine karo — read aur write interfaces alag rakho. Yeh advanced level ISP hai.

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom):

```
1. USER ACTION (Angular)
   └─▶ User signup karta hai → Backend response milta hai "Check your email"
       └─▶ User email mein link click karta hai
           └─▶ Angular /verify route load → ActivatedRoute queryParamMap
               └─▶ Loading state dikhata hai (UX)

2. API REQUEST (Spring Boot / .NET)
   └─▶ POST /api/auth/verify with {userId, token}
       └─▶ Controller method invoked
           └─▶ EmailVerificationService.verifyToken(userId, rawToken)

3. BUSINESS LOGIC (Java / C#)
   └─▶ @Transactional starts
       └─▶ Fetch active token by userId
           └─▶ BCrypt.checkpw(rawToken, hashedToken)
               └─▶ Validate expiry + used flag

4. DATABASE (SQL)
   └─▶ SELECT WITH (UPDLOCK) on verification_tokens
       └─▶ UPDATE tokens SET used = 1
           └─▶ UPDATE users SET email_verified = 1
               └─▶ COMMIT (atomic)

5. EVENT PUBLISHING (System Design)
   └─▶ Publish UserEmailVerifiedEvent to Kafka
       └─▶ Async consumers react:
           ├─▶ Welcome email worker
           ├─▶ Analytics tracker
           └─▶ Coupon-grant service

6. RESPONSE (All layers)
   └─▶ 200 OK → Angular success state → countdown → redirect to dashboard
```

**What Breaks If You Skip ANY Layer**:

- **Skip SecureRandom (Java)**: Predictable tokens → mass account takeover
- **Skip @Transactional**: Token marked used, but `email_verified` not set → user stuck
- **Skip UPDLOCK (SQL)**: Race condition → two requests verify same token
- **Skip Angular state handling**: Blank white screen → user panic → support tickets
- **Skip message queue (System Design)**: Email send blocks API → slow signup → user abandons

---

## 🧭 MENTAL MAP — How to Memorize This

```
                [EMAIL VERIFICATION]
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   [Frontend]      [Backend]        [Database]
   Angular         Spring/.NET      SQL Server
        │               │               │
   Route guard     SecureRandom    token table
   queryParams     BCrypt hash     filtered idx
   3 states        @Transactional  UPDLOCK
   Resend          EmailSender I.  cleanup job
        │               │               │
        └───────────────┼───────────────┘
                        │
                [SYSTEM DESIGN]
                Async Kafka events
                Retry + DLQ
                
                [OOP LENS]
                ISP: small focused interfaces
                EmailSender ≠ NotificationService
```

**Mental Story to Remember (Roman Urdu)**: 

"Bhai, socho tum Daraz ke courier system bana rahe ho. User ne signup kiya — woh ek **shaadi ka invitation card** print karwana chahta hai. 

- **Frontend (Angular)** = Card design (user ne dekha)
- **Backend (Spring/.NET)** = Printer machine (card actually print karta hai)
- **SecureRandom** = Unique invitation number (predictable nahi)
- **Database** = Wedding planner ka register (kaunse cards bheje, kis ko)
- **Email (Kafka async)** = Courier service (cards deliver karta hai)
- **ISP** = Different services for different jobs (courier alag, printer alag, designer alag)

Agar koi bhi missing ho — fake card chal jayega, duplicate invitations jayenge, ya guest confused ho jayega."

**Acronym/Mnemonic**: 
**STIVE** = **S**ecureRandom + **T**oken hash + **I**nterface segregated + **V**erify expiry + **E**vents async

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Design an email verification system for a Daraz-scale e-commerce platform. Walk me through the full flow — frontend, backend, database, and how you handle scale."

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Iss scenario ko handle karne ke liye, hum 5 layers ko coordinate karenge — secure token generation backend pe, hashed storage in DB, asynchronous email delivery via queue, user-friendly states in Angular, aur ISP-based interface design taake email provider switch karna trivial ho."

**Body** (60 sec):
1. **Frontend** (Angular): User clicks link → `ActivatedRoute` se token + userId extract → 3 states: loading/success/error → resend button on expiry
2. **Backend API** (Spring/.NET): `POST /verify` controller → service layer mein `@Transactional`
3. **Business Logic** (Java/C#): `SecureRandom` 32-byte token → `BCrypt` hash → `EmailSender` interface (ISP) inject → BCrypt verify on validation
4. **Database** (SQL): Separate `verification_tokens` table → filtered index on `(user_id, expires_at) WHERE used = 0` → `UPDLOCK` for atomic mark-used → daily cleanup job
5. **Architecture**: `UserRegisteredEvent` to Kafka → async email worker → retry queue → DLQ for failures

**Closing** (10 sec):
"By combining these layers with Interface Segregation aur async event-driven architecture, hum guarantee karte hain ke verification secure hai, scalable hai 10M users tak, aur email provider switching zero-impact hai."

### Under-the-Hood Concepts You MUST Know

1. **SecureRandom vs Random**: OS entropy source vs predictable LCG — security critical
2. **BCrypt cost factor**: Work factor 10 = ~100ms per hash; balance security vs latency
3. **Filtered Index**: Index only on `WHERE used = 0` rows — saves space, faster active-token queries
4. **UPDLOCK semantics**: Acquires update lock at SELECT — prevents lost updates without escalating to serializable
5. **Spring AOP @Transactional proxy**: Runtime CGLIB subclass intercepts methods — that's why `private` methods don't get transaction

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

**Counter Q1 (Trade-off focused)**: "Why hash the token in DB? Token is one-time use anyway."
**Your Answer**: 
"Bhai, defense-in-depth principle hai. Agar DB leak ho jaye (insider attack ya SQL injection), attacker ke paas **all unused tokens** ka plain text mil jayega — woh seconds mein 100,000 accounts verify kar sakta hai before tokens expire. BCrypt hash karne se leak ke baad bhi tokens useless hain — attacker ko brute force karna parega per token. Cost: ~100ms extra per verification. Worth it. **Stripe, GitHub, Auth0 — sab yeh karte hain.**"

---

**Counter Q2 (Scale focused)**: "How do you handle 10 million signups per day? Token table will explode."
**Your Answer**: 
"Teen strategies:

(1) **Partitioning**: `verification_tokens` table ko `created_at` pe monthly partition karo. Old partitions DROP karna fast hai vs row-by-row DELETE.

(2) **Aggressive cleanup**: Daily job se 7-day-old expired tokens delete. Used tokens 30 din baad delete. Avg active row count manageable rehta hai.

(3) **Move to TTL store**: Redis/DynamoDB TTL pe rakho tokens — auto-expire built-in. SQL Server pe sirf user-level data. Read-heavy verification flow Redis pe sub-millisecond ho jata hai. Trade-off: cache miss ke liye fallback chahiye.

10M signups = ~115/sec sustained. Single Postgres instance bhi handle kar sakta hai with proper indexing — RDS db.r5.2xlarge enough."

---

**Counter Q3 (Failure mode focused)**: "What if SendGrid is down for 2 hours during a flash sale?"
**Your Answer**: 
"ISP pattern + async queue saves us:

(1) **Synchronous wouldn't work** — every signup API would block 30s timeout → users abandon → revenue loss.

(2) **Our design**: signup completes, event publishes to Kafka, email worker tries SendGrid. If fails: **exponential backoff retry** (2s, 4s, 8s, 16s...). After 5 retries → DLQ.

(3) **Multi-provider failover**: Because we used `EmailSender` interface (ISP!), runtime mein switch kar sakte hain — primary SendGrid, secondary AWS SES. Code change zero.

(4) **Status page**: 'Email may be delayed' banner show karo Angular pe — proactive communication.

Real example: Twilio outage 2023, **Notion** ka email worker auto-switched to SES because they had interface abstraction — zero customer impact."

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "What if a user clicks the verification link from inside Gmail's preview pane — Gmail bot might auto-click it. How do you prevent the token from being burned before the user actually clicks?"

**Junior Answer**: "Add captcha or rate limit per IP."

**Senior Answer**: "Bhai, yeh subtle real-world bug hai — Gmail/Outlook URL scanners auto-fetch links for safety preview, accidentally consuming our tokens. **Solution**:

(1) **GET vs POST split**: Link mein `/verify-page?token=xxx` ho — GET endpoint sirf React/Angular SPA serve kare, **token consume nahi kare**. SPA load hone ke baad **user action (button click)** se POST `/verify` call ho — actual verification.

(2) **Confirmation page**: Click karne pe '✅ Confirm Verification' button dikhao. User explicitly click kare. Auto-bots single click nahi karte usually.

(3) **Detect bot signature**: User-Agent check (`GoogleImageProxy`, `Outlook`, etc.) → reject silently from GET endpoint with cache headers, but allow real browsers.

**Stripe, Auth0, Vercel** — sab yeh do-step pattern follow karte hain. Single-click verification ek **anti-pattern** hai modern email security mein."

### Red Flag Signals (Don't Say These!)

- ❌ "I'll use `Math.random()` for the token" — Why: Not cryptographically secure, easily predictable, attack vector
- ❌ "Store token in plain text in DB — it's temporary anyway" — Why: DB leak = mass account compromise, no defense-in-depth
- ❌ "Email send is part of the same transaction" — Why: SMTP can take 30s, will block API, can't rollback sent emails
- ❌ "Use Singleton for `EmailSender`" — Why: Singleton + state = testing nightmare; use DI instead
- ❌ "One big `INotificationService` is fine, less files" — Why: ISP violation, every change ripples, forced UnsupportedOperationException

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ Why `SecureRandom` is mandatory vs `java.util.Random` for token generation
2. ✅ How BCrypt hashing protects tokens even if database is compromised
3. ✅ How `@Transactional` + `UPDLOCK` together prevent race conditions in token validation
4. ✅ How Interface Segregation Principle enables provider-switching (SendGrid ↔ SES) without code changes
5. ✅ Why async email sending via Kafka is essential for scale and resilience

---

## 🔗 Tomorrow's Connection Hint

Tomorrow we'll cover: **Day 15 — Phone OTP Verification (SMS-based)** with OOP focus on **SOLID — Dependency Inversion Principle (D)**

Aaj ke concept se kaise connected hai: 
Aaj humne `EmailSender` interface banaya (ISP). Kal `SmsSender` interface banayenge — same pattern, different channel. Aur DIP layer add karenge: `OtpService` will depend on **abstractions** (interfaces), not concrete Twilio/Vonage classes. Aaj jo ISP seekha, kal usko DIP ke saath combine karenge — together they form the **D + I** half of SOLID.

---

## 📚 Progress Tracker

```
🟢 Beginner     [▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░] Day 14/20
🟡 Intermediate [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] Day 0/30
🟠 Advanced     [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] Day 0/40
🔴 Expert       [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] Day 0/50
⚫ Master       [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] Day 0/60
🟣 Architect    [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] Day 0/70
💎 Principal    [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] Day 0/95
```
