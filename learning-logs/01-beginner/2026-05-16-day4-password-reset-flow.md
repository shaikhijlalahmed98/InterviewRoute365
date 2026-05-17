# 🎯 🟢 Day 4 of Beginner (Level 1 of 7): Password Reset Flow (Secure Token-Based)

**Overall Day**: Day 4 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 4 of 20 in this level
**Today's Theme**: Secure password reset using one-time tokens — aur Polymorphism (method overriding) ka use jisse notification system pluggable banta hai.

---

## 📖 The Brainer Scenario (Real Problem) — Time Pe Step By Step

Bhai, soch ek situation. **Daraz** pe Ahmed bhai ka account hai. Kal raat tak login ho raha tha. Aaj subah password bhool gaya. Ab reset karna hai.

Yeh "Forgot Password" feature dikhne mein simple lagta hai — 2 textbox aur 1 button. Lekin **andar 11 cheezein ho rahi hain**. Agar koi bhi 1 cheez galat ho gayi, toh security breach ho jata hai.

**Pura flow step-by-step (clock ke saath ke kab kya hua):**

```
T+0ms     Ahmed Daraz pe "Forgot Password" link click karta hai
T+1ms     Browser ek form dikhata hai: "Apna email daalein"
T+2000ms  Ahmed "ahmed@gmail.com" type karta hai, Submit click karta hai
T+2010ms  Browser HTTP POST request bhejta hai Daraz backend ko
T+2015ms  Backend check karta hai: "Is IP se 1 ghante mein 3 se zyada requests aayi hain?"
T+2020ms  Backend check karta hai: "Kya 'ahmed@gmail.com' users table mein hai?"
T+2025ms  HAAN exist karta hai → ek secret token banao (32 random bytes)
T+2026ms  Token ka HASH (fingerprint) banao, HASH ko DB mein save karo (raw token nahi)
T+2030ms  Kafka mein event publish karo: "User X ko reset link bhejo"
T+2040ms  Browser ko HTTP 200 return karo (FAST — email ka wait mat karo)
T+2045ms  Browser show karta hai: "Agar email registered hai, link bhej diya"
T+5000ms  (Background mein) Email service Kafka event uthata hai
T+7000ms  (Background mein) Email Ahmed ke inbox mein pohnch jata hai with RAW token URL mein
T+30000ms Ahmed email mein link click karta hai
T+30050ms Browser open karta hai: daraz.pk/reset?token=abc123xyz...
T+45000ms Ahmed naya password type karta hai, Submit click karta hai
T+45050ms Backend received token ko hash karta hai, DB mein dekhta hai
T+45055ms Mil gaya + expired nahi + used nahi → atomically used=true mark karo
T+45060ms User ka password_hash naye BCrypt hash se update karo
T+45065ms Ahmed ke saare active sessions DELETE karo (har device pe logout)
T+45070ms Success response → Browser login page pe redirect ho jata hai
```

Iss 11-second drama ke andar **11 critical decisions** chupe hain. Aaj hum har ek ka "kyun" depth mein samjhenge.

---

### The 4 Attacks Hackers Use (Gotchas Explained With Stories)

#### 🪤 Attack 1: Timing Attack (Stopwatch Se Enumeration)

**"Enumeration" ka matlab kya hai pehle samjho**: "Enumeration" = list bana lena. **User enumeration attack** matlab — hacker yeh figure out kar leta hai ke konse emails Daraz pe registered hain (matlab valid accounts). Phir woh saare emails ko target karta hai password guessing ya phishing ke liye.

**Attack kaise hota hai**:

Hacker ek script likhta hai jo 10,000 random emails Daraz ke "Forgot Password" endpoint pe bhejta hai aur **response time stopwatch se measure** karta hai.

```
Test email: random1234567@gmail.com (fake)
   Server time: 50ms (kyunki bas DB mein dhundha, nahi mila, return)

Test email: ahmed@gmail.com (real)
   Server time: 250ms (DB lookup + token generate + hash + DB insert + Kafka publish)
```

Difference: **200ms**. Hacker ke script ko clear pata chal gaya — `ahmed@gmail.com` real account hai.

**Yeh kyun matter karta hai**:

Hacker ab 10,000 emails mein se shayad 500 confirmed real emails identify kar leta hai. Phir wo karta hai **credential stuffing attack** — "credential stuffing" matlab leaked passwords ki list (jo 2014-2024 ke breaches mein leak hui) se same emails Daraz, Facebook, banking sites pe try karta hai. Bohot users **same password** har jagah use karte hain. Result: massive account takeover.

**Fix**: Dono code paths (email exists / nahi exists) ko **same time** lena chahiye. Ya kam-se-kam obvious leaks eliminate karo (same response message, same HTTP status code, similar response size).

---

#### 🪤 Attack 2: Plaintext Token Storage (DB Leak Ka Nightmare)

**"Plaintext" ka matlab**: Bina encryption ya hashing ke, original readable form mein. Jaise tum likhte ho "MyPassword123" aur woh exactly aise hi store ho jata hai DB mein — to woh plaintext storage hai.

**Attack scenario**:

Daraz ka database breach ho jata hai. Common causes:
- **SQL injection** — hacker website ke search box mein special SQL command daal deta hai jo database access karwati hai
- **Insider leak** — koi unhappy employee database export kar ke bech deta hai
- **Backup theft** — backup file kisi insecure server pe rakhi thi, hacker ne download kar liya

Agar tokens **plaintext** mein store the:

```
DB row:
  token: "abc123xyz789..."     ← Hacker ko mil gaya raw token
  user_id: 5
  expires_at: 2026-05-16 12:30:00
  used: false
```

Hacker turant URL banata hai: `https://daraz.pk/reset?token=abc123xyz789...`
URL open karta hai, naya password set kar deta hai. **15 minutes ke andar har active user ka account kha gaya**.

**Yeh kyun matter karta hai**:

Tokens basically temporary passwords hote hain. Inko bhi same care chahiye jo passwords ko chahiye. Industry rule: **never store secrets in plaintext, ever, anywhere**.

**Fix**: **Hashed storage** — token ka SHA-256 hash store karo, raw token sirf email mein bhejo. Hashing ka detailed concept aage Stack 1 mein aayega — abhi yaad rakho ke hash ek **one-way fingerprint** hota hai — fingerprint se asli token wapas nahi nikal sakti.

---

#### 🪤 Attack 3: Double-Click Race Condition (TOCTOU Bug)

**"TOCTOU" ka matlab** — yeh ek classic concurrency bug ka acronym hai:

- **T** — Time
- **O** — Of
- **C** — Check
- **T** — Time
- **O** — Of
- **U** — Use

**Asaan tareeqe se samjho**: Tum ek condition **check** karte ho (e.g., "kya token unused hai?"), phir **use** karte ho (mark as used). **Check aur use ke beech mein** koi aur process state change kar deta hai. Iss "beech ke time" mein bug hota hai.

**Real example — bank withdrawal**:

```
Tumhare account mein 1000 hain.
Tumhara code:
   balance = SELECT balance FROM accounts WHERE id=X;  -- returns 1000
   if (balance >= 500):
       UPDATE accounts SET balance = balance - 500;
       cash de do 500
```

Tum aur tumhari biwi same waqt withdraw karte ho 500-500 each:

```
You (T+0ms):    SELECT balance → 1000  ✓ (>= 500, OK)
Wife (T+1ms):   SELECT balance → 1000  ✓ (>= 500, OK)
You (T+2ms):    UPDATE balance = 1000 - 500 = 500
Wife (T+3ms):   UPDATE balance = 1000 - 500 = 500
You (T+4ms):    Cash 500 nikal gaya
Wife (T+5ms):   Cash 500 nikal gaya
Final balance: 500
Total withdrawn: 1000
Bank lost: 500 ← BUG!
```

Yeh hua TOCTOU. Check (balance >= 500) aur use (balance update) ke beech main state change ho gayi.

**Password reset mein same problem**:

Ahmed ne email phone aur laptop dono pe open kar di. Galti se dono pe simultaneously link click kar diya:

```
Phone request (T+0ms):    SELECT used FROM tokens WHERE hash=X → used=false ✓
Laptop request (T+1ms):   SELECT used FROM tokens WHERE hash=X → used=false ✓
Phone (T+2ms):            UPDATE tokens SET used=true
Laptop (T+3ms):           UPDATE tokens SET used=true
Phone (T+4ms):            naya password set ✓
Laptop (T+5ms):           naya password set bhi ✓ ← Double consumption!
```

**Yeh kyun matter karta hai**:

Real attack scenario: hacker ne Ahmed ka email compromise kar liya, link nikal liya. Ahmed bhi same waqt try kar raha hai. **Dono successfully reset kar dete hain**. Hacker ka set kiya hua password latest hai, Ahmed lock out ho jata hai.

**Fix**: **Atomic** check-and-update — single SQL statement mein dono kaam karwao. SQL engine row pe lock leta hai, sirf ONE update success hota hai, doosra rowcount=0 return karta hai. Detailed explanation Stack 3 (SQL) mein aayega.

---

#### 🪤 Attack 4: Purane Sessions Reset Ke Baad Bhi Zinda

**"Session" ka matlab kya hota hai pehle samjho**:

Jab tum login karte ho kisi website pe, server tumhe ek "session token" deta hai (cookie ya JWT). Yeh basically ek "ticket" hai jo prove karta hai "yeh user already login ho chuka hai". Har subsequent request mein browser yeh ticket bhejta hai, server check karta hai, access deta hai.

Sessions ke 2 types:
- **Server-side sessions**: Server DB mein record rakhta hai "session abc123 belongs to Ahmed". Validation = DB lookup.
- **JWT (JSON Web Token)**: Token khud apne andar info contain karta hai (cryptographically signed). Server DB lookup nahi karta — bas signature verify karta hai. Aage detail aayegi.

**Attack scenario**:

Hacker ne Ahmed ka password 2 din pehle steal kar liya tha (phishing ya keylogger se). Login kar liya tha, **active session JWT** abhi bhi valid hai (24-hour expiry).

Ahmed kuch suspicious activity dekhta hai — "kisi ne mera profile picture badla?!" Turant password reset karta hai.

**Lekin hacker ka session JWT abhi bhi 22 ghante valid hai!** Hacker still has full access. Reset useless ho gaya.

**Yeh kyun matter karta hai**:

Password reset ka **whole point** unauthorized access kaatna hai. Agar purane sessions zinda hain, reset sirf ek illusion hai — security theater.

**Fix**: Password change pe **saare active sessions** invalidate karo. 4 different techniques hain (server sessions, refresh tokens, token versioning, Redis blocklist) — detail Stack 5 + Senior Question mein aayegi.

---

### Why This Matters In Production (Real Companies)

- **Stripe** (payments): 1-hour token expiry. Tokens **HMAC-signed** — HMAC = Hash-based Message Authentication Code = hash + secret key se signature banta hai jo prove karta hai "yes I generated this". Server signature verify karke validate karta hai bina DB lookup ke (stateless).

- **GitHub**: Same response chahe email registered ho ya na — explicitly mentioned in their security docs as enumeration prevention. 3-hour expiry.

- **Google**: Resets invalidate **saare OAuth refresh tokens + saari browser sessions globally**. Industry mein strongest implementations mein se ek.

- **AWS Cognito**: Long URL ki bajaye 6-digit OTP code. Hashed storage. Pehli galat attempt pe token auto-mark used (brute force prevention).

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Goal of this section**: Pehle 5 foundational concepts samjho. Yeh foundation hai — woh hi banaye ge tumhe code samajhne ke kaabil.

### Foundation Concept 1: Random Numbers Ka Pura Drama

#### Step 1.1: Pehle yeh samjho ke computer "random" bana hi nahi sakta

Computer fundamentally **deterministic machine** hai. "Deterministic" ka matlab — same input do, same output milega, **hamesha**. Yeh **predictable** hai by design (warna programs unreliable ho jate).

