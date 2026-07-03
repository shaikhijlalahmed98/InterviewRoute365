# 📝 Day 4 Quiz — Password Reset Flow (Practical Edition)

**📖 Lesson**: [day-004-password-reset.md](../lessons/day-004-password-reset.md)
**🔑 Revision Keys**: [revision/day-004-password-reset.md](../revision/day-004-password-reset.md)
**⏱️ Suggested time**: 50-70 minutes
**📊 Total questions**: 50
**🎯 Format**: Code analysis · Dry-run traces · Bug spotting · Scenarios · Concept mastery

---

## 📋 Instructions

1. Replace `___` with letter for each question (`**Your Answer**: B`)
2. Difficulty marked: ⭐ (Easy) · ⭐⭐ (Medium) · ⭐⭐⭐ (Hard)
3. Commit + push when done — mentor auto-evaluates next day
4. Don't peek at answer key until done

---

## 🧪 Section A: Code Output Prediction (10 Qs)

### Q1. ⭐ Output kya hoga?
```java
SecureRandom random = new SecureRandom();
byte[] tokenBytes = new byte[32];
random.nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
System.out.println(token.length());
```
A) 32
B) 43
C) 44
D) 64

**Your Answer**: ___

---

### Q2. ⭐⭐ Yeh code 2 baar run kar — `h1.equals(h2)` ka result?
```java
BCryptPasswordEncoder enc = new BCryptPasswordEncoder();
String h1 = enc.encode("MySecret123");
String h2 = enc.encode("MySecret123");
System.out.println(h1.equals(h2));
System.out.println(enc.matches("MySecret123", h1));
System.out.println(enc.matches("MySecret123", h2));
```
A) true, true, true (deterministic)
B) false, true, true (different salts, but matches() handles both)
C) true, false, false
D) Compile error

**Your Answer**: ___

---

### Q3. ⭐⭐ SHA-256 ka output:
```java
String h1 = sha256("hello");
String h2 = sha256("hello");
String h3 = sha256("Hello");  // capital H
System.out.println(h1.equals(h2));
System.out.println(h1.equals(h3));
```
A) true, true
B) true, false
C) false, false
D) false, true

**Your Answer**: ___

---

### Q4. ⭐⭐⭐ Spring `@Transactional` self-invocation bug — actual behavior?
```java
@Service
public class TokenService {
    @Transactional
    public void outerMethod() {
        // some DB work
        this.innerMethod();  // same class call!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // expects NEW transaction
    }
}
```
A) Both methods run in separate transactions as declared
B) `innerMethod()` ka `@Transactional` ignore — runs in `outerMethod()`'s transaction (proxy bypassed)
C) Compile error
D) Runtime exception "Cannot invoke @Transactional internally"

**Your Answer**: ___

---

### Q5. ⭐⭐ Hibernate is JPA code ke liye konsa SQL generate karega?
```java
@Entity
public class Token {
    @Id Long id;
    boolean used;
    @Version Long version;
}

// Service code:
Token t = repo.findById(5L).get();  // version is 3
t.setUsed(true);
repo.save(t);
```
A) `UPDATE token SET used=1 WHERE id=5`
B) `UPDATE token SET used=1, version=4 WHERE id=5 AND version=3`
C) `UPDATE token SET used=1, version=version+1 WHERE id=5`
D) `INSERT INTO token (used, version) VALUES (1, 4)`

**Your Answer**: ___

---

### Q6. ⭐⭐ Polymorphic dispatch — output?
```java
class Animal { public String sound() { return "generic"; } }
class Dog extends Animal { public String sound() { return "woof"; } }
class Puppy extends Dog { public String sound() { return "yip"; } }

Animal a = new Puppy();
Dog d = new Puppy();
Puppy p = new Puppy();
System.out.println(a.sound() + " " + d.sound() + " " + p.sound());
```
A) generic woof yip
B) yip yip yip
C) generic generic yip
D) woof woof yip

**Your Answer**: ___

---

