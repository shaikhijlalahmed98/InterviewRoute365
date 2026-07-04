# Day 2 — Woh Memory Jo Wapas Nahi Aayi

> 🎬 **ASAL LESSON: [`day-002-app.html`](./day-002-app.html) — browser mein kholo.** 3 core ideas (stack, heap, static/lifetime) + C# IMemoryCache code + SQL table/queries + Singleton pattern + 2-server architecture twist — Danish ke static Dictionary ke muqadme ki kahani mein. Yeh file = record + revision reference.
>
> **Problem:** 3 hafte baad TICH-KOLEY phir slow, phir OOM — warm-up ka masla nahi • **Waqt:** ~1:30 • **Drill jawab: optional** (tracked)

---

## 🔍 W4 Recap — aaj ka core concept

- **WHY:** Program ko do umar ki memory chahiye — chhoti (method ke locals) aur lambi (objects jo baad mein bhi chahiye).
- **WHAT:** Stack = plates ka meenar (locals + references; return = plate gone). Heap = godown (objects; parchi/reference se pahunchte ho).
- **HOW:** Object zinda jab tak koi reachable parchi hai; lawaris ko GC uthata hai; static parchi kabhi nahi phatti — us se bandhi cheez kabhi nahi marti (= leak).
- **WHEN:** Jo bhi jama karo (Map/cache/SQL table) — usi waqt poochho "niklegi kab?" (bound + expire). Memory chart sirf upar = restart nahi, heap dekho. Shared state (counters/sessions) kabhi per-server memory mein nahi — DB/shared cache.

## ⚛️ Atoms

1. **A-002-1** — Stack = plates ka meenar: har method call ki plate (locals+refs); return pe poori plate gone — isi liye tez.
2. **A-002-2** — Objects heap (godown) mein; stack pe sirf reference (parchi). Do parchiyan, ek object — mumkin.
3. **A-002-3** — Object tab tak zinda jab tak koi reachable parchi; lawaris ko GC uthata hai.
4. **A-002-4** — Static reference kabhi nahi marta — barhti hui static collection = classic leak ("bhoola hua reference").
5. **A-002-5** — Har jama hone wali cheez pe: "niklegi kab?" (bound + expire — Map/IMemoryCache/SQL retention job); leak ka pehla symptom slowness (GC pressure).
6. **A-002-6** — 2+ servers pe per-instance memory sab ko alag dikhti hai — shared counters/controls DB ya shared cache mein, warna security rules chupchaap tootte hain.
7. **A-002-7** — Singleton = EK instance poore program mein; mutable data ke saath global-state anti-pattern — DI container (AddSingleton / Spring @Service default) yeh manage karta hai.

## 🎤 Interview Drill (sawal — tafseel app mein)

- **W1–W3 (warm-up, kal ke skipped):** stack/heap placement • leak ka pehla symptom slowness kyun • Integer `==` vs `.equals()`
- **C1 (trap):** method khatam — return hua object zinda kaise? *(plate pe parchi thi, object nahi)*
- **C2 (trap):** `update(c)` mein setName + `c = new ...` — caller kya dekhega? *(parchi ki copy — A-000-02 ka hisaab)*
- **K1 (chain):** stack tez kyun → "to sab kuch stack pe kyun nahi?"
- **K2 (chain):** GC ke hote leak kaise → "pehla symptom slowness kyun?"
- **S1 (SQL):** pichle 30 min ki failed attempts ka COUNT — query likho
- **P1 (pattern chain):** Singleton + khatra → "framework mein kaise milta hai?"
- **D1 (design):** in-memory tracking ke 2 hifazati kaam
- **D2 (architecture twist):** 2 servers, 10 attempts, lock kyun nahi hua? *(do dukaanein, alag register)*

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
D2:
```

**Time laga:** ___ min • **Energy (1–5):** ___

<details>
<summary>🔑 Key (mukhtasar — poori wazahat app mein)</summary>

W1: x→stack, c (parchi)→stack, object→heap • W2: GC bar bar bhaagta hai, utha kuch nahi pata — bekaar mehnat CPU khaati hai • W3: `==` true (50 cached, same object), `.equals()` true (value) — `==` pata, `.equals()` qeemat
C1: plate pe sirf parchi thi; return ne parchi ki copy caller ko di — object ki zinda parchi maujood → zinda *(plate pe object nahi, parchi thi)*
C2: "Ali" — parchi ke raste object badla (dikha); apni parchi pe naya pata likha (nahi dikha). Reference ki COPY pass hoti hai *(parchi ki copy)*
K1: rakhna/uthana = meenar ka sirra badalna, poori plate ek saath phenki jati; cross: jo cheez method ke baad chahiye ya share karni ho, woh plate pe nahi reh sakti
K2: GC sirf LAWARIS uthata hai — static se bandha kabhi lawaris nahi (leak = bhoola hua reference); cross: bharta godown = GC ki lambi-bar bar nakaam safai = CPU = slowness
S1: `SELECT COUNT(*) FROM FailedLogins WHERE Cnic=@cnic AND AttemptAt > DATEADD(MINUTE,-30,SYSUTCDATETIME())`
P1: EK instance poore program mein; khatra mutable global state; cross: DI container — AddSingleton / @Service default singleton
D1: bound (size limit) + expire (TTL) — IMemoryCache/Caffeine + memory alert
D2: har server ka apna register — 5/5 batt gaye; fix: shared jagah (FailedLogins table / shared cache) *(do dukaanein, alag register)*

</details>

---

**Ho gaya (ya jitna hua)?** `git add -A && git commit -m "day-002 done"` — kal `AGLA DIN`.

*Kal ka hook: Day 3 — C# ka sab se bara farq: **struct** — jo godown jaate hi nahi, poore ke poore plate pe rehte hain. Aur Day-0 ke "TRUE TRUE" ka asli post-mortem: **boxing**. Bilal bhai ka hisaab poora hoga.*