To "random number" kaise banayega computer? Asli random impossible hai mathematically. Solution: **pseudo-random** ("pseudo" = naqli/fake) numbers banao — kuch formula se number generate karo jo **human ko random lage** but reality mein formula-driven hai.

#### Step 1.2: Java mein 2 random classes — `Random` vs `SecureRandom`

🟥 **`Random` class** — kaise kaam karti hai:

Yeh ek simple math formula use karti hai jisko bolte hain **Linear Congruential Generator (LCG)**:

```
next_number = (current_number × 1103515245 + 12345) % 2147483648
```

Bas. Itna hi. Ek multiplication, ek addition, ek modulo. Yeh 1960s ka formula hai.

**Iska problem**: Agar tum mujhe 3-4 outputs do, main reverse-engineer kar sakta hoon formula ko, aur **saare aage ke numbers predict** kar sakta hoon.

```java
Random r = new Random(42);  // "42" is the "seed" (starting value)
r.nextInt();  // -1170105035
r.nextInt();  // 234785527
// Agar hacker yeh dono numbers dekhe → woh next nikal sakta hai!
```

**Use case**: Sirf games, dice rolls, simulations, non-security stuff ke liye. **Security ke liye kabhi use mat karo**.

🟩 **`SecureRandom` class** — yeh different tareeqe se kaam karti hai:

Yeh apna formula nahi chalati. Yeh **Operating System se "real randomness" maangti hai**.

#### Step 1.3: "Real randomness" OS ke paas kahan se aati hai? — Yeh hai **Entropy**

**"Entropy" ka simple matlab**: "physical world ki choti choti chaos jo predict nahi ho sakti."

Tumhara OS (Linux ya Windows) background mein continuously yeh cheezein **measure** karta rehta hai:

- ⌨️ **Keyboard timings** — tum kab key dabate ho, exact microsecond (1 second ka 10 lakh-vaan part) tak
- 🖱️ **Mouse movements** — exact path aur speed har milli-second mein
- 💾 **Disk read latencies** — hard disk se data padhne mein kitne nanoseconds lage (electrical noise affect karta hai, har baar thoda alag)
- 🌐 **Network packet arrival times** — kab packet aaya
- 🌡️ **CPU temperature fluctuations** — tiny variations
- 🔌 **Interrupt timings** — hardware interrupts ka exact moment

Yeh sab cheezein **physically unpredictable** hain. Tum khud nahi predict kar sakte ke tumhari agli keystroke kab hogi millisecond-precise. Hacker to bilkul nahi.

OS in saari "chaos values" ko ek container mein collect karta hai. Iss container ko bolte hain **"entropy pool"** — basically random bits ka thaila jo OS slowly bharta rehta hai background mein.

#### Step 1.4: `/dev/urandom` kya hai? — Entropy Pool Ka "Tap"

Linux mein **`/dev/urandom`** ek special "file" hai. Lekin yeh **asal mein disk pe file nahi hai**.

Ek concept samjho: Linux ka philosophy hai **"everything is a file"** — kernel apne saare interfaces ko file ki shakal mein expose karta hai, taake same simple read/write commands sab pe kaam karen.

`/dev/urandom` is concept ka example hai — yeh **kernel ka interface** hai jo file ki shakal mein dikhta hai. Jab koi program is "file" ko **read** karta hai:

1. Linux kernel entropy pool se kuch random bits uthata hai
2. Unko **cryptographic hash function** se mix karta hai (taake aur secure ho jaye — agar pool kuch predictable bhi ho, hash unpredictable bana deta hai)
3. Caller ko random bytes return karta hai

`/dev/urandom` se bytes maange = OS ke physical chaos pool se randomness mile = **truly unpredictable**.

#### Step 1.5: "Wrap karta hai" ka matlab

**"Wrap" ka matlab software mein**: ek class/function jo doosri cheez ke upar ek **layer add karke usko use-karne-mein-asaan banata hai**. English mein "wrapper" — kuch chhupa ke chhota interface dene wala.

Tumhe `/dev/urandom` se bytes chahiye Java code mein. **Without wrapper**, tumhe yeh karna parta:

```java
// AGAR WRAPPER NA HOTA — manual file I/O karna parta
FileInputStream fis = new FileInputStream("/dev/urandom");
byte[] bytes = new byte[32];
fis.read(bytes);
fis.close();
// Aur agar Windows pe code chala? Windows mein /dev/urandom hi nahi hai!
// Windows ke liye alag API hai (CryptGenRandom)
// Mac ke liye alag (SecRandomCopyBytes)
// Tumhe 3 versions likhne parte cross-platform support ke liye.
```

Bohot kaam. OS-specific code. Error-prone.

**Java ki `java.security.SecureRandom` class yeh sab tumse hide karti hai** — yeh hai "wrapper". Tum sirf likhte ho:

```java
SecureRandom random = new SecureRandom();
byte[] bytes = new byte[32];
random.nextBytes(bytes);  // Done!
```

Andar **`SecureRandom` khud decide karti hai**:
- Linux pe ho? → `/dev/urandom` se padh lo
- Windows pe ho? → CryptGenRandom API call karo
- Mac pe ho? → SecRandomCopyBytes call karo

Tumhe OS-specific complexity ka pata nahi chalta. **Yeh hi "wrap karna" hai** — complexity hide karna behind a simple interface.

**Roman Urdu analogy**: Tum Careem book karte ho. App tumhe ek simple button deta hai "Book Ride". Andar app: Google Maps API call kar raha, driver matching algorithm chala raha, payment gateway integrate kar raha, GPS track kar raha, real-time ETA calculate kar raha. Yeh sab tumhe nahi dikhta — app ne "wrap" kar diya hai pura complexity ko ek button ke pichhe.

#### Step 1.6: Token Banana — 32 Bytes Kyun?

```java
SecureRandom random = new SecureRandom();
byte[] tokenBytes = new byte[32];  // ← yahi line
random.nextBytes(tokenBytes);
```

32 bytes kyun? 32 bytes = **256 bits** = 2^256 possible values.

**Yeh number kitna bara hai? Concretely samjho**:

- 2^256 = ~10^77 (10 ke saath 77 zero)
- Pure universe mein atoms ki estimated count = ~10^80
- Sun ki age = ~5 billion years = ~10^17 seconds
- Modern fastest supercomputer = ~10^18 operations/second

Agar koi supercomputer pure universe ka age tak guess karta rahe non-stop, woh sirf **10^17 × 10^18 = 10^35** guesses kar sakta hai. Yeh 2^256 ka **microscopic fraction** hai.

**Conclusion**: 32-byte token brute force karna **mathematically impossible** hai. Yeh hi reason hai 32 bytes industry standard hai security tokens ke liye.

---

### Foundation Concept 2: Hash Functions Ki Magic (SHA-256)

#### Step 2.1: "Hash function" kya hai? — Asaan Analogy

Imagine kar ek **magic shredder machine**. Tum koi bhi document daalo (1 page ya 1000 pages), machine ek **fixed-size unique receipt** banake deti hai — bas 32 characters ka. Receipt unique hai (har different document ki different receipt), aur **receipt se asli document banana impossible** hai.

Yeh hai hash function. Computer science mein:

```
Input:  "Hello World"           (any length)
Output: "a591a6d40bf420404a..." (fixed length, 64 hex chars for SHA-256)

Input:  pura "Quran-e-Pak" PDF (millions of bytes)
Output: "7d8a4b9f2e5c1a..."    (still 64 hex chars)
```

Output ki length hamesha **same** — chahe input 1 byte ho ya 1 GB.

#### Step 2.2: Hash Function Ki 3 Critical Properties

**Property 1: Deterministic (predictable consistency)**

Same input → **always same hash**, har machine pe, har time pe.

```
SHA-256("hello") = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
   Tum bhi run karo, mein bhi karoon, 100 saal baad bhi — same output.
```

**Property 2: One-way / Pre-image resistant**

Hash dekh ke original input **wapas nahi nikal sakte**. Mathematically computationally impossible (would take longer than age of universe).

```
"2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
   ↑ Iss hash ko dekh ke kabhi nahi pata chalega ke input "hello" tha.
```

Yeh "one-way" property hi security ka backbone hai.

**Property 3: Avalanche effect**

Input mein **1 bit** change → output mein **~50% bits** change. Output unpredictably different ho jata hai.

```
SHA-256("hello")  = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
SHA-256("hello!") = ce06092fb948d9ffac7d1a376e404b26b7575bcc11ee05a4615fef4fec3a308b
                    ↑↑↑↑↑↑↑↑ bilkul alag, koi pattern nahi
```

Tum hash dekh ke andaaz nahi laga sakte ke input mein chhoti change thi ya badi.

#### Step 2.3: Yeh Properties Tokens Pe Kaise Apply Hoti Hain?

Hum ne kaha "raw token email mein bhejo, hash DB mein store karo." Ab clear hoga **kyun**:

```
T+0ms:   SecureRandom 32 bytes generate karta hai → "abc123xyz..."
T+1ms:   Hum SHA-256(raw token) calculate karte hain → "9f86d081884c..."
T+2ms:   DB mein store karte hain: hash = "9f86d081884c..."
T+3ms:   Email Ahmed ko jata hai with raw token = "abc123xyz..." in URL
```

**Ab agar DB breach ho gaya**: Hacker ko dikhta hai sirf hash. Hash se raw token wapas nahi nikal sakta (one-way property). URL banane ke liye raw token chahiye — hacker ke paas nahi hai. **Tokens bach gaye.**

**Ab agar Ahmed link click karta hai**: Browser raw token send karta hai → server SHA-256 hash calculate karta hai → DB mein stored hash se compare karta hai → match? Token valid. Yeh hi reason hai deterministic property zaroori hai — same raw token always same hash banayega.

#### Step 2.4: SHA-256 vs BCrypt — Tokens Mein SHA-256, Passwords Mein BCrypt Kyun?

Yeh confusing point hai jo bohot logon ko ulta padta hai. Carefully samjho.

**SHA-256** — fast. 1 microsecond mein hash banata hai modern CPU pe. Designed for **integrity checking** (verify ke file change nahi hui), digital signatures, etc.

**BCrypt** — slow on purpose. 100-200 milliseconds (yes, milliseconds, not microseconds) per hash. **100,000x slower than SHA-256.**

**Kyun? Aur kiske liye konsa?**

**For passwords (BCrypt use karo)**:

Human passwords low-entropy hote hain. "Entropy" yahan matlab "unpredictability". Insaan likhte hain "Password123", "Ahmed@2024", "Karachi786" — **predictable patterns**. Hacker ke paas "rainbow tables" hoti hain — pre-computed hashes of millions of common passwords.

Agar tumne SHA-256 use kiya passwords ke liye:
- Hacker DB breach mein hashes le jata hai
- Apni rainbow table mein lookup karta hai
- Common passwords (jo 80% users use karte hain) **seconds mein crack**
- 1 second mein 1 billion guesses possible (GPU-accelerated)

BCrypt ko **deliberately slow** banaya gaya hai. 1 hash check = 200ms = 5 guesses/second per CPU core. 1 billion guesses karne mein **6+ years**. Hacker give up kar deta hai.

**For tokens (SHA-256 use karo)**:

Tokens **high-entropy** hote hain — 32 random bytes from CSPRNG. 2^256 possibilities. Brute force impossible regardless of hash speed.

To hash speed matter nahi karta security ke liye. **Lekin tum tokens validate karte ho frequently** (har email click pe). Agar BCrypt use karoge to har validation 200ms slow — user UX kharab + server pe load.

**Rule**:
- Low-entropy input (passwords) → **slow hash** (BCrypt, Argon2, scrypt)
- High-entropy input (random tokens) → **fast hash** (SHA-256, SHA-512)

#### Step 2.5: "Salt" Kya Hota Hai Aur Kyun?

Bonus concept jo passwords ke liye critical hai.

**Problem without salt**:

