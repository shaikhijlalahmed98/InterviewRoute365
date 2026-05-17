# 🎯 🟢 Day 4 of Beginner (Level 1 of 7): Password Reset Flow (Secure Token-Based)

**Overall Day**: Day 4 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 4 of 20 in this level
**Today's Theme**: Secure password reset using one-time tokens — and how Polymorphism (method overriding) makes the notification system pluggable.

---

## 📖 The Brainer Scenario (Real Problem) — Step by Step

Bhai, situation yeh hai. **Daraz** pe Ahmed bhai ka account hai. Kal raat tak login ho raha tha. Aaj subah password bhool gaya. Ab usko reset karna hai.

Yeh "Forgot Password" feature dikhne mein 2 textbox aur 1 button hai — but **andar 11 cheezein ho rahi hain** jo agar koi bhi galat ho jaye, security breach ho jata hai.

**Step-by-step kya hota hai (clock ke saath):**

```
T+0ms     Ahmed clicks "Forgot Password" link on Daraz
T+1ms     Browser shows form: "Apna email daalein"
T+2000ms  Ahmed types "ahmed@gmail.com", clicks Submit
T+2010ms  Browser sends POST request to Daraz backend
T+2015ms  Backend checks: "Is this IP spamming? More than 3 requests/hour?"
T+2020ms  Backend checks: "Does ahmed@gmail.com exist in users table?"
T+2025ms  YES exists → Generate a secret token (32 random bytes)
T+2026ms  Hash the token (SHA-256), save HASH (not raw token) to DB
T+2030ms  Publish event to Kafka: "Send reset link to user X"
T+2040ms  Return HTTP 200 to browser (FAST — don't wait for email)
T+2045ms  Browser shows: "If this email is registered, link sent"
T+5000ms  (Async) Email service picks up Kafka event
T+7000ms  (Async) Email arrives in Ahmed's inbox with the RAW token in URL
T+30000ms Ahmed clicks the email link
T+30050ms Browser opens: daraz.pk/reset?token=abc123xyz...
T+45000ms Ahmed types new password, clicks Submit
T+45050ms Backend hashes the received token, looks up in DB
T+45055ms Found + not expired + not used → atomically mark used=true
T+45060ms Update user's password_hash with new BCrypt hash
T+45065ms Delete ALL active sessions for Ahmed (force logout everywhere)
T+45070ms Return success → Browser redirects to login page
```

**11 critical things hidden in here**: rate limiting, anti-enumeration, secure randomness, token hashing for storage, async email dispatch, expiry timing, single-use enforcement, atomic SQL update, password hashing, session invalidation, polymorphic notification.

Aaj hum **har step ke "kyun"** ko depth mein samjhenge.

---

### The Real Challenge (The "Gotchas" — 4 Attacks Hackers Use)

#### 🪤 Gotcha 1: Timing Attack (User Enumeration via Stopwatch)

**Attack**: Hacker tries `random123@gmail.com` (fake email). Server takes 50ms to respond. Hacker tries `ahmed@gmail.com` (real email). Server takes 250ms (because it generated token, saved to DB, queued email — extra work). Hacker now **knows** Ahmed has an account on Daraz.

**Why it matters**: Hacker collects a list of confirmed real emails. Phir credential stuffing attack — wahi emails Facebook, gmail, banking sites pe try karta hai with leaked passwords from other breaches. Many users **reuse passwords** → mass account takeover.

