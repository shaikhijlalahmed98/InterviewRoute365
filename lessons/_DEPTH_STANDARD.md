# 🎯 Depth Standard (Quality Bar for Every Lesson)

This is the **non-negotiable quality bar** for every daily lesson. Mentor must meet ALL these criteria before pushing.

---

## ✅ Mandatory Rules

### 1. **Inline Jargon Definition — NEVER Assume Knowledge**

❌ **BAD**: "Spring Boot mein SecureRandom (CSPRNG) JVM ke java.security.SecureRandom ko wrap karta hai."
*Reader doesn't know: CSPRNG, JVM, "wrap", entropy. Has to alt-tab and Google.*

✅ **GOOD**: "Pehle yeh samjho ke 'CSPRNG' kya hai. C = Cryptographically Secure, P = Pseudo (naqli), R = Random, N = Number, G = Generator. 'Pseudo' isliye kyunki computer asli random nahi bana sakta — woh deterministic machine hai. To woh formula se 'fake random' banata hai jo human ko random lage. 'Cryptographically secure' matlab — even after observing millions of past outputs, attacker can't predict the next bit. Aur 'wrap' ka matlab software mein..."

**Rule**: First mention of any technical term → 2-3 sentence inline explanation. No assumptions.

### 2. **Foundation Concepts BEFORE Code**

Don't dump code first. Build foundation in this order:
1. **Why** is this concept needed? (problem statement)
2. **What** is the concept? (definition + analogy)
3. **How** does it work? (step-by-step mechanism)
4. **Where** does it apply? (today's scenario)
5. **Now** show code (code becomes obvious after foundation)

### 3. **Roman Urdu Analogies for Every Abstract Concept**

Every abstract concept must have a Pakistani-context analogy:
- Encapsulation = ATM machine
- Polymorphism = Careem ride types (Go/Plus/Bike)
- Locking = Pandit's marriage register
- Hashing = Magic shredder machine
- DI = "Tum mere liye bana ke do, main nahi banaunga"

### 4. **Step-By-Step Mechanism Breakdown**

For internal mechanisms (vtable, change detection, transactions), use this format:
```
T+0: <what happens at time 0>
T+1: <next step>
T+2: <next step>
...
```

### 5. **Story Format Over Bullet Dumps**

❌ Lists of facts.
✅ "Imagine kar yeh scenario hai. Hacker yeh karta hai. Server yeh karta hai. Issue yeh ban jata hai. Solution yeh hai."

### 6. **Compare Bad Code vs Good Code**

Every principle/pattern section must show:
- ❌ BAD code (without principle)
- ✅ GOOD code (with principle)
- Why bad is bad (concrete failures)
- Why good is good (concrete benefits)

### 7. **Concrete Numbers, Not Vague Adjectives**

❌ "It's fast" → ✅ "1 microsecond per operation"
❌ "It's secure" → ✅ "2^256 possibilities = 10^77 combos = brute force impossible"
❌ "It scales" → ✅ "100M users, 50+ channels, 1B requests/day"

### 8. **Production Examples From Real Companies**

Every concept must reference real-world usage:
- Stripe uses HMAC-signed tokens with 1-hour expiry
- Netflix notification service handles 100M+ users via Strategy pattern
- Slack magic-link uses random + SHA-256 hashed in DB

---

## 📏 Quality Self-Check (Before Pushing)

Mentor must answer YES to all:

- [ ] Har technical term jo pehli baar aaya, inline define hua?
- [ ] Foundation concepts code se PEHLE explain hue?
- [ ] Har abstract concept ka Pakistani-context analogy hai?
- [ ] Internal mechanisms ka step-by-step breakdown hai?
- [ ] Story format use kiya, bullet dump nahi?
- [ ] ❌ vs ✅ code comparison hai for principles?
- [ ] Concrete numbers diye, vague adjectives nahi?
- [ ] Real company examples reference kiye?
- [ ] Roman Urdu memory hook diya?
- [ ] Cross-questioning drills with confident answers hain?
- [ ] Mental map + acronym mnemonic diya?

If any answer is NO → don't push. Rewrite.

---

## 🚫 Phrases to Avoid (Cause Confusion)

These shortcuts assume knowledge. Replace with inline explanation:

| ❌ Avoid | ✅ Replace With |
|----------|-----------------|
| "X wraps Y" | "X is a layer over Y. Y is OS-specific/low-level. X gives a simple interface so..." |
| "It's just AOP magic" | "Spring runtime pe ek proxy class banata hai jo tumhari class extend karti hai..." |
| "Standard CSPRNG behavior" | "CSPRNG = Cryptographically Secure PRNG. Yeh OS entropy use karta hai jo..." |
| "Uses the vtable" | "Vtable = virtual function table = har class ka array of function pointers..." |
| "Standard WAL approach" | "WAL = Write-Ahead Log. DB pehle log file mein change likhta hai..." |

---

## 📈 Evolution

This standard was established after Day 4 feedback: *"tmhari explanations vague hain aur depth lack karti hai. Beech mein technical words use karte ho jo mujhe alag se search karne parte hain."*

From Day 4 onwards, every lesson follows this standard.

**Evolution log**:
- Day 1-3: First attempts, surface-level
- Day 4 v1: Better structure but still jargon-heavy
- Day 4 v2: Rewrite with this standard applied ← **baseline established**
- Day 5+: This standard mandatory
