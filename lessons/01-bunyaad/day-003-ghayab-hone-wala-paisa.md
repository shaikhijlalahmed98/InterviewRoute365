# Day 3 — Ghayab Hone Wala Paisa

> 🎬 **ASAL LESSON: [`day-003-app.html`](./day-003-app.html) — browser mein kholo.** 3 core ideas (value vs reference, copy-semantics ka bug, boxing + Integer cache) + record/immutability code + SQL DECIMAL + Value Object pattern — Fatima ke ghayab waiver ki kahani mein. Yeh file = record + revision reference.
>
> **Problem:** Fee waiver diya, phir bhi poora fee kata — na error, na log • **Waqt:** ~1:30 • **Drill jawab: optional** (tracked)

---

## 🔍 W4 Recap — aaj ka core concept

- **WHY:** Har cheez heap bhejna mehnga (dabba+GC) — C# ne value types diye; magar copy-semantics na samajhna = chupke se ghalat paisa.
- **WHAT:** Value type (struct) = photocopy; reference type (class) = parchi. Boxing = value ko dabbe mein band kar ke godown bhejna.
- **HOW:** Struct pe assignment/pass/list-access = COPY — copy pe change asal ko nahi lagta. Java −128..127 ke Integer dabbay cache karta hai — 127 `==` true, 128 `==` false.
- **WHEN:** Paisa/tareekh = immutable record (Value Object); dabbon pe `==` kabhi nahi; loop mein wrappers nahi; money = decimal/BigDecimal/DECIMAL(19,4) teeno manzilon pe.

## ⚛️ Atoms

1. **A-003-1** — C# struct = value type: copy milti hai (photocopy); class = reference (parchi). Java mein sirf primitives value.
2. **A-003-2** — List se struct nikaalna COPY hai — change asal ko nahi lagta ("photocopy pe dastakhat"); mutable struct = evil.
3. **A-003-3** — Boxing = value ka heap-dabba; loop mein wrapper types = dabba factory (slow + GC).
4. **A-003-4** — Integer cache −128..127: 127 `==` true, 128 `==` false; dabbon pe hamesha `.equals()`.
5. **A-003-5** — Paisa exact PURE raste: C# `decimal` ↔ Java `BigDecimal` ↔ SQL `DECIMAL(19,4)` — float kabhi nahi.
6. **A-003-6** — Value Object: pehchan qeemat hai → immutable record, value-equality; Entity ki apni ID hoti hai.
7. **A-003-7** — Paise ka hisaab EK jagah (backend); UI/jobs sirf dikhate hain — warna rounding ke jhagray.

## 🎤 Interview Drill (sawal — tafseel app mein)

- **W1–W3:** Integer 127/128 (teesri dafa!) • double balance kyun nahi • SQL money column
- **C1 (trap):** list se struct nikaal ke −100 — print kya hoga? *(photocopy pe dastakhat)*
- **C2 (trap):** `Long total` loop — sahi jawab, reject kyun? *(dabba factory)*
- **K1 (chain):** struct kab → "mutable struct evil kyun?"
- **K2 (chain):** 0.1+0.2 ≠ 0.3 kyun → "float exist kyun karta hai phir?"
- **S1 (SQL):** Accounts table CREATE — balance type + ek constraint
- **P1 (pattern chain):** Value Object vs Entity → "sab se asaan auzaar?"
- **D1 (design):** 3 jagah fee calculation, rounding ka farq — fix?

## ✍️ Mere Jawab (optional — jitna diya, utna grade hoga)

```
W1:
W2:
W3:
C1:
C2:
K1:
K2:
S1:
P1:
D1:
```

**Time laga:** ___ min • **Energy (1–5):** ___

<details>
<summary>🔑 Key (mukhtasar — poori wazahat app mein)</summary>

W1: true phir false — cache −128..127, `==` dabba compare • W2: double binary approx — `decimal` • W3: `DECIMAL(19,4)`; FLOAT approximation, audit fail
C1: **500** — list ne photocopy di, waiver copy pe laga
C2: Long = har += pe naya dabba (10 lakh objects); fix: `long` (chhota l)
K1: chhoti value-jaisi immutable cheezein; cross: mutable struct har access pe copy deta hai — chupke se kuch nahi badalta
K2: binary mein 0.1 exact nahi (jaise decimal mein 1/3); cross: float scientific/graphics/ML ke liye — jahan approx theek hai
S1: `Balance DECIMAL(19,4) NOT NULL` + `CHECK (Balance >= 0)`
P1: VO = qeemat hi pehchan (Rs 500 = Rs 500), Entity = ID/shanakht; cross: `record`
D1: hisaab EK jagah (backend Money/service) — baqi sirf display

</details>

---

**Ho gaya (ya jitna hua)?** `git add -A && git commit -m "day-003 done"` — kal `AGLA DIN`.

*Kal ka hook: Day 4 — safai-wale (GC) ki poori kahani: usay kaise pata kaun kachra hai, generations, stop-the-world — aur heap dump mein Danish ke Dictionary ki laash. Forensic day. 🔬*
