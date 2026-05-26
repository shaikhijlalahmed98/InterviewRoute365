# 📝 Day 14 Quiz — Email Verification Flow

**📖 Lesson**: [day-014-email-verification-flow.md](../lessons/01-beginner/2026-05-26-day14-email-verification-flow.md)
**🔑 Revision**: [day-014-email-verification-flow.md](../revision/day-014-email-verification-flow.md)
**Total**: 50 MCQs | **Difficulty**: 8⭐ + 28⭐⭐ + 14⭐⭐⭐
**Goal**: 80% (40/50) before moving to Day 15. Below 70% = re-read lesson.

---

## 📋 Sections

- **A. Code Output Prediction** (10) — "Yeh code chala — output kya?"
- **B. Bug Spotting** (8) — "Is code mein kya galat hai?"
- **C. Dry-Run Tracing** (10) — Step-by-step execution trace
- **D. Scenario-Based Decisions** (10) — Real production situation
- **E. Concept Mastery** (12) — Direct conceptual MCQs

---

## Section A: Code Output Prediction (10 MCQs)

### Q1. ⭐⭐ Yeh Java code ka output kya hoga?
```java
byte[] tokenBytes = new byte[32];
new SecureRandom().nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
System.out.println(token.length());
```
A) 32
B) 43
C) 44
D) 64

**Your Answer**: ___

---

### Q2. ⭐⭐ Same input pe yeh kya return karega?
```java
String s = "abc123";
boolean result = MessageDigest.isEqual(s.getBytes(), s.getBytes());
System.out.println(result);
```
A) true
B) false
C) NullPointerException
D) IllegalArgumentException

**Your Answer**: ___

---

### Q3. ⭐⭐⭐ Yeh C# code ka result?
```csharp
var bytes = new byte[16];
RandomNumberGenerator.Fill(bytes);
var base64 = Convert.ToBase64String(bytes);
Console.WriteLine(base64.Length);
```
A) 16
B) 22
C) 24
D) 32

**Your Answer**: ___

---

### Q4. ⭐⭐ SQL output? Initial: 1 unused token row.
```sql
UPDATE verification_tokens 
SET is_used = 1 
WHERE token = 'abc' AND is_used = 0;
SELECT @@ROWCOUNT;
```
A) 0
B) 1
C) -1
D) NULL

**Your Answer**: ___

---

### Q5. ⭐⭐⭐ Same query, second time pe `@@ROWCOUNT` kya hoga?
A) 1 (token already updated)
B) 0 (already used, condition fails)
C) Error (constraint violation)
D) NULL

**Your Answer**: ___

---

### Q6. ⭐ Yeh Spring code mein kya inject hoga?
```java
@Service
public class VerifyService {
    @Autowired
    private EmailSender emailSender;
}
```
A) SendGrid SDK class
B) The bean implementing EmailSender interface
C) Compile error - interface can't be autowired
D) null - manual instantiation needed

**Your Answer**: ___

---

### Q7. ⭐⭐ Angular template output state?
```typescript
state = 'loading';
// ... HTTP returns 410 ...
catchError(err => { 
  this.state = err.status === 410 ? 'expired' : 'invalid'; 
  return EMPTY; 
})
```
A) 'loading'
B) 'success'
C) 'expired'
D) 'invalid'

**Your Answer**: ___

---

### Q8. ⭐⭐ Java BCrypt hashing same password 2 baar:
```java
String h1 = BCrypt.hashpw("pass", BCrypt.gensalt());
String h2 = BCrypt.hashpw("pass", BCrypt.gensalt());
System.out.println(h1.equals(h2));
```
A) true
B) false
C) Compile error
D) NullPointerException

**Your Answer**: ___

---

### Q9. ⭐⭐⭐ C# atomic upsert via MERGE — pehli baar phone="+92-300" ke saath:
```sql
MERGE otp AS t USING (SELECT '+92-300' AS phone) AS s
ON t.phone = s.phone
WHEN MATCHED THEN UPDATE SET attempts = 0
WHEN NOT MATCHED THEN INSERT (phone, attempts) VALUES (s.phone, 0);
SELECT COUNT(*) FROM otp WHERE phone = '+92-300';
```
A) 0
B) 1
C) 2
D) Error

**Your Answer**: ___

---

