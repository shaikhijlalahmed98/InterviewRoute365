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

**Total memorized**: 5 acronyms (will grow daily).