### Q7. ⭐⭐⭐ Method overloading vs overriding — yeh compile karega?
```java
class Parent {
    public Number calculate() { return 10; }
}
class Child extends Parent {
    @Override
    public Integer calculate() { return 20; }  // narrower return type
}
```
A) Compile error — return type must match exactly
B) Compiles fine — covariant return types allowed in Java since 5.0
C) Compiles but @Override fails at runtime
D) Compiles only without @Override

**Your Answer**: ___

---

### Q8. ⭐⭐ RxJS pipe — output sequence?
```typescript
import { of } from 'rxjs';
import { map, filter, take } from 'rxjs/operators';

of(1, 2, 3, 4, 5).pipe(
  filter(n => n % 2 === 1),
  map(n => n * 10),
  take(2)
).subscribe(v => console.log(v));
```
A) 10, 20, 30, 40, 50
B) 10, 30
C) 30, 50
D) 10, 30, 50

**Your Answer**: ___

---

### Q9. ⭐⭐⭐ Cold Observable — kitne HTTP calls hote hain?
```typescript
const data$ = this.http.get('/api/users');
data$.subscribe(u => console.log('a', u));
data$.subscribe(u => console.log('b', u));
data$.subscribe(u => console.log('c', u));
```
A) 1 (cached)
B) 3 (cold = each subscribe fires)
C) 0 (Observable not "hot")
D) Compile error

**Your Answer**: ___

---

### Q10. ⭐⭐ `debounceTime` timeline:
```typescript
formControl.valueChanges.pipe(
  debounceTime(300)
).subscribe(console.log);

// User types: 'a' at 0ms, 'b' at 100ms, 'c' at 200ms, 'd' at 600ms, stops
```
**Kya emit hota hai aur kab?**
A) 'a' at 0ms, 'b' at 100ms, 'c' at 200ms, 'd' at 600ms
B) 'abc' at 500ms, 'abcd' at 900ms
C) 'abc' at 500ms (200+300), then 'abcd' at 900ms (600+300)
D) Sirf 'abcd' at 900ms

**Your Answer**: ___

---

## 🐛 Section B: Bug Spotting (8 Qs)

### Q11. ⭐⭐⭐ Is code mein konsa security bug hai?
```java
@Service
public class PasswordResetService {
    public void confirmReset(String rawToken, String newPassword) {
        Token t = tokenRepo.findByHash(sha256(rawToken));
        if (t != null && !t.isUsed() && t.getExpiry().isAfter(Instant.now())) {
            // mark used
            t.setUsed(true);
            tokenRepo.save(t);
            // update password
            userRepo.updatePassword(t.getUserId(), newPassword);
        }
    }
}
```
A) Password not hashed before save
B) TOCTOU race — two concurrent clicks can both pass `if`, both update
C) Token expiry check wrong direction
D) Both A and B

**Your Answer**: ___

---

### Q12. ⭐⭐ Yeh entity mein kya missing hai for proper concurrency?
```java
@Entity
public class Account {
    @Id Long id;
    BigDecimal balance;
    // BankApp has 100K concurrent users transferring money
}
```
A) `@Index` on balance
B) `@Version Long version` for optimistic locking (otherwise lost-update bugs)
C) `@Transient` on id
D) Nothing missing

**Your Answer**: ___

---

### Q13. ⭐⭐⭐ Constructor mein overridable method call — kya hoga?
```java
class Parent {
    public Parent() { init(); }
    public void init() { System.out.println("parent init"); }
}
class Child extends Parent {
    private String name = "default";
    @Override
    public void init() { System.out.println("child init: " + name.toUpperCase()); }
}

new Child();
```
A) "parent init"
B) "child init: DEFAULT"
C) "child init: " then NullPointerException (`name` is null when Parent constructor runs)
D) Compile error

**Your Answer**: ___

---

### Q14. ⭐⭐ Yeh equals() override mein kya bug hai?
```java
public class User {
    private String email;
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof User)) return false;
        return this.email.equals(((User) o).email);
    }
}
// Then:
Set<User> users = new HashSet<>();
users.add(new User("a@b.com"));
boolean has = users.contains(new User("a@b.com"));  // expected: true
```
A) `equals` wrong — should compare by `==`
B) `hashCode()` not overridden — HashSet bucketing breaks, `contains` returns false
C) `instanceof` wrong — should use `getClass()`
D) HashSet doesn't use equals