### Q10. ⭐⭐ Spring `@Transactional` method calls another `@Transactional` method in SAME class. Transaction count?
```java
@Service
public class X {
    @Transactional public void a() { b(); }
    @Transactional(propagation = REQUIRES_NEW) public void b() { /* ... */ }
}
// External caller invokes a()
```
A) 1 transaction (b's @Transactional ignored — self-invocation)
B) 2 transactions (REQUIRES_NEW creates new)
C) 0 transactions (proxy not applied)
D) Compile error

**Your Answer**: ___

---

## Section B: Bug Spotting (8 MCQs)

### Q11. ⭐⭐ Is code mein kya bug hai?
```java
public String generateToken() {
    return UUID.randomUUID().toString() + System.currentTimeMillis();
}
```
A) Token too short
B) UUID + timestamp = predictable enough for security; should use SecureRandom with cryptographic entropy
C) Should use `+` not `.toString()`
D) Missing salt

**Your Answer**: ___

---

### Q12. ⭐⭐ Bug spot karo:
```java
@Transactional
public void verify(String token) {
    VerificationToken vt = tokenRepo.findByToken(token).orElseThrow();
    if (vt.isExpired()) throw new TokenExpiredException();
    vt.markUsed();
    emailSender.send(vt.getUser().getEmail(), "Welcome!");
}
```
A) Missing isUsed() check — same token reusable
B) emailSender inside @Transactional → SMTP failure rolls back verification + dual-write problem
C) Both A and B
D) Code is fine

**Your Answer**: ___

---

### Q13. ⭐⭐⭐ Bug:
```java
public boolean compareTokens(String submitted, String stored) {
    return submitted.equals(stored);  // both already hashed
}
```
A) Should use `==`
B) `.equals()` short-circuits on first mismatched char → timing attack leaks token
C) Should hash again before compare
D) Missing null check

**Your Answer**: ___

---

### Q14. ⭐⭐ SQL bug:
```sql
SELECT user_id FROM verification_tokens WHERE token = @t;
-- check expired in app code
UPDATE verification_tokens SET is_used = 1 WHERE token = @t;
```
A) Index missing
B) TOCTOU race — between SELECT and UPDATE, another request could verify the same token
C) Wrong column name
D) Missing transaction

**Your Answer**: ___

---

### Q15. ⭐⭐⭐ Angular bug:
```typescript
ngOnInit() {
  const token = this.route.snapshot.paramMap.get('token');
  this.auth.verifyEmail(token!).subscribe(() => this.state = 'success');
}
```
A) Missing error handler — verification failures show no feedback to user
B) Wrong route reading
C) Missing async pipe
D) state should be Observable

**Your Answer**: ___

---

### Q16. ⭐⭐ Bug:
```java
public interface NotificationService {
    void sendEmail(String to, String body);
    void sendSms(String phone, String msg);
    void sendPush(String deviceId, String msg);
    void sendWhatsApp(String phone, String template);
}

@Service
public class EmailVerification {
    @Autowired NotificationService notifier;
    public void verify(User u) {
        notifier.sendEmail(u.getEmail(), "verify");
    }
}
```
A) Wrong DI scope
B) Fat interface violates ISP — EmailVerification depends on SMS/Push/WhatsApp methods it doesn't use
C) Should use class not interface
D) Missing @Async

**Your Answer**: ___

---

### Q17. ⭐⭐⭐ C# bug:
```csharp
public async Task VerifyAsync(string token) {
    var vt = await _db.VerificationTokens.FirstOrDefaultAsync(v => v.Token == token);
    if (vt == null) throw new InvalidTokenException();
    vt.IsUsed = true;
    vt.User.EmailVerified = true;
    await _db.SaveChangesAsync();
}
```
A) Missing `Include(v => v.User)` → vt.User is null, NullReferenceException
B) Wrong async
C) Should use sync version
D) Code is fine

**Your Answer**: ___

---

### Q18. ⭐⭐ Bug:
```java
@Service
public class EmailVerificationService {
    private final TwilioRestClient twilio = new TwilioRestClient("sid", "token");
    
    public void sendVerification(String email, String token) {
        // ... uses twilio directly
    }
}
```
A) Twilio is SMS, not email
B) Hard-coded credentials in code + tight coupling to Twilio = DIP violation, untestable, vendor lock-in
C) Should use new on each call
D) Missing @Autowired

**Your Answer**: ___

---

## Section C: Dry-Run Tracing (10 MCQs)