Imagine 100 users ne password "password123" use kiya (sad but reality). SHA-256("password123") sab ke liye **same hash** banayega.

```
DB:
  ahmed   → hash_X
  ali     → hash_X
  fatima  → hash_X
  ...100 more users → hash_X
```

Hacker DB breach mein dekhta hai — 100 users ka same hash. Ek crack karo, **100 crack**.

**"Salt" ka solution**:

Password ke saath ek **random string** (salt) add karo before hashing. Har user ka different random salt.

```
ahmed:   salt="rxQz81" → BCrypt("password123" + "rxQz81") → hash_A
ali:     salt="pL7m2k" → BCrypt("password123" + "pL7m2k") → hash_B (totally different!)
fatima:  salt="9nVy3w" → BCrypt("password123" + "9nVy3w") → hash_C
```

Ab DB mein **same password ka 100 different hashes**. Rainbow tables useless. Hacker ko har account separately attack karna parega.

**BCrypt automatically salt handle karta hai** — `encode()` method internally salt generate karta hai aur hash mein embed kar deta hai. Stored format: `$2a$10$saltsaltsalt$hashhashhash`.

Tum manually salt manage nahi karte BCrypt use karte time — yeh hi reason hai BCrypt convenient hai vs raw SHA-256.

---

### Foundation Concept 3: Spring Ka @Transactional — "Transaction" Kya Hai?

Yahan tum likhoge `@Transactional` annotation method pe, lekin pehle samjho yeh kar kya raha hai.

#### Step 3.1: Pehle "Transaction" Concept Database Mein Samjho

**Transaction** = ek **group of database operations** jinhe **ek single unit** ki tarah treat karna hai. Yaa **sab successful**, yaa **sab fail** (rollback).

**Classic example — bank transfer**:

Tum Ali ko 500 transfer karte ho. 2 operations:

```
Operation 1: UPDATE accounts SET balance = balance - 500 WHERE id = 'you';
Operation 2: UPDATE accounts SET balance = balance + 500 WHERE id = 'ali';
```

Imagine kar Operation 1 succeed ho gaya, phir power chala gaya, Operation 2 nahi hua. Result: **tumhare 500 udd gaye, Ali ko mile nahi**. Bank loss.

**Transaction yeh problem solve karti hai**:

```
BEGIN TRANSACTION;
   UPDATE accounts SET balance = balance - 500 WHERE id = 'you';
   UPDATE accounts SET balance = balance + 500 WHERE id = 'ali';
COMMIT;
-- ya phir if anything fails:
ROLLBACK;
```