**Your Answer**: ___

---

### Q15. ⭐⭐ Token storage code — kya galat hai?
```java
String rawToken = Base64.getEncoder().encodeToString(secureRandom.generateSeed(32));
PasswordResetToken token = new PasswordResetToken();
token.setUserId(user.getId());
token.setToken(rawToken);  // ← yeh line
token.setExpiresAt(Instant.now().plus(15, ChronoUnit.MINUTES));
tokenRepo.save(token);
emailService.send(user.getEmail(), "Click: " + rawToken);
```
A) Token expiry too short
B) DB mein **raw token** save kar raha — DB breach pe saare active tokens leak. Should store SHA-256 hash.
C) Base64 wrong encoding
D) generateSeed() is correct usage

**Your Answer**: ___

---

### Q16. ⭐⭐⭐ Login endpoint — kya security issue?
```java
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest req) {
    User user = userRepo.findByEmail(req.getEmail());
    if (user == null) {
        return ResponseEntity.status(404).body("Email not found");
    }
    if (!encoder.matches(req.getPassword(), user.getPasswordHash())) {
        return ResponseEntity.status(401).body("Wrong password");
    }
    return ResponseEntity.ok(jwtUtil.generate(user));
}
```
A) Password should be encrypted in transit
B) **User enumeration**: different error messages reveal which emails exist → attacker harvests valid emails
C) JWT not validated
D) findByEmail too slow

**Your Answer**: ___

---

### Q17. ⭐⭐ Yeh notification dispatch mein OCP violation kahan hai?
```java
public void sendReset(User user, String link, String channel) {
    if (channel.equals("EMAIL")) {
        emailClient.send(user.getEmail(), link);
    } else if (channel.equals("SMS")) {
        smsClient.send(user.getPhone(), link);
    } else if (channel.equals("PUSH")) {
        pushClient.send(user.getDeviceId(), link);
    }
    // Adding WhatsApp = modify this file + add new else-if
}
```
A) Method too long
B) **Open/Closed violated** — adding new channel requires modifying existing code instead of just adding new class
C) Variable names bad
D) No error handling

**Your Answer**: ___

---

### Q18. ⭐⭐ `@Override` annotation chhoot gaya — bug spot karo:
```java
public class EmailNotifier implements NotificationSender {
    public void Send(User user, Message msg) {  // capital S in 'Send'
        // email logic
    }
}
```
**Result kya hai?**
A) Compile error (signature mismatch)
B) `Send` aur `send` dono available — but `NotificationSender` interface ka `send` unimplemented → if interface has default empty `send`, that empty method runs in production → emails never sent → silent bug
C) Runtime exception
D) Both methods work fine

**Your Answer**: ___

---

## 🔬 Section C: Dry-Run Tracing (10 Qs)

### Q19. ⭐⭐⭐ DB initial state aur 2 concurrent requests:
```
Initial: tokens row {id=5, used=0, version=1}

T+0ms:   Request A: SELECT * FROM tokens WHERE id=5;  → version=1
T+0ms:   Request B: SELECT * FROM tokens WHERE id=5;  → version=1
T+10ms:  Request A: UPDATE tokens SET used=1, version=2 WHERE id=5 AND version=1;
T+11ms:  Request B: UPDATE tokens SET used=1, version=2 WHERE id=5 AND version=1;
```
**Final state aur dono requests ka result?**
A) Both succeed; row final state: used=1, version=2
B) A succeeds (rowcount=1, version becomes 2); B fails (rowcount=0) → throws OptimisticLockException
C) B succeeds; A fails
D) Deadlock

**Your Answer**: ___

---

