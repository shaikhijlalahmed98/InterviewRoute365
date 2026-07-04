# Day 0 — Nizaam ka Tour + Baseline Diagnostic

> **Level:** 🪨 Bunyaad • **Waqt:** ~50–60 min (tour 10 + diagnostic 30 + naam & khat 10)
> **Aaj koi lesson nahi.** Aaj hum do kaam karte hain: (1) tumhe system ki chabi dete hain, (2) tumhara asli naqsha banate hain — kahan mazboot ho, kahan kacha hai. Ghalat jawab yahan **inaam** hai: har ghalti saal bhar ke liye tumhara personal review card ban jayegi.

---

## 1️⃣ System ka Tour (10 min — sirf parho)

**Yeh system kaise chalta hai:**

- **`AGLA DIN`** likho → main pehle tumhara pichla kaam check karta hoon, phir naya din deta hoon. **RULE 0 — Airlock:** jab tak pichle din ke jawab nahi bhare, naya din NAHI milega. V1 isi liye mara tha — quiz kabhi check nahi hue. Ab structurally namumkin hai.
- **REVIEW-LEDGER:** har lesson ke aakhir mein 3–5 "atoms" (chhoti chhoti testable baatein) bante hain. Har atom Leitner box mein jata hai — sahi jawab pe interval barhta hai (+1 → +3 → +7 → +16 → +35...), ghalat pe wapas girta hai. Roz ka warm-up ledger se aata hai. **Bhoolna is system mein allowed hi nahi.**
- **Masroof din?** `BUSY DAY` (15-min micro-day, max 4/mahina). **Gap ho gaya?** `WAPSI` — koi sharmindagi nahi, narm re-entry. **Lab mein phanse?** `MADAD`. **Kuch samajh na aaye?** `SAMJHAO X`.
- **Wall Week ka waada:** Days 13–17 jaan boojh ke halke honge — wahi jagah jahan V1 (Day 15) mara tha. Day 16 pe hum record tootne ka jashn manayenge.
- **Git hi streak hai:** har din ke aakhir mein commit. `git log --oneline` tumhari chain hai — usay tootne mat dena.

---

## 2️⃣ Baseline Diagnostic — 30 Sawal (30 min, CLOSED BOOK)

**Usool:** Google band, notes band, IDE band. Jo nahi aata, **khali chhor do ya "nahi pata" likho** — yeh bhi qeemti data hai, sharam ki baat nahi. Jawab chhote rakho (1–2 jumlay). Urdu ya English, jo tez ho. Answer key neeche `<details>` mein hai — **jawab likhne se pehle kholna cheating apne aap se hai.**

### Stack A — Java

1. Yeh code kya print karega? <!-- A-000-01 -->
   ```java
   Integer a = 127, b = 127;
   Integer c = 128, d = 128;
   System.out.println((a == b) + " " + (c == d));
   ```
2. Java "pass-by-value" hai — phir method ke andar `list.add(5)` caller ko nazar aata hai, lekin `list = new ArrayList<>()` nahi. Kyun? <!-- A-000-02 -->
3. Ek `@Transactional` method ussi class ke doosre method se seedha call ho rahi hai — transaction lagta hi nahi. Kyun? <!-- A-000-03 -->
4. Loop mein 10,000 dafa `s += i` (String pe) sust kyun hai? Ilaaj kya? <!-- A-000-04 -->
5. Kisi class mein `equals()` override kiya magar `hashCode()` nahi — `HashMap` mein use karne pe kya masla hoga? <!-- A-000-05 -->
6. Spring `@Transactional` default mein kis qism ki exception pe rollback karta hai — checked ya unchecked? <!-- A-000-06 -->

### Stack B — .NET / C#

7. Yeh code kya print karega? <!-- A-000-07 -->
   ```csharp
   struct Point { public int X; }
   var p1 = new Point { X = 1 };
   var p2 = p1;
   p2.X = 99;
   Console.WriteLine(p1.X);
   ```
8. `async void` method `async Task` ke muqable mein khatarnak kyun hai? <!-- A-000-08 -->
9. Account balance `double` mein rakhna bug kyun hai? Sahi type kya hai? <!-- A-000-09 -->
10. LINQ: `var q = list.Where(x => x > 5);` ke BAAD list mein naya element add karo, phir `q.Count()` chalao — naya element count hoga ya nahi, aur kyun? <!-- A-000-10 -->
11. ASP.NET Core pipeline mein `UseAuthentication()` ko `UseAuthorization()` se pehle likhna zaroori kyun hai? <!-- A-000-11 -->
12. `object o = 42;` — is line mein andar kya hota hai, aur iski cost kya hai? <!-- A-000-12 -->

### Stack C — SQL Server

