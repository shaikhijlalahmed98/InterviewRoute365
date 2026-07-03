# 💡 Memory Hooks — All Lessons

Roman Urdu mnemonics collected from every lesson. Quick recall for interviews.

---

## Day 1 — User Registration

### **R.U.V.E.S** (Registration Flow)
- **R**eactive Form (Angular — validate first)
- **U**nique Constraint (SQL — no duplicates)
- **V**erification Token (UUID, 24hr expiry)
- **E**mail Send (async/queue for production)
- **S**ecure Hash (BCrypt — never plain text)

### **OOP**: Encapsulation = "DABBA"
"Data Access By Backed Access" — sirf approved tareeqe se data milega.
Analogy: ATM machine — andar ka mechanism dikhta nahi, sirf card daalo aur paise nikalo.

---

## Day 2 — User Login with JWT

### **VIBES** (Login Flow)
- **V**erify password (BCrypt)
- **I**ssue token (JWT sign)
- **B**earer header (Angular interceptor)
- **E**xpiry short (15 min)
- **S**ignature check (stateless verify)

### **JWT structure**: "Hath.Pet.Mohar"
- **Hath**er (header — algorithm info)
- **Pe**yload (claims — userId, role)
- **Mohar** (signature — proves authenticity)

### **OOP**: Abstraction = "Naqsha aur Naqshanavees"
Naqsha (interface) dikhta hai; naqshanavees (implementation) chhupa hota hai.

---

## Day 3 — User Profile Update

### **PCAVR** (Concurrent Update Protection)
- **P**ATCH semantics (partial update)
- **C**oncurrency token (@Version)
- **A**udit trail (BaseAuditableEntity)
- **V**alidation (DTO + business rules)
- **R**etry on conflict (UX)

### **OOP**: Inheritance = "Khandani Wirasat"
Parent ke gun (fields/methods) bachhon ko milte hain bina dohraye.
But "Liskov rule": bachhe ko parent ki promise tornay ka haq nahi.

---

## Day 4 — Password Reset Flow

### **TRACE** (Password Reset Security)
- **T**oken (SecureRandom, 32 bytes, stored as hash)
- **R**ate limit (3 attempts/hour)
- **A**tomic single-use (SQL UPDATE WHERE used=0)
- **C**onsistent response (no enumeration)
- **E**xpiry + session invalidation

"Hacker ki TRACE mat chhodo."

### **OOP**: Polymorphism = "BARTAN BADLO, KHAANA WOHI"
Bartan (concrete class) badle, khaana (interface contract) wohi.
- **Overloading** = "Sab apne pyaalon mein chai" (compile-time, same class)
- **Overriding** = "Beta baap ki recipe mein twist" (runtime, parent-child)

### **Hash function** = "Magic shredder machine"
Koi bhi document daalo, fixed-size unique receipt mile. Receipt se asli document banana impossible.

### **DI** = "Tum mere liye bana ke do, main nahi banaunga"
Class apni dependencies khud nahi banati — framework provide karta hai.

### **Optimistic Locking** = "Pandit ka Register"
Pandit puchta: "Tum kis version pe baat kar rahe ho?" Version mismatch = reject.

---

## Day 12 — Upload Profile Picture

### **SNAP** (Profile Picture Upload — photo khinchne wali SNAP!)
- **S**ignature + Size validate (magic bytes, NOT Content-Type/extension — spoofable)
- **N**o local disk → object storage (S3/Blob); DB stores URL/key only
- **A**sync transform: resize + EXIF strip (privacy/GPS) off the request thread
- **P**ublish via CDN with cache-busted versioned URL (`?v=N`)

### **OOP**: Open/Closed = "EXTEND HAAN, MODIFY NAA" + "USB-C PORT"
Naya device lagao (extend), phone mat kholo (modify). Port = interface; device = implementation.
New storage provider / transform = nayi class, `ProfilePictureService` untouched.

### **Magic bytes** = "File ki asli pehchaan, rename-proof"
JPEG=`FF D8 FF`, PNG=`89 50 4E 47`. Extension/Content-Type jhoot bol sakte hain.

### **Object storage** = "Godown + tag number"
File bade godown (S3) mein tag laga ke rakho; diary (DB) mein sirf tag number.

### **Orphan cleanup** = "Pehle likho, phir mitaao"
DB commit FIRST, phir purani file delete. Ulta kiya + commit fail = broken image forever.

---

## Day 13 — Form Validation Full-Stack

### **CASE-D** (Defense-in-Depth Validation)
- **C**lient validation = UX only (trust=0)
- **A**PI = security boundary (must re-validate)
- **S**ervice = business rules layer
- **E**nforce in DB = integrity guarantee
- **D**efense-in-depth across all layers

### **OOP**: Liskov Substitution = "NAYA KHILARI, WOHI RULES"
Subtype base ki jagah substitutable. Cricket team mein naya bowler aaye, wickets ki rules wahi rahein.
Validation engine living example: har validator same contract (`null`/`error`, never throw) → polymorphic loop safe.

### **TOCTOU race in signup** = "Pehle dekho, phir banao" (galat)
Check-then-insert mein race window — 2 requests dono "available" dekh ke dono insert kar dein.
Fix: DB UNIQUE constraint — single source of truth.

---

## Day 14 — Email Verification Flow