### Q20. ⭐⭐⭐ Atomic UPDATE pattern — dono concurrent requests:
```
Initial: tokens {id=5, used=0, expires_at=2030-01-01}

T+0ms:   Request A: UPDATE tokens SET used=1
                    WHERE id=5 AND used=0 AND expires_at > NOW();
T+0ms:   Request B: UPDATE tokens SET used=1
                    WHERE id=5 AND used=0 AND expires_at > NOW();
```
**Internal lock behavior aur result?**
A) Both succeed (no locking)
B) A acquires U→X lock; B waits; A commits (rowcount=1, used=1); B gets lock, sees used=1, WHERE fails, rowcount=0
C) Both fail
D) Race condition — random winner

**Your Answer**: ___

---

### Q21. ⭐⭐⭐ Isolation level READ COMMITTED — T2 kya dekhega?
```
T1 starts:
  UPDATE accounts SET balance = balance - 500 WHERE id = 5;  (not yet committed)

T2 starts (READ COMMITTED):
  SELECT balance FROM accounts WHERE id = 5;
```
**T2 ka result?**
A) Updated value (balance - 500) — sees uncommitted data
B) Old value (original balance) — only committed data visible at READ COMMITTED
C) Blocks T2 forever
D) Throws error

**Your Answer**: ___

---

### Q22. ⭐⭐ RxJS stream — output sequence?
```typescript
import { interval } from 'rxjs';
import { take, map, filter } from 'rxjs/operators';

interval(100).pipe(
  filter(n => n > 0),
  map(n => n * n),
  take(3)
).subscribe(v => console.log(v));
```
**1 second mein console mein kya print hoga?**
A) 0, 1, 4
B) 1, 4, 9
C) 100, 400, 900
D) 0, 1, 4, 9

**Your Answer**: ___

---

### Q23. ⭐⭐ Zone.js + change detection trace:
```typescript
@Component({ template: '<p>{{ count }}</p>' })
class MyComp {
  count = 0;
  ngOnInit() {
    setTimeout(() => {
      this.count = 5;  // change happens
    }, 100);
  }
}
```
**Kya hota hai?**
A) `count = 5` set hota but UI nahi update hoti (Angular notice nahi karta)
B) Zone.js setTimeout ko monkey-patch karta hai → callback complete pe Angular ko notify → change detection runs → UI shows 5
C) Manual `changeDetectorRef.detectChanges()` zaruri hai
D) `count` private nahi reachable

**Your Answer**: ___

---

### Q24. ⭐⭐⭐ Kafka consumer — crash recovery scenario:
```
Topic: orders, partition 0
Messages: [m1, m2, m3, m4, m5]

Consumer reads:
  - Reads m1, processes, commits offset 1
  - Reads m2, processes, commits offset 2
  - Reads m3, processing CRASHES (no commit)
  - Consumer restarts...
```
**Restart pe consumer kahan se shuru karega?**
A) From m1 (start of partition)
B) From m3 (offset 3, last committed +1)
C) From m4 (next after last read)
D) From m5 (latest)

**Your Answer**: ___

---

### Q25. ⭐⭐ Token bucket simulation:
```
Bucket capacity: 5 tokens
Refill rate: 1 token/sec
Initial: bucket full (5/5)

T+0ms:    Request → consume → bucket=4
T+100ms:  Request → consume → bucket=3
T+200ms:  Request → consume → bucket=2
T+300ms:  Request → consume → bucket=1
T+400ms:  Request → consume → bucket=0
T+500ms:  Request → ???
T+1500ms: Request → ???
```
**Last 2 requests ka result?**
A) Both rejected
B) Both accepted
C) T+500ms rejected (bucket empty); T+1500ms accepted (~1 token refilled in 1 sec)
D) Both accepted from queue

**Your Answer**: ___

---

### Q26. ⭐⭐⭐ Vtable lookup trace:
```java
class A { public String m() { return "A"; } }
class B extends A { public String m() { return "B"; } }
class C extends B { /* no override */ }

A obj = new C();
obj.m();  // ?
```
**JVM ka vtable lookup kya return karega aur kyun?**
A) "A" — declared type wins
B) "B" — C inherits B's m(), no own override; vtable[m] points to B.m()
C) "C" — most-derived
D) Ambiguous — compile error

**Your Answer**: ___

---