**Fix**: Both code paths (exists / doesn't exist) take **equal time**. Or accept the tiny leak but eliminate the **obvious** leaks (different response message, different status code).

---

#### 🪤 Gotcha 2: Token Storage in Plaintext

**Attack**: Daraz DB gets breached (SQL injection, insider leak, backup theft). If reset tokens are stored in plaintext, attacker copies all active tokens → uses them to reset every active user's password in next 15 minutes → mass account takeover before anyone notices.

**Why it matters**: Tokens are **credentials**. Treat them like passwords — never store plaintext.

**Fix**: Store **SHA-256 hash** of the token. When user clicks link, hash the token they send and compare hashes. (More on what hash means below.)

---

#### 🪤 Gotcha 3: Double-Click Race Condition (TOCTOU Bug)

**TOCTOU** = **Time Of Check, Time Of Use** — bug where you *check* a condition (e.g. "is token unused?"), then *use* it (mark as used), but **between** check and use, something changes.

**Attack**: Ahmed opens email on phone aur laptop dono pe. Email link click karta hai dono jagah simultaneously (race). Server gets 2 requests at same millisecond:

```
Request A (phone):    SELECT used FROM tokens WHERE hash=X → used=false → OK
Request B (laptop):   SELECT used FROM tokens WHERE hash=X → used=false → OK
Request A:            UPDATE tokens SET used=true WHERE hash=X
Request B:            UPDATE tokens SET used=true WHERE hash=X
                      Both requests proceed to set password!
```

**Why it matters**: Double password reset. If Ahmed's email was compromised AND he himself clicks link, hacker can hijack the second use.

**Fix**: **Atomic** check-and-update in single SQL statement: `UPDATE tokens SET used=true WHERE hash=X AND used=false`. SQL engine takes a row lock — only ONE update succeeds, other gets rowcount=0.

---

#### 🪤 Gotcha 4: Old Sessions Survive Password Change

**Attack**: Hacker stole Ahmed's password 2 days ago, logged in, has an active session/JWT. Ahmed notices weird activity, resets password. **But hacker's session JWT is still valid for next 24 hours** (until it expires) — hacker still has access!

**Why it matters**: Password reset ka **whole point** unauthorized access kaatna hai. Agar old sessions zinda hain, reset useless hai.

**Fix**: On password change, invalidate **all** active sessions for this user. (We'll cover 4 techniques for this in the Senior Question.)

---

### Why This Matters in Production

- **Stripe** (payments): 1-hour expiry, HMAC-signed tokens. HMAC = **H**ash-based **M**essage **A**uthentication **C**ode = a hash combined with a secret key, so server can verify "yes I generated this token" without DB lookup.
- **GitHub**: Returns identical response for both valid and invalid emails — explicitly mentioned in their security docs as enumeration prevention.
- **Google**: Resets invalidate ALL OAuth refresh tokens + all browser sessions globally — one of the strongest implementations in industry.
- **AWS Cognito**: Uses 6-digit OTP code instead of long link, stored hashed, auto-marked used on first attempt (right or wrong).

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: Secure random token generation, one-way hashing, atomic single-use enforcement, pluggable notifier via interfaces.

### Under the Hood — Yeh Kaise Kaam Karta Hai (Step by Step)

Pehle 4 cheezein samjho — yeh hi foundation hai:

#### 1. `Random` vs `SecureRandom` — Kya Farak Hai?

Java mein `Random` class **predictable** hai. Internally yeh ek **Linear Congruential Generator (LCG)** use karta hai — matlab next number = `(previous * a + c) % m`. Math formula hai. Agar tum 5-6 outputs dekho, attacker formula reverse engineer karke aage ke saare numbers predict kar sakta hai.

`SecureRandom` ek **CSPRNG** use karta hai = **C**ryptographically **S**ecure **P**seudo-**R**andom **N**umber **G**enerator. "Pseudo" matlab "fake hai lekin asli jaisa lagta hai". "Cryptographically secure" matlab — **even if you see a million past outputs, you can't predict the next bit**. This is the mathematical guarantee.

Internally Linux pe `SecureRandom` `/dev/urandom` se bytes uthata hai. **`/dev/urandom` kya hai?** — yeh ek special "file" hai jo Linux OS provide karta hai. OS hardware se **entropy** collect karta hai (keyboard timings, mouse movements, disk read latencies, network packet arrival times — yeh sab physical chaos hai jo predict nahi ho sakta). Yeh chaos accumulate hota hai ek pool mein. Jab tum `/dev/urandom` read karte ho, OS is pool se hashing through random bytes deta hai.

**Bottom line**: For tokens, **always** `SecureRandom`. `Random` use karna = `Math.random()` use karna = security ka qatal.

#### 2. SHA-256 — One-Way Hash Function Kya Hota Hai?

**Hash function** ka kaam: koi bhi input do (1 byte ya 1 GB), fixed-size output deta hai (SHA-256 ka case mein 256 bits = 32 bytes).

3 properties critical hain:

- **Deterministic** — same input ne always same hash deta hai. `SHA-256("hello") = 2cf24dba5fb0a30e26e83b...` har baar.
- **One-way / pre-image resistant** — hash dekh ke original input nikalna **computationally impossible** (would take billions of years).
- **Avalanche effect** — input mein 1 bit change → output mein ~50% bits change.

```
SHA-256("hello")  = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
SHA-256("hello!") = ce06092fb948d9ffac7d1a376e404b26b7575bcc11ee05a4615fef4fec3a308b
```

Bilkul alag dikh raha hai sirf `!` add karne se.

**Why we hash tokens before storing**: If DB leaks, attacker sees hashes only. To use a token, you need the RAW value (which was only ever in user's email). Hash se raw value nahi nikal sakti. So leaked DB = useless tokens.

**Why SHA-256 (and not BCrypt) for tokens?**: BCrypt is slow on purpose (designed to take 100-200ms). Passwords are short and low-entropy (humans pick weak ones), so slowness prevents brute force. Tokens are 32 random bytes — already 2^256 possibilities. No brute force possible. Fast SHA-256 is fine — and fast matters because we validate tokens on every reset click.

#### 3. BCrypt for Passwords — Why Different?

**BCrypt** is a **password hashing algorithm** specifically designed to be **slow** (configurable via "cost factor"). It has 2 built-in features:

- **Built-in salt** — "salt" = random bytes added to password before hashing. So even if Ahmed and Ali both use password "12345", their stored hashes are different because their salts are different. Without salt, attacker can use a "rainbow table" (precomputed hashes of common passwords) to crack thousands of passwords instantly. With salt, attacker must crack each one separately.
- **Cost factor** (typically 10-12) — exponential slowness. Cost 10 = 2^10 = 1024 internal rounds. Cost 12 = 4096 rounds. Doubles every increment. Makes brute force 10000x slower than SHA-256.

`BCryptPasswordEncoder` Spring class ka method `encode(rawPassword)` automatically random salt generate karta hai, hash karta hai, aur saari info ek single string mein pack karta hai jisme prefix bhi hota hai (`$2a$10$...`). `matches(raw, stored)` method same salt extract karke verify karta hai.

#### 4. `@Version` Aur Optimistic Locking — Kaise Kaam Karta Hai

**Optimistic locking** ka philosophy: "Main assume karta hoon ke koi conflict nahi hoga. Check karunga commit ke time. Agar tab tak koi aur ne update kar diya, main fail ho jaunga."

**Pessimistic locking** ka opposite philosophy: "Pehle row LOCK karo, phir koi aur touch nahi kar sakta. Safe but slow."

JPA mein `@Version` annotation lagao ek field pe (`Long version`). Hibernate hr UPDATE ke saath WHERE clause mein version check add kar deta hai:

```sql
-- Tumne likha:
UPDATE tokens SET used = true WHERE id = 5;

-- Hibernate actually executes:
UPDATE tokens SET used = true, version = 6 WHERE id = 5 AND version = 5;
```

Agar `version = 5` match nahi karta (kyunki kisi aur ne update kiya, version 6 ban gaya), rowcount = 0 ho jata hai. Hibernate yeh detect karta hai aur `OptimisticLockException` throw karta hai. Iska matlab "kisi aur ne pehle modify kar diya, tumhara update reject."

Real-life analogy: shaadi ka pandit. Pehle wala bola "Rajesh ki shaadi Priya se." Pandit: "Theek hai, mere register mein note kar leta hoon — version 1." Doosra bola "Rajesh ki shaadi Sonia se." Pandit: "Ek minute, mere register mein abhi to Priya likha hai (version 1). Tum kis version pe baat kar rahe ho?" Conflict detected.

---

### Bhai, Simple Mein Samjho (One-Paragraph Mental Model)

Soch: Tum ek school principal ho. Bachcha bolta hai "Mera report card chahiye." Tum:
1. **Random reference number generate karte ho** (32-character koi bhi guess na kar sake) — yeh hai SecureRandom token.
2. Reference number ka **fingerprint** apne register mein likhte ho, asli number bachey ki maa ko sealed envelope mein dete ho — yeh hai SHA-256 hashing for storage.
3. Bolte ho "15 minute mein wapas aana, warna naya number lena parega" — yeh hai expiry.
4. Maa wapas aati hai, asli number deti hai, tum fingerprint match karte ho, **register mein "used" mark karte ho usi waqt** — atomic single-use.
5. Doosra bachcha duplicate maa banake aaya same number lekar? Register mein already used hai, reject. Race condition handled.

---

### Code Pattern (Full Working Example with Inline Comments)

```java
// ============ ENTITY (Database table mapping) ============
@Entity
@Table(name = "password_reset_tokens",
    indexes = {
        // Index = DB ka "phone directory" — fast lookup
        @Index(name = "idx_token_hash", columnList = "tokenHash", unique = true),
        @Index(name = "idx_user_expiry", columnList = "userId, expiresAt")
    })
public class PasswordResetToken {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;  // UUID = 128-bit random ID, collision practically impossible

    @Column(nullable = false, length = 64)
    private String tokenHash;  // SHA-256 hex = 64 characters always

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private Instant expiresAt;  // Instant = UTC timestamp, timezone-independent

    @Column(nullable = false)
    private boolean used = false;

    @Version
    private Long version;  // For optimistic locking — explained above
}

// ============ SERVICE (Business logic layer) ============
@Service
@RequiredArgsConstructor  // Lombok — generates constructor with all final fields
public class PasswordResetService {

    private final UserRepository userRepo;
    private final PasswordResetTokenRepository tokenRepo;
    private final PasswordEncoder passwordEncoder;  // Interface — BCrypt actual impl
    private final SessionService sessionService;
    private final NotificationSender notifier;  // Interface — polymorphism magic!
    private final SecureRandom random = new SecureRandom();  // CSPRNG instance

    /**
     * Called when user clicks "Forgot Password" and submits email.
     * GOAL: generate token, save hash, send email/SMS.
     * SECURITY: must look IDENTICAL whether email exists or not (anti-enumeration).
     */
    @Transactional  // Spring wraps this method in DB transaction (auto rollback on exception)
    public void requestReset(String email) {

        Optional<User> userOpt = userRepo.findByEmail(email);

        if (userOpt.isPresent()) {
            User user = userOpt.get();

            // STEP 1: Generate 32 random bytes from CSPRNG
            // 32 bytes = 256 bits = 2^256 possibilities ≈ 10^77
            // (universe mein atoms 10^80 hain — practically uncrackable)
            byte[] tokenBytes = new byte[32];
            random.nextBytes(tokenBytes);

            // STEP 2: Convert bytes to URL-safe string
            // Base64 URL encoding: uses A-Z a-z 0-9 - _ (no + / which break URLs)
            // 32 bytes → 43 character string
            String rawToken = Base64.getUrlEncoder()
                .withoutPadding()
                .encodeToString(tokenBytes);

            // STEP 3: Hash the token before storing
            // RAW token goes only to user's email. DB has only HASH.
            String tokenHash = sha256Hex(rawToken);

            // STEP 4: Kill any previous unused tokens for this user
            // (User clicked "Forgot Password" 5 times? Only LATEST link works.)
            tokenRepo.markAllUserTokensAsUsed(user.getId());

            // STEP 5: Save new token with 15-minute expiry
            PasswordResetToken token = new PasswordResetToken();
            token.setUserId(user.getId());
            token.setTokenHash(tokenHash);
            token.setExpiresAt(Instant.now().plus(15, ChronoUnit.MINUTES));
            tokenRepo.save(token);

            // STEP 6: Send notification via polymorphic dispatch
            // notifier could be EmailSender, SmsSender, WhatsAppSender — service doesn't care
            String resetLink = "https://daraz.pk/reset?token=" + rawToken;
            notifier.send(user, new PasswordResetMessage(resetLink));
        }

        // CRITICAL: We don't throw exception or return different status if user not found.
        // Same code path = same response time = no enumeration leak.
    }

    /**
     * Called when user clicks email link and submits new password.
     * GOAL: validate token, update password, kill all sessions.
     * SECURITY: atomic single-use enforcement, no token reuse possible.
     */
    @Transactional
    public void confirmReset(String rawToken, String newPassword) {

        // Hash incoming token to compare with stored hash
        String tokenHash = sha256Hex(rawToken);

        // Find token: must be unused, not expired
        // Note: just SELECT here, atomic enforcement is in setUsed below
        PasswordResetToken token = tokenRepo
            .findByTokenHashAndUsedFalseAndExpiresAtAfter(tokenHash, Instant.now())
            .orElseThrow(() -> new InvalidTokenException("Token invalid or expired"));

        User user = userRepo.findById(token.getUserId())
            .orElseThrow(() -> new InvalidTokenException("User not found"));

        // Hash and save new password
        // BCrypt internally generates random salt, embeds it in output
        user.setPasswordHash(passwordEncoder.encode(newPassword));
        userRepo.save(user);

        // Mark token used. @Version protects against race:
        // If another concurrent request already marked it used (incrementing version),
        // this save() throws OptimisticLockException.
        token.setUsed(true);
        tokenRepo.save(token);

        // Critical: kill all active sessions/JWTs for this user
        // Without this, hacker who knew old password keeps access.
        sessionService.invalidateAllSessions(user.getId());
    }

    /**
     * SHA-256 hash, return as 64-char hex string.
     */
    private String sha256Hex(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hashBytes = md.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hashBytes);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 should always be available", e);
        }
    }
}
```

**Interview phrasing**:
"Iss scenario mein raw token sirf email mein jata hai, DB mein sirf SHA-256 hash store hota hai — taake DB breach pe tokens leak na hon. Single-use guarantee `@Version` ke saath optimistic locking se enforce hoti hai. Aur user enumeration roknay ke liye, exists / not-exists dono code paths same response return karte hain."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: `RandomNumberGenerator`, `IPasswordHasher<TUser>`, EF Core concurrency tokens, polymorphic resolution via DI.

### Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai

`RandomNumberGenerator.Fill()` exact same security guarantee deta hai jo Java ka `SecureRandom` deta hai. Windows pe yeh **CNG** = **C**ryptography **N**ext **G**eneration use karta hai (Windows ka modern crypto API, replaced old CryptoAPI). Linux pe yeh bhi `/dev/urandom` use karta hai. Same OS entropy source.

EF Core ka `[Timestamp]` attribute SQL Server mein `ROWVERSION` column banata hai. **ROWVERSION kya hai?** — SQL Server ka special data type (8 bytes). Har baar jab row update hoti hai, SQL Server **automatically** is column ko increment kar deta hai — tum manually set nahi karte. Yeh database-wide unique counter hai (har row pe nahi, pure database mein ek hi counter).

EF Core jab UPDATE generate karta hai, automatic WHERE clause mein original rowversion add kar deta hai (just like Hibernate `@Version`):

```sql
UPDATE tokens SET used = 1
WHERE id = @id AND rowversion = @originalRowVersion;
```

Agar update ne 0 rows affect kiye (kyunki rowversion mismatch — kisi aur ne pehle update kar diya), EF Core `DbUpdateConcurrencyException` throw karta hai.

**Polymorphic DI in .NET**: Agar tum constructor mein `IEnumerable<INotificationSender>` inject karo, .NET ka DI container **saari** registered implementations ko collection mein de deta hai. .NET 8+ mein "keyed services" feature aaya — tum naam de ke specific implementation maang sakte ho:

```csharp
services.AddKeyedScoped<INotificationSender, EmailNotificationSender>("email");
var emailSender = serviceProvider.GetRequiredKeyedService<INotificationSender>("email");
```

### Bhai, .NET Mein Yeh Kaise Hota Hai

Logic 100% same — sirf framework syntax change. EF Core ka `SaveChangesAsync()` Hibernate ke `save()` jaisa hai — change tracker dekh ke automatic SQL generate karta hai. **Change tracker kya hai?** — EF Core background mein note rakhta hai konsi entity property change hui. Tum `entity.Name = "new"` likhte ho, change tracker note kar leta hai "Name modified". `SaveChangesAsync` pe sirf modified fields ka UPDATE generate karta hai (efficient).

### Code Pattern

```csharp
// ============ ENTITY ============
public class PasswordResetToken
{
    public Guid Id { get; set; }                          // .NET ka UUID = Guid
    public string TokenHash { get; set; } = default!;
    public long UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool Used { get; set; }

    [Timestamp]  // EF Core: this is the optimistic concurrency token
    public byte[] RowVersion { get; set; } = default!;
}

// ============ SERVICE ============
public class PasswordResetService
{
    private readonly AppDbContext _db;
    private readonly IPasswordHasher<User> _hasher;
    private readonly ISessionService _sessions;
    private readonly INotificationSender _notifier;  // Polymorphic interface

    public PasswordResetService(
        AppDbContext db,
        IPasswordHasher<User> hasher,
        ISessionService sessions,
        INotificationSender notifier)
    {
        _db = db;
        _hasher = hasher;
        _sessions = sessions;
        _notifier = notifier;
    }

    public async Task RequestResetAsync(string email)
    {
        var user = await _db.Users
            .FirstOrDefaultAsync(u => u.Email == email);

        if (user != null)
        {
            // STEP 1: 32 cryptographically secure random bytes
            var tokenBytes = new byte[32];
            RandomNumberGenerator.Fill(tokenBytes);

            // STEP 2: URL-safe Base64 encoding
            var rawToken = Base64UrlEncoder.Encode(tokenBytes);

            // STEP 3: Hash for storage
            var tokenHash = Sha256Hex(rawToken);

            // STEP 4: Invalidate old unused tokens — single SQL UPDATE, no entity loading
            // ExecuteUpdateAsync (EF Core 7+) = bulk update without change tracker overhead
            await _db.PasswordResetTokens
                .Where(t => t.UserId == user.Id && !t.Used)
                .ExecuteUpdateAsync(s => s.SetProperty(t => t.Used, true));

            // STEP 5: Insert new token
            _db.PasswordResetTokens.Add(new PasswordResetToken
            {
                Id = Guid.NewGuid(),
                UserId = user.Id,
                TokenHash = tokenHash,
                ExpiresAt = DateTime.UtcNow.AddMinutes(15),
                Used = false
            });

            await _db.SaveChangesAsync();

            // STEP 6: Polymorphic notification dispatch
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

        // HashPassword internally generates salt, embeds in output
        user.PasswordHash = _hasher.HashPassword(user, newPassword);
        token.Used = true;

        try
        {
            // EF Core compares RowVersion in WHERE clause automatically
            await _db.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            // Concurrent click won the race — this attempt fails cleanly
            throw new InvalidTokenException("Token already used");
        }

        await _sessions.InvalidateAllSessionsAsync(user.Id);
    }

    private static string Sha256Hex(string input)
    {
        using var sha = SHA256.Create();  // SHA-256 instance, disposed after use
        var bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(bytes).ToLowerInvariant();  // 64-char hex
    }
}
```

### Java vs .NET Comparison Table (With Plain-English Explanations)

| Feature | Java/Spring | .NET/C# | What it does |
|---------|-------------|---------|--------------|
| Secure random | `SecureRandom.nextBytes()` | `RandomNumberGenerator.Fill()` | Gives bytes unpredictable to attacker |
| Password hashing | `BCryptPasswordEncoder` | `IPasswordHasher<TUser>` (PBKDF2 default) | Slow hash designed to resist brute force |
| SHA-256 | `MessageDigest.getInstance("SHA-256")` | `SHA256.Create()` | Fast one-way hash for non-password data |
| Optimistic lock | `@Version` on field | `[Timestamp]` on `byte[]` field | Detect concurrent updates, reject conflict |
| Async pattern | `@Async` + `TaskExecutor` | `async/await` + `Task` | Non-blocking I/O, free thread for other work |
| Polymorphic DI | `@Qualifier` or `List<Interface>` | `IEnumerable<Interface>` or keyed services | Inject multiple implementations of same interface |
| URL-safe Base64 | `Base64.getUrlEncoder()` | `Base64UrlEncoder.Encode` | Convert bytes to URL-friendly string |
| Bulk update | `@Modifying` + JPQL | `ExecuteUpdateAsync` (EF 7+) | Single SQL UPDATE without loading entities |

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: Atomic check-and-update, B-tree indexes, isolation levels, row versioning for optimistic locking.

### Under the Hood — SQL Engine Yeh Kaise Karta Hai (Step by Step)

#### 1. UPDATE Statement Locking — Frame by Frame

Jab tum likhte ho:

```sql
UPDATE tokens SET used = 1 WHERE token_hash = 'abc...' AND used = 0;
```

SQL Server (or any RDBMS) yeh karta hai:

```
Step 1: Parser checks SQL syntax
Step 2: Query Optimizer dekhta hai indexes available — picks idx_token_hash
Step 3: Engine seeks to that token's row using B-tree index
Step 4: Acquires "U-Lock" (Update Lock) on that row
        — U-Lock = "I might update this, no one else lock for write"
Step 5: Reads current row, checks WHERE clause (used = 0?)
Step 6: If condition matches:
        a. Converts U-Lock to X-Lock (Exclusive Lock = "no one read or write")
        b. Modifies row data
        c. Writes change to Write-Ahead Log (WAL) first — durability
        d. Marks row dirty in buffer pool (will flush to disk later)
        e. Releases X-Lock at transaction commit
        f. Returns rowcount = 1
Step 7: If condition doesn't match (used was already 1):
        a. Releases U-Lock
        b. Returns rowcount = 0
```

**Write-Ahead Log (WAL)** kya hai? Database har change pehle ek sequential log file mein likhta hai, **phir** actual data file mein. Kyun? Crash recovery. Power chala gaya middle of update mein? On restart, DB log file padhta hai aur incomplete transactions ko replay/rollback karke consistent state mein restore karta hai. This is the "D" in **ACID** = Durability.

**B-tree index** kya hai? Imagine kar phone directory — naam alphabetically sorted. Tum middle se start karte ho, decide karte ho left ya right side. Har step mein search space half ho jata hai. 1 million entries mein search = ~20 steps (log₂ of 1M). B-tree wahi hai, lekin disk-page-friendly structure mein. Index = pre-sorted lookup ka shortcut, full table scan se 1000x faster.

#### 2. Atomic Check-and-Update — Race Ko Eliminate Karna

Yeh teen scenarios compare karo:

**❌ Approach 1: SELECT then UPDATE (TOCTOU bug)**

```sql
-- App code:
SELECT used FROM tokens WHERE token_hash = 'abc';  -- Returns false
-- App checks: "OK, not used yet"
UPDATE tokens SET used = 1 WHERE token_hash = 'abc';  -- Always updates
```

Race: 2 concurrent requests both SELECT (both see used=false), both UPDATE. Double consumption.

**❌ Approach 2: SELECT FOR UPDATE then UPDATE (Pessimistic Locking — works but slower)**

```sql
BEGIN TRANSACTION;
SELECT used FROM tokens WHERE token_hash = 'abc' FOR UPDATE;  -- Locks row
-- App checks: "OK, not used yet"
UPDATE tokens SET used = 1 WHERE token_hash = 'abc';
COMMIT;
```

Lock holds across multiple statements. Other requests **wait**. Safe but creates contention.

**✅ Approach 3: Atomic UPDATE with WHERE (Best)**

```sql
UPDATE tokens SET used = 1 WHERE token_hash = 'abc' AND used = 0;
-- Check rowcount
```

Single statement. SQL Server internally acquires U-Lock, checks `used=0`, converts to X-Lock if match, updates. **All in one atomic operation**. Two concurrent requests? One succeeds (rowcount=1), other fails (rowcount=0). No waiting, no contention beyond microseconds.

This is the **gold standard pattern** for single-use tokens, counter increments, claim flags, etc.

#### 3. Index Design for This Table

Tumhare table mein 3 query patterns hain:

```sql
-- Pattern A: Lookup by token hash (every confirm reset)
WHERE token_hash = ?

-- Pattern B: Find user's active tokens (during request reset, to invalidate old)
WHERE user_id = ? AND used = 0

-- Pattern C: Cleanup expired tokens (cron job)
WHERE expires_at < NOW()
```

Indexes:

```sql
-- Unique index on token_hash: O(log n) lookup, also enforces no duplicate hashes
CREATE UNIQUE INDEX idx_token_hash ON password_reset_tokens(token_hash);

-- Composite index for user_id queries
-- Order matters: leftmost columns must be in WHERE
CREATE INDEX idx_user_active ON password_reset_tokens(user_id, used, expires_at);

-- For cleanup job
CREATE INDEX idx_expires ON password_reset_tokens(expires_at);
```

**Composite index leftmost-prefix rule** kya hai? Index `(user_id, used, expires_at)` ka matlab DB ne data sort kiya: first by user_id, then by used, then by expires_at. Tum query kar sakte ho:
- `WHERE user_id = ?` ✅ Uses index
- `WHERE user_id = ? AND used = 0` ✅ Uses index
- `WHERE user_id = ? AND used = 0 AND expires_at > NOW()` ✅ Uses index
- `WHERE used = 0` ❌ Cannot use (skipped leftmost column user_id)
- `WHERE expires_at > NOW()` ❌ Cannot use (skipped first two)

Yeh "leftmost-prefix rule" hai — sirf left-to-right columns use kar sakte ho, beech ka skip nahi.

### Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai

Two flows yaad rakh:

1. **Token banao** — Pehle purane unused tokens mark `used=1` (single user 5 baar click kare to sirf latest valid). Phir naya insert.
2. **Token use karo** — **Atomic UPDATE** with WHERE. `rowcount=1` matlab success. `rowcount=0` matlab token invalid/expired/used.

### SQL Example (Full Working)

```sql
-- ============ TABLE DEFINITION ============
CREATE TABLE password_reset_tokens (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    user_id         BIGINT NOT NULL,
    token_hash      CHAR(64) NOT NULL,            -- SHA-256 hex, always 64 chars
    expires_at      DATETIME2 NOT NULL,
    used            BIT NOT NULL DEFAULT 0,       -- BIT = 1 bit boolean
    used_at         DATETIME2 NULL,
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    row_version     ROWVERSION,                   -- Auto-managed concurrency token

    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- ============ INDEXES ============
CREATE UNIQUE INDEX idx_token_hash
    ON password_reset_tokens(token_hash);

CREATE INDEX idx_user_active
    ON password_reset_tokens(user_id, used, expires_at);

CREATE INDEX idx_cleanup
    ON password_reset_tokens(expires_at);

-- ============ ATOMIC SINGLE-USE CONSUMPTION ============
BEGIN TRANSACTION;

-- Atomic: only succeeds if token unused AND not expired
UPDATE password_reset_tokens
SET used = 1,
    used_at = SYSUTCDATETIME()
WHERE token_hash = @tokenHash
  AND used = 0
  AND expires_at > SYSUTCDATETIME();

IF @@ROWCOUNT = 0
BEGIN
    -- Either token doesn't exist, is expired, or already used
    -- Same error for all 3 → no info leak to attacker
    ROLLBACK TRANSACTION;
    THROW 51000, 'Token invalid, expired, or already used', 1;
END

-- Now safely update the password
DECLARE @userId BIGINT = (
    SELECT user_id FROM password_reset_tokens
    WHERE token_hash = @tokenHash
);

UPDATE users
SET password_hash = @newPasswordHash,
    password_changed_at = SYSUTCDATETIME()
WHERE id = @userId;

-- Also kill all sessions for this user
DELETE FROM user_sessions WHERE user_id = @userId;

COMMIT TRANSACTION;
```

### The Gotcha (Detailed Explanation)

**TOCTOU bug ka real-world example**: ATM cash withdraw imagine karo without atomic check.

```
Bhai code:
  balance = SELECT balance FROM accounts WHERE id = X;  -- Returns 1000
  if (balance >= 500):
      UPDATE accounts SET balance = balance - 500 WHERE id = X;
      dispense_cash(500);
```

Tum aur tumhari biwi same time withdraw karte ho 500 each:

```
You:    SELECT balance → 1000  ✓ ≥ 500
Wife:   SELECT balance → 1000  ✓ ≥ 500
You:    UPDATE balance = 1000-500 = 500
Wife:   UPDATE balance = 1000-500 = 500
You:    Cash dispensed 500
Wife:   Cash dispensed 500
Final balance: 500. But you withdrew 1000 total! Bank lost 500.
```

**Atomic fix**:

```sql
UPDATE accounts SET balance = balance - 500
WHERE id = X AND balance >= 500;
-- Check rowcount
```

SQL engine row pe X-lock leta hai before checking. Second request waits, sees balance=500, condition `balance >= 500` fails, rowcount=0. Bank safe.

Same pattern, same lifesaver, for token consumption.

### Isolation Level Choice — Plain English

**Isolation level** = "How much do concurrent transactions see each other's uncommitted changes?" 4 standard levels:

| Level | What you see | Use case |
|-------|--------------|----------|
| READ UNCOMMITTED | Even uncommitted changes by others ("dirty reads") | Almost never use |
| READ COMMITTED | Only committed data, but data can change between reads in your transaction | **Default, fine for most** |
| REPEATABLE READ | Once you read a row, it stays the same in your transaction | When you SELECT same row twice and need consistency |
| SERIALIZABLE | Transactions appear to run one-by-one | When you need absolute correctness over performance |

For password reset: **READ COMMITTED is enough**. Why? Because our atomic UPDATE statement is itself atomic at row level — SQL Server takes X-lock on the row, no other transaction can interleave. Higher isolation = more locks = worse performance, no extra safety here.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Reactive Forms, RxJS operators, route param extraction, debouncing for performance, identical error UX.

### Under the Hood — Angular Yeh Kaise Karta Hai

#### 1. Reactive Forms — Form As an Observable

Angular ke 2 form types hain: **Template-Driven** (form in HTML, simple) aur **Reactive** (form in TypeScript, powerful). For anything beyond basic, use Reactive.

`FormControl` ka `valueChanges` ek **Observable<value>** hai. **Observable kya hai?** — sochne ka tarika: Observable ek "future stream of values" hai. Tum `subscribe` karte ho, jab bhi value change hoti hai, tumhara callback chalta hai. Like newsletter subscription — naya issue aaye, tumhe deliver hota hai. Promise sirf ek baar resolve hota hai, Observable multiple times values emit kar sakta hai.

#### 2. `debounceTime` — Spam Roknay Ki Magic

Soch: User password field mein type kar raha hai "MySecret123". Har keystroke pe agar tum password strength API call karo, **10 API calls** ho jayenge for one password.

`debounceTime(300)` solution: "Last keystroke ke 300ms baad agar koi naya keystroke nahi aaya, tab proceed karo. Drmiyaan mein agar aur key dabe, timer reset ho jaye."

Internally yeh `setTimeout` use karta hai aur `clearTimeout` se cancel karta hai. Result: 10 keystrokes → 1 final emit after 300ms idle.

```typescript
this.form.get('password')!.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged()  // skip if value is same as last emitted
).subscribe(value => calculateStrength(value));
```

#### 3. Zone.js — Angular Ka Change Detection Engine

Angular ko kaise pata chalta hai "kuch change hua, UI refresh karna hai"? **Zone.js** library JavaScript ke saare async APIs (setTimeout, fetch, addEventListener) ko **monkey-patch** karti hai = unke around wrapper add karti hai. Jab bhi koi async operation complete hota hai, Zone.js Angular ko notify karta hai "ho gaya, check kar le". Angular `tick()` chalata hai jo component tree traverse karta hai aur dirty components ko re-render karta hai.

Yeh "Magic" hai jo `setTimeout(() => this.x = 'new')` ke baad UI automatically update hoti hai bina tumhare kuch likhe.

#### 4. HttpClient Observables — Cold vs Hot

`this.http.post(...)` ek **cold Observable** return karta hai. "Cold" matlab — **subscribe karne tak request fire hi nahi hoti**. Subscribe karte hi HTTP call jata hai. Har naye subscriber pe naya HTTP call fire hota hai (potentially dangerous if you forget).

`async` pipe HTML mein (`{{ data$ | async }}`) automatically subscribe + unsubscribe handle karta hai component destroy pe — memory leak avoid.

### Bhai, Frontend Pe Yeh Kaise Handle Karein

User enumeration roknay ke liye **frontend pe bhi same message** chahiye, chahe email registered ho ya na. Token URL se uthao, naye password ka form dikhao, real-time strength check (debounced for performance).

### Code Pattern

```typescript
// ============ REQUEST RESET COMPONENT ============
@Component({
  selector: 'app-forgot-password',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email"
             type="email"
             placeholder="Apna email daalein" />

      <!-- Show validation error -->
      <div *ngIf="form.get('email')?.invalid &&
                  form.get('email')?.touched"
           class="error">
        Valid email daalein
      </div>

      <button [disabled]="form.invalid || loading">
        {{ loading ? 'Bhej raha hai...' : 'Reset Link Bhejo' }}
      </button>
    </form>

    <!-- IDENTICAL message regardless of whether email exists -->
    <div *ngIf="submitted" class="success">
      Agar yeh email registered hai, reset link bhej diya gaya hai.
      Inbox check karein (spam folder bhi).
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

    this.auth.requestPasswordReset(this.form.value.email!)
      .pipe(
        // finalize runs whether success or error — always reset loading
        finalize(() => this.loading = false)
      )
      .subscribe({
        // CRITICAL: same UI behavior on both success AND error
        // If we show different messages, attacker uses DevTools to enumerate
        next: () => this.submitted = true,
        error: () => this.submitted = true
      });
  }
}

// ============ CONFIRM RESET COMPONENT ============
@Component({
  selector: 'app-reset-password',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="password"
             type="password"
             placeholder="Naya Password (8+ characters)" />

      <!-- Real-time strength meter -->
      <div class="strength-bar" [ngClass]="(strength$ | async) || 'weak'">
        Strength: {{ strength$ | async }}
      </div>

      <input formControlName="confirm"
             type="password"
             placeholder="Password confirm karein" />

      <div *ngIf="form.errors?.['mismatch'] && form.get('confirm')?.touched"
           class="error">
        Passwords match nahi karte
      </div>

      <button [disabled]="form.invalid || loading">
        {{ loading ? 'Set kar raha hai...' : 'Naya Password Set Karein' }}
      </button>
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
    // Extract token from URL: /reset?token=abc...
    this.token = this.route.snapshot.queryParamMap.get('token') ?? '';

    if (!this.token) {
      // No token in URL → useless page, redirect
      this.router.navigate(['/login']);
      return;
    }

    // Real-time password strength
    // debounceTime(250): wait 250ms after last keystroke
    // distinctUntilChanged: skip if same value as before
    // map: transform value to strength rating
    this.strength$ = this.form.get('password')!.valueChanges.pipe(
      debounceTime(250),
      distinctUntilChanged(),
      map(pw => this.calculateStrength(pw ?? ''))
    );
  }

  submit() {
    this.loading = true;

    this.auth.confirmReset(this.token, this.form.value.password!)
      .pipe(finalize(() => this.loading = false))
      .subscribe({
        next: () => {
          // Success: redirect to login with success message
          this.router.navigate(['/login'], {
            queryParams: { resetSuccess: 'true' }
          });
        },
        error: err => {
          // Generic error — could be expired, used, or invalid
          // Don't leak which one to attacker
          alert(err.error?.message ?? 'Link expired ya invalid. Dobara request karein.');
        }
      });
  }

  // Custom validator: check password === confirm
  private matchValidator(group: AbstractControl): ValidationErrors | null {
    const pw = group.get('password')?.value;
    const cf = group.get('confirm')?.value;
    return pw === cf ? null : { mismatch: true };
  }

  // Password strength calculator
  // 4 criteria: uppercase, lowercase, digit, symbol
  private calculateStrength(pw: string): 'weak' | 'medium' | 'strong' {
    if (pw.length < 8) return 'weak';

    const hasUpper = /[A-Z]/.test(pw);
    const hasLower = /[a-z]/.test(pw);
    const hasDigit = /\d/.test(pw);
    const hasSymbol = /[^A-Za-z0-9]/.test(pw);

    const score = [hasUpper, hasLower, hasDigit, hasSymbol]
      .filter(Boolean).length;

    if (score >= 3 && pw.length >= 12) return 'strong';
    if (score >= 2) return 'medium';
    return 'weak';
  }
}
```

### UX Concern

Agar tum frontend pe alag message dikhao ("Email not found" vs "Reset sent"), attacker DevTools open karta hai, Network tab dekhta hai, fields/messages diff karta hai → enumeration done. **Identical response** har layer pe non-negotiable hai. UX team agar bole "user ko clear message do" — to bhi security wins. Industry standard yahi hai (GitHub, Stripe, Google sab same approach).

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: Strategy Pattern (Polymorphism in action) + Event-Driven async + Rate Limiting.

### Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai

#### 1. Strategy Pattern — Polymorphism Ka Practical Use

**Strategy Pattern** = "Algorithm ko alag classes mein wrap karo, interchangeable banao at runtime". Yeh literally polymorphism + intent ka combo hai.

Tumhare paas ek `NotificationSender` interface hai. Concrete implementations: `EmailNotificationSender`, `SmsNotificationSender`, `WhatsAppSender`. Service code sirf `INotificationSender` reference rakhta hai. Runtime pe DI container decide karta hai konsa actual class inject karna hai (user preference, configuration, etc.).

**Benefit**: Naya channel add karna = naya class likhna. Service code **untouched**. This is the **Open/Closed Principle** in action (Day 12 mein detail).

#### 2. Event-Driven Async — Kafka/Queue Ka Use

**Async architecture** ka philosophy: "Slow operations ko main request thread se hatao." API user ka response 50ms mein return kar de, slow email sending (2-5 seconds SMTP delay) **background** mein chale.

**Kafka kya hai?** — Distributed message broker. Producer message publish karta hai ek "topic" pe. Consumer subscribe karta hai. Messages disk pe persist hote hain (durable). Kafka guarantees: ordering within partition, at-least-once delivery, replay capability (read old messages).

**Flow**:
```
API:        Save token + publish event "PasswordResetRequested" → respond 200 in 50ms
Kafka:      Stores event durably
Consumer:   Email service picks event, sends SMTP, takes 3 seconds — no problem
Failure:    If SMTP fails 3 times → message moves to Dead Letter Queue (DLQ)
DLQ:        Special queue for failed messages, ops team reviews, retries manually
```

**Dead Letter Queue (DLQ)** = bin where messages go after exhausting retries. Without DLQ, failed messages either get retried infinitely (clogging system) or are silently dropped (lost data). DLQ = "I gave up, human please look."

#### 3. Token Bucket Rate Limiter

**Token Bucket Algorithm** explanation:

Imagine ek bucket jo X tokens hold kar sakta hai (capacity = 100). Bucket mein Y tokens/sec refill hote hain (rate = 10/sec). Har incoming request 1 token consume karta hai. Bucket khaali hai? Request rejected (429 Too Many Requests).

For password reset endpoint: capacity = 3, refill = 1/hour. Yani max 3 requests per hour per user/IP. Brute force attempt? Banda 4th try pe block ho jata hai.

Redis pe implement karte hain — Redis atomic INCR + EXPIRE commands have us covered:

```
Key: "rate_limit:forgot_password:USER_IP"
Value: counter, TTL: 1 hour
On request: INCR key, EXPIRE key 3600 IF not set
If counter > 3: reject
```

### Bhai, Architecture Level Pe Yeh Kaise Design Karein

Reset request aaye → **synchronously** token banao + DB save → **asynchronously** notification dispatch via event/queue. User ko 50ms response, email background mein. Notification service crash ho jaye → DLQ mein chala jaye for retry. Polymorphic sender allows runtime channel selection.

### Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│  USER (Browser/Mobile App)                                     │
│  Angular Forms: /forgot-password and /reset?token=...          │
└─────────────────────────┬──────────────────────────────────────┘
                          │ HTTPS POST
                          ▼
┌────────────────────────────────────────────────────────────────┐
│  API GATEWAY                                                   │
│  + Rate Limiter (Token Bucket: 3 requests/hour/IP)             │
│  + TLS termination, request validation                         │
└─────────────────────────┬──────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────────┐
│  AUTH SERVICE (Spring Boot / .NET)                             │
│                                                                │
│  requestReset(email):                                          │
│    1. Lookup user                                              │
│    2. SecureRandom 32 bytes → raw token                        │
│    3. SHA-256 hash → save to DB                                │
│    4. Publish event to Kafka                                   │
│    5. Return 200 (don't wait for email)                        │
│                                                                │
│  confirmReset(token, password):                                │
│    1. Hash token, atomic UPDATE used=1                         │
│    2. Update password_hash (BCrypt)                            │
│    3. Delete all sessions                                      │
│    4. Return 200                                               │
└─────────┬────────────────────────────┬─────────────────────────┘
          │                            │
          ▼                            ▼
┌──────────────────────┐   ┌──────────────────────────────────┐
│  SQL DATABASE        │   │  KAFKA (or AWS SNS/RabbitMQ)     │
│                      │   │  Topic: "password.reset.events"  │
│  - users             │   │  Durable, replayable             │
│  - reset_tokens      │   └──────────┬───────────────────────┘
│  - sessions          │              │
│                      │              ▼
│  Indexes:            │   ┌──────────────────────────────────┐
│  - token_hash UNIQUE │   │  NOTIFICATION SERVICE            │
│  - user_id+used+exp  │   │                                  │
└──────────────────────┘   │  INotificationSender (interface) │
                           │     ▲                            │
                           │     │ implements                 │
                           │     ├─ EmailSender (AWS SES)     │
                           │     ├─ SmsSender (Twilio)        │
                           │     ├─ PushSender (Firebase FCM) │
                           │     └─ WhatsAppSender (Meta API) │
                           │                                  │
                           │  Polymorphic dispatch based on   │
                           │  user.preferredChannel           │
                           └───────────┬──────────────────────┘
                                       │
                                  on failure (3 retries exhausted)
                                       ▼
                           ┌──────────────────────────────────┐
                           │  DEAD LETTER QUEUE               │
                           │  Ops alert + manual review       │
                           └──────────────────────────────────┘
```

### Trade-offs (Detailed)

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| **Sync email send** | Simple, immediate confirmation | API slow (SMTP delays 2-5s), single point of failure | Tiny apps, internal tools |
| **Async via queue** | Fast response, retryable, scalable, decoupled | Eventual delivery, infra cost, debugging complex | **Default for production** |
| **JWT-based token (stateless)** | No DB lookup per validation, scalable | Cannot revoke before expiry, replay-able if leaked | Magic links with short expiry |
| **Random + DB token (stateful)** | Revocable, single-use enforced, audit trail | DB hit per validation, requires schema | **Default for security-sensitive** |
| **6-digit OTP code** | User-friendly, works without email click | Brute-force-able (need attempt limits), 6 digits = 1M combos | Mobile-first apps |
| **Long random token in URL** | Cannot be brute-forced (32 bytes = 10^77 combos) | Requires email/SMS link click | **Daraz-style apps** |

### Real Companies Using This

- **Stripe**: Async via internal Kafka. Multi-channel (email + SMS). 1-hour expiry. HMAC tokens (no DB lookup, but cannot revoke).
- **GitHub**: Sync email send (their scale allows it). 3-hour expiry. Explicit anti-enumeration in docs.
- **Slack**: Magic-link login uses this exact pattern. Random token, SHA-256 hashed in DB, single-use, async via SendGrid.
- **AWS Cognito**: Hashed 6-digit codes, attempt-limited (5 wrong = lock), async via SES.

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Cross-Questioning Proof)

**Today's OOP/Pattern Focus**: **Polymorphism (Method Overriding)** — Day 4 of OOP Fundamentals

### Principle/Pattern Definition (In Depth)

**Polymorphism** = Greek "poly" (many) + "morph" (shape) = "many shapes". In code, it means: **same method call, different behavior depending on the actual object at runtime**.

3 types of polymorphism in OOP:

1. **Compile-time polymorphism (Method Overloading)** — same method name, different parameters in **same class**. Compiler picks the right one based on argument types.

2. **Runtime polymorphism (Method Overriding)** — child class **redefines** a method declared in parent class/interface. The actual object's class (not the reference type) decides which method runs **at runtime**.

3. **Parametric polymorphism (Generics)** — same code works for any type. `List<String>`, `List<Integer>` use same `List` implementation.

Aaj **#2 (Method Overriding)** focus hai.

### How Method Overriding Actually Works (Step by Step)

```java
NotificationSender sender = new EmailNotificationSender();
sender.send(user, message);
```

Yeh 1 line code mein 3 things happen:

```
1. Compile time:
   Compiler dekhta hai: "sender" is of type NotificationSender (interface)
   "send()" method NotificationSender interface mein declared hai? YES
   OK, code valid hai.
   But compiler DOESN'T KNOW which actual class's send() will run.

2. Runtime — JVM ka kaam:
   Object actually EmailNotificationSender hai (was created with `new`).
   Every Java object has a hidden pointer to its class's "vtable".

3. Virtual Table (vtable) lookup:
   Vtable = array of function pointers, one per method.
   EmailNotificationSender's vtable mein send() ke against
   EmailNotificationSender.send() ka pointer hai.
   JVM jumps to that pointer, executes the method.
```

**Vtable kya hai (depth)**: Har class apna vtable rakhti hai memory mein. Methods ke pointers store karti hai. Subclass jab method override karti hai, uska vtable parent ke jaisa hota hai except overridden method ka pointer apni implementation ki taraf point karta hai.

```
NotificationSender's vtable:
  [0] → send (abstract — no pointer)

EmailNotificationSender's vtable:
  [0] → EmailNotificationSender.send (actual code in memory)

SmsNotificationSender's vtable:
  [0] → SmsNotificationSender.send (different code in memory)
```

Object creation ke time class pointer set ho jata hai. Method call ke time JVM:
1. Object se class pointer follow karta hai
2. Class se vtable follow karta hai
3. Vtable mein method ka index dekh ke function pointer leta hai
4. Pointer ko jump karta hai → method execute

Ye sab nano-seconds mein hota hai but technically virtual dispatch ka cost hai. JIT compiler sometimes inline kar deta hai if it sees only one implementation ever runs (called "monomorphic call site optimization").

### Bhai, Simple Mein Samjho

Imagine kar ek hotel mein **chef** hai. Customer kehta hai "Chef sahab, khaana banao." Chef sirf bolta hai "OK". Lekin actual mein:
- Agar chef ko **Pakistani chef** hire kiya gaya tha → biryani banegi
- Agar **Chinese chef** hire kiya tha → noodles banenge
- Agar **Italian chef** tha → pasta banegi

Customer (caller code) ko parwaah nahi kaunsa chef hai. Customer ka request same hai (`makeFood()`). Chef ki actual identity (subclass) decide karti hai actual khaana.

**Yeh hi polymorphism hai**. Caller interface ko jaanta hai, implementation ko nahi.

### Real-Life Analogy (Pakistani Context — Detailed)

**Careem app analogy (depth)**:

Tum Careem khol ke "Ride book" button click karte ho. App tumhe puchti hai:
- Careem Go (small car)
- Careem Plus (sedan)
- Careem XL (SUV)
- Careem Bike (motorcycle)
- Rickshaw

Tum koi bhi select karo, button **same** kaam karta hai — `bookRide()`. App ka code:

```
selectedVehicle.bookRide();   // selectedVehicle is "Vehicle" type
```

Lekin har vehicle ke piché alag implementation:
- `CareemGo.bookRide()` → small car driver assign, fare = distance * 25
- `CareemBike.bookRide()` → bike rider assign, fare = distance * 12, helmet provide
- `Rickshaw.bookRide()` → rickshaw driver, fare = bargained, no AC

Same button click, different actual behavior, runtime decided based on `selectedVehicle` ki **actual type**.

Tumhara app code (caller) **kabhi nahi badlta**. Kal Careem ne `CareemElectric` add kiya? Wahi `bookRide()` button kaam karega — naya class, same interface. **Yeh hi polymorphism ka power hai**: extensibility without modification.

---

### Aaj Ke Brainer Mein Yeh Pattern/Principle Kahan Hai?

Aaj ke Password Reset scenario mein **polymorphism literally har jagah** hai:

**1. `NotificationSender` interface** — yeh hi sabse obvious example hai. `EmailSender`, `SmsSender`, `PushSender`, `WhatsAppSender` — sab ne `send()` method override kiya hai. Service code sirf interface jaanta hai. WhatsApp kal add karna ho? Naya class likho, DI register karo, **service code untouched**.

**2. `PasswordEncoder` (Spring) / `IPasswordHasher<T>` (.NET)** — Spring ke paas hain:
- `BCryptPasswordEncoder`
- `Argon2PasswordEncoder`
- `Pbkdf2PasswordEncoder`
- `SCryptPasswordEncoder`

Sab ne `encode()` aur `matches()` override kiye. Tum apne service code mein sirf `PasswordEncoder encoder` use karte ho. DI configuration mein BCrypt se Argon2 switch karna ho? Sirf bean configuration badlo:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new Argon2PasswordEncoder();  // was BCryptPasswordEncoder
}
```

Saare service code **untouched**. Polymorphism ke without, har service mein `BCrypt.hash(...)` directly call hota — har jagah migration karna parta.

**3. `MessageDigest.getInstance("SHA-256")`** — JVM internally polymorphic dispatch karta hai. `MessageDigest` abstract base class hai, har algorithm (`SHA-256`, `SHA-512`, `MD5`) ki concrete subclass hai. Tum sirf algorithm name pass karte ho, JVM correct implementation return karta hai.

Polymorphism is the **secret sauce** that makes our `PasswordResetService` **Open for extension, Closed for modification** (foreshadowing Day 12 — Open/Closed Principle).

---

### Code Mein Yeh Principle Kaise Apply Hota Hai?

#### ❌ BAD (Polymorphism NOT Used — `if/else` Hell)

```java
@Service
public class PasswordResetService {

    public void sendResetNotification(User user, String link, String channel) {
        // 😱 Tight coupling, hard to extend
        if (channel.equals("EMAIL")) {
            JavaMailSender mailSender = ...;
            MimeMessage msg = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(msg);
            helper.setTo(user.getEmail());
            helper.setSubject("Reset");
            helper.setText("Click: " + link, true);
            mailSender.send(msg);
        }
        else if (channel.equals("SMS")) {
            TwilioRestClient twilio = ...;
            Message smsMessage = Message.creator(
                new PhoneNumber(user.getPhone()),
                new PhoneNumber("+923001234567"),
                "Reset link: " + link
            ).create();
        }
        else if (channel.equals("PUSH")) {
            FirebaseMessaging fcm = ...;
            Notification notification = Notification.builder()
                .setTitle("Password Reset")
                .setBody("Tap to reset")
                .build();
            Message pushMsg = Message.builder()
                .setNotification(notification)
                .setToken(user.getFcmToken())
                .build();
            fcm.send(pushMsg);
        }
        // 🤬 WhatsApp add karna hai? Yahin chain mein else-if add karo!
        // Service file ka size grow karta jayega, kabhi clean nahi hoga.
    }
}
```

**Problems list**:
1. **Open/Closed violated** — naya channel = is class ko modify karna (file diff mein dikhega, code review headache).
2. **Single Responsibility violated** — ek class email + SMS + push sab know karti hai.
3. **Unit testing impossible** — `JavaMailSender`, `TwilioRestClient`, `FirebaseMessaging` saare hard-coded, mock karna painful.
4. **Imports bloat** — har channel ka SDK import — class bahut bhari ho jata hai.
5. **Coupling** — Reset service ka existence Twilio/Firebase/JavaMail pe depend karta hai. Twilio API change kare → reset service compile fail.

#### ✅ GOOD (Polymorphism Used Properly)

```java
// ============ INTERFACE — the contract ============
public interface NotificationSender {
    void send(User user, NotificationMessage message);
    NotificationChannel getChannel();
}

public enum NotificationChannel {
    EMAIL, SMS, PUSH, WHATSAPP
}

// ============ EMAIL IMPLEMENTATION ============
@Component
@RequiredArgsConstructor
public class EmailNotificationSender implements NotificationSender {

    private final JavaMailSender mailSender;

    @Override  // tells compiler "must match interface signature"
    public void send(User user, NotificationMessage message) {
        try {
            MimeMessage mime = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(mime, true);
            helper.setTo(user.getEmail());
            helper.setSubject(message.getSubject());
            helper.setText(message.getHtmlBody(), true);
            mailSender.send(mime);
        } catch (MessagingException e) {
            throw new NotificationException("Email send failed", e);
        }
    }

    @Override
    public NotificationChannel getChannel() {
        return NotificationChannel.EMAIL;
    }
}

// ============ SMS IMPLEMENTATION ============
@Component
@RequiredArgsConstructor
public class SmsNotificationSender implements NotificationSender {

    private final TwilioClient twilio;

    @Override
    public void send(User user, NotificationMessage message) {
        twilio.messages().create(
            new PhoneNumber(user.getPhone()),
            new PhoneNumber("+923001234567"),
            message.getSmsBody()
        );
    }

    @Override
    public NotificationChannel getChannel() {
        return NotificationChannel.SMS;
    }
}

// ============ PUSH IMPLEMENTATION ============
@Component
@RequiredArgsConstructor
public class PushNotificationSender implements NotificationSender {

    private final FirebaseMessaging fcm;

    @Override
    public void send(User user, NotificationMessage message) {
        Message msg = Message.builder()
            .setNotification(Notification.builder()
                .setTitle(message.getSubject())
                .setBody(message.getPushBody())
                .build())
            .setToken(user.getFcmToken())
            .build();
        try {
            fcm.send(msg);
        } catch (FirebaseMessagingException e) {
            throw new NotificationException("Push send failed", e);
        }
    }

    @Override
    public NotificationChannel getChannel() {
        return NotificationChannel.PUSH;
    }
}

// ============ SERVICE — consumes via interface, knows no concrete types ============
@Service
public class PasswordResetService {

    private final Map<NotificationChannel, NotificationSender> sendersByChannel;

    // Spring magic: List<NotificationSender> auto-fills with ALL @Component
    // implementations of NotificationSender. We index by channel for lookup.
    public PasswordResetService(List<NotificationSender> allSenders) {
        this.sendersByChannel = allSenders.stream()
            .collect(Collectors.toMap(
                NotificationSender::getChannel,
                Function.identity()
            ));
    }

    public void sendResetNotification(User user, String link) {
        // User has a preferred channel (saved in their profile)
        NotificationSender sender = sendersByChannel.get(user.getPreferredChannel());

        if (sender == null) {
            // Fallback to email if preference unavailable
            sender = sendersByChannel.get(NotificationChannel.EMAIL);
        }

        // ⭐ POLYMORPHIC CALL — runtime decides which send() runs
        // Service has NO knowledge of Email/SMS/Push internals
        sender.send(user, new PasswordResetMessage(link));
    }
}
```

**Now adding WhatsApp**:

```java
// Just write ONE new class. Service code unchanged.
@Component
@RequiredArgsConstructor
public class WhatsAppNotificationSender implements NotificationSender {
    private final WhatsAppBusinessClient whatsApp;

    @Override
    public void send(User user, NotificationMessage message) {
        whatsApp.sendTemplate(user.getPhone(), "password_reset",
            Map.of("link", message.getWhatsAppLink()));
    }

    @Override
    public NotificationChannel getChannel() {
        return NotificationChannel.WHATSAPP;
    }
}
```

Spring automatically detects the new `@Component`, injects it into the service's list, map gets a new entry. **Zero changes to existing code, zero deployment risk on existing channels.** This is the textbook **Open/Closed Principle**.

---

### Java vs .NET Mein Yeh Principle Kaise Implement Hota Hai?

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Override keyword | `@Override` annotation (optional but strongly recommended) | `override` keyword (**mandatory**) |
| Base method virtual? | Methods are **virtual by default** in Java | Base must be `virtual` or `abstract` explicitly |
| Hide vs override | Java always overrides (no hiding) | C# `new` keyword to hide (different semantics, dangerous) |
| Interface methods | Implicitly abstract (or `default` in Java 8+) | Implicit, or `default` interface methods (C# 8+) |
| Sealed override | `final` keyword on method blocks further override | `sealed override` blocks further override |
| Polymorphic DI | `List<INotifier>` or `Map<Key, INotifier>` | `IEnumerable<INotifier>` or keyed services (.NET 8+) |

```csharp
// C# equivalent — full working example

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

// ============ REGISTRATION (Program.cs in .NET 8) ============
// Keyed services allow looking up by key (channel)
builder.Services.AddKeyedScoped<INotificationSender, EmailNotificationSender>(
    NotificationChannel.Email);
builder.Services.AddKeyedScoped<INotificationSender, SmsNotificationSender>(
    NotificationChannel.Sms);

// ============ SERVICE ============
public class PasswordResetService
{
    private readonly IServiceProvider _sp;

    public PasswordResetService(IServiceProvider sp) => _sp = sp;

    public async Task NotifyAsync(User user, string link)
    {
        // Resolve correct sender at runtime based on user preference
        var sender = _sp.GetRequiredKeyedService<INotificationSender>(
            user.PreferredChannel);

        // Polymorphic dispatch — actual implementation decided at runtime
        await sender.SendAsync(user, new PasswordResetMessage(link));
    }
}
```

**Important Java vs C# gotcha**: In Java, all methods are virtual by default — child can override anything. In C#, parent method must be marked `virtual` to allow override. C# defaults to "sealed" — methods cannot be overridden unless explicitly allowed. This is a **design philosophy difference**: Java trusts subclasses by default, C# requires explicit opt-in for safety.

---

### 🎯 Cross-Questioning Drill (Interview Defense)

#### Cross Q1: "Polymorphism aur Method Overloading mein kya difference hai?"

**Confident Answer**:

"Bhai, dono ka naam similar lagta hai but **completely different concepts** hain:

**Method Overloading (compile-time polymorphism)**:
- Same class mein same method name
- **Different parameter list** (different number, types, or order)
- Compiler decide karta hai which to call, by looking at arguments at compile time
- No inheritance involved
- Example:
```java
class Printer {
    void print(int x) { ... }
    void print(String x) { ... }
    void print(int x, int y) { ... }
}
```
Call `print(5)` → compiler sees int → picks first method. Decision made at compile time. JVM doesn't think.

**Method Overriding (runtime polymorphism)**:
- Parent class/interface ki method ko child class redefine karti hai
- **Same signature** — same name, same parameters, same return type
- JVM decide karta hai which to call by looking at actual object's class **at runtime**
- Inheritance/implements required
- Example:
```java
class Animal { void sound() { print('generic'); } }
class Dog extends Animal { void sound() { print('woof'); } }
class Cat extends Animal { void sound() { print('meow'); } }

Animal a = new Dog();
a.sound();  // prints 'woof' — runtime decided
```

Compile time pe `a` ka type `Animal` hai, lekin runtime pe actual object `Dog` hai, Dog ka `sound()` chalega.

**Yeh hi 'virtual dispatch' kehte hain** — JVM uses vtable (virtual function table) lookup."

---

#### Cross Q2: "Tumne strategy pattern use kiya. Alternative kya tha?"

**Confident Answer**:

"Alternatives the:

**1. `switch` statement / if-else chain**:
```java
switch(channel) {
    case EMAIL: /* email code */; break;
    case SMS: /* sms code */; break;
}
```
Problems:
- Every new channel = modify this method (Open/Closed violated)
- Testing requires mocking all dependencies in one class
- Single Responsibility violated

**2. Functional approach (Java 8+)**:
```java
Map<Channel, BiConsumer<User, Message>> handlers = Map.of(
    EMAIL, (u, m) -> { /* email logic */ },
    SMS, (u, m) -> { /* sms logic */ }
);
handlers.get(channel).accept(user, message);
```
Better than switch but still concentrates all logic in one file. Hard to inject dependencies cleanly. Lambdas can't easily hold their own state.

**3. Strategy + Polymorphism (chosen approach)**:
- Each strategy = own class = own file = own tests
- DI container handles wiring
- Adding new channel = new file, zero modifications
- Stack traces show actual class name (better debugging)

**Why I chose Strategy**: enterprise codebases mein clarity > brevity. Functional approach Day 1 mein attractive lagta hai but year 2 mein 50 channels ke saath maintenance nightmare ban jata hai."

---

#### Cross Q3: "Polymorphism violate kab kar sakte ho jaan-boojh ke?"

**Confident Answer**:

"3 legitimate reasons:

**1. Performance-critical hot paths**:
Virtual dispatch ka cost hota hai — typical 1-5 nanoseconds per call due to vtable lookup. High-frequency trading mein 1 billion calls/sec hote hain — 5ns × 1B = 5 seconds overhead per second. Solutions:
- Mark class `final` in Java / `sealed` in C# → compiler can devirtualize
- JIT compiler agar dekhta hai sirf ek implementation hai runtime mein, automatic devirtualize karta hai (monomorphic call site optimization)

**2. Single implementation forever**:
Agar tumhe pakka pata hai sirf ek implementation kabhi banegi (e.g., `MainConfig`), interface banane se code complexity bina value badhti hai. **YAGNI principle** (You Aren't Gonna Need It) yeh sikhata hai.

**3. Marker interfaces / type tags**:
Sometimes you don't need polymorphic behavior, you just need type checking (`instanceof Serializable`). Interface has no methods, no polymorphism, just metadata.

**Common mistake**: Premature abstraction. Junior developer dekhta hai 1 implementation, lekin sochta hai 'maybe future mein doosri ho'. 3 saal baad doosri kabhi nahi banti. Result: useless interface, indirection, harder to navigate code.

**Rule of thumb**: Wait for **second** implementation to appear. Then extract interface. Two examples reveal the right abstraction; one example is just a class."

---

#### Cross Q4: "Yeh principle Spring/JPA mein automatically follow hota hai ya manual?"

**Confident Answer**:

"Spring **heavily** polymorphism pe based hai — pura framework hi interface contracts ke around bana hai:

**1. Bean Container**: Spring ka `BeanFactory` interface hai, `ApplicationContext` extend karta hai, `AnnotationConfigApplicationContext`/`WebApplicationContext` etc. concrete implementations hain. Tum interface use karte ho, runtime pe correct impl mil jati hai.

**2. JPA Abstraction**: `EntityManager` interface hai, Hibernate ka `SessionImpl` concrete impl. EclipseLink, OpenJPA dusre impls. Tum apna code Hibernate-specific likhne ki bajaye `EntityManager` use karte ho — vendor switch possible without code change.

**3. Spring AOP — proxy generation**: Spring runtime mein **dynamic proxy classes** generate karta hai. Tumhari `UserService` class hai jisme `@Transactional` method hai. Spring runtime pe `UserService` ko extend karke ek subclass banata hai (using CGLib). Is subclass ke method mein extra code add karta hai (open transaction → call super method → commit). Tumhare clients ko yeh subclass inject hota hai — they think it's `UserService`, but actually proxy hai jo extra behavior add karta hai. **Yeh polymorphism + composition + proxy pattern ka combo hai**.

**4. JDBC Templates**: `JdbcTemplate.query(sql, RowMapper<T>)` — RowMapper interface hai, tum apni implementation pass karte ho. Polymorphic callback.

**Spring polymorphism enforce nahi karta, lekin uska entire design polymorphism PE built hai**. Tum bhi same patterns follow karoge — interface define karo, multiple impls likho, DI se inject."

---

#### Cross Q5: "Production mein polymorphism scale kaise karta hai?"

**Confident Answer**:

"Polymorphism class-level concept hai, lekin **service-level pe bhi extend hota hai** distributed systems mein:

**Service-level polymorphism example**:
Imagine Daraz ka **Payment Service**. JazzCash, EasyPaisa, Stripe, COD, Bank Transfer — sab `PaymentProvider` interface implement karte hain. Naya provider add karna ho (Sadapay) → new microservice deploy karo, service registry mein register karo, payment orchestrator dynamically resolve karta hai. **Zero downtime, zero code change in orchestrator**.

**Real numbers**:
- Netflix ka notification service ~100M+ users handle karta hai
- 50+ channels (email, SMS, push, in-app, browser, smart TV notification, etc.)
- Polymorphism + Strategy = ek single dispatcher 50 channels handle karta hai cleanly

**Performance cost at scale**:
- Per-call overhead ~5-10 nanoseconds (vtable lookup)
- 1 trillion calls/day = 10 trillion ns overhead = 10,000 seconds/day total
- Spread across 1000 servers = 10 seconds/server/day → negligible
- JIT compiler optimization aur further reduce karta hai

**Catch — polymorphism scales badly when**:
- Deep inheritance chains (Class → SubClass → SubSubClass → ...) → maintenance nightmare. Solution: favor composition over deep inheritance.
- Too many implementations (300+ subclasses) → understanding which to use becomes hard. Solution: subdivide into multiple interfaces.

**Bottom line**: well-designed polymorphism scales beautifully. Anti-pattern is forcing polymorphism where it doesn't belong."

---

### 🧩 Pattern Combination (How Today's Pattern Pairs With Others)

**Pairs Well With**:

- **Strategy Pattern (Day 62)** — Strategy literally polymorphism use karta hai. Strategy = polymorphism + "intent of swapping algorithms".
- **Factory Method (Day 32)** — Factory returns polymorphic type. `NotifierFactory.create(channel)` returns `NotificationSender`, hiding the concrete class.
- **Template Method (Day 66)** — Base class skeleton fix karta hai, subclass specific steps override karti hai. Polymorphism enables variation in fixed structure.
- **Open/Closed Principle (Day 12)** — Polymorphism **primary mechanism** hai OCP achieve karne ka. Extension via subclassing, no modification of parent.
- **Dependency Inversion (Day 15)** — "Depend on abstractions, not concretions" = depend on polymorphic interfaces, not concrete classes.

**Tension With**:

- **YAGNI (Day 18)** — Don't create interface for single implementation. Polymorphism has cognitive cost (one more file to navigate).
- **KISS (Day 17)** — Sometimes a direct `switch` is clearer than 5 classes for 5 channels (if channels never change).
- **Performance-critical code** — Virtual dispatch prevents some JIT optimizations. Hot loops sometimes need `final`/`sealed`.

---

### 🎓 Real Production Code Where This Matters

**Spring's `DelegatingPasswordEncoder` — masterclass in polymorphism**:

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}

// Concrete implementations:
// BCryptPasswordEncoder, Argon2PasswordEncoder, Pbkdf2PasswordEncoder,
// SCryptPasswordEncoder, NoOpPasswordEncoder (testing only)

// Spring Security ka registry:
@Bean
public PasswordEncoder passwordEncoder() {
    return new DelegatingPasswordEncoder("bcrypt", Map.of(
        "bcrypt", new BCryptPasswordEncoder(),
        "argon2", new Argon2PasswordEncoder(),
        "pbkdf2", new Pbkdf2PasswordEncoder("")
    ));
}
```

**Magic of `DelegatingPasswordEncoder`**:

Stored password format: `{algorithmId}encodedValue`
Examples:
- `{bcrypt}$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy`
- `{argon2}$argon2id$v=19$m=4096,t=3,p=1$...`

When `matches()` called:
1. Extract prefix `{bcrypt}` from stored value
2. Look up the encoder for that algorithm in the map (polymorphism)
3. Delegate to that encoder for verification

**Why this matters**: Tumhari app 2020 mein BCrypt use kar rahi thi. 2025 mein industry recommends Argon2. Tum migrate karna chahte ho **without forcing users to reset passwords**. With `DelegatingPasswordEncoder`:
- New passwords hashed with Argon2 (default updated)
- Old BCrypt hashes still verifiable (`{bcrypt}` prefix recognized)
- On successful login with BCrypt, you can re-hash with Argon2 and update

**Polymorphism + Strategy + Factory pattern all in one**, enabling **seamless migration at scale**. Yeh hi senior engineering hai.

---

### 💡 Memory Hook for This Principle/Pattern

**POLYMORPHISM = "BARTAN BADLO, KHAANA WOHI"**

Bhai, polymorphism aisi cheez hai: **bartan** (concrete class) badal jaye, **khaana** (interface contract) wohi rahe.

- Pateele mein biryani, handi mein biryani, degh mein biryani — bartan alag, function (biryani pakana) same.

Ya phir: **"EK CALL, KAYI CHEHRE"** — same method call, different faces showing up at runtime.

Yaad rakhne ka tareeqa:
- **Overloading** = "Sab apne pyaalon mein chai" (same chai, alag pyaale — compile-time, same class)
- **Overriding** = "Beta baap ki recipe mein twist" (parent method ko redefine, runtime decide)

**Mental shortcut for interview**:
- "Compile time" = "Compiler ne already decide kar liya" = overloading
- "Runtime" = "Runtime pe object dekhke decide hoga" = overriding

---

### ⚠️ Common Mistakes Jo Aaj Bachne Hain

#### ❌ Mistake 1: `@Override` annotation skip karna in Java

**Why it's wrong**:
Agar tum method signature galat likhte ho (typo, wrong parameter type, wrong case), compiler **chup rahega** — silently new method ban jayegi parent ke alongside, override nahi hoga. Production mein parent ka method chalega, tumhari child wali kabhi nahi.

```java
// BAD — typo, no @Override → silent bug
public class EmailNotifier implements NotificationSender {
    public void Send(User user, Message msg) {  // capital S — DIFFERENT method!
        // Email sending code
    }
}
// Result: NotificationSender interface ka send() (lowercase s) unimplemented hai!
// Compiler ne error nahi diya kyunki interface mein default method ho sakta hai.
// Runtime pe interface ka default send() chalega — empty body — email NEVER sent!
// Debug karne mein hafte lag jate hain.
```

**Correct approach**:
```java
public class EmailNotifier implements NotificationSender {
    @Override  // compiler catches signature mismatch
    public void send(User user, Message msg) {
        // ...
    }
}
```

Compiler error: "Method does not override method from its superclass." Bug caught at compile time, never reaches production.

**Rule**: ALWAYS use `@Override`. C# mein `override` keyword mandatory hai, so this mistake is impossible. Java mein discipline required.

---

#### ❌ Mistake 2: Calling overridable methods from constructor

**Why it's wrong**:
Constructor mein `this.someMethod()` call karte ho jo subclass mein override hai. Subclass ka constructor abhi run nahi hua, subclass ki fields uninitialized hain. Override version runs but on incomplete object → NullPointerException or weird behavior.

**Concrete example**:
```java
// BAD
public class Parent {
    public Parent() {
        initialize();  // calls overridable method
    }
    public void initialize() {
        System.out.println("Parent init");
    }
}

public class Child extends Parent {
    private String name = "default";  // initialized AFTER parent constructor

    @Override
    public void initialize() {
        System.out.println(name.toUpperCase());  // NPE!
        // 'name' is null when parent constructor runs because
        // child field initialization hasn't happened yet
    }
}

new Child();  // CRASH with NullPointerException
```

**Order of operations**:
1. `new Child()` triggered
2. Child's parent (Parent) constructor runs first
3. Inside Parent constructor: calls `this.initialize()` — polymorphism kicks in, Child's `initialize()` runs
4. Child's `initialize()` accesses `name` → but `name = "default"` hasn't executed yet → name is still null (default)
5. `null.toUpperCase()` → NPE

**Correct approach**:
- Constructor mein only `private` or `final` methods call karo (cannot be overridden, behavior guaranteed)
- Or use a separate `init()` method called after construction
- Or use builder/factory pattern

```java
// GOOD
public class Parent {
    public Parent() {
        // No virtual calls
    }
    public final void initialize() {  // final = cannot override
        System.out.println("Parent init");
    }
}
```

---

#### ❌ Mistake 3: Override `equals()` but forget `hashCode()`

**Why it's wrong**:
Java's contract: "if two objects are equal (`equals()` returns true), they MUST have same hashCode".

HashMap, HashSet internally use hashCode to bucket objects. Equal objects with different hashCodes → end up in different buckets → set thinks they're different → duplicate entries. Map lookup fails.

```java
// BAD
public class User {
    private String email;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return email.equals(((User) o).email);
    }
    // hashCode() NOT overridden! Default uses object identity (memory address)
}

User u1 = new User("ahmed@gmail.com");
User u2 = new User("ahmed@gmail.com");

u1.equals(u2);  // true
u1.hashCode() == u2.hashCode();  // FALSE (different memory addresses)

Set<User> users = new HashSet<>();
users.add(u1);
users.contains(u2);  // FALSE! Even though equals says they're equal!
// HashSet looks in u2's hash bucket, doesn't find u1 (it's in different bucket)
```

**Correct approach**:
```java
@Override
public int hashCode() {
    return Objects.hash(email);
}
```

Both methods together. Or use:
- Lombok `@EqualsAndHashCode`
- IDE auto-generate
- Java records (auto-generated equals/hashCode)
- Apache Commons `EqualsBuilder`/`HashCodeBuilder`

**Why this is polymorphism-related**: `equals()` and `hashCode()` are methods declared in `Object` (base class of everything). All classes inherit them and can override. Breaking the contract violates Liskov Substitution (Day 13) — your subclass should behave correctly wherever parent type is expected.

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom — Every Layer's Role):

```
1. USER ACTION (Angular)
   └─▶ Click "Forgot Password"
       └─▶ Reactive Form submit, loading spinner
       └─▶ Same UI message on success OR error (anti-enumeration)

2. NETWORK (HTTPS)
   └─▶ TLS encryption protects email in transit
       └─▶ Rate limiter at API gateway (3/hour/IP)

3. API REQUEST (Spring Boot / .NET)
   └─▶ POST /api/auth/forgot-password
       └─▶ @Transactional wrapper begins DB transaction

4. BUSINESS LOGIC (Java / C#)
   └─▶ SecureRandom 32-byte token generated
       └─▶ SHA-256 hash created
       └─▶ Polymorphic notifier.send(user, msg) dispatched

5. DATABASE (SQL)
   └─▶ INSERT password_reset_tokens (hash, expires_at)
       └─▶ Unique index prevents collisions
       └─▶ Composite index makes user-token queries fast

6. ASYNC EVENT (System Design)
   └─▶ Publish to Kafka topic "password.reset"
       └─▶ Notification service consumes
       └─▶ EmailSender OR SmsSender (polymorphic dispatch)
       └─▶ Failure → DLQ for ops review

7. USER CLICKS LINK (15-min window)
   └─▶ GET /reset?token=xyz → Angular form
       └─▶ Token extracted from URL
       └─▶ Password strength meter (debounced RxJS)

8. CONFIRM RESET
   └─▶ POST /api/auth/reset {token, newPassword}
       └─▶ Atomic UPDATE tokens SET used=1 WHERE used=0 AND not expired
       └─▶ @@ROWCOUNT check ensures single-use
       └─▶ BCrypt hash new password (10-12 cost factor)
       └─▶ Delete all sessions (force logout everywhere)

9. SUCCESS RESPONSE
   └─▶ 200 OK → Angular redirects to login
       └─▶ Old JWTs/sessions invalid → must log in fresh
```

**What Breaks If You Skip ANY Layer**:

| Skip This | Consequence |
|-----------|-------------|
| Angular generic response | User enumeration via UI (attackers harvest emails) |
| Backend rate limit | DDoS via "Forgot password" spam → mailbox flooding |
| Token hashing | DB breach = all active tokens leaked = mass takeover |
| Atomic SQL update | Race condition → token used twice → account compromise |
| Async queue | SMTP slowdown → API timeout → 504 errors → bad UX |
| Session invalidation | Hacker with stolen old password keeps access after reset |
| TLS encryption | Reset link visible to network attacker (WiFi sniffer) |
| Password hashing (BCrypt) | DB leak = plaintext passwords visible → reused on Facebook, banking |

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
   Strength meter      Polymorphic Notifier     used=1 WHERE used=0
   debounceTime        @Version optimistic      ROWVERSION concurrency
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
                          │
                    Polymorphic dispatch
                    runtime decides which
```

**Mental Story (Roman Urdu — Detailed)**:

Bhai, password reset ko **shaadi ke invitation card system** jaisa socho:

- **Angular** = Card ka design (kaisa dikhega user ko, validation, strength meter)
- **Backend (Spring/.NET)** = Card printer (token banata hai, expiry print karta hai)
- **Token generation (Java/C#)** = Unique serial number generator (SecureRandom guarantees no two cards have same number)
- **SQL** = Guest registry (kis ko bheja, kab expire hoga, RSVP done ya nahi — all tracked)
- **System Design** = Delivery selection — DHL (Email), TCS (SMS), khud jaa ke dena (Push notification). **Sender alag, kaam wohi — guest tak invitation pohnchana**.

**Polymorphism** = chahe DHL bheje ya TCS, **method same**: `deliver(invitation, guest)`. Tum bas bolo "deliver", sender khud decide karega kaise.

Agar koi step missing — shaadi mein:
- No card design (Angular) → guest confused
- No printer (Backend) → no cards exist
- No serial number (token) → duplicates, fraud
- No registry (DB) → forget who's invited
- No delivery (notification) → guest never knows

**Acronym for Password Reset Security: "TRACE"**

- **T** — Token (SecureRandom, 32 bytes, stored as hash)
- **R** — Rate limit (3 attempts/hour, prevents brute force)
- **A** — Atomic single-use (SQL UPDATE WHERE used=0)
- **C** — Consistent response (no enumeration, identical UI)
- **E** — Expiry + session invalidation (15-min + logout all devices)

Yaad karne ke liye: "Hacker ki **TRACE** mat chhodo" — trace = footprint, koi clue mat de.

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Design karo Daraz ke liye complete password reset flow. Security considerations bhi explain karo aur batao notification multi-channel kaise pluggable banaoge?"

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Iss scenario ko handle karne ke liye, hum 5 layers ko coordinate karenge with security at every step — frontend pe enumeration prevention, backend pe secure token generation, DB pe atomic single-use, async notification dispatch, aur polymorphic sender for channel flexibility."

**Body** (60 sec):

1. **Frontend** (Angular):
   - Reactive Form for email input
   - **Identical success response** chahe email exist kare ya na — taake user enumeration leak na ho
   - Reset page token ko URL query param se uthaye
   - Password strength real-time check with `debounceTime(250)` for performance
   - Generic error messages on confirm (don't leak token state)

2. **Backend API** (Spring/.NET):
   - Rate-limited endpoint (3 req/hour/IP via token bucket on Redis)
   - `SecureRandom` se 32 bytes (256-bit) token generate
   - Raw token sirf email mein, **SHA-256 hash DB mein**
   - `@Transactional` enforce atomicity
   - Async notification dispatch via Kafka event

3. **Business Logic** (Java/C#):
   - **Polymorphic `NotificationSender` interface** — Email, SMS, Push, WhatsApp implementations
   - DI container resolves correct sender based on user preference
   - Naya channel = bas naya class + register = zero existing code change (Open/Closed Principle)

4. **Database** (SQL):
   - `password_reset_tokens` table with `UNIQUE INDEX` on hash, composite index on `(user_id, used, expires_at)`
   - Token consumption **atomic**: `UPDATE WHERE used=0 AND expires_at > NOW()` — `@@ROWCOUNT=1` = safe
   - `ROWVERSION`/`@Version` for optimistic locking double-defense
   - On password update: also `DELETE FROM sessions WHERE user_id = X`

5. **Architecture**:
   - Strategy Pattern (polymorphism in action) + Event-driven via Kafka
   - API publishes `PasswordResetRequested` event
   - Notification Service consumes asynchronously
   - DLQ for failed sends, ops monitoring

**Closing** (10 sec):
"By combining these layers with polymorphic notification dispatch, hum guarantee karte hain ke yeh flow secure, scalable, aur infinitely extensible hai — kal WhatsApp add karna ho, zero existing code change chahiye."

### Under-the-Hood Concepts You MUST Know

1. **`SecureRandom` vs `Random`**: `Random` uses Linear Congruential Generator (predictable math). `SecureRandom` uses OS entropy (`/dev/urandom`) — cryptographically unpredictable.

2. **SHA-256 vs BCrypt — when to use which**: SHA-256 fast (1 microsecond), suitable for high-entropy inputs like random tokens. BCrypt slow on purpose (100-200ms), suitable for low-entropy inputs like human passwords (slowness defeats brute force).

3. **Optimistic vs Pessimistic Locking**: Optimistic (`@Version`) = no lock during read, conflict detected at commit (better for low-contention). Pessimistic (`SELECT FOR UPDATE`) = lock during read (safer for high-contention but blocks other readers).

4. **Virtual dispatch (vtable)**: Polymorphism's runtime cost — vtable lookup adds ~1-5ns per call. JIT compiler can devirtualize if monomorphic (only one implementation observed).

5. **TOCTOU (Time Of Check, Time Of Use) bug**: Classic concurrency vulnerability where you check a condition, then act on it, but state changes between. Fixed with atomic check-and-act (single SQL statement with WHERE).

6. **Token bucket rate limiting**: Bucket holds N tokens, refills at R/sec, each request consumes 1. Empty bucket = reject. Implemented with Redis atomic INCR + EXPIRE.

7. **CSPRNG (Cryptographically Secure Pseudo-Random Number Generator)**: Algorithms that produce output indistinguishable from true randomness even after observing millions of outputs.

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

#### Counter Q1 (Trade-off focused):

**Q**: "JWT-based reset token (stateless) ya DB-stored random token (stateful) — kya choose karoge aur kyun?"

**Your Answer**:

"Dono ke trade-offs hain, aur **choice scenario pe depend karta hai**:

**JWT-based reset token (stateless)**:
JWT = JSON Web Token = signed token containing user info. Tum token mein userId + expiry encode karte ho, HMAC-SHA256 se sign karte ho with server secret. Verify karne ke liye DB lookup nahi chahiye — bas signature verify aur expiry check.

✅ Pros:
- No DB write/read per token
- Stateless = horizontally scalable
- Simple architecture

❌ Cons:
- **Cannot revoke before expiry** — agar user dobara reset request kare aur purana token bhi valid rahe, hacker dono use kar sakta hai
- **Replay attack vulnerable** — token leaked? Use kar lo jab tak expiry na ho. Single-use enforce karna na-mumkin without state.
- **Token size large** — 200+ bytes vs 43-byte random. Email mein lambi URL.

**DB-stored random token (stateful)**:
SecureRandom token, SHA-256 hash, DB mein store. Verify = DB lookup.

✅ Pros:
- **Single-use atomic enforcement** via SQL UPDATE
- **Revocable** — kal banake aaj invalidate kar sakte ho
- **Audit trail** — kab create hua, kab use hua, sab DB mein
- Smaller token size

❌ Cons:
- DB hit per validation (~1-5ms with index)
- Requires schema, migrations

**My choice for password reset**: **DB-stored random token, hands down**.

Why? Password reset is **security-critical, single-use** operation. Convenience ke chakkar mein security compromise nahi karta. JWT ka 'no DB hit' optimization 5ms save karta hai per request, lekin password reset throughput low hai (few per second even at Daraz scale) — yeh optimization irrelevant hai.

Slack ka magic-link login bhi same DB-stored approach use karta hai for the exact same reason. JWT is great for session tokens (high throughput, short-lived) but wrong tool for reset tokens."

---

#### Counter Q2 (Scale focused):

**Q**: "10 million users, 1% per day password reset karte hain = 100K requests/day. SMTP slow hai 2-3 seconds per send. Architecture kya hogi?"

**Your Answer**:

"Numbers analyze karte hain pehle:

- 100K requests/day = ~1.2 req/sec average
- Peak (e.g., morning rush) = 10x average = ~12 req/sec
- SMTP latency 3 sec/email

**Synchronous architecture (BAD)**:
12 concurrent requests × 3 sec = need 12 SMTP connections constantly. SES limits 14/sec by default — barely fits, no headroom. If SMTP slows to 5 sec, queue builds up, API requests pile up, **502/504 timeouts**. User sees errors. Awful.

**Async architecture (CORRECT)**:

```
API request:
  1. Save token + publish Kafka event → <50ms response to user
  2. User sees "link sent" instantly

Kafka consumer (notification service):
  3. Pool of 20 worker threads
  4. Each picks event from queue, sends SMTP
  5. 20 workers × 3 sec = 6.7 sends/sec sustained
  6. Burst of 100 events = queued, processed in 15 seconds
  7. User typically doesn't notice 15-sec delay in email
```

**Capacity planning**:
- Average load: 1.2 sends/sec — 2 workers sufficient
- Peak load: 12 sends/sec — 20 workers handle 10x peak comfortably
- 10x growth (1M reset/day) = ~12 req/sec average = same 20 workers
- 100x growth = horizontal scale Kafka consumers + multiple SMTP providers

**Failure handling**:
- SMTP fails → retry with exponential backoff (1s, 5s, 25s)
- 3 retries exhausted → message to DLQ
- DLQ monitored by ops, alerts trigger investigation
- User won't suffer — they can request reset again

**99th percentile API latency**: <100ms (just token DB write + Kafka publish). User experience: snappy, professional.

10x scale comfortable. 100x scale needs regional sharding — each region has own notification service + local SMTP relay."

---

#### Counter Q3 (Failure mode focused):

**Q**: "User ne reset link click kiya, password type kiya, submit click kiya — server crash. User dobara try karta hai. Kya hoga?"

**Your Answer**:

"Bhai, depends on **kab** server crash hua. 4 scenarios analyze karte hain:

**Scenario 1: Crash before any DB write**
- Token still unused in DB, password unchanged
- User retries → succeeds normally
- **Idempotent. No issue.**

**Scenario 2: Crash AFTER `UPDATE tokens SET used=1` but BEFORE `UPDATE users SET password_hash=...`**
- Token marked used, password unchanged
- User retries → 'token invalid' error
- **Bad UX** — user requests new reset link, waits, tries again
- **Solution**: wrap both UPDATEs in **same SQL transaction**. Either both commit or both rollback. SQL Server WAL ensures durability + atomicity (ACID 'A' and 'D').

**Scenario 3: Crash AFTER both UPDATEs but BEFORE response sent**
- Both done in DB, but user sees 'failed' (no response received)
- User retries → 'token invalid' error
- **But password actually updated!** User confused, doesn't try login with new password
- **Solution**: idempotency — retry with same token should detect "already done" and return success
- Implementation: after marking token used, also save `password_changed_via_token` link. On retry, check this link exists → return success.

**Scenario 4: Crash after sending response but before client receives**
- DB consistent, password changed, sessions killed
- Client never got 200
- User refreshes, sees login page (sessions killed, redirected)
- Tries login with new password → works!
- **No issue. The system is self-correcting.**

**Best practices**:
1. **Same transaction**: both token UPDATE and password UPDATE atomic
2. **WAL durability**: SQL Server's Write-Ahead Log persists changes before acknowledging commit
3. **Idempotent retries**: same request twice = same result
4. **Client-side timeout + retry**: with idempotency key (we'll learn this in Day 39!)

Crash recovery is built into modern DBs (PostgreSQL, SQL Server). Application just needs to **trust ACID** and **design for retries**."

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "Password reset ke baad active sessions kaise invalidate karoge across multiple servers/devices?"

**Junior Answer**: "Database mein sessions table hai, sab delete kar denge user_id ke liye. JWT case mein... uhh... kuch nahi kar sakte? JWT expire hone ka wait karna parega?"

**Senior Answer**:

"Multi-pronged approach, depending on session strategy:

**Approach 1: Stateful sessions (DB-stored)**
- Simple: `DELETE FROM sessions WHERE user_id = ?`
- Next request from old session → 401 Unauthorized
- Used by: traditional web apps, banking applications
- Pro: instant invalidation
- Con: DB hit per request to validate session

**Approach 2: Stateless JWT — solutions**

JWT can't be directly revoked. Three techniques:

**2a. Short access token + refresh token pattern**:
- Access token = JWT, expires in 15 min
- Refresh token = random string in DB, expires in 7 days
- On password change: delete all refresh tokens from DB
- Worst case: hacker has 15 more minutes (until access token expires)
- Most apps acceptable trade-off
- Used by: Auth0, most SaaS apps

**2b. Token version in JWT claims**:
- User entity has `tokenVersion` integer field
- Every issued JWT includes `tokenVersion` claim
- Validation: `jwt.tokenVersion === user.tokenVersion` ? valid : invalid
- On password change: `user.tokenVersion++` → all old JWTs invalid immediately
- Requires DB lookup per JWT validation (defeats stateless purpose somewhat)
- Solution: cache user.tokenVersion in Redis with TTL

**2c. Redis blocklist**:
- Add JWT's `jti` (unique ID claim) to Redis set when revoked
- TTL = JWT's remaining lifetime
- Validation checks blocklist (~1ms Redis hit)
- Pro: instant revocation, minimal storage
- Con: extra dependency on Redis
- Used by: Slack, Discord

**Approach 3: Real-time push to clients**
- Server pushes "re-authenticate" message via WebSocket/Server-Sent Events
- Frontend immediately redirects to login
- Backup to other approaches — handles edge case of clients with cached UI

**Approach 4: Audit logging**
- Log every session termination event
- Required for compliance (GDPR, PCI-DSS, SOC2)
- Forensics — if account compromise reported, audit log shows exactly when reset happened, what sessions were killed

**My recommendation for Daraz**:
- Stateful sessions for web (cookies + DB-backed session store)
- Refresh token + token versioning for mobile API (best of both worlds)
- Redis blocklist for hot revocations
- WebSocket push for connected admin users

**Why this matters at senior level**: junior thinks 'kill the session'. Senior thinks 'how does this scale across 100 servers, 10 regions, mobile + web + IoT devices, with compliance and audit requirements?' Yeh hi differentiator hai."

### Red Flag Signals (Don't Say These!)

❌ **"Email mein plaintext token bhej do, DB mein bhi plaintext save kar do"**
**Why wrong**: DB breach instantly leaks all active reset tokens, complete account takeover within 15-min expiry window.

❌ **"Agar email registered nahi to 'Email not found' error dikhao"**
**Why wrong**: User enumeration attack. Attacker harvests valid emails for credential stuffing on other sites.

❌ **"Token expire hi nahi karega, user jab chahe use kare"**
**Why wrong**: Email screenshot/forward, 6 months baad bhi exploitable. Industry standard 15min-1hour for reset tokens.

❌ **"Math.random() / Random() se token bana lo"**
**Why wrong**: Predictable PRNG. Attacker who observes few tokens can predict future ones. ALWAYS `SecureRandom` / `RandomNumberGenerator`.

❌ **"Token validate karne ke baad alag transaction mein use mark karenge"**
**Why wrong**: TOCTOU race condition, double-use possible. Atomic single-statement UPDATE essential.

❌ **"Sessions kya hoti hain reset ke baad, woh alag concern hai"**
**Why wrong**: Senior engineers think **holistically**. Password reset without session invalidation = incomplete security — hacker stays logged in.

❌ **"Frontend pe alag message dikhao for better UX"**
**Why wrong**: DevTools inspection enables enumeration even if backend is hardened. UX team ko convince karna parta hai security wins.

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ **CSPRNG** kya hai, `SecureRandom` vs `Random` ka difference, aur tokens ke liye kyun ek hi acceptable hai?
2. ✅ **Hash function** (SHA-256) ka one-way property kaise tokens ko DB breach se bachata hai?
3. ✅ **BCrypt vs SHA-256** — passwords ke liye slow hash kyun chahiye but tokens ke liye fast hash kafi hai?
4. ✅ **User enumeration attack** — kaise hoti hai timing + response + status code se? Aur kaise prevent karte hain frontend + backend + network teen jagah pe?
5. ✅ **TOCTOU bug** kya hai aur **atomic UPDATE** pattern kaise eliminate karta hai?
6. ✅ **Optimistic vs Pessimistic locking** — `@Version` kaise kaam karta hai internally?
7. ✅ **Polymorphism (method overriding)** kya hai vs overloading, aur vtable runtime pe kaise dispatch karta hai?
8. ✅ **Strategy Pattern** real-world `NotificationSender` mein kaise apply hota hai aur kaise Open/Closed achieve karta hai?
9. ✅ **JWT vs DB-stored tokens** ka trade-off — kab konsa use karna chahiye?
10. ✅ **Session invalidation strategies** — stateful (DB), JWT (refresh token, token version, Redis blocklist) all 4 approaches?

---

## 🔗 Tomorrow's Connection Hint

Tomorrow we'll cover: **Day 5 — Basic Product Listing + Composition over Inheritance**

Aaj humne **polymorphism (inheritance/interface-based)** dekha — kaise child class parent ke method ko override karke alag behavior deti hai. Kal hum **composition over inheritance** principle dekhenge — yeh principle kehta hai "jab bhi possible ho, inheritance ki bajaye composition use karo".

Kal ka scenario: Daraz pe product listing page. Categories, filters, sort options — yeh sab hum **compose** karke build karenge instead of deep inheritance hierarchy (`Product → ElectronicProduct → MobilePhone → Smartphone`). SQL pe **pagination strategies** (offset-based vs cursor-based) aur **index optimization** for product search.

Aaj polymorphism ka power dekha — kal usi ka **abuse** se kaise bachna hai, woh seekhenge. Inheritance + polymorphism powerful hain but easily misused — deep inheritance trees turn into "diamond problem" nightmares. Composition simpler, more flexible, more testable. **"Has-a" beats "Is-a" 90% of the time** — yeh principle pakka karenge.

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

**Streak**: 🔥 4 days strong, bhai! Consistency hi senior banayegi. Aaj bhi do baar padhna — pehli baar concepts, dosri baar interview perspective.