### Q19. ⭐⭐⭐ User clicks verify link 3 times within 100ms (parallel requests). DB state initially: 1 unused token.
```
T+0ms:   Req1 starts UPDATE WHERE is_used=0
T+10ms:  Req1 acquires row X-lock
T+20ms:  Req2 starts UPDATE WHERE is_used=0 → blocked
T+30ms:  Req3 starts UPDATE WHERE is_used=0 → blocked
T+50ms:  Req1 commits, releases lock, @@ROWCOUNT = 1
T+51ms:  Req2 acquires lock, condition is_used=0 fails
T+52ms:  Req2 @@ROWCOUNT = 0
```
**Question**: How many verifications succeed?
A) 0
B) 1 (Req1 only)
C) 2
D) All 3

**Your Answer**: ___

---

### Q20. ⭐⭐⭐ Token expiry timeline. Token created at T=0, expires at T+24h.
```
T+0       Token generated, sent via email
T+1h      User opens email, clicks link
T+24h     System cleanup job runs
T+25h     Attacker tries token from leaked email
```
**Question**: Attacker's verify attempt at T+25h returns?
A) Success (token valid forever)
B) 410 Gone (expired, single-purpose error)
C) 500 (race condition)
D) 200 OK (idempotent)

**Your Answer**: ___

---

### Q21. ⭐⭐⭐ Trace this @Transactional scenario:
```
T+0   service.verifyToken("abc") called from controller
T+1   Spring AOP proxy intercepts call
T+2   txManager.begin() — autocommit OFF
T+3   findByToken("abc") — returns VerificationToken vt
T+4   vt.markUsed() — entity state changed, NOT yet in DB
T+5   vt.getUser().setEmailVerified(true)
T+6   method returns
T+7   Spring AOP proxy: commit()
T+8   Hibernate flushes dirty entities: UPDATE vt + UPDATE user
T+9   COMMIT
```
**Question**: Agar T+5 pe exception aaye?
A) Only token update commits, user update rolls back
B) Both updates rolled back — single transaction atomicity
C) Only user update commits
D) Hangs indefinitely

**Your Answer**: ___

---

### Q22. ⭐⭐ Outbox pattern dry-run:
```
T+0    User signs up — POST /users
T+1    UserService starts transaction
T+2    INSERT users (...)
T+3    INSERT outbox_events ("SEND_VERIFICATION_EMAIL", ...)
T+4    COMMIT
T+5    HTTP 201 returned to client
T+10   Outbox worker polls table — finds new event
T+11   Worker calls SMTP — fails (SMTP down)
T+12   Worker leaves event for retry
T+5min Worker retries — SMTP recovered, email sent
T+5min Worker marks event as processed
```
**Question**: At T+5, is the user account created?
A) No, SMTP failure rolled it back
B) Yes — outbox decouples SMTP from user creation
C) Partially — user with no email
D) Crashed server

**Your Answer**: ___

---

### Q23. ⭐⭐⭐ Rate limit Redis trace. Initial: counter=0.
```
T+0    User clicks "Resend" — Redis INCR otp:req:+92300 → returns 1, set EXPIRE 1hr
T+10s  Click 2 — INCR returns 2
T+20s  Click 3 — INCR returns 3
T+25s  Click 4 — INCR returns 4 → THROW RateLimitExceededException
T+1hr  Key auto-expires
T+1hr+1s  Click 5 — INCR returns 1 (key recreated)
```
**Question**: Limit is 3/hour. How many SMS sent?
A) 5
B) 4
C) 3
D) 6

**Your Answer**: ___

---

### Q24. ⭐⭐⭐ Idempotency trace:
```
T+0    User clicks verify link — token verified, is_used=1, user.verified=true
T+5s   User clicks link again (still in inbox)
T+5s   Backend: findByToken returns vt, isUsed=true
T+5s   IF (vt.isUsed()) return; // idempotent
T+5s   HTTP 200 OK to client
```
**Question**: What should UI show on second click?
A) Error "already verified"
B) Success "email verified" — same friendly outcome
C) 500 error
D) Redirect to signup

**Your Answer**: ___

---

### Q25. ⭐⭐ Anti-enumeration trace:
```
Scenario A: User signs up with existing-verified email "a@x.com"
Scenario B: User signs up with existing-unverified email "b@x.com"
Scenario C: User signs up with new email "c@x.com"
```
**Question**: Response should be?
A) A: "exists", B: "verify pending", C: "check email" — different
B) All three: same generic "check your email for next steps"
C) A: 409, B: 200, C: 201 — different status codes
D) Only fail for A

**Your Answer**: ___

---