### Q27. ⭐⭐⭐ Composite index trace — konsi queries index use karengi?
```sql
CREATE INDEX idx_user_active ON tokens(user_id, used, expires_at);

Q1: SELECT * FROM tokens WHERE user_id = 5;
Q2: SELECT * FROM tokens WHERE used = 0;
Q3: SELECT * FROM tokens WHERE user_id = 5 AND used = 0;
Q4: SELECT * FROM tokens WHERE expires_at > NOW();
Q5: SELECT * FROM tokens WHERE user_id = 5 AND expires_at > NOW();
```
**Index kis Q's mein use hoga?**
A) All 5
B) Q1, Q3, Q5 only (leftmost prefix `user_id` present)
C) Q3 only (all 3 columns)
D) None

**Your Answer**: ___

---

### Q28. ⭐⭐⭐ JIT devirtualization decision:
```java
interface Worker { void work(); }
class EmailWorker implements Worker { public void work() { /*...*/ } }

// In hot loop, executed 1 million times:
for (int i = 0; i < 1_000_000; i++) {
    Worker w = getWorker();  // always returns EmailWorker in this codebase
    w.work();  // polymorphic call
}
```
**JIT compiler kya optimization karega aur kyun?**
A) Nothing — virtual call hamesha vtable lookup
B) Devirtualizes — agar JIT detect kare ke monomorphic call site (sirf EmailWorker chal raha) → vtable skip karke direct call (perf boost)
C) Inlines all interfaces
D) Removes the loop

**Your Answer**: ___

---

## 🎯 Section D: Scenario-Based Decisions (10 Qs)

### Q29. ⭐⭐ Production issue: Users report password reset emails kabhi nahi aate. Setup:
```
1. /forgot-password endpoint saves token to DB + publishes Kafka event
2. Notification service consumes Kafka, sends email via SMTP
3. SMTP provider returns 250 OK
```
**Most likely cause?**
A) Database is slow
B) Kafka topic doesn't exist or consumer crashed (events accumulating but not processed)
C) Browser caching
D) Email format wrong

**Your Answer**: ___

---

### Q30. ⭐⭐⭐ Daraz DB breach happened. Attackers got entire `password_reset_tokens` table. Tokens stored as `token_hash` (SHA-256 of raw token). 15-min expiry.
**Risk assessment?**
A) Catastrophic — all reset tokens compromised
B) **Tokens safe** — hash one-way, attackers can't reverse to raw tokens. URLs require raw token. (Other DB tables may still be compromised separately.)
C) Risk only for users who reset in last 15 min
D) Need to revoke all tokens immediately as precaution

**Your Answer**: ___

---

### Q31. ⭐⭐⭐ 10x traffic spike on `/forgot-password`. Current setup: sync SMTP send (3s/email). What fails first?
A) DB connection pool
B) **API timeouts** — sync 3s wait + 10x concurrent = thread pool exhausted → 504 Gateway Timeout
C) Disk space
D) Frontend cache

**Your Answer**: ___

---

### Q32. ⭐⭐⭐ User screenshots reset email and shares accidentally. Token still valid. Saved by:
A) HMAC signature
B) **Single-use enforcement** (atomic UPDATE WHERE used=0) — first click wins, screenshot link becomes useless after legitimate user uses it
C) Rate limiting
D) Browser security

**Your Answer**: ___

---

### Q33. ⭐⭐ User reports "Reset link expired or invalid" error every time. They request fresh link each time. Likely cause?
A) Bug in expiry calculation
B) **User clicks old link first** (current logic: requesting new link invalidates old ones; if old link in inbox is clicked, it's already marked used by the new-link request)
C) DB down
D) JavaScript disabled

**Your Answer**: ___

---

### Q34. ⭐⭐⭐ After password reset, user reports "old browser tab still logged in". Setup uses JWT (15-min expiry). What's needed?
A) Cookie clearing
B) **Session invalidation strategy**: increment user's `tokenVersion` claim → all old JWTs invalid (or maintain Redis blocklist of revoked tokens)
C) Browser refresh
D) Wait 15 min for natural expiry — acceptable trade-off

**Your Answer**: ___

---