DB engine guarantee karta hai: dono operations ya to **sab commit** honge atomically (one moment they're not, next moment they are), ya **dono rollback** honge (jaise kabhi hua hi nahi).

Yeh hai **"A" in ACID** = Atomicity.

#### Step 3.2: ACID Properties — Pure Family

**ACID** = 4 properties jo har serious database guarantee karta hai:

- **A — Atomicity**: Saari operations ek group mein hain. Sab success ya sab fail. Beech mein interrupt nahi.

- **C — Consistency**: Transaction ke baad database ek **valid state** mein hi hoga. Foreign keys, unique constraints, business rules — sab maintained. Agar invalid state hone wala hai, transaction reject ho jata hai.

- **I — Isolation**: 2 concurrent transactions ek doosre ki **uncommitted changes** nahi dekhte. Tumhara transaction chal raha hai, dosra bhi chal raha hai — dono ek doosre ko ignore karte hain jab tak commit nahi ho jate. Isolation ke 4 "levels" hote hain — detail Stack 3 mein.

- **D — Durability**: Ek baar commit ho gaya, **kabhi nahi khoyega**. Power chali jaye, server crash kare, RAM ud jaye — committed data disk pe persisted hai aur recovery pe wapas mil jayega. Yeh "Write-Ahead Log" se hota hai (Stack 3 mein detail).

#### Step 3.3: Spring Ka @Transactional Annotation — Yeh Kya Karta Hai?

Spring mein agar tum method pe `@Transactional` likho:

```java
@Service
public class PasswordResetService {
    @Transactional
    public void confirmReset(String rawToken, String newPassword) {
        // Step 1: token mark as used
        // Step 2: password update
        // Step 3: sessions delete
    }
}
```

**Yeh annotation Spring ko bolta hai**:

> "Bhai, jab koi is method ko call kare, **uss se pehle** automatically `BEGIN TRANSACTION` chala dena. Method successfully complete ho jaye to `COMMIT` kar dena. Agar koi exception throw ho method ke andar, automatically `ROLLBACK` kar dena."

Tumhe manually BEGIN/COMMIT/ROLLBACK nahi likhna parta. Annotation se Spring automatically transaction boundaries handle karta hai.

#### Step 3.4: Yeh Annotation Kaam Kaise Karti Hai? — "Proxy" Magic

Yeh thoda mind-blowing concept hai, lekin samjho — interviews mein bohot poocha jata hai.

**Problem**: Tumne `confirmReset()` method likhi, kahin bhi `BEGIN TRANSACTION` nahi likha. Spring kahan se interrupt karta hai?

**Solution — "Proxy Pattern"**:

Jab Spring application start hota hai, woh dekhta hai tumhari class mein `@Transactional` annotation hai. Spring **runtime pe ek nayi class dynamically generate karta hai** jo tumhari class extend karti hai. Yeh nayi class call hoti hai **"proxy"**.

```
Conceptually:
class PasswordResetService_Proxy extends PasswordResetService {
    @Override
    public void confirmReset(String rawToken, String newPassword) {
        // Spring ka magic code:
        transactionManager.begin();
        try {
            super.confirmReset(rawToken, newPassword);  // tumhari original method
            transactionManager.commit();
        } catch (Exception e) {
            transactionManager.rollback();
            throw e;
        }
    }
}
```

Jab koi `PasswordResetService` inject karta hai Spring se, **Spring asli class ki bajaye proxy inject karta hai**. Caller ko pata bhi nahi chalta — usse lagta hai woh asli class use kar raha hai.

**Method call ka actual flow**:
```
Caller code: passwordResetService.confirmReset(...)
   ↓
Actually calling: PasswordResetService_Proxy.confirmReset(...)
   ↓
Proxy starts transaction
   ↓
Proxy calls original PasswordResetService.confirmReset(...)
   ↓
Original method runs
   ↓
Proxy commits or rolls back transaction
   ↓
Caller gets response
```

Yeh hi **Spring AOP** (Aspect-Oriented Programming) ka core mechanism hai. AOP = behavior add karna code ko bina actual code modify kiye, using proxies. Spring Security ke `@PreAuthorize`, Spring Cache ke `@Cacheable`, sab same mechanism use karte hain.

---

### Foundation Concept 4: Optimistic Locking (@Version)

#### Step 4.1: "Locking" Kya Hai Database Mein?

**Problem**: Do users same data ko same waqt modify karna chahte hain. Without coordination, data corrupt ho jata hai (TOCTOU bug jo hum dekh chuke).

**Solution**: **Locking** — coordination mechanism jo ensure karta hai data race-free updates ho.

2 philosophies hain:

#### Step 4.2: Pessimistic vs Optimistic Locking

**Pessimistic Locking** ka philosophy:

> "Main maan ke chal raha hoon ke conflict hoga. Isliye pehle hi row lock karta hoon. Jab tak main kaam khatam nahi karta, koi aur touch nahi kar sakta."

```sql
BEGIN TRANSACTION;
SELECT * FROM tokens WHERE id=5 FOR UPDATE;  -- Row locked!
-- Sab kuch wait karta hai jab tak main commit/rollback nahi karta
-- Do something with row
UPDATE tokens SET used=true WHERE id=5;
COMMIT;  -- Lock released
```

- ✅ **Pros**: Conflicts impossible. Simple to reason about.
- ❌ **Cons**: Throughput kam. Doosre transactions wait karte hain. Long-held locks deadlock risk.

**Optimistic Locking** ka philosophy:

> "Main maan ke chal raha hoon ke conflict nahi hoga (kyunki actually low contention hai). Lock nahi leta. Lekin commit ke time check karunga — agar kisi aur ne meanwhile modify kar diya, main fail ho jaunga."

```sql
-- Read karte time version number bhi padho
SELECT *, version FROM tokens WHERE id=5;
-- Returns: token data + version=3

-- Update karte time version check karo
UPDATE tokens SET used=true, version=4
WHERE id=5 AND version=3;  -- ← version check!

-- Agar kisi aur ne pehle update kar diya tha:
-- DB mein version=4 ho gaya hoga
-- Tumhara UPDATE WHERE version=3 → 0 rows affected
-- Detect: conflict!
```

- ✅ **Pros**: No locking overhead. High throughput. Better for low-contention scenarios.
- ❌ **Cons**: Failed transactions retry karne parte hain agar conflict ho.

**Password reset jaisa scenario — optimistic better hai** kyunki:
- Conflicts rare (Ahmed kabhi-kabhi double-click karega, sirf 0.001% requests mein)
- Throughput chahiye (high traffic Daraz)
- Conflict detected ho to "token already used" error theek hai (just retry)

#### Step 4.3: JPA Ka @Version Annotation Kaise Kaam Karta Hai

**JPA** = Java Persistence API = standard interface jo databases se kaam karne ka. **Hibernate** is the most popular implementation of JPA.

Tum entity class mein ek field pe `@Version` annotation lagao:

```java
@Entity
public class PasswordResetToken {
    @Id
    private UUID id;

    private String tokenHash;
    private boolean used;

    @Version
    private Long version;  // ← yahi optimistic lock ka magic
}
```

**Ab kya hota hai automatically**:

Jab tum entity update karte ho aur save karte ho:

```java
PasswordResetToken token = repo.findById(tokenId);
// Suppose version = 3 at this moment
token.setUsed(true);
repo.save(token);
```

Hibernate **automatically** SQL generate karta hai:

```sql
UPDATE password_reset_tokens
SET used = true, version = 4   -- version increment
WHERE id = ?
  AND version = 3              -- ← original version match karo!
```

Agar yeh UPDATE 0 rows affect karta hai (kyunki kisi aur ne pehle update kar diya, version ab 4 ho chuka hai), Hibernate detect karta hai aur **`OptimisticLockException`** throw karta hai.

**Pandit analogy**:

Pandit ke paas register hai. Pehla banda aata hai: "Rajesh ki shaadi Priya se." Pandit register check karta hai, current version 1 hai, likh deta hai version 2 banake.

Doosra banda aata hai: "Rajesh ki shaadi Sonia se." Pandit puchta hai: "Tum kis version pe baat kar rahe ho?" Doosra: "Version 1." Pandit: "Beta, register mein abhi version 2 hai (Priya wali). Tumhara update purane state pe based hai — reject."

Yeh exactly optimistic locking hai. Concurrent updates detect ho jate hain by version mismatch.

---

### Foundation Concept 5: Dependency Injection (DI) Aur Interfaces

#### Step 5.1: "Dependency" Kya Hai?

Code mein **dependency** = ek class jo doosri class pe **depend** karti hai kaam karne ke liye.

```java
public class PasswordResetService {
    private EmailSender emailSender;  // ← PasswordResetService depends on EmailSender

    public void sendResetLink(User user) {
        emailSender.send(user.getEmail(), "Reset link...");
    }
}
```

`PasswordResetService` ki dependency hai `EmailSender`. Bina `EmailSender` ke, `PasswordResetService` apna kaam nahi kar sakta.

#### Step 5.2: Without DI — Pareshani Wala Tareeqa

Naive approach: class apni dependencies khud banaye.

```java
public class PasswordResetService {
    private EmailSender emailSender = new EmailSender();  // ← khud banaya

    public void sendResetLink(User user) {
        emailSender.send(user.getEmail(), "...");
    }
}
```

**Problems**:

1. **Testing impossible**: Tum unit test likhna chahte ho `PasswordResetService` ka without actually sending emails. Lekin yeh hard-coded `new EmailSender()` har time real email bhejega. Mock nahi kar sakte.

2. **Tight coupling**: Agar kal tum `EmailSender` ko `BetterEmailSender` se replace karna chaho, **har class mein change** karna parega jisne `new EmailSender()` use kiya hai. Probably 50 jagah.

3. **Configuration nightmare**: `EmailSender` ko host, port, password chahiye. Har class jo `new EmailSender()` karti hai, woh configuration provide karna parta hai. Repetition.

#### Step 5.3: DI Ka Solution — "Tum Mere Liye Bana Ke Do, Main Nahi Banaunga"

**Dependency Injection** = class apni dependencies khud nahi banati. **Bahar se "inject" hoti hain** (provide hoti hain).

```java
public class PasswordResetService {
    private final EmailSender emailSender;  // ← interface or class

    // Constructor receives dependency from outside
    public PasswordResetService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }

    public void sendResetLink(User user) {
        emailSender.send(user.getEmail(), "...");
    }
}
```

Ab `PasswordResetService` ko parwaah nahi `EmailSender` kahan se aaya — koi bhi de sakta hai.

**Test mein**:
```java
EmailSender mockSender = mock(EmailSender.class);  // fake one
PasswordResetService service = new PasswordResetService(mockSender);
service.sendResetLink(user);
verify(mockSender).send("ahmed@gmail.com", "...");  // verify mock was called
```

Real email nahi gaya. Test fast aur isolated.

#### Step 5.4: Spring's DI Container — Magic Behind The Scenes

Manually `new PasswordResetService(new EmailSender())` har jagah likhna painful hai badi application mein. Hierarchy deep ho gayi to constructor calls disgusting ho jate.

**Spring's DI container ka solution**:

1. Tum classes pe `@Service`, `@Component`, `@Repository` annotations lagao.
2. Spring application start ke time **scan** karta hai saari annotations.
3. **Spring khud objects banata hai aur connections wire karta hai**.

```java
@Service  // Spring ko bolta hai "yeh class manage karo"
public class PasswordResetService {
    private final EmailSender emailSender;

    public PasswordResetService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
    // ...
}

@Component
public class EmailSender {
    // ...
}
```

Spring boot karte time:
1. Spring scan karta hai → `@Service PasswordResetService` mila, `@Component EmailSender` mila.
2. Spring sochta hai: "PasswordResetService ko EmailSender chahiye constructor mein."
3. Spring pehle `EmailSender` ka instance banata hai.
4. Phir `new PasswordResetService(emailSenderInstance)` call karta hai.
5. Dono instances Spring's "container" mein stored hain.

Tumhe kahin bhi `new` nahi karna parta. Spring sab kuch automatically wire kar deta hai. **Yeh hi "Dependency Injection container" hai**.

#### Step 5.5: Interface Ka Power — Polymorphism Setup

Magic level up: instead of `EmailSender` concrete class, use **interface**:

```java
public interface NotificationSender {  // contract only
    void send(User user, String message);
}

@Component
public class EmailSender implements NotificationSender {
    @Override
    public void send(User user, String message) {
        // email-specific logic
    }
}

@Component
public class SmsSender implements NotificationSender {
    @Override
    public void send(User user, String message) {
        // SMS-specific logic
    }
}

@Service
public class PasswordResetService {
    private final NotificationSender notifier;  // ← interface, not concrete class

    public PasswordResetService(NotificationSender notifier) {
        this.notifier = notifier;
    }
}
```

Spring confusion: "do implementations hain `NotificationSender` ki — `EmailSender` aur `SmsSender`. Konsi inject karoon?"

Tum decide karte ho `@Primary` annotation se ya `@Qualifier` se. Ya inject karwa do **List<NotificationSender>** to get all of them — yeh hi polymorphism setup ka raasta hai jo aaj ke OOP section mein detailed cover hoga.

**Yeh hi reason hai DI + Interfaces saath chalti hain** — polymorphic dispatch enables extensibility without modification.

---

### Bhai, Yeh Sab Yaad Karne Ka Mental Model

Pichle 5 concepts ek story mein bandh kar yaad rakh:

> "Ek **secret reference number** (SecureRandom) banao. Uska **fingerprint** (SHA-256 hash) safe-keeping mein rakho, asli reference number bahar bhejo. Iss sab kaam ko ek **transaction** (atomic unit) ke andar do — ya sab succeed ya sab fail. Concurrent modifications detect karne ke liye **version number** (optimistic locking) use karo. Aur saara setup **interfaces + DI** se pluggable banao — kal naya tareeqa add karna ho, sirf nayi class likho."

Iss story mein har keyword tumhe Daraz scenario ke saath jodna chahiye.

---

### Ab Code Padho (Inline Comments Ke Saath)

Yeh code padhne se pehle ke woh 5 concepts pakke ho jane chahiye — warna code random magic lagega.

```java
// ============ ENTITY (Database table mapping) ============
// @Entity = Spring/JPA ko bolta hai "yeh class ek DB table represent karti hai"
@Entity
@Table(
    name = "password_reset_tokens",
    indexes = {
        // Index = phone directory ki tarah, fast lookup ka tareeqa
        // Bina index ke, query pure table ko scan karti hai (slow on millions of rows)
        // unique = true → DB enforce karega ke koi do tokens ka same hash na ho
        @Index(name = "idx_token_hash", columnList = "tokenHash", unique = true),
        // Composite index — multiple columns. Query pattern: WHERE user_id=? AND ...
        @Index(name = "idx_user_expiry", columnList = "userId, expiresAt")
    }
)
public class PasswordResetToken {

    @Id  // Primary key — har row uniquely identify karta hai
    @GeneratedValue(strategy = GenerationType.UUID)  // ID khud generate kare DB
    private UUID id;  // UUID = 128-bit unique ID, collision practically impossible

    // SHA-256 hex string is always exactly 64 characters
    // length=64 storage optimize karta hai (varchar fixed)
    @Column(nullable = false, length = 64)
    private String tokenHash;

    @Column(nullable = false)
    private Long userId;

    // Instant = UTC timestamp, timezone-independent
    // Don't use LocalDateTime — timezone bugs cause production headaches
    @Column(nullable = false)
    private Instant expiresAt;

    @Column(nullable = false)
    private boolean used = false;

    // @Version = optimistic locking magic
    // Hibernate auto-increment karta hai aur WHERE clause mein check karta hai
    @Version
    private Long version;
}

// ============ SERVICE (Business logic layer) ============
@Service  // Spring ko bolta hai "yeh class manage karo, DI mein register karo"
@RequiredArgsConstructor  // Lombok: auto-generate constructor with all final fields
public class PasswordResetService {

    // final = constructor mein set hoga, baad mein change nahi
    // Spring DI in dependencies ko inject karega constructor ke through
    private final UserRepository userRepo;
    private final PasswordResetTokenRepository tokenRepo;
    private final PasswordEncoder passwordEncoder;  // interface! BCrypt actual impl
    private final SessionService sessionService;
    private final NotificationSender notifier;  // interface! Polymorphism magic

    // SecureRandom instance — one per service, thread-safe
    private final SecureRandom random = new SecureRandom();

    @Transactional  // Spring proxy automatic BEGIN/COMMIT/ROLLBACK manage karega
    public void requestReset(String email) {

        // Optional<User> = either has User or empty (Java 8+ null-safe pattern)
        Optional<User> userOpt = userRepo.findByEmail(email);

        if (userOpt.isPresent()) {
            User user = userOpt.get();

            // STEP 1: 32 random bytes generate karo CSPRNG se
            byte[] tokenBytes = new byte[32];
            random.nextBytes(tokenBytes);

            // STEP 2: Bytes ko URL-safe string mein convert karo
            // 32 bytes → 43-character string
            String rawToken = Base64.getUrlEncoder()
                .withoutPadding()
                .encodeToString(tokenBytes);

            // STEP 3: Token ko hash karo store karne se pehle
            String tokenHash = sha256Hex(rawToken);

            // STEP 4: Pehle ke unused tokens kill kar do
            tokenRepo.markAllUserTokensAsUsed(user.getId());

            // STEP 5: Naya token save karo with 15-minute expiry
            PasswordResetToken token = new PasswordResetToken();
            token.setUserId(user.getId());
            token.setTokenHash(tokenHash);
            token.setExpiresAt(Instant.now().plus(15, ChronoUnit.MINUTES));
            tokenRepo.save(token);

            // STEP 6: Notification bhejo via polymorphic dispatch
            String resetLink = "https://daraz.pk/reset?token=" + rawToken;
            notifier.send(user, new PasswordResetMessage(resetLink));
        }

        // CRITICAL: agar user nahi mila, hum exception throw nahi karte
        // Same code path = same response time = no enumeration leak
    }

    @Transactional
    public void confirmReset(String rawToken, String newPassword) {

        String tokenHash = sha256Hex(rawToken);

        PasswordResetToken token = tokenRepo
            .findByTokenHashAndUsedFalseAndExpiresAtAfter(tokenHash, Instant.now())
            .orElseThrow(() -> new InvalidTokenException("Token invalid or expired"));

        User user = userRepo.findById(token.getUserId())
            .orElseThrow(() -> new InvalidTokenException("User not found"));

        // BCrypt internally salt generate karta hai, hash mein embed karta hai
        user.setPasswordHash(passwordEncoder.encode(newPassword));
        userRepo.save(user);

        // @Version protects against concurrent click race
        token.setUsed(true);
        tokenRepo.save(token);

        sessionService.invalidateAllSessions(user.getId());
    }

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

**Interview Phrasing**:

"Iss scenario mein raw token sirf email mein jata hai, DB mein sirf SHA-256 hash store hota hai — taake DB breach pe tokens leak na hon. Single-use guarantee `@Version` ke saath optimistic locking se enforce hoti hai. User enumeration roknay ke liye — exists/not-exists dono code paths same response return karte hain."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Goal**: Same logic, different framework. .NET ki terminology aur internals samjho.

### Foundation Concept 1: .NET Mein Random Number — `RandomNumberGenerator`

Java ke `SecureRandom` ka equivalent .NET mein hai **`RandomNumberGenerator`** class (namespace: `System.Security.Cryptography`).

Same exact security guarantees:
- Predictable nahi
- OS entropy use karta hai

Windows mein OS-level API ka naam different hai: **CNG** = **Cryptography Next Generation**. Yeh Windows ka modern crypto API hai jo purani CryptoAPI ki jagah aaya (early 2000s mein). Yeh same kaam karta hai jo Linux ka `/dev/urandom` — hardware entropy collect karke unpredictable bytes provide karta hai.

`RandomNumberGenerator.Fill()` static method automatically OS-appropriate path use karta hai. Same "wrapper" concept jo humne Java mein dekha.

```csharp
var tokenBytes = new byte[32];
RandomNumberGenerator.Fill(tokenBytes);  // OS entropy from CNG / urandom
```

### Foundation Concept 2: `IPasswordHasher<TUser>` — .NET Ka Password Hashing

.NET Core 3.0+ mein password hashing ka standard interface hai **`IPasswordHasher<TUser>`**. **Default implementation** PBKDF2 use karta hai.

**PBKDF2** = **P**assword-**B**ased **K**ey **D**erivation **F**unction **2**. Yeh BCrypt ka cousin hai — same purpose (slow, salted password hashing). Slightly different math but same concept.

### Foundation Concept 3: EF Core Aur Change Tracker

**Entity Framework Core (EF Core)** = .NET ka ORM. **ORM** = **O**bject-**R**elational **M**apper = library jo aapke C# objects ko SQL tables mein automatically map karta hai aur SQL queries generate karta hai aap ki bajaye.

**Java mein equivalent**: Hibernate / JPA.

**"Change Tracker" ka concept**:

EF Core background mein har loaded entity ko **track** karta hai. Yaad rakhta hai entity ka original state. Jab tum koi property modify karte ho:

```csharp
var user = await dbContext.Users.FindAsync(5);
// EF Core records original state

user.Name = "Ahmed Khan";
// Change tracker note karta hai: "Name modified"

await dbContext.SaveChangesAsync();
// EF Core sirf modified columns ka UPDATE generate karta hai:
// UPDATE Users SET Name = 'Ahmed Khan' WHERE Id = 5;
```

**Smart**: Sirf badle hue fields update hote hain.

### Foundation Concept 4: `[Timestamp]` Attribute — .NET Mein Optimistic Locking

Java ka `@Version` ka equivalent .NET mein `[Timestamp]` attribute hai.

```csharp
public class PasswordResetToken
{
    public Guid Id { get; set; }
    public string TokenHash { get; set; }
    public bool Used { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

**`[Timestamp]`** SQL Server mein **`ROWVERSION`** column type use karta hai. ROWVERSION database-wide unique 8-byte counter hai — har baar koi bhi row update hoti hai, ROWVERSION auto-increment hota hai.

Jab EF Core UPDATE generate karta hai:

```sql
UPDATE password_reset_tokens
SET used = 1
WHERE id = @id
  AND rowversion = @originalRowVersion;  -- ← concurrency check
```

Agar mismatch — `DbUpdateConcurrencyException`.

### Foundation Concept 5: .NET Mein DI

.NET Core mein built-in DI container hai. Spring se conceptually same hai.

```csharp
// Program.cs
builder.Services.AddScoped<IPasswordHasher<User>, PasswordHasher<User>>();
builder.Services.AddScoped<INotificationSender, EmailNotificationSender>();
builder.Services.AddScoped<PasswordResetService>();

// Service
public class PasswordResetService
{
    private readonly INotificationSender _notifier;

    public PasswordResetService(INotificationSender notifier)
    {
        _notifier = notifier;
    }
}
```

**Scopes**:
- `Singleton` — pura app ek instance
- `Scoped` — har HTTP request ka apna instance (default for web)
- `Transient` — har injection pe naya

### .NET Code Pattern

```csharp
public class PasswordResetToken
{
    public Guid Id { get; set; }
    public string TokenHash { get; set; } = default!;
    public long UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool Used { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = default!;
}

public class PasswordResetService
{
    private readonly AppDbContext _db;
    private readonly IPasswordHasher<User> _hasher;
    private readonly ISessionService _sessions;
    private readonly INotificationSender _notifier;

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
            var tokenBytes = new byte[32];
            RandomNumberGenerator.Fill(tokenBytes);
            var rawToken = Base64UrlEncoder.Encode(tokenBytes);
            var tokenHash = Sha256Hex(rawToken);

            // Bulk update without loading entities (EF Core 7+)
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

### Java vs .NET Comparison Table

| Feature | Java/Spring | .NET/C# | Plain English |
|---------|-------------|---------|---------------|
| CSPRNG | `SecureRandom.nextBytes()` | `RandomNumberGenerator.Fill()` | Predictable nahi bytes from OS |
| Password hashing | `BCryptPasswordEncoder` | `IPasswordHasher<TUser>` (PBKDF2) | Slow hash with salt |
| SHA-256 | `MessageDigest.getInstance("SHA-256")` | `SHA256.Create()` | Fast one-way hash |
| Optimistic lock | `@Version` on field | `[Timestamp]` on byte[] field | Concurrent update detection |
| Async I/O | `@Async` + `CompletableFuture` | `async/await` + `Task` | Non-blocking operations |
| ORM | Hibernate (JPA) | EF Core | Object ↔ SQL mapping |
| Transaction | `@Transactional` | `using var tx = await db.Database.BeginTransactionAsync()` | Atomic group |
| DI container | Spring @Service, @Autowired | builder.Services.Add* | Constructor injection |

---

## 🔧 Stack 3: SQL (Data Layer)

**Goal**: Database internals samjho — locks, isolation, indexes, transactions.

### Foundation Concept 1: Database "Lock" Kya Hota Hai?

Database concurrent access ke liye **locks** use karta hai. Lock = "yeh resource main use kar raha hoon, dosre wait karo".

**3 main types**:

#### 🔵 Shared Lock (S-Lock) — "Padh Raha Hoon, Tum Bhi Padh Sakte Ho"

Read karte time S-Lock. **Multiple transactions** same row pe S-Lock le sakte hain. Lekin koi write nahi kar sakta.

```
A: SELECT ... id=5  → S-Lock on row 5
B: SELECT ... id=5  → S-Lock on row 5 (allowed)
C: UPDATE ... id=5  → Wait!
```

#### 🔴 Exclusive Lock (X-Lock) — "Sirf Main"

Modify karte time X-Lock. **Sirf ONE transaction** X-Lock le sakta hai. Doosre wait.

```
A: UPDATE ... id=5  → X-Lock on row 5
B: SELECT ... id=5  → Wait!
C: UPDATE ... id=5  → Wait!
```

#### 🟡 Update Lock (U-Lock) — Intermediate

DB engine UPDATE statement execute karte time pehle U-Lock leta hai (safely read karne ke liye), agar update karna hai to X-Lock mein convert karta hai. Yeh deadlock prevent karta hai.

### Foundation Concept 2: ACID Properties (Stack 1 Mein Touch Kiya Tha)

**ACID** = 4 promises:

- **A — Atomicity**: Sab commit ya sab rollback
- **C — Consistency**: Constraints maintained
- **I — Isolation**: Concurrent transactions ek doosre se isolated
- **D — Durability**: Committed data crash mein bhi safe

### Foundation Concept 3: Write-Ahead Log (WAL) — Durability Ka Mechanism

**Problem**: COMMIT kiya, 1 second baad power off. Data RAM mein tha, disk pe nahi. **Lost?**

**Solution — WAL**:

DB har change **pehle sequential log file** mein likhta hai, **phir** actual data file mein. Log file append-only = fast.

```
1. Transaction modifying data in memory
2. COMMIT triggered
3. WAL mein log entry write (synchronously to disk):
   "Transaction#42 committed: UPDATE tokens SET used=1 WHERE id=5"
4. WAL write confirmed → COMMIT acknowledged
5. (Async) Actual data file later updated
```

**Crash recovery**:
1. DB restart
2. WAL padhta hai end to end
3. Committed-but-not-flushed transactions replay
4. Consistent state restore

WAL sequential write = fast. Actual data files ka random write slow — async kar diye.

### Foundation Concept 4: Indexes — B-Tree Ki Magic

#### Step 4.1: Bina Index Ke Search Slow

```sql
SELECT * FROM tokens WHERE token_hash = 'abc123...';
```

**Without index** = **full table scan** — har row check. 1 million rows = 1 million comparisons.

#### Step 4.2: Phone Directory Analogy

**Index** = pre-sorted lookup structure. Phone directory jaisa — naam sorted. Tum middle se start karte ho, half eliminate karte ho har step. 1 million names = ~20 steps.

DB index **B-tree** structure use karta hai (sorted, balanced tree).

#### Step 4.3: B-Tree Visual

```
                    [Page 1: 'm']
                   /              \
              [Page 2: 'g']      [Page 3: 's']
              /     |     \      /     |     \
         [a-d]   [e-f]  [h-l]  [n-q] [r]   [t-z]
         leaves with actual data pointers
```

Search "tahir":
1. Root: 't' > 'm', right
2. Next: 't' > 's', right
3. Find in leaf "t-z"
4. Done — 3 steps for huge table

"B" = balanced — tree height equal har taraf, ensuring consistent log(n) performance.

#### Step 4.4: Disk Pages

DB disk se data **pages** mein read karta hai (typically 8KB blocks). B-tree nodes = disk pages. Wide fanout (dozens of children per node) → shallow tree (3-4 levels even for billions of rows).

Practical: lookup = 3-4 disk reads = milliseconds. Vs full scan = seconds/minutes.

### Foundation Concept 5: Composite Index Aur Leftmost-Prefix Rule

```sql
CREATE INDEX idx_user_active
    ON password_reset_tokens(user_id, used, expires_at);
```

Internally sorted: pehle `user_id`, phir `used`, phir `expires_at`.

#### Leftmost-Prefix Rule

✅ **Index used** (left-to-right columns in WHERE):
- `WHERE user_id = ?`
- `WHERE user_id = ? AND used = false`
- `WHERE user_id = ? AND used = false AND expires_at > NOW()`

❌ **Index NOT used**:
- `WHERE used = false` (skipped leftmost)
- `WHERE expires_at > NOW()` (skipped first two)

**Why**: Index sorted by user_id first. Without user_id condition, DB ko start point nahi pata — full scan.

### Foundation Concept 6: Isolation Levels — 4 Levels

ACID ka "I" = Isolation. 4 standard levels:

| Level | Behavior | Use case |
|-------|----------|----------|
| **READ UNCOMMITTED** | Doosre ki uncommitted changes bhi dekho (dirty reads) | Almost never |
| **READ COMMITTED** (default in SQL Server, Oracle, Postgres) | Sirf committed data. Same query different result possible. | Most apps |
| **REPEATABLE READ** (MySQL default) | Padhi hui row frozen for your transaction. Phantom rows possible. | When you re-read same row |
| **SERIALIZABLE** | Transactions appear one-by-one | Banking, critical financial |

**Password reset**: **READ COMMITTED enough**. Single-statement atomic UPDATE row-level lock leta hai automatically.

### Foundation Concept 7: Atomic UPDATE Pattern

#### ❌ TOCTOU (Multi-Statement)

```sql
SELECT used FROM tokens WHERE hash='abc';  -- returns false
-- Application: "OK, not used"
UPDATE tokens SET used=1 WHERE hash='abc';  -- always updates
```

Race: 2 concurrent SELECTs both see false, both UPDATE.

#### ✅ Atomic UPDATE (Single Statement)

```sql
UPDATE tokens
SET used = 1
WHERE token_hash = 'abc'
  AND used = 0;       -- ← check AND update atomic
```

**Internal flow**:

```
1. DB seeks row using idx_token_hash
2. Acquires U-Lock
3. Reads row, checks: used=0?
4. If YES:
   - Convert U-Lock → X-Lock
   - Modify row: used=1
   - Write to WAL
   - Release X-Lock at COMMIT
   - rowcount=1
5. If NO:
   - Release U-Lock
   - rowcount=0
```

**Concurrent race**:

```
A: UPDATE → U-Lock → used=0 → X-Lock → updates → rowcount=1 ✓
B: UPDATE → wants U-Lock → BLOCKED by A's X-Lock → WAIT
   A commits, releases X-Lock
   B gets U-Lock → sees used=1 (changed by A) → WHERE fails → rowcount=0 ✗
```

One succeeds, other cleanly rejected. No corruption.

### Pura SQL Code

```sql
-- ============ TABLE ============
CREATE TABLE password_reset_tokens (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    user_id         BIGINT NOT NULL,
    token_hash      CHAR(64) NOT NULL,  -- SHA-256 hex always 64 chars
    expires_at      DATETIME2 NOT NULL,
    used            BIT NOT NULL DEFAULT 0,
    used_at         DATETIME2 NULL,
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    row_version     ROWVERSION,
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- ============ INDEXES ============
CREATE UNIQUE INDEX idx_token_hash ON password_reset_tokens(token_hash);
CREATE INDEX idx_user_active ON password_reset_tokens(user_id, used, expires_at);
CREATE INDEX idx_cleanup ON password_reset_tokens(expires_at);

-- ============ ATOMIC CONSUMPTION ============
BEGIN TRANSACTION;

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

DECLARE @userId BIGINT = (
    SELECT user_id FROM password_reset_tokens WHERE token_hash = @tokenHash
);

UPDATE users
SET password_hash = @newPasswordHash, password_changed_at = SYSUTCDATETIME()
WHERE id = @userId;

DELETE FROM user_sessions WHERE user_id = @userId;

COMMIT TRANSACTION;
```

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Goal**: Frontend-specific concepts samjho.

### Foundation Concept 1: Reactive Programming Aur Observable

#### Step 1.1: "Reactive Programming" Kya Hai?

**Traditional (imperative)**: step-by-step instructions.

```javascript
let total = 0;
total = price * quantity;
// Phir price ya quantity change ho, manually recalculate
```

**Reactive**: data ke beech **relationships** declare karo. Change pe **automatic update**.

Mental model: spreadsheet. A3 = `=A1*A2`. Tum A1 change karo, A3 automatically recompute.

#### Step 1.2: "Observable" — Newsletter Analogy

**Observable** = "future stream of values". Tum **subscribe** karte ho — value emit ho, tumhara callback chalta hai.

Newsletter subscription: subscribe karo. Future mein naya issue → mil jata hai. Doosra bhi subscribe karta hai, woh bhi receive karta hai.

```typescript
const counter$: Observable<number> = interval(1000);
counter$.subscribe(value => console.log(value));
// 0 (after 1s), 1 (after 2s), 2 (after 3s)...
```

#### Step 1.3: Observable vs Promise

**Promise** = ek single future value. Resolve once, khatam.

**Observable** = stream of values. Kitne bhi values emit over time. Cancellable.

```typescript
fromEvent(button, 'click').subscribe(event => handle(event));
// Click 1, Click 2, Click 3...
```

#### Step 1.4: RxJS Aur Operators

**RxJS** = library for Observables + operators.

**Operators** = functions jo streams transform karte hain (like SQL for streams).

```typescript
interval(1000).pipe(
    filter(n => n % 2 === 0),
    map(n => n * 10)
).subscribe(console.log);
// 0, 20, 40, 60...
```

### Foundation Concept 2: `debounceTime` — Spam Roknay Ki Magic

**Problem**: search box mein har keystroke pe API call = 5 calls for "ahmed".

**Solution**:

```typescript
searchInput.valueChanges.pipe(
    debounceTime(300)
).subscribe(value => api.search(value));
```

**Step-by-step**:

```
T+0ms:    Type 'a' → emit → timer set (300ms)
T+100ms:  Type 'h' → cancel timer, new emit → new timer
T+200ms:  Type 'm' → cancel, emit → new timer
T+300ms:  Type 'e' → cancel, emit → new timer
T+400ms:  Type 'd' → cancel, emit → new timer
T+700ms:  No input for 300ms → final emit → subscriber called
```

5 keystrokes → 1 API call. Internally setTimeout/clearTimeout.

### Foundation Concept 3: Angular Reactive Forms

**2 form types**:
- **Template-Driven**: simple, state in HTML
- **Reactive**: powerful, state in TypeScript

`FormControl`'s **`valueChanges`** = Observable. Value change → emit.

```typescript
this.form.get('email')!.valueChanges.subscribe(value => {
    console.log('Email:', value);
});
```

### Foundation Concept 4: Change Detection Aur Zone.js

#### Angular Ko Kaise Pata Chalta Hai Variable Change Hua?

```typescript
@Component({ template: `<p>{{ count }}</p>` })
class MyComponent {
    count = 0;
    increment() { this.count++; }
}
```

Button click → `count++` → UI update. **Kaise?**

#### Zone.js — JavaScript Ka Spy

**Zone.js** = library jo JavaScript ke async APIs ko intercept karti hai. "Monkey-patching" karti hai — standard JS functions replace karti hai apne versions se.

**Monkey-patching example**:

```javascript
const originalSetTimeout = window.setTimeout;
window.setTimeout = function(callback, delay) {
    return originalSetTimeout(function() {
        callback();
        notifyAngularToCheckForChanges();  // Zone notifies Angular
    }, delay);
};
```

Zone.js patches: setTimeout, addEventListener, fetch, Promises — **har async API**.

#### Full Mechanism

1. App start → Zone.js loads, monkey-patches everything
2. User clicks → Zone intercepts
3. Click handler: `this.count++`
4. Handler done → Zone notifies Angular
5. Angular runs change detection — traverses component tree
6. `{{ count }}` re-renders with new value

### Foundation Concept 5: HttpClient Cold Observables

```typescript
const data$ = this.http.get('/api/users');  // NO request yet
data$.subscribe(...);  // NOW request fires
data$.subscribe(...);  // ANOTHER request (separate!)
```

**Cold** = subscribe nahi karte to nothing happens. 2 subscribes = 2 requests.

**Async pipe**:

```html
<div *ngFor="let user of users$ | async">
  {{ user.name }}
</div>
```

`async` pipe auto-subscribe + auto-unsubscribe on component destroy. **No memory leak**.

### Angular Code

```typescript
@Component({
  selector: 'app-forgot-password',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" type="email" placeholder="Email" />
      <button [disabled]="form.invalid || loading">
        {{ loading ? 'Bhej raha hai...' : 'Reset Link' }}
      </button>
    </form>
    <!-- IDENTICAL message — anti-enumeration -->
    <div *ngIf="submitted" class="success">
      Agar email registered hai, reset link bhej diya gaya hai.
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
      .pipe(finalize(() => this.loading = false))
      .subscribe({
        // CRITICAL: same UI on success AND error
        next: () => this.submitted = true,
        error: () => this.submitted = true
      });
  }
}

@Component({
  selector: 'app-reset-password',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="password" type="password" />
      <div class="strength" [ngClass]="(strength$ | async) || 'weak'">
        Strength: {{ strength$ | async }}
      </div>
      <input formControlName="confirm" type="password" />
      <div *ngIf="form.errors?.['mismatch']">Passwords nahi match</div>
      <button [disabled]="form.invalid">Set Password</button>
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
    if (!this.token) { this.router.navigate(['/login']); return; }

    // Real-time strength — debounced
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
        next: () => this.router.navigate(['/login']),
        error: err => alert(err.error?.message ?? 'Link expired')
      });
  }

  private matchValidator(group: AbstractControl): ValidationErrors | null {
    return group.get('password')?.value === group.get('confirm')?.value
      ? null : { mismatch: true };
  }

  private calculateStrength(pw: string): 'weak' | 'medium' | 'strong' {
    if (pw.length < 8) return 'weak';
    const score = [/[A-Z]/, /[a-z]/, /\d/, /[^A-Za-z0-9]/].filter(r => r.test(pw)).length;
    if (score >= 3 && pw.length >= 12) return 'strong';
    if (score >= 2) return 'medium';
    return 'weak';
  }
}
```

### UX Concern — Anti-Enumeration Frontend Pe Bhi

Different message dikhao → DevTools → Network tab → enumeration. **Identical response har layer pe** non-negotiable.

---

## 🔧 Stack 5: System Design

**Goal**: System level — messaging, async, rate limiting.

### Foundation Concept 1: Synchronous vs Asynchronous

**Sync**: tum karte ho, **wait** karte ho, aage badhte ho.

```
Client → Server (DB+SMTP 2200ms) → Response
Total: 2.2s — user dekhta loading spinner — UX kharab
```

**Async**: slow operations background mein, user ko fast response.

```
Client → Server (DB + publish event 21ms) → Response
Total: 21ms — fast!

Background: Queue consumer → SMTP (2s) → email delivered
```

User ko 21ms mein "link sent". Email background mein. No waiting.

**Trade-offs**:
- ✅ Fast response, scalable, resilient
- ❌ Complex architecture, eventual delivery, debugging harder

### Foundation Concept 2: Message Broker (Kafka)

**Message broker** = middleware jo messages **producers** se receive karke **consumers** tak deliver karta hai.

Analogy: **Post office**. Tum letter dete ho, post office store karta hai, recipient tak deliver karta hai.

**Apache Kafka** distributed broker. Multiple servers ("brokers") milke cluster banate hain. Messages replicate hote hain durability ke liye.

**Key concepts**:
- **Topic** = category/channel (`password.reset.events`)
- **Producer** = publishes (Auth Service)
- **Consumer** = subscribes and reads (Notification Service)
- **Partition** = topic subdivision for parallelism
- **Offset** = sequential message ID

**Durability**: messages disk pe write immediately. Crash mein safe. Configurable retention (default 7 days) — replay possible.

### Foundation Concept 3: Dead Letter Queue (DLQ)

**Problem**: notification fail ho gayi (SMTP down). Kya karein message ka?

❌ Retry infinitely → clog system
❌ Silently drop → data loss

**✅ DLQ**: separate queue for **failed messages**. After N retries, message moves to DLQ.

```
Original queue: process → fail → retry → fail → retry → fail
                                                          ↓
DLQ:                                                  message here
Ops team:    monitors DLQ → reviews → manually retry or fix
```

Graceful degradation. System keeps running, problems isolated.

### Foundation Concept 4: Rate Limiting — Token Bucket Algorithm

**Rate limiting** = max N requests per time window per user/IP.

**Token Bucket**:

Imagine bucket holding X tokens (capacity 100). Refills Y tokens/sec (rate 10/sec). Each request consumes 1.

```
Time 0s:   Bucket [100/100]
           Request → consume 1 → [99/100]
Time 0.1s: Auto-refill 1 → [100/100]
           10 requests → consume 10 → [90/100]
...
Heavy traffic:
           Bucket drains → 0 → requests REJECTED (429)
           Traffic stops → bucket refills → service resumes
```

Self-recovering. Allows bursts, sustains long-term limits.

**Redis implementation**: in-memory key-value store, microseconds latency. Atomic INCR + EXPIRE commands. Lua script for race-free updates.

### Architecture Diagram

```
┌──────────────────────────────────────────────────────┐
│  USER (Browser/Mobile)                               │
│  Angular: /forgot-password and /reset?token=...      │
└─────────────────────┬────────────────────────────────┘
                      │ HTTPS POST
                      ▼
┌──────────────────────────────────────────────────────┐
│  API GATEWAY                                         │
│  + Rate Limiter (Token Bucket: 3/hour/IP on Redis)   │
│  + TLS termination                                   │
└─────────────────────┬────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│  AUTH SERVICE (Spring/.NET)                          │
│  requestReset: DB lookup → token gen → save → KAFKA  │
│  confirmReset: atomic UPDATE → password → kill sessions │
└─────┬──────────────────────────────────┬─────────────┘
      │                                  │
      ▼                                  ▼
┌──────────────┐         ┌────────────────────────────┐
│  SQL DB      │         │  KAFKA                     │
│  - users     │         │  Topic: password.reset     │
│  - tokens    │         └──────────┬─────────────────┘
│  - sessions  │                    │
└──────────────┘                    ▼
                          ┌────────────────────────────┐
                          │  NOTIFICATION SERVICE      │
                          │                            │
                          │  INotificationSender       │
                          │     ├─ EmailSender (SES)   │
                          │     ├─ SmsSender (Twilio)  │
                          │     ├─ PushSender (FCM)    │
                          │     └─ WhatsAppSender      │
                          │                            │
                          │  Polymorphic dispatch      │
                          └──────────┬─────────────────┘
                                     │ on failure
                                     ▼
                          ┌────────────────────────────┐
                          │  DEAD LETTER QUEUE         │
                          │  Ops alert + manual review │
                          └────────────────────────────┘
```

### Trade-offs

| Approach | Pros | Cons | When |
|----------|------|------|------|
| Sync email | Simple, immediate | Slow API, single point of failure | Tiny apps |
| Async via queue | Fast response, retryable, scalable | Complex, eventual delivery | **Production default** |
| JWT token | No DB lookup, stateless | Cannot revoke before expiry | Short-expiry magic links |
| DB-stored token | Revocable, single-use enforced | DB hit per validation | **Security-sensitive default** |
| 6-digit OTP | User-friendly | Brute-forceable, needs attempt limits | Mobile-first apps |
| Long URL token | 10^77 combos, unbrute-forceable | Needs click | **Daraz-style** |

### Real Companies

- **Stripe**: Async Kafka, multi-channel, 1-hour HMAC tokens
- **GitHub**: Sync send, 3-hour expiry, explicit anti-enumeration
- **Slack**: Magic-link, random + SHA-256, async SendGrid
- **AWS Cognito**: Hashed 6-digit, attempt-limited, async SES

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Polymorphism Detail)

**Today's Focus**: **Polymorphism (Method Overriding)** — Day 4 of OOP Fundamentals

### Foundation 1: Polymorphism Ka Pura Concept

#### Step 1.1: Naam Ka Matlab

**Polymorphism** = Greek **"poly" (many) + "morph" (shape)** = "many shapes".

Programming mein: **same method call, different actual behavior** depending on actual object's type at runtime.

#### Step 1.2: 3 Types (Aaj Sirf 1)

1. **Compile-time (Overloading)** — same name, different parameters, same class
2. **Runtime (Overriding)** — child redefines parent's method
3. **Parametric (Generics)** — same code, multiple types

**Aaj #2 focus.**

#### Step 1.3: Hotel Chef Analogy

Customer kehta: "Chef sahab, khaana banao." Chef: "OK."

- Pakistani chef → biryani
- Chinese chef → noodles
- Italian chef → pasta

**Same request, different delivery.** Caller knows interface, not implementation.

### Foundation 2: Method Overloading Vs Overriding

#### Method Overloading (Compile-Time)

**Same class**, same method name, **different parameter lists**.

```java
class Printer {
    void print(int x) { ... }
    void print(String x) { ... }
    void print(int x, int y) { ... }
}

Printer p = new Printer();
p.print(5);        // Picks print(int) — compile time
p.print("hello");  // Picks print(String) — compile time
```

Compiler dekhta arguments → exact method choose karta. **Saara decision compile time pe**.

#### Method Overriding (Runtime)

**Child class** parent's same-signature method **redefines**.

```java
class Animal {
    public void sound() { print("generic"); }
}

class Dog extends Animal {
    @Override
    public void sound() { print("Woof!"); }
}

class Cat extends Animal {
    @Override
    public void sound() { print("Meow!"); }
}

Animal myPet = new Dog();
myPet.sound();  // "Woof!" — RUNTIME decided
```

Compile time pe compiler: "myPet is Animal type, sound() exists in Animal — valid."
Runtime pe JVM: "actual object is Dog, call Dog's sound()."

#### Comparison

| Aspect | Overloading | Overriding |
|--------|-------------|------------|
| Same class? | Yes | No — parent + child |
| Same parameters? | No | Yes — identical signature |
| Decision when? | Compile time | **Runtime** |
| Mechanism | Compiler picks | JVM vtable lookup |
| Inheritance? | No | Yes |

### Foundation 3: Vtable — Runtime Polymorphism Internal Mechanism

#### Step 3.1: Confusion Kya Hai?

```java
Animal myPet = new Dog();
myPet.sound();  // prints "Woof!"
```

**Question**: `myPet` ka declared type Animal hai. Compiler ko sirf `Animal.sound()` pata. Phir JVM kaise jaane Dog ki sound chalani hai?

**Answer**: **Virtual function table (vtable)**.

#### Step 3.2: Vtable Kya Hai?

Har class ka apna **vtable** hota hai memory mein. **Array of function pointers** — har overrideable method ka pointer.

```
Class Animal's vtable:
  index 0: → Animal.sound() code address
  index 1: → Animal.eat() code address

Class Dog's vtable (extends Animal):
  index 0: → Dog.sound() code address    ← OVERRIDDEN
  index 1: → Animal.eat() code address    ← inherited

Class Cat's vtable (extends Animal):
  index 0: → Cat.sound() code address    ← OVERRIDDEN
  index 1: → Animal.eat() code address    ← inherited
```

Note: **same index** for sound(), different pointers.

#### Step 3.3: Method Call Flow

```java
Animal myPet = new Dog();
myPet.sound();
```

```
T+0: Object create: new Dog()
     Dog object in memory + hidden "class pointer" → Dog's vtable

T+1: Variable assignment: Animal myPet = ...
     myPet → points to Dog object

T+2: myPet.sound()
     Compile time: compiler checks sound() exists in Animal → valid
     Runtime — JVM:
       a. myPet → follow pointer → Dog object
       b. From object → class pointer → Dog's vtable
       c. Look up index 0 → Dog.sound() pointer
       d. Jump to that address → execute
       e. Prints "Woof!"
```

Nanoseconds, but **vtable lookup** is the cost of virtual dispatch.

#### Step 3.4: JIT Devirtualization

**JIT** = **J**ust-**I**n-**T**ime compiler. JVM component converting bytecode to native machine code at runtime.

JIT smart: if a call site always sees same implementation, it **devirtualizes** — direct call instead of vtable lookup. "Monomorphic call site optimization."

### Foundation 4: Aaj Ke Code Mein Polymorphism

#### 1. `NotificationSender` Interface

```java
public interface NotificationSender {
    void send(User user, NotificationMessage message);
}

class EmailNotificationSender implements NotificationSender { ... }
class SmsNotificationSender implements NotificationSender { ... }
class PushNotificationSender implements NotificationSender { ... }
```

Service:
```java
private final NotificationSender notifier;  // interface
notifier.send(user, message);  // polymorphic dispatch
```

WhatsApp add? **Naya class. Done.** Service code untouched.

#### 2. `PasswordEncoder` (Spring Built-In)

Spring: `BCryptPasswordEncoder`, `Argon2PasswordEncoder`, `Pbkdf2PasswordEncoder`, `SCryptPasswordEncoder`. Sab override `encode()` aur `matches()`.

DI config se switch:
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new Argon2PasswordEncoder();  // was BCrypt
}
```

**Saare service code untouched.**

#### 3. `MessageDigest.getInstance("SHA-256")`

JVM polymorphic dispatch. Abstract base + concrete subclasses (SHA-256, SHA-512, MD5).

---

### Code: ❌ BAD vs ✅ GOOD

#### ❌ BAD: If/Else Hell

```java
@Service
public class PasswordResetService {
    public void sendResetNotification(User user, String link, String channel) {
        if (channel.equals("EMAIL")) {
            JavaMailSender mailSender = ...;
            // 20 lines of email code
        }
        else if (channel.equals("SMS")) {
            TwilioRestClient twilio = ...;
            // 15 lines of SMS code
        }
        else if (channel.equals("PUSH")) {
            FirebaseMessaging fcm = ...;
            // 25 lines of push code
        }
        // WhatsApp add karna? Yahi chain extend karo!
    }
}
```

**Problems**: Open/Closed violated, Single Responsibility violated, untestable, imports bloat, tight coupling.

#### ✅ GOOD: Polymorphism + Strategy

```java
// INTERFACE
public interface NotificationSender {
    void send(User user, NotificationMessage message);
    NotificationChannel getChannel();
}

// EMAIL IMPLEMENTATION
@Component
@RequiredArgsConstructor
public class EmailNotificationSender implements NotificationSender {
    private final JavaMailSender mailSender;

    @Override
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

// SMS IMPLEMENTATION
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

// SERVICE
@Service
public class PasswordResetService {
    private final Map<NotificationChannel, NotificationSender> sendersByChannel;

    // Spring magic: List<NotificationSender> auto-fills with ALL implementations
    public PasswordResetService(List<NotificationSender> allSenders) {
        this.sendersByChannel = allSenders.stream()
            .collect(Collectors.toMap(NotificationSender::getChannel, Function.identity()));
    }

    public void sendResetNotification(User user, String link) {
        NotificationSender sender = sendersByChannel.get(user.getPreferredChannel());
        if (sender == null) sender = sendersByChannel.get(NotificationChannel.EMAIL);

        // ⭐ POLYMORPHIC CALL
        sender.send(user, new PasswordResetMessage(link));
    }
}
```

**Ab WhatsApp add karna**:

```java
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
    public NotificationChannel getChannel() { return NotificationChannel.WHATSAPP; }
}
```

Spring detects new `@Component`, injects to service's list, map gets new entry. **Zero changes to existing code.** Textbook **Open/Closed Principle**.

### Java vs .NET

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Override syntax | `@Override` (optional but recommended) | `override` (**mandatory**) |
| Base virtual? | Default virtual | Must be `virtual`/`abstract` |
| Hide vs override | Always overrides | `new` keyword can hide |
| Prevent further override | `final` | `sealed override` |
| Polymorphic DI | `List<INotifier>` | `IEnumerable<INotifier>` or keyed services |

---

### 🎯 Cross-Questioning Drill

#### Q1: "Polymorphism aur Method Overloading mein farak?"

"Bhai, dono ka naam similar but completely alag concepts:

**Overloading (compile-time)**:
- Same class, same name, **different parameters**
- **Compiler decides** by argument types
- No inheritance
- `print(int)`, `print(String)` — same class

**Overriding (runtime)**:
- Parent's method redefined in child
- **Same signature**
- **JVM decides at runtime** via vtable lookup
- Inheritance required

Compile time pe compiler validates signature exists. Runtime pe JVM looks up actual object's vtable."

#### Q2: "Strategy pattern use kiya. Alternatives?"

"3 alternatives:

1. **Switch/if-else**: Open/Closed violated, hard to test, single class knows all channels.
2. **Functional (lambdas)**: Better but still concentrates logic in one file. Hard to inject deps.
3. **Strategy + Polymorphism (chosen)**: Each strategy own class, own tests, DI handles wiring. New channel = new file.

Enterprise mein clarity > brevity. Functional approach Day 1 attractive but Year 2 mein 50 channels = nightmare."

#### Q3: "Polymorphism violate kab kar sakte ho?"

"3 legitimate reasons:

1. **Performance-critical hot paths**: vtable cost ~1-5ns/call. HFT mein 1B calls/sec adds up. Mark `final`/`sealed` for devirtualization.
2. **Single implementation forever**: YAGNI — don't create interface for one impl.
3. **Marker interfaces**: type tagging without polymorphism (`Serializable`).

**Rule**: Wait for second implementation before extracting interface."

#### Q4: "Spring/JPA mein polymorphism manual ya automatic?"

"Spring **heavily** polymorphism pe based:
1. Bean container — interfaces everywhere
2. JPA — `EntityManager` interface, Hibernate impl
3. **Spring AOP proxies** — runtime-generated classes (CGLib) wrap your methods for `@Transactional` etc. Polymorphism + proxy + composition combo.
4. JDBC templates — `RowMapper` polymorphic callback.

Spring expects you to define interfaces. Framework itself built ON polymorphism."

#### Q5: "Production scale kaise?"

"Class-level + service-level:

**Service-level**: Daraz Payment Service. `PaymentProvider` interface — JazzCash, EasyPaisa, Stripe. New provider = new microservice + register. Zero orchestrator change.

**Numbers**: Netflix notification = 100M+ users, 50+ channels via Strategy + polymorphism.

**Performance cost**: ~5-10ns per call × 1T calls/day / 1000 servers = 10s/server/day = negligible.

**Bad scaling**: deep inheritance chains, 300+ subclass explosion. Solution: composition + multiple smaller interfaces."

---

### 🧩 Pattern Combinations

**Pairs Well With**:
- Strategy (Day 62) — strategy = polymorphism + intent
- Factory Method (Day 32) — factory returns polymorphic type
- Template Method (Day 66) — fixed skeleton, varying steps
- **Open/Closed (Day 12)** — polymorphism is the mechanism
- Dependency Inversion (Day 15) — depend on polymorphic interfaces

**Tension With**:
- YAGNI (Day 18) — premature interface = bad
- KISS (Day 17) — sometimes switch clearer
- Performance code — virtual dispatch prevents some JIT optimizations

---

### 🎓 Real Production: Spring's DelegatingPasswordEncoder

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}

@Bean
public PasswordEncoder passwordEncoder() {
    return new DelegatingPasswordEncoder("bcrypt", Map.of(
        "bcrypt", new BCryptPasswordEncoder(),
        "argon2", new Argon2PasswordEncoder(),
        "pbkdf2", new Pbkdf2PasswordEncoder("")
    ));
}
```

Stored format: `{algorithmId}encodedValue`
- `{bcrypt}$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy`
- `{argon2}$argon2id$v=19$m=4096,t=3,p=1$...`

**Magic**: `matches()` extracts prefix, looks up encoder in map, delegates (polymorphism).

**Why matters**: 2020 mein BCrypt, 2025 mein Argon2. **Migrate without forcing user resets**. Old hashes still verifiable, new ones Argon2. On successful login, silently re-hash with Argon2 and update.

**Polymorphism + Strategy + Factory all in one**. Seamless migration. Senior engineering.

---

### 💡 Memory Hook

**POLYMORPHISM = "BARTAN BADLO, KHAANA WOHI"**

Bartan (concrete class) badle, khaana (interface contract) wohi. Pateele/handi/degh — sab biryani pakate hain.

**"EK CALL, KAYI CHEHRE"**

- Overloading = "Same chai, alag pyaale" (compile-time, same class)
- Overriding = "Beta baap ki recipe mein twist" (runtime, parent-child)

---

### ⚠️ Common Mistakes

#### ❌ Mistake 1: `@Override` skip karna in Java

Typo (capital S in `Send`) without `@Override` → silent bug. Parent's empty method runs. Hours of debugging.

**Fix**: Always `@Override`. Compiler catches signature mismatch.

#### ❌ Mistake 2: Constructor mein overridable methods call karna

```java
class Parent {
    public Parent() { initialize(); }
    public void initialize() { ... }
}
class Child extends Parent {
    private String name = "default";
    @Override
    public void initialize() {
        System.out.println(name.toUpperCase());  // NPE! name null
    }
}
new Child();  // CRASH
```

Parent constructor → calls initialize() → JVM uses Dog's vtable → Dog's initialize runs → accesses uninitialized name.

**Fix**: Constructor mein only `private`/`final` methods call karo.

#### ❌ Mistake 3: Override equals() but forget hashCode()

Java contract: equal objects MUST have same hashCode. HashMap/HashSet break otherwise.

```java
User u1 = new User("ahmed@gmail.com");
User u2 = new User("ahmed@gmail.com");
u1.equals(u2);  // true
u1.hashCode() == u2.hashCode();  // FALSE (memory addresses)
Set<User> users = new HashSet<>();
users.add(u1);
users.contains(u2);  // FALSE!!
```

**Fix**: Both together. Lombok `@EqualsAndHashCode` or Java records.

---

## 🧠 Complete Solution Interlocking

```
1. USER ACTION (Angular)
   └─▶ Click "Forgot Password" → form submit → loading spinner
       └─▶ Same UI message success OR error (anti-enumeration)

2. NETWORK (HTTPS)
   └─▶ TLS encryption protects token in transit
       └─▶ Rate limiter at gateway (3/hour/IP)

3. API REQUEST (Spring/.NET)
   └─▶ POST → @Transactional begins DB transaction

4. BUSINESS LOGIC (Java/C#)
   └─▶ SecureRandom 32 bytes (CSPRNG)
       └─▶ SHA-256 hash created
       └─▶ Polymorphic notifier.send() dispatched

5. DATABASE (SQL)
   └─▶ INSERT tokens (hash, expires_at)
       └─▶ Unique index prevents hash collisions
       └─▶ Composite index for fast user queries

6. ASYNC EVENT (System Design)
   └─▶ Publish to Kafka
       └─▶ Notification service consumes
       └─▶ Email OR SMS sender (polymorphism)
       └─▶ Failure → DLQ for ops

7. USER CLICKS LINK (within 15-min window)
   └─▶ Angular form, token from URL
       └─▶ Password strength (debounced RxJS)

8. CONFIRM RESET
   └─▶ POST → atomic UPDATE tokens SET used=1 WHERE used=0
       └─▶ @@ROWCOUNT ensures single-use
       └─▶ BCrypt hash new password
       └─▶ DELETE all sessions

9. SUCCESS
   └─▶ 200 OK → redirect to login
       └─▶ Old JWTs/sessions invalid
```

**Skip Any Layer → Consequence**:

| Skip | Consequence |
|------|-------------|
| Angular generic response | UI enumeration |
| Rate limit | DDoS via spam |
| Token hashing | DB breach = all tokens leaked |
| Atomic UPDATE | Race → double consumption |
| Async queue | SMTP slowdown → API timeouts |
| Session invalidation | Old hacker session survives |
| TLS | Network sniffing |
| BCrypt | Plaintext passwords leaked |

---

## 🧭 Mental Map

```
                [PASSWORD RESET]
                       │
       ┌───────────────┼───────────────┐
       │               │               │
   [FRONTEND]      [BACKEND]       [DATABASE]
       │               │               │
   ReactiveForms   SecureRandom    token_hash CHAR(64)
   Same response   SHA-256 hash    UNIQUE INDEX
   Token from URL  @Transactional  Atomic UPDATE
   Strength meter  Polymorphic     used=1 WHERE used=0
                   @Version        ROWVERSION
       │               │               │
       └───────────────┼───────────────┘
                       │
               [SYSTEM DESIGN]
                       │
        Strategy Pattern + Event-Driven
        ┌────────┬────────┬────────┐
        │ Email  │  SMS   │  Push  │
        └────────┴────────┴────────┘
              all implement INotifier
              polymorphic dispatch
```

**Mental Story**:

Password reset = **shaadi ke invitation card system**:
- **Angular** = Card design
- **Backend** = Card printer
- **Token gen** = Unique serial number
- **SQL** = Guest registry
- **System Design** = Delivery (DHL/TCS/khud) — **sender alag, deliver same**

**Polymorphism** = chahe DHL ya TCS, method same `deliver(invitation, guest)`. Sender khud decide karega kaise.

**Acronym: TRACE**
- **T** — Token (SecureRandom, 32 bytes, hashed)
- **R** — Rate limit (3/hour)
- **A** — Atomic single-use (UPDATE WHERE used=0)
- **C** — Consistent response (no enumeration)
- **E** — Expiry + session invalidation

"Hacker ki **TRACE** mat chhodo."

---

## 🎤 INTERVIEW DRILL

### Main Question

**Q**: "Daraz ke liye password reset flow design karo. Security + multi-channel notification pluggable kaise?"

**Answer Structure (60-90 sec)**:

**Opening**: "5 layers coordinate karenge — frontend anti-enumeration, backend secure tokens, DB atomic single-use, async dispatch, polymorphic sender."

**Body**:
1. **Angular**: Reactive Form, identical success response, debounced strength meter.
2. **Spring/.NET**: Rate-limited endpoint (3/hr/IP), SecureRandom 32 bytes, SHA-256 hash, async via Kafka.
3. **Java/C#**: Polymorphic `NotificationSender`, DI resolves by user preference, new channel = new class (OCP).
4. **SQL**: Unique index on hash, composite (user_id, used, expires_at), atomic UPDATE pattern, ROWVERSION.
5. **Architecture**: Strategy Pattern + event-driven Kafka, DLQ for failures.

**Closing**: "Polymorphic dispatch ensures kal WhatsApp add karna ho — zero existing code change."

### Must-Know Concepts

1. **CSPRNG vs LCG**: SecureRandom uses OS entropy; Random uses predictable formula
2. **SHA-256 vs BCrypt**: fast for high-entropy tokens; slow for low-entropy passwords
3. **Optimistic vs Pessimistic locking**: version check vs row lock
4. **Vtable**: ~1-5ns per virtual call, JIT can devirtualize
5. **TOCTOU**: Time Of Check, Time Of Use — fix with atomic UPDATE
6. **Token bucket**: capacity + refill rate, self-recovering
7. **WAL**: log first, data later — durability mechanism
8. **Spring AOP proxies**: runtime-generated wrappers for `@Transactional`

### Counter-Questions

#### Q1 (Trade-off): JWT vs DB-stored reset token?

"JWT: no DB hit, scalable. BUT cannot revoke, replay-able.
DB-stored: revocable, single-use enforced. DB hit ~5ms.

**Choice for reset**: DB-stored. Security > convenience. Reset throughput low — 5ms irrelevant. Slack uses same approach."

#### Q2 (Scale): 10M users, 100K resets/day, SMTP 3s — architecture?

"100K/day = 1.2/sec avg, 12/sec peak.

**Sync (BAD)**: 12 concurrent × 3s = SES limit barely fits.

**Async (CORRECT)**: Save + publish Kafka → 50ms response. 20 workers parallel = 6.7 sends/sec sustained. 99th percentile <100ms. DLQ for failures.

10x scale: same architecture. 100x: regional sharding."

#### Q3 (Failure): Server crash mid-reset, user retries?

"4 scenarios:
1. Crash before any DB write → retry succeeds. Idempotent.
2. Crash after token UPDATE, before password UPDATE → token consumed, password unchanged → retry fails. **Fix**: both in same transaction.
3. Crash after both UPDATE, before response → DB consistent, user confused → idempotent retry detects 'already done'.
4. Crash after response, before client received → DB fine, user refresh → login works.

ACID + same transaction + idempotency = crash safety."

### Senior Differentiator

**Q**: "Sessions invalidate kaise after password reset?"

"Multi-pronged:

1. **Stateful sessions (DB)**: DELETE WHERE user_id = ? Simple, instant.

2. **JWT — 3 solutions**:
   - **Refresh tokens**: short-expiry access (15 min) + refresh token in DB. Delete refresh tokens on password change. Worst case: hacker 15 more min.
   - **Token versioning**: user.tokenVersion in JWT claim. Increment on password change → all old JWTs invalid. Cache version in Redis.
   - **Redis blocklist**: add jti to Redis set on revoke. TTL = remaining lifetime. Validation checks blocklist.

3. **Real-time push**: WebSocket/SSE notify clients to re-authenticate.

4. **Audit logging**: GDPR/PCI-DSS/SOC2 compliance.

**For Daraz**: stateful sessions for web + refresh tokens for mobile + Redis blocklist + WebSocket for connected admin.

Junior thinks 'kill the session'. Senior thinks 'across 100 servers, 10 regions, web+mobile+IoT, with compliance.'"

### Red Flags (Don't Say!)

- ❌ "Plaintext token storage" → DB leak = takeover
- ❌ "Email not found error" → enumeration
- ❌ "Never expire token" → screenshot exploitable
- ❌ "Math.random() / Random()" → predictable
- ❌ "Validate then mark used in separate steps" → TOCTOU
- ❌ "Sessions are separate concern" → incomplete security
- ❌ "Frontend show different messages for UX" → DevTools enumeration

---

## 📊 What You Should Explain After Today

1. ✅ CSPRNG kya hai, SecureRandom vs Random?
2. ✅ Hash function ki 3 properties, tokens kaise bachate hain?
3. ✅ BCrypt vs SHA-256 — kab konsa?
4. ✅ Salt — same password different hash kaise?
5. ✅ User enumeration attack + 3-layer prevention
6. ✅ TOCTOU bug + atomic UPDATE solution
7. ✅ Optimistic vs Pessimistic locking, @Version internals
8. ✅ WAL — crash durability mechanism
9. ✅ B-tree + leftmost-prefix rule
10. ✅ Polymorphism vs overloading, vtable mechanism
11. ✅ Strategy Pattern + Open/Closed via polymorphism
12. ✅ Spring AOP proxies — @Transactional internals
13. ✅ JWT vs DB-stored — trade-offs
14. ✅ Session invalidation strategies — 4 approaches

---

## 🔗 Tomorrow's Hint

**Day 5 — Product Listing + Composition over Inheritance**

Aaj polymorphism (inheritance-based) dekha. Kal **composition over inheritance** — kab inheritance avoid karke composition use karein. Scenario: Daraz product listing with filters/sort. SQL pagination (offset vs cursor) + index optimization.

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

**Streak**: 🔥 4 days strong, bhai! Aaj 2-3 baar padhna — pehle concepts, phir code, phir interview drill.