### Q26. ⭐⭐⭐ SecureRandom vs Random predictability:
```java
Random r = new Random(42);  // seed = 42
System.out.println(r.nextInt());  // -1170105035
System.out.println(r.nextInt());  // 234785527
// Attacker observes outputs, reverse-engineers seed, predicts next
```
**Question**: SecureRandom mein same predictable?
A) Yes — same algorithm
B) No — uses OS entropy (/dev/urandom), not deterministic seed
C) Only if same seed
D) Compile error — no seed allowed

**Your Answer**: ___

---

### Q27. ⭐⭐ Trace Java token generation:
```java
SecureRandom rnd = new SecureRandom();
byte[] bytes = new byte[32];
rnd.nextBytes(bytes);  // 32 random bytes
String b64 = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
// Math: 32 bytes × 8 bits = 256 bits; Base64 = 6 bits per char; 256/6 = 42.67 → 43 chars
```
**Question**: Total entropy in bits?
A) 256 bits
B) 128 bits
C) 64 bits
D) 32 bits

**Your Answer**: ___

---

### Q28. ⭐⭐⭐ Concurrent verify attempts trace:
```
Two browser tabs, same user, both have verify link.
T+0   Tab1: UPDATE WHERE is_used=0 → row locked
T+0   Tab2: UPDATE WHERE is_used=0 → BLOCKED waiting for lock
T+5   Tab1: COMMIT → row unlocked
T+5   Tab2: lock acquired → WHERE clause re-evaluated
T+5   Tab2: is_used now = 1 → condition fails → @@ROWCOUNT = 0
```
**Question**: Tab2 ka outcome?
A) Verification succeeds (lock-and-wait then continue)
B) Verification fails — single-use enforced; UI shows already-verified message
C) Deadlock detected, both roll back
D) Database crash

**Your Answer**: ___

---

## Section D: Scenario-Based Decisions (10 MCQs)

### Q29. ⭐⭐⭐ Production issue: Daraz verify link mein attacker brute-forces tokens. Logs show 100k token attempts per minute from rotating IPs. What's the best defense?
A) Block all IPs
B) Increase token entropy to 64 bytes
C) Token entropy of 32 bytes already mathematically unguessable (2^256 = 10^77 possibilities); attacker can't brute force, this is logging artifact or false positive
D) Disable verification temporarily

**Your Answer**: ___

---

### Q30. ⭐⭐ FoodPanda signup spike — 10k new users per hour. SMTP server starts queuing emails. Verification emails delayed 30 min. What's the senior engineer fix?
A) Increase SMTP timeout
B) Implement Outbox pattern with multiple worker threads, scale workers horizontally
C) Send emails synchronously
D) Move to a paid SMTP

**Your Answer**: ___

---

### Q31. ⭐⭐⭐ Verification token leak — backup of `verification_tokens.csv` accidentally pushed to public Git. 100k tokens exposed. Impact?
A) Massive — all unverified accounts can be hijacked
B) None if SHA-256 hashes stored (raw token never in DB); attacker has hashes, can't reverse
C) Only affects expired tokens
D) Email notification protects users

**Your Answer**: ___

---

### Q32. ⭐⭐ Careem manager: "Add WhatsApp verification alongside email." How to add with minimum code change?
A) Modify EmailVerificationService to also send WhatsApp
B) Create new `WhatsAppVerificationService` implementing same `IVerificationChannel` interface; inject both via `List<IVerificationChannel>`; Strategy Pattern
C) Fork the codebase
D) Use stored procedures

**Your Answer**: ___

---

### Q33. ⭐⭐⭐ Microservices: User service creates user; Email service sends verification. Communication via REST. User service POST /users, then POST /email-service/send. Email service returns 503. What goes wrong?
A) Nothing — eventually retried
B) Dual-write problem — user committed in User DB but email never sent. Solution: Outbox pattern or saga
C) User service crashes
D) Returns 200 anyway

**Your Answer**: ___

---

### Q34. ⭐⭐ Saudi compliance: PDPL requires audit log of every verification email sent (who, when, IP). Where to add?
A) Inside EmailSender concrete implementations
B) Cross-cutting concern — implement as Spring AOP aspect on @VerificationAudited or as decorator wrapping EmailSender; keep core logic clean
C) In the Controller
D) In Angular frontend

**Your Answer**: ___

---

### Q35. ⭐⭐⭐ Bug report: "Verification works first time, but if user clicks link twice, second click shows error 500." Root cause?
A) Database deadlock
B) Service throws exception on already-used token instead of returning idempotent success
C) Spring config wrong
D) Email cached