### Q35. ⭐⭐⭐ 50K password resets/hour at peak. SMTP can do 14/sec. Sync architecture analysis:
A) Fine — 50K/3600 = ~14/sec average
B) **Sync architecture risky** — peak burst > 14/sec, no headroom. Async via queue mandatory + 20 worker pool for resilience.
C) Use bigger SMTP
D) Reject excess users

**Your Answer**: ___

---

### Q36. ⭐⭐ Frontend dev push back: "Different message for 'email not found' is better UX." Your response?
A) Agree — UX matters most
B) **Disagree — anti-enumeration trumps UX**. Identical response prevents attackers from harvesting valid emails. Industry standard (GitHub, Stripe).
C) Use CAPTCHA instead
D) Add delay only

**Your Answer**: ___

---

### Q37. ⭐⭐⭐ SMTP provider goes down for 2 hours. Async architecture with DLQ. User experience?
A) Catastrophic — 2 hours of users locked out
B) **Graceful**: API still returns 200 fast; emails queue up; when SMTP recovers, backlog processes; failed-after-3-retries → DLQ for ops. Users see "link sent" — actual delivery delayed but successful.
C) Auto-fallback to SMS
D) Refund users

**Your Answer**: ___

---

### Q38. ⭐⭐ New requirement: add WhatsApp as notification channel. Polymorphic NotificationSender design — what changes?
A) Modify existing `PasswordResetService` to add WhatsApp logic
B) **Add one new class** `WhatsAppNotificationSender implements NotificationSender`, mark `@Component`, register in DI — zero existing code change (Open/Closed achieved)
C) Rewrite Email + SMS senders too
D) Fork the entire notification module

**Your Answer**: ___

---

## 📚 Section E: Concept Mastery (12 Qs)

### Q39. ⭐ CSPRNG ka full form?
A) Computer-Standard PRNG
B) Cryptographically Secure Pseudo-Random Number Generator
C) Common Safe PRNG
D) Cycle-Synced PRNG

**Your Answer**: ___

---

### Q40. ⭐⭐ `java.util.Random` security ke liye kyun acceptable nahi?
A) Slow
B) Uses Linear Congruential Generator — predictable after observing few outputs; attacker can guess future outputs
C) Returns negative numbers
D) Not thread-safe

**Your Answer**: ___

---

### Q41. ⭐ SHA-256 ki avalanche property:
A) Hash gets faster over time
B) Output appears to "avalanche" if too many inputs
C) 1-bit input change → ~50% bits change in output (no predictable pattern)
D) Outputs increase monotonically

**Your Answer**: ___

---

### Q42. ⭐⭐ Passwords ke liye BCrypt over SHA-256 kyun?
A) BCrypt is encrypted, SHA-256 is hashed
B) BCrypt deliberately slow (100-200ms) + built-in salt + cost factor — defeats brute force on low-entropy human passwords
C) SHA-256 deprecated
D) BCrypt returns shorter strings

**Your Answer**: ___

---

### Q43. ⭐⭐ Salt ka purpose hashing mein:
A) Encryption strength
B) Random bytes per user → same password → different hashes → rainbow tables useless
C) Improve hash speed
D) User identification

**Your Answer**: ___

---

### Q44. ⭐⭐⭐ Spring `@Transactional` works via:
A) Bytecode rewriting at compile time
B) Runtime-generated proxy class (CGLib/JDK proxy) that wraps method with begin/commit/rollback
C) Database trigger
D) Annotation processor

**Your Answer**: ___

---

### Q45. ⭐⭐ `@Version` field is auto-managed by:
A) Developer manually increments
B) Hibernate auto-increments on save + auto-adds `WHERE version=N` to UPDATE
C) SQL database trigger
D) Spring proxy

**Your Answer**: ___

---

### Q46. ⭐ ACID ka "D":
A) Distinct
B) Distributed
C) Durability (committed data survives crashes via WAL)
D) Direct

**Your Answer**: ___

---

### Q47. ⭐⭐⭐ Write-Ahead Log (WAL) kaise crash recovery enable karta hai:
A) Periodic snapshots
B) Changes pehle sequential log file mein commit (durable on disk before COMMIT ack); restart pe log replay = consistent state restore
C) RAID mirroring
D) Slave replication

