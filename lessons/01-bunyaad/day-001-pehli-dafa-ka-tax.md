# Day 1 — Pehli Dafa ka Tax (Beginner se shuru)

> 🎬 **ASAL LESSON: [`day-001-app.html`](./day-001-app.html) — browser mein kholo.** Aaj sirf 3 naye ideas: compiler, bytecode, JIT — launch-night ki kahani ke through. Yeh file = record + revision reference.
>
> **Problem:** TICH-KOLEY ki pehli request 2.1 second — kyun? • **Waqt:** ~30–40 min • **Drill jawab: optional** (tracked)

---

## 🔍 W4 Recap — aaj ka core concept

- **WHY:** Har CPU/OS ki machine code alag thi — har platform ke liye alag compile karna azaab tha ("write once, debug everywhere").
- **WHAT:** Beech ki zubaan (bytecode/IL) + har machine pe tarjuman (JVM/CLR) = ek hi file har jagah chalti hai. (C# = wohi idea: csc, IL, CLR — .NET Core/8+.)
- **HOW:** JVM pehle interpret karta hai (foran start, sust) aur bar-bar chalne wali "hot" methods ko JIT machine code bana deta hai (tez) — is "garam hone" (warm-up) mein waqt lagta hai.
- **WHEN:** "Deploy ke baad slow" suno to pehla shak warm-up pe karo — restart pe nahi (restart = cook ka pehla din wapas). Ilaaj: launch se pehle warm-up requests.

## ⚛️ Atoms

1. **A-001-1** — CPU sirf machine code samajhta hai; compiler tumhari zubaan ka tarjuma karta hai.
2. **A-001-2** — Java/C# DO dafa compile hote hain: build pe bytecode/IL, runtime pe JIT se machine code.
3. **A-001-3** — Bytecode/.class portable hai (har OS pe wohi file) — machine code har platform ka JVM/CLR khud banata hai.
4. **A-001-4** — JIT sirf HOT methods compile karta hai — pehli requests slow (warm-up), baad mein tez; restart = warm-up dobara.
5. **A-001-5** — "Pehli dafa ka tax" har jagah hai (SQL plan cache, browser JS) — deploy ke baad slow dekho to pehla shak warm-up pe.

## 🎤 Interview Drill (sawal — tafseel app mein)

- **W1–W3 (warm-up):** stack/heap • static Dictionary leak • Integer cache ulta sawal
- **C1 (code trap):** `parse()` timing 900µs → 3µs — "bug hai?" *(thanda benchmark)*
- **C2 (code trap):** Windows pe bani `.class` — Linux pe chalegi? *(portability confusion)*
- **K1 (concept + chain):** bytecode vs machine code → "JVM khud kaise chalti hai?"
- **K2 (concept + chain):** pehli request slow kyun → "restart se theek hoga?"
- **D1 (design):** agle launch pe pehle customer ko bhi 40ms kaise dein?

## ✍️ Mere Jawab (optional — jitna diya, utna grade hoga)

```
W1: skipped
W2: skipped
W3: skipped
C1: warm up time he
C2: ghalat. jvm kaam krle ga
K1: byte code beech ki language he or machine ki language machine code he jo har macine k lye alag
K2: warm up hota he system jvm intrepret kr rha hota he
D1: pehle se warmup req krdein
```

**Time laga:** 1 min • **Energy (1–5):** 4

<details>
<summary>🔑 Key (mukhtasar — poori wazahat app mein)</summary>

W1: count→stack, reference→stack, object→heap • W2: static kabhi delete nahi → pehle sust, phir OOM • W3: `true` phir `false` (chhote numbers cached; `==` pata compare karta hai)
C1: bug nahi — **JIT warm-up**; naapne se pehle garam karo *(thanda benchmark)*
C2: Danish **ghalat** — .class bytecode hai, OS ki cheez nahi; Linux ka JVM tarjuma kar lega *(portability confusion)*
K1: bytecode = beech ki zubaan (JVM chalata hai); machine code = CPU ki zubaan; JVM khud pehle se compiled aam program hai (C++) — kahin na kahin seedha machine code hi chalta hai
K2: JIT ne abhi compile nahi kiya (sab interpret) → slow; restart se theek NAHI — ulta warm-up dobara
D1: launch se pehle warm-up requests — JIT garam karo, phir customers

</details>

---

**Ho gaya (ya skip bhi theek hai)?** `git add -A && git commit -m "day-001 done"` — kal `AGLA DIN`.

---

## 🧑‍🏫 Mentor Evaluation (graded D1, 2026-07-05)

**Score: 5/5 attempted — sab sahi.** ✅ (Warm-up W1–W3 skipped — koi jurm nahi, woh atoms kal ke warm-up mein wapas aayenge.)

| Sawal | Verdict | Note |
|---|---|---|
| C1 (thanda benchmark) | ✅ | "warm up time he" — bilkul. JIT abhi garam nahi tha. |
| C2 (portability) | ✅ | "ghalat, JVM kaam kar lega" — perfect. Danish ko bacha liya tumne. |
| K1 (bytecode vs machine code) | ✅ (cross adhoora) | Farq sahi likha. Cross-chain ("JVM khud kaise chalti hai?") ka jawab nahi likha — yaad rakho: JVM khud C++ ka pehle-se-compiled aam program hai; kahin na kahin seedha machine code hi chalta hai. |
| K2 (slow kyun + restart?) | ✅ (cross adhoora) | Warm-up/interpret sahi. Restart wale cross ka jawab likhna reh gaya: restart se theek NAHI hota — ulta warm-up dobara shuru. |
| D1 (design) | ✅ | "pehle se warmup req" — yehi jawab tha. |

**Mentor note:** Kal 30 sawalon mein 1 sahi tha. Aaj 5 mein 5. Concepts jab hazam-size ke hon to tum bilkul pakar lete ho — system ab tumhare size ka hai. Aadat banao: **cross-chain wala doosra hissa bhi likha karo** — asli interview mein wohi follow-up hi banda girata hai.

**Ledger:** A-001-1 se A-001-5 paida hue (Due D2). W1–W3 ke atoms (A-000-25, A-000-30, A-000-01) ab bhi due hain — Day 2 ke warm-up mein naye libaas mein milenge.

*Kal ka hook: TICH-KOLEY 3 hafte baad PHIR slow — aur is dafa garam hone se theek nahi hoga, kyunke masla memory ka hoga. Day 2: "Woh memory jo kabhi wapas nahi aayi" — stack aur heap ki asli kahani + Danish ke static Dictionary ka muqadma.*