13. App mein code hai: "pehle SELECT se check karo CNIC hai ya nahi, na ho to INSERT karo." Phir bhi do parallel requests se duplicate CNIC ghus gaya. Kyun hua, aur pakka ilaaj kya hai? <!-- A-000-13 -->
14. Clustered aur nonclustered index ka farq ek jumle mein. <!-- A-000-14 -->
15. `WHERE UPPER(email) = 'ALI@X.COM'` — email pe index hone ke bawajood scan hota hai. Kyun? <!-- A-000-15 -->
16. `COMMIT` kamyab hua, ussi second bijli chali gayi. Data bachega? SQL Server yeh guarantee KAISE deta hai? <!-- A-000-16 -->
17. Paison ka column database mein kis type ka hona chahiye — aur `FLOAT` kyun nahi? <!-- A-000-17 -->
18. `GETDATE()` ke bajaye `SYSUTCDATETIME()` kyun — ek banking system mein is faisle ki wajah kya hai? <!-- A-000-18 -->

### Stack D — Web / HTTP / Angular

19. 401 aur 403 ka farq ek jumle mein. <!-- A-000-19 -->
20. Client ne `POST /transfer` bheja, timeout hua, retry kiya — paisa do dafa chala gaya. API design mein kya kami thi? <!-- A-000-20 -->
21. JWT ko browser ke `localStorage` mein rakhna khatarnak kyun hai? Behtar jagah kya hai? <!-- A-000-21 -->
22. PUT aur POST mein se kaunsa contract ke mutabiq idempotent hai — matlab kya hua iska? <!-- A-000-22 -->
23. Email mein link hai: `GET /verify?token=abc123` — is design mein DO masle hain. Kaunse? <!-- A-000-23 -->
24. Angular 17 signals: `count = signal(0)` aur aam class property `count = 0` mein template-update ke hawale se bunyaadi farq kya hai? <!-- A-000-24 -->

### Stack E — Bunyaadein (memory, GC, DS&A)

25. Java method ke andar: `int x = 5; var obj = new Customer();` — `x`, `obj` (reference), aur Customer object — teeno kahan rehte hain (stack/heap)? <!-- A-000-25 -->
26. GC ka "generational hypothesis" ek jumle mein kya kehta hai? <!-- A-000-26 -->
27. Two Sum ko O(n) mein hal karne ke liye kaunsa data structure use hota hai, aur woh kya bachaata hai? <!-- A-000-27 -->
28. HashMap ka lookup O(1) kaise hota hai — aur do keys ka hash takra jaye (collision) to kya hota hai? <!-- A-000-28 -->
29. Sorted array (10 lakh items) mein ek number dhoondhna hai — kaunsa algorithm, kitni complexity? <!-- A-000-29 -->
30. Service pehle slow hui, phir OutOfMemory se mar gayi. Heap dump mein ek `static Map` mila jis mein croron entries thin. Kya hua tha, aur ilaaj kya? <!-- A-000-30 -->

---

## 3️⃣ Apne Bank ka Naam Rakho

Yeh saal bhar tumhara capstone hai — asli digital bank jo TUM banao ge, scratch se. Isko naam do jo repo, packages, aur Demo Day tak chale. (Chingari chahiye to: *Bunyaad Bank, Meezan-e-Code, Qila Pay, Daryaa Bank* — magar naam TUMHARA hoga.)

## 4️⃣ Day-365 ke Naam Khat (3 jumlay)

Teen jumlay likho: **"Day 365 pe main kaun banna chahta hoon?"** Yeh khat STATE.md mein pin hoga aur saal ke aakhir mein tumhe wapas milega.

---

## ✍️ Mere Jawab

> Har number ke aage jawab likho. Nahi aata to "nahi pata" — khali line se behtar hai.

```
A1:
A2:
A3:
A4:
A5:
A6:
B7:
B8:
B9:
B10:
B11:
B12:
C13:
C14:
C15:
C16:
C17:
C18:
D19:
D20:
D21:
D22:
D23:
D24:
E25:
E26:
E27:
E28:
E29:
E30:
```

**Bank ka naam:**

**Day-365 khat (3 jumlay):**
1.
2.
3.

**Time laga:** ___ min • **Energy (1–5):** ___ • **Closed book raha? (haan/nahi):** ___

---

<details>
<summary>🔑 Answer Key — SIRF jawab likhne ke BAAD kholo</summary>