**Your Answer**: ___

---

### Q48. ⭐⭐ Polymorphism = "many shapes" — runtime decision mechanism:
A) Compile-time type binding
B) **Vtable** — class's array of function pointers; object holds class pointer; JVM looks up actual method at runtime
C) Reflection only
D) Annotation processing

**Your Answer**: ___

---

### Q49. ⭐⭐ Method overloading vs overriding:
A) Both compile-time
B) Overloading: compile-time (compiler picks by parameter types). Overriding: runtime (JVM picks by actual object type via vtable)
C) Both runtime
D) Overloading needs inheritance

**Your Answer**: ___

---

### Q50. ⭐⭐⭐ Strategy Pattern + DI + Polymorphism — combined benefit:
A) Faster execution
B) **Open/Closed Principle** achieved — new strategy = new class, zero modification to existing code; DI container auto-wires
C) Lower memory
D) Simpler code

**Your Answer**: ___

---

## 🔒 Answer Key (SCROLL ONLY WHEN DONE!)

<details>
<summary>⚠️ Click to reveal — complete all 50 first!</summary>

### Section A: Code Output Prediction
1. **B (43)** — 32 bytes Base64 URL-encoded without padding = 43 chars (32 × 4/3 = 42.67 → 43 with no padding)
2. **B** — BCrypt embeds random salt in hash, so `h1 ≠ h2` but `matches()` extracts salt and verifies both
3. **B** — SHA-256 deterministic (same input → same hash); avalanche means "Hello" produces totally different hash than "hello"
4. **B** — Self-invocation bypasses Spring's proxy; `this.innerMethod()` is direct call, not through proxy, so `@Transactional` annotation ignored
5. **B** — Hibernate auto-adds `WHERE version=3` and increments to 4
6. **B** — Polymorphism: actual object is Puppy regardless of declared reference type; Puppy.sound() runs in all 3 cases
7. **B** — Java 5+ allows covariant return types (Integer is subtype of Number)
8. **B** — Filter odd → 1,3,5; map ×10 → 10,30,50; take(2) → 10,30
9. **B** — Cold Observable: each subscribe triggers separate execution → 3 HTTP calls
10. **D** — debounceTime cancels previous timer on each emit; only emits after 300ms idle; first 3 keystrokes cancel each other; final 'd' at 600ms, idle reaches 900ms

### Section B: Bug Spotting
11. **B** — Classic TOCTOU: SELECT check + separate UPDATE = race condition. Fix: atomic `UPDATE WHERE used=0`
12. **B** — Without `@Version`, concurrent transfers can lost-update each other's balance changes
13. **C** — Parent's constructor runs first, calls overridable `init()`, JVM dispatches to Child's override, but Child's `name` field not yet initialized → NPE on `name.toUpperCase()`
14. **B** — Contract: `equals` true must imply same `hashCode`. Without overriding `hashCode()`, default uses memory address → different buckets → HashSet.contains() can't find equal objects
15. **B** — Raw token in DB = breach disaster. Should store SHA-256 hash only.
16. **B** — User enumeration: 404 vs 401 reveals whether email is registered → attacker builds list for credential stuffing
17. **B** — Open/Closed Principle violated; new channel modifies existing service
18. **B** — Without `@Override`, typo creates new unrelated method; interface contract `send()` unimplemented; subtle production bug