**Your Answer**: ___

---

### Q36. ⭐⭐ Test scenario: How to unit test `EmailVerificationService` without hitting real SMTP?
A) Use real SMTP test account
B) Mock `EmailSender` interface (Mockito/Moq), verify .send() called with expected arguments. ISP + DIP make this trivial.
C) Skip tests for email logic
D) Use integration test only

**Your Answer**: ___

---

### Q37. ⭐⭐⭐ Token expiry choice: 1 hour, 24 hours, 7 days, never expire. Best for email verification?
A) Never expire — most convenient
B) 24 hours — balance UX (user might check email next morning) and security (limited exploit window if email forwarded)
C) 1 hour — too aggressive, users miss window
D) 7 days — too lax, screenshot exploitable

**Your Answer**: ___

---

### Q38. ⭐⭐ Architecture review: Service has `List<EmailSender> senders` injected. Method iterates `for (EmailSender s : senders) s.send(...)`. What pattern + concern?
A) Strategy + risk of sending email multiple times
B) Chain of Responsibility / Composite — useful for fallback chain (try primary, on fail try secondary). Idempotency required on receivers.
C) Observer
D) Visitor

**Your Answer**: ___

---

## Section E: Concept Mastery (12 MCQs)

### Q39. ⭐ Why store SHA-256 hash of token in DB instead of raw token?
A) Faster lookup
B) DB leak protection — attacker gets hashes but can't reverse to raw tokens (one-way hash)
C) Saves disk space
D) Required by JDBC

**Your Answer**: ___

---

### Q40. ⭐ `SecureRandom` vs `Random` — main difference?
A) Speed
B) `SecureRandom` uses OS-level entropy (cryptographically secure); `Random` uses LCG (predictable)
C) Thread safety
D) Memory usage

**Your Answer**: ___

---

### Q41. ⭐⭐ What does ISP (Interface Segregation Principle) say?
A) Use interfaces over classes
B) Many small focused interfaces beat one fat interface; clients shouldn't depend on methods they don't use
C) Always extend from interfaces
D) Interfaces must have at least one method

**Your Answer**: ___

---

### Q42. ⭐ Memory hook for ISP?
A) "EK BANDA, EK KAAM"
B) "BHARI INTERFACE MAT BHEJO" / "CHOTA INTERFACE, BARA SUKOON"
C) "BARTAN BADLO, KHAANA WOHI"
D) "EXTEND HAAN, MODIFY NAA"

**Your Answer**: ___

---

### Q43. ⭐⭐ What HTTP status code for expired verification link?
A) 400 Bad Request
B) 410 Gone — semantically "resource was there but no longer available"
C) 404 Not Found
D) 500 Internal Server Error

**Your Answer**: ___

---

### Q44. ⭐⭐ Idempotency in verify endpoint means?
A) Returns same response every call
B) Already-verified user clicking link again gets 200 OK (success), not error — safe to repeat
C) Database insert only
D) JWT token issued

**Your Answer**: ___

---

### Q45. ⭐⭐⭐ Outbox Pattern solves which problem?
A) Database deadlock
B) Dual-write problem — atomically commit business data + external event publication via single DB transaction; background worker dispatches to external system with retry
C) Message ordering
D) Schema migration

**Your Answer**: ___

---

### Q46. ⭐⭐ Why constant-time comparison for token check?
A) Faster
B) Defeats timing attacks — `.equals()` short-circuits on mismatch position, leaking which characters match
C) Required by SQL
D) Spring annotation requirement

**Your Answer**: ___

---

### Q47. ⭐ Anti-enumeration on signup means?
A) Block signups
B) Same generic response whether email exists or not — prevents attacker from harvesting valid emails
C) Captcha required
D) Rate limit per IP

**Your Answer**: ___

---

### Q48. ⭐⭐⭐ `@Transactional` Spring mechanism?
A) Compile-time bytecode weaving
B) Runtime AOP proxy (CGLib subclass / JDK dynamic proxy) wrapping method with begin/commit/rollback
C) JVM agent attached
D) JNDI lookup

**Your Answer**: ___

---

### Q49. ⭐⭐ Composite/Filtered index `WHERE is_used = 0` — purpose?
A) Hides used tokens
B) Smaller index size (only ~current unused rows), faster lookup, less memory
C) Required for foreign keys
D) Better INSERT performance

**Your Answer**: ___

---