1. `true false` — **Integer cache trap:** −128..127 cached objects hain, `==` reference compare karta hai; 128 pe naye objects bante hain. `.equals()` use karo.
2. Java reference ki **copy** (value) pass karta hai — copy se object mutate ho sakta hai, magar copy ko naya object dena caller ka reference nahi badalta. *Trap: "pass-by-reference" kehna.*
3. **Self-invocation:** Spring ka transaction proxy sirf bahar se aane wali calls intercept karta hai; `this.method()` proxy ko bypass kar deta hai.
4. String **immutable** hai — har `+=` naya object banata hai, poora copy hota hai (O(n²)). Ilaaj: `StringBuilder`.
5. Barabar objects alag buckets mein girenge — `map.get()` daal ke bhi **nahi milega**, duplicates bhi ban sakte hain. Contract: equals ⇒ same hashCode.
6. Sirf **unchecked** (RuntimeException/Error) pe. Checked exception pe default rollback NAHI hota — `rollbackFor` dena parta hai.
7. `1` — struct **value type** hai: `p2 = p1` poori **copy** banata hai; p2 badalne se p1 nahi badalta. (Java mein yeh class hota to `99` aata.)
8. `async void` ko **await nahi kar sakte** aur uski exception caller tak nahi aati — process gira sakti hai. Sirf event handlers ke liye; warna hamesha `async Task`.
9. `double` binary floating-point hai — 0.1 jaise amounts **exactly store nahi hote**, paisa leak hota hai. Sahi: `decimal`.
10. **Hoga** — LINQ query **deferred** hai: `Where` tab chalta hai jab enumerate karo (`Count()`), definition ke waqt nahi.
11. Authentication pehle **user ki pehchan** banata hai (`HttpContext.User`); Authorization ussi pehchan pe faisla karta hai — ulta karo to har authorize check anonymous user dekhega.
12. **Boxing:** int (value type) heap pe ek naye object mein lapet diya jata hai — allocation + baad mein GC cost. Loops mein qatil.
13. **TOCTOU race:** dono requests ne ek hi waqt check kiya (dono ko "nahi hai" mila), phir dono ne insert kiya. Pakka ilaaj sirf database ka **UNIQUE constraint** — app-level check hamesha haar sakta hai.
14. Clustered index **khud table hai** (rows usi tarteeb mein rakhi hain, is liye ek hi ho sakta hai); nonclustered alag structure hai jo rows ki taraf **ishara** karta hai.
15. **Sargability:** column pe function lagate hi index ki sorted tarteeb bekar ho jati hai — har row pe `UPPER` chala ke compare karna parta hai (scan).
16. Bachega. **Write-Ahead Logging:** COMMIT se pehle transaction log disk pe pakka likha jata hai; crash ke baad recovery log se replay karti hai. Durability ka "D" yehi hai.
17. `DECIMAL(19,4)` — exact base-10. `FLOAT` binary approximation hai: paise ghalat jama honge, audit fail. *(Yeh rule saal bhar quiz mein wapas aayega.)*
18. **UTC + precision:** GETDATE() server ke local timezone pe hai — DST/region badalne pe audit trail ka bharosa toot jata hai. Banking mein waqt ka ek hi sach hona chahiye: UTC.
19. **401** = pehchan hi nahi (login/token nahi ya ghalat); **403** = pehchan hai, magar **ijazat nahi**.
20. **Idempotency key nahi thi.** POST idempotent nahi hota — client ek unique key bheje, server usay UNIQUE index se pakre aur dobara aane pe pehla hi result lautaye.
21. localStorage ko **koi bhi XSS script parh sakti hai** — token chori = account chori. Behtar: **HttpOnly + Secure + SameSite cookie** (JavaScript se parhi hi nahi ja sakti).
22. **PUT.** Idempotent = ek dafa karo ya das dafa, **aakhri haalat wohi** — is liye PUT retry-safe hai, POST nahi.
23. (a) **State-changing GET** — GET safe hona chahiye; email scanners/prefetchers link khud khol ke verify kar dete hain. (b) **Secret query string mein** — logs, browser history, referrer headers mein leak hota hai.
24. Signal **trackable container** hai — `count.set(5)` pe Angular ko theek pata hai kya badla aur sirf wohi update hota hai; aam property pe change detection ko andaza lagana parta hai (zones ka purana daur).
25. `x` → **stack** • `obj` reference → **stack** • Customer object → **heap**. *(Yeh Bunyaad ka pehla mantra banega.)*
26. **Zyada tar objects jawani mein marte hain** — is liye GC nayi generation ko baar baar (sasta) aur purani ko kam (mehnga) check karta hai.
27. **HashMap** (value → index): har element pe "complement dekha hai?" O(1) mein poochte ho — nested loop ka O(n²) bach jata hai.
28. Key ka **hash** bucket ka pata deta hai (array index — O(1)). Collision pe ussi bucket mein chain/list banti hai (Java 8+: lambi ho to red-black tree).
29. **Binary search** — O(log n). 10 lakh items ≈ sirf ~20 muqablay.
30. **Unbounded static cache = memory leak:** static Map kabhi GC nahi hota, barhta gaya → GC thrash (slow) → OOM (maut). Ilaaj: bounded cache (LRU/TTL — Caffeine/MemoryCache) ya cache bahar (Redis). *(Week 1 ka poora scenario yehi hai — Lab 1 mein isay khud maroge.)*

</details>

---

*Word count: ~1,150 (Day 0 special — tour + diagnostic; aam weekday cap ≤900 Day 1 se lagu hoga.)*

**Ho gaya? Ab:** `git add -A && git commit -m "day-000 done"` — phir kal `AGLA DIN` likhna. Main check karke Day 1 dunga: **"Compiler se CPU tak."**