### **VERIFY** (Email Verification Security)
- **V**alidate token (SecureRandom 32-byte, SHA-256 hashed in DB)
- **E**xpiry enforced (24 hours, 410 Gone after)
- **R**ace-condition safe (atomic UPDATE WHERE is_used=0)
- **I**dempotent (already-verified = 200 OK, not error)
- **F**ire async email (Outbox pattern, not blocking)
- **Y**ield clear UI feedback (loading/success/expired/invalid)

### **OOP**: Interface Segregation = "BHARI INTERFACE MAT BHEJO" / "CHOTA INTERFACE, BARA SUKOON"
Bara fat `INotificationService` (sendEmail+sendSms+sendPush+sendWhatsApp) tor ke chote interfaces banao: `IEmailSender`, `ISmsSender`, `IPushNotifier`.
Analogy: shaadi vendors — har vendor ka apna interface, fat "IWeddingVendor" mat banao.

### **Outbox Pattern** = "DABBA-WALA email delivery"
Tiffin ka dabba (outbox row) tumhare ghar (DB transaction) mein safe rakho. Dabba-wala (background worker) le ja ke deliver karega — agar SMTP down ho, dobara try karega.

### **Anti-enumeration** = "Ghar ka chowkidar"
Bahar wala koi bhi puchne aaye — chowkidar same jawab deta hai. Existing/non-existing email pe same response = attacker email list nahi bana sakta.

### **Constant-time compare** = "Har baar 6 chars check, chahe pehla mismatch ho"
`.equals()` shortcuts on mismatch — timing leak. `MessageDigest.isEqual()` always full-length = no info leak.

---

## Day 15 — Phone OTP Verification

### **OTP-SECURE** (Phone OTP Defense)
- **O**TP hashed in DB (SHA-256, not plain)
- **T**hrottled requests (3/hour per phone via Redis INCR)
- **P**roviders chained (Twilio → Veevo → Jazz fallback)
- **S**ingle-use (DELETE on successful verify)
- **E**xpiry enforced (5 min)
- **C**onstant-time compare (FixedTimeEquals / isEqual)
- **U**pserted atomically (MERGE / ON CONFLICT)
- **R**edis rate-limited (atomic INCR single-thread)
- **E**rrors clear in UI (countdown timer, resend cooldown)

### **OOP**: Dependency Inversion = "INTERFACE PE BHAROSA, IMPLEMENTATION PE NAHI" / "SOCKET PATTERN"
Electric kettle wall socket pe depend karti hai, na ke specific power source (WAPDA/solar/K-Electric).
`OtpService` `ISmsSender` pe depend kare, na ke concrete `TwilioClient`. Provider swap = zero business code change.

### **Circuit Breaker** = "3 baar fail = 60s rukna" (ATM card block jaisa)
3 consecutive failures in 10s → circuit OPEN → 60s fail-fast → HALF-OPEN test → CLOSED ya wapas OPEN.

### **Token Bucket** = "Pani ki tanki + tap"
Tanki mein N tokens, har second R tokens add hote hain. Request aaye = 1 token consume. Tanki khali = reject. Burst allowed + sustained rate enforced.

### **Hexagonal Architecture** = "Core ke chaaron taraf adapter"
Business logic central hexagon mein. DB, SMS, queue — sab adapters jo port (interface) implement karte hain. Tech swap = adapter badlo, core untouched.

---

## Recurring Power-Words

### **CSPRNG** = "Cryptographically Secure Pseudo Random Number Generator"
OS entropy se bytes leta hai. Predictable nahi.

### **WAL** = "Write-Ahead Log"
DB pehle log file mein change likhta hai, phir actual data file. Crash recovery ka backbone.

### **ACID** = "Atomic, Consistent, Isolated, Durable"
- A — sab ya kuch nahi
- C — valid state always
- I — concurrent transactions don't see each other's uncommitted changes
- D — committed = persisted, crash mein bhi safe

### **TOCTOU** = "Time Of Check, Time Of Use"
Check aur use ke beech mein state change = bug. Fix: atomic single-statement UPDATE.

### **AOP** = "Aspect-Oriented Programming"
Behavior add karna code ko **bina modify kiye**, using proxies. Spring's `@Transactional` is the textbook example.

### **DI Container** = "Spring/.NET framework jo automatically classes wire karta hai"
Tum `@Service`/`@Component` annotation lagao, framework `new` keyword tumhare bajaye chala leta hai.

---

## Memory Anchor: Lesson Acronyms Quick List

| Day | Acronym | Meaning |
|-----|---------|---------|
| 1 | R.U.V.E.S | Registration flow steps |
| 2 | VIBES | Login flow steps |
| 3 | PCAVR | Concurrent update protection |
| 4 | TRACE | Password reset security |
| 12 | SNAP | Profile picture upload (Signature, No-disk, Async, Publish-CDN) |
| 13 | CASE-D | Defense-in-depth validation (Client, API, Service, Enforce, Defense) |
| 14 | VERIFY | Email verification security (Validate, Expiry, Race-safe, Idempotent, Fire async, Yield UI) |
| 15 | OTP-SECURE | Phone OTP defense (OTP hashed, Throttled, Providers chained, Single-use, Expiry, Constant-time, Upsert atomic, Redis-limited, Errors clear) |

**Total memorized**: 8 acronyms (will grow daily).
