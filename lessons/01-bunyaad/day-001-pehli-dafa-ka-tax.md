# Day 1 — Pehli Dafa ka Tax (Ek Problem × 5 Stacks)

> 🎬 **ASAL LESSON: [`day-001-app.html`](./day-001-app.html) — browser mein kholo.** Poori kahani wahan hai: launch night, Java/JVM (T+0 breakdown), .NET/CLR, SQL Server (plan cache + buffer pool), Angular/V8, System Design (Netflix/SnapStart/Dragonwell), Interlock table, twisted Interview Drill. Yeh file = record + revision reference.
>
> **Problem:** TICH-KOLEY ki pehli request 2.1 second — kyun? • **Waqt:** ~50 min • **Drill jawab: optional** (tracked)

---

## 🔍 W4 Recap — aaj ka core concept

- **WHY:** 1990s mein har CPU+OS ke liye alag binary — porting ka azaab ("write once, debug everywhere") ne beech-ki-zubaan ka idea paida kiya (1995 Java; 1997 muqadma → 2000 C#/Hejlsberg).
- **WHAT:** Do-phase compilation — build pe CPU-neutral bytecode/IL; runtime (JVM/CLR) chalte-chalte JIT se machine code banata hai.
- **HOW:** interpret + counters → C1 → C2 (profiling: inlining/devirtualization) — 900µs → 3µs (300x). Wohi pattern SQL plan cache + buffer pool aur browser V8 (Ignition→TurboFan) mein.
- **WHEN:** JIT = lamba chalne wala process (warm-up amortize, peak profiling); AOT = startup-critical (serverless/CLI; GraalVM/NativeAOT/SnapStart); design: cold instance ko traffic kabhi nahi — warm-up → LB join.

## ⚛️ Atoms

1. **A-001-1** — Java/C# DO dafa compile: build pe source→bytecode/IL (javac/Roslyn), runtime pe JIT→machine code (JVM/CLR).
2. **A-001-2** — Bytecode OS ke across portable, JVM-version forward NAHI (naya bytecode + purana JVM = UnsupportedClassVersionError).
3. **A-001-3** — JIT tiered: interpret+counters → C1 → C2 (300x) — warm-up slow; ilaaj: warm-up traffic ya AOT (qeemat: profiling nahi).
4. **A-001-4** — "Pehli dafa ka tax" HAR layer pe: SQL plan cache + cold buffer pool, V8, Angular AOT — ek hi pattern.
5. **A-001-5** — Design usool: cold instance ko traffic kabhi nahi — warm-up → readiness → LB join (Netflix, Lambda SnapStart, Dragonwell JWarmup).

## 🎤 Interview Drill (sawal — details app mein)

- **W1–W3 (warm-up):** stack/heap placement • static Dictionary leak • Integer cache ulta sawal
- **C1 (code trap):** `parse()` timing 900µs → 3µs — "bug hai?" *(thanda benchmark)*
- **C2 (code trap):** Java 21 compile, Java 8 JVM pe run — chalega? *(portability ka dhoka)*
- **K1 (concept + chain):** bytecode vs machine code → "JVM khud kis mein likhi hai?"
- **K2 (concept + chain):** JIT kab AOT se behtar → "phir Lambda pe AOT kyun?"
- **D1 (design twist):** rolling deploy, har naye pod pe pehli 100 requests slow — 2 fixes

## ✍️ Mere Jawab (optional — jitna diya, utna grade hoga)

```
W1:
W2:
W3:
C1:
C2:
K1:
K2:
D1:
```

**Time laga:** ___ min • **Energy (1–5):** ___

<details>
<summary>🔑 Key (mukhtasar — poori wazahat app mein)</summary>

W1: count→stack, reference→stack, object→heap • W2: static kabhi GC nahi → slowness (GC thrash) → OOM • W3: `true` phir `false` (cache −128..127; `==` reference)
C1: bug nahi, **JIT warm-up** — benchmark warm hone ke baad, percentiles mein *(thanda benchmark)*
C2: **nahi** — `UnsupportedClassVersionError`; bytecode OS-portable hai, JVM-version-forward nahi *(portability ka dhoka)*
K1: bytecode JVM chalata hai, machine code CPU khud; JVM khud C++ → machine code — turtles yahan rukte hain
K2: lamba process = JIT (warm-up amortize + profiling); serverless = AOT/SnapStart (tax har invocation pe parta)
D1: warm-up before LB join (readiness gate) + canary/gradual rollout (+ AOT/R2R/JWarmup bonus)

</details>

---

**Ho gaya (ya skip bhi theek hai)?** `git add -A && git commit -m "day-001 done"` — kal `AGLA DIN`.

*Kal ka hook: 3 hafte baad TICH-KOLEY phir slow — aur is dafa garam hone se theek NAHI hota. Day 2: "Woh memory jo kabhi wapas nahi aayi" — Java heap andar se + Danish ke static Dictionary ka muqadma.*