### Section C: Dry-Run Tracing
19. **B** — Optimistic locking: A's UPDATE matches version=1, succeeds, version becomes 2. B's UPDATE WHERE version=1 finds version=2 now, 0 rows affected, exception
20. **B** — Atomic UPDATE: A acquires U-Lock → converts to X-Lock → updates → commits → releases. B waits, then sees used=1, WHERE fails, rowcount=0
21. **B** — READ COMMITTED only shows committed data; T1's uncommitted change invisible to T2
22. **C** — interval emits 0,1,2,3,... at 100ms intervals. filter `n>0` → 1,2,3,... map n² → 1,4,9,... take(3) → 1,4,9
23. **B** — Zone.js monkey-patches setTimeout; callback completes → Zone notifies Angular → change detection runs → UI re-renders
24. **B** — Kafka consumer resumes from last committed offset; offsets 1 and 2 committed; next read = offset 3 (m3)
25. **C** — At 500ms, bucket=0 → request rejected. At 1500ms, ~1 second refill = 1 token available → request accepted
26. **B** — C has no `m()` override, so inherits from B; C's vtable[m] points to B.m()
27. **B** — Leftmost-prefix rule: index sorted by user_id first; Q1, Q3, Q5 use `user_id` first column. Q4 starts at expires_at = index can't be used; Q5 uses user_id range, then expires_at within
28. **B** — JIT detects monomorphic call site → can devirtualize, replacing vtable lookup with direct call (huge speedup in hot paths)

### Section D: Scenarios
29. **B** — Most actionable cause; check Kafka consumer status, topic existence, DLQ for stuck messages
30. **B** — SHA-256 one-way; raw tokens needed for URL exploitation; hashes alone useless. (Other DB damage separately concerning)
31. **B** — Sync request waits 3s; 10x concurrent + slow tasks = thread pool exhausted = timeouts
32. **B** — Single-use atomic flag: first click consumes token; subsequent clicks (including screenshot scenario) fail
33. **B** — Race: requesting new token invalidates old; user has old link in inbox; clicks it first; system says "already used"
34. **B** — JWT can't be revoked directly; token versioning or Redis blocklist required for instant invalidation
35. **B** — Peak burst > steady rate; sync architecture leaves no headroom; async + worker pool provides elasticity
36. **B** — Enumeration enables credential stuffing → mass account takeover. Identical response is industry standard.
37. **B** — Async + DLQ provides resilience; users see success; backlog drains when SMTP recovers
38. **B** — Polymorphism + DI + Strategy = textbook Open/Closed; new class auto-registers, zero existing code modification

### Section E: Concept Mastery
39. **B** — Cryptographically Secure Pseudo-Random Number Generator
40. **B** — LCG is deterministic math formula; attacker can derive seed from outputs
41. **C** — 1-bit input change → ~50% output bits change → unpredictable output relationship
42. **B** — Slow (cost factor) + auto-salt; defeats brute force; SHA-256 too fast for low-entropy passwords
43. **B** — Random bytes per user; same password produces different hashes; defeats rainbow tables
44. **B** — CGLib generates subclass at runtime; method calls go through proxy that wraps with transaction logic
45. **B** — JPA auto-manages via reflection; you only declare `@Version`, framework handles increment + WHERE
46. **C** — Durability: committed transactions persist through crashes via WAL
47. **B** — Sequential log write before data files; log → recoverable; restart replays committed-but-not-flushed transactions
48. **B** — Vtable lookup at runtime; object's class pointer → vtable → method address
49. **B** — Overloading = compile-time (parameter types); Overriding = runtime (actual object type via vtable)
50. **B** — Open/Closed: extend behavior via new class, zero modification of existing code; DI container wires automatically

</details>

---

## 📊 Scoring Self-Assessment

| Score | Level | Action |
|-------|-------|--------|
| 45-50 | ⭐⭐⭐ Master | Interview ready for this topic |
| 38-44 | ⭐⭐ Solid | Review wrong answers, retake in 1 week |
| 28-37 | ⭐ Developing | Re-read lesson + revision file, retake in 3 days |
| <28 | 🔴 Foundational | Re-study entire lesson, focus weak sections |

**Difficulty distribution**:
- ⭐ (Easy): 8 questions
- ⭐⭐ (Medium): 26 questions
- ⭐⭐⭐ (Hard): 16 questions

**Stack distribution**:
- Java/Spring: 14
- .NET/C#: 0 (Day 4 had less .NET-unique content)
- SQL: 10
- Angular: 6
- System Design: 10
- OOP: 10

---

## 📤 Submission

Just commit + push this file with your answers. Mentor's next routine will auto-evaluate and append `## 📊 Evaluation — <date>` section here.

**No need to paste answers in chat.**