### Q50. ⭐⭐ Real company that uses Outbox pattern for verification emails?
A) Shopify (order confirmations), GitHub (background workers via Resque)
B) Only banks
C) Only Stripe
D) No one in production

**Your Answer**: ___

---

## 🎯 Score Tracker

```
Section A (Code Output):     ___/10
Section B (Bug Spotting):    ___/8
Section C (Dry-Run):         ___/10
Section D (Scenarios):       ___/10
Section E (Concept):         ___/12
TOTAL:                       ___/50
```

**Target**: 40+/50 = Strong (move on)  
**35-39/50** = OK (review weak areas)  
**<35/50** = Re-read lesson

---

<details>
<summary>⚠️ Answer Key — Click only after completing all 50!</summary>

### Section A: Code Output Prediction
1. **B (43)** — 32 bytes Base64URL-encoded without padding = ⌈32×4/3⌉ = 43 chars
2. **A (true)** — `MessageDigest.isEqual` returns true for byte-equal arrays
3. **C (24)** — 16 bytes Base64 with padding: ⌈16/3⌉×4 = 24 chars
4. **B (1)** — Token found, condition met, 1 row updated
5. **B (0)** — `is_used = 0` condition now fails (already 1)
6. **B** — Spring autowires the bean implementing the interface
7. **C ('expired')** — status 410 triggers expired branch
8. **B (false)** — BCrypt embeds random salt, each hash unique
9. **B (1)** — MERGE inserts (no match), 1 row exists
10. **A (1 transaction)** — Self-invocation bypasses proxy, b's @Transactional ignored

### Section B: Bug Spotting
11. **B** — UUID+timestamp predictable; use SecureRandom 32+ bytes
12. **C (both A and B)** — Missing isUsed() + dual-write inside @Transactional
13. **B** — `.equals()` short-circuits leaking info via timing
14. **B** — TOCTOU race between SELECT and UPDATE
15. **A** — Missing error handler leaves user stuck
16. **B** — Fat interface violates ISP
17. **A** — Missing `.Include(v => v.User)` causes NullReferenceException
18. **B** — DIP violation: hard-coded credentials + tight coupling

### Section C: Dry-Run Tracing
19. **B (1 only)** — Atomic UPDATE serializes via row lock, only Req1 wins
20. **B (410 Gone)** — Token expired, cleaned up
21. **B (both rolled back)** — Single transaction atomicity
22. **B (yes)** — Outbox decouples SMTP from user creation
23. **C (3)** — Limit 3/hour; 4th throws exception, no SMS
24. **B (success message)** — Idempotent UX prevents user panic
25. **B (all same generic response)** — Anti-enumeration
26. **B** — SecureRandom uses OS entropy, non-deterministic
27. **A (256 bits)** — 32 bytes × 8 = 256 bits entropy
28. **B** — Tab2 sees is_used=1, atomic UPDATE returns 0 rows; UI shows verified message

### Section D: Scenario-Based Decisions
29. **C** — Token entropy 2^256 is mathematically unguessable; address logging or DoS not brute force concern
30. **B** — Outbox + horizontal worker scaling
31. **B** — None if SHA-256 hashes stored (one-way), raw tokens never in DB
32. **B** — Strategy Pattern with multiple impls of IVerificationChannel
33. **B** — Dual-write problem; needs Outbox or saga
34. **B** — Cross-cutting via AOP/decorator keeps core clean
35. **B** — Should return idempotent success on already-used, not throw
36. **B** — Mock EmailSender interface; ISP + DIP enable easy mocking
37. **B (24 hours)** — Balance UX (overnight check) and security (limited exploit window)
38. **B** — Composite/Chain of Responsibility for fallback; receivers must be idempotent

### Section E: Concept Mastery
39. **B** — One-way hash protects against DB leak
40. **B** — OS entropy vs LCG difference
41. **B** — ISP definition: small focused interfaces
42. **B** — "BHARI INTERFACE MAT BHEJO" / "CHOTA INTERFACE, BARA SUKOON"
43. **B (410 Gone)** — Semantically correct for expired resource
44. **B** — Idempotent = repeat-safe with same outcome
45. **B** — Outbox solves dual-write atomically
46. **B** — Constant-time defeats timing attacks
47. **B** — Same generic response prevents email harvesting
48. **B** — Spring AOP runtime proxy mechanism
49. **B** — Smaller filtered index = faster + lighter
50. **A** — Shopify, GitHub well-known Outbox users

</details>
