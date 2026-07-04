# Day 0 — Pehla Interview: Jo Aaj Tum Haaroge

> **Level:** 🪨 Bunyaad • **Waqt:** ~45 min • **Format:** Simulated interview loop (5 rounds, 30 sawal)
> Yeh exam nahi hai. Yeh **woh interview hai jo aaj ke tum clear nahi kar sakte** — aur Day 365 pe yehi exact loop dobara hoga. Us din ka result aaj se likha ja raha hai.

---

## 📞 Call aati hai...

Jumeraat ki shaam hai. LinkedIn pe ek recruiter ka message: **Zarr Technologies** — Karachi ka sab se garam fintech, SBP license mila hai, digital bank launch kar rahe hain. Position: **Senior Software Engineer, Core Banking**. Paisa itna ke tum message do dafa parhte ho. Stack: Java + .NET + SQL Server + Angular. *Tumhara* stack.

Recruiter kehti hai: *"Profile strong lag rahi hai. Monday ko full loop rakh dein? 5 rounds — Java, .NET, Data, Web, aur aakhir mein CTO ke saath."*

Tum "haan" keh dete ho. Ab Monday hai. Office ki 14th manzil, sheeshay ki deewarein, aur panel tayyar hai.

**Khel ke rules (yeh asli interview hai, is liye):**
- **Google band, notes band, IDE band** — interview mein bhi yehi hota hai.
- Har jawab 1–2 jumlay. Nahi aata to **"nahi pata"** likho — interview mein bluff pakra jata hai, yahan "nahi pata" tumhara dost hai: har ghalat/khali jawab saal bhar ka personal review card banega. **Yahan haarna hi jeetne ka pehla qadam hai.**
- Jawab neeche `## ✍️ Mere Jawab` mein. Answer key `<details>` mein hai — pehle kholna apne aap se cheating.

---

## Round 1 — Java Panel: Bilal bhai (Principal Engineer)

*Thanda banda, dheemi awaaz, lekin sawal ustray jaise. Kehta hai: "Chaliye, halke se shuru karte hain."*

1. Yeh code kya print karega? <!-- A-000-01 -->
   ```java
   Integer a = 127, b = 127;
   Integer c = 128, d = 128;
   System.out.println((a == b) + " " + (c == d));
   ```
2. Java "pass-by-value" hai — phir method ke andar `list.add(5)` caller ko nazar aata hai, lekin `list = new ArrayList<>()` nahi. Kyun? <!-- A-000-02 -->
3. *"Humare ek junior ka bug hai yeh —"* ek `@Transactional` method ussi class ke doosre method se seedha call ho rahi hai, transaction lagta hi nahi. Kyun? <!-- A-000-03 -->
4. Loop mein 10,000 dafa `s += i` (String pe) sust kyun hai? Ilaaj kya? <!-- A-000-04 -->
5. Kisi class mein `equals()` override kiya magar `hashCode()` nahi — `HashMap` mein use karne pe kya masla hoga? <!-- A-000-05 -->
6. Spring `@Transactional` default mein kis qism ki exception pe rollback karta hai — checked ya unchecked? <!-- A-000-06 -->

## Round 2 — .NET Panel: Sana (Staff Engineer)

*Java se .NET migration khud jheli hai. Muskura ke kehti hai: "Aap bhi Java se aaye hain na? Phir yeh aasaan hoga..."*

7. Yeh code kya print karega? <!-- A-000-07 -->
   ```csharp
   struct Point { public int X; }
   var p1 = new Point { X = 1 };
   var p2 = p1;
   p2.X = 99;
   Console.WriteLine(p1.X);
   ```
8. `async void` method `async Task` ke muqable mein khatarnak kyun hai? <!-- A-000-08 -->
9. *"Yeh humara favourite reject-sawal hai:"* account balance `double` mein rakhna bug kyun hai? Sahi type kya hai? <!-- A-000-09 -->
10. LINQ: `var q = list.Where(x => x > 5);` ke BAAD list mein naya element add karo, phir `q.Count()` chalao — naya element count hoga ya nahi, aur kyun? <!-- A-000-10 -->
11. ASP.NET Core pipeline mein `UseAuthentication()` ko `UseAuthorization()` se pehle likhna zaroori kyun hai? <!-- A-000-11 -->
12. `object o = 42;` — is line mein andar kya hota hai, aur iski cost kya hai? <!-- A-000-12 -->

## Round 3 — Data Panel: Rehan sahab (DBA, 15 saal)

*Kursi pe peeche ho ke baithta hai: "Beta, main woh banda hoon jo tum jaise developers ki queries raat 3 baje theek karta hoon. Batao —"*

13. App mein code hai: "pehle SELECT se check karo CNIC hai ya nahi, na ho to INSERT karo." Phir bhi do parallel requests se duplicate CNIC ghus gaya. Kyun hua, aur pakka ilaaj kya hai? <!-- A-000-13 -->
14. Clustered aur nonclustered index ka farq ek jumle mein. <!-- A-000-14 -->
15. `WHERE UPPER(email) = 'ALI@X.COM'` — email pe index hone ke bawajood scan hota hai. Kyun? <!-- A-000-15 -->
16. `COMMIT` kamyab hua, ussi second bijli chali gayi. Data bachega? SQL Server yeh guarantee KAISE deta hai? <!-- A-000-16 -->
17. Paison ka column database mein kis type ka hona chahiye — aur `FLOAT` kyun nahi? <!-- A-000-17 -->
18. `GETDATE()` ke bajaye `SYSUTCDATETIME()` kyun — ek banking system mein is faisle ki wajah kya hai? <!-- A-000-18 -->

## Round 4 — Web Panel: Maha (Full-stack, security-obsessed)

*Laptop khol ke tumhari taraf ghuma deti hai: "Main aapko production ke asli zakham dikhati hoon —"*

19. 401 aur 403 ka farq ek jumle mein. <!-- A-000-19 -->
20. Client ne `POST /transfer` bheja, timeout hua, retry kiya — paisa do dafa chala gaya. API design mein kya kami thi? <!-- A-000-20 -->
21. JWT ko browser ke `localStorage` mein rakhna khatarnak kyun hai? Behtar jagah kya hai? <!-- A-000-21 -->
22. PUT aur POST mein se kaunsa contract ke mutabiq idempotent hai — matlab kya hua iska? <!-- A-000-22 -->
23. Email mein link hai: `GET /verify?token=abc123` — is design mein DO masle hain. Kaunse? <!-- A-000-23 -->
24. Angular 17 signals: `count = signal(0)` aur aam class property `count = 0` mein template-update ke hawale se bunyaadi farq kya hai? <!-- A-000-24 -->

## Round 5 — CTO Round: "Bar Raiser"

*CTO khud aata hai. Koi laptop nahi, sirf chai. "Framework sab bhool jao. Mujhe batao tumhe MACHINE samajh aati hai ya nahi —"*

25. Java method ke andar: `int x = 5; var obj = new Customer();` — `x`, `obj` (reference), aur Customer object — teeno kahan rehte hain (stack/heap)? <!-- A-000-25 -->
26. GC ka "generational hypothesis" ek jumle mein kya kehta hai? <!-- A-000-26 -->
27. Two Sum ko O(n) mein hal karne ke liye kaunsa data structure use hota hai, aur woh kya bachaata hai? <!-- A-000-27 -->
28. HashMap ka lookup O(1) kaise hota hai — aur do keys ka hash takra jaye (collision) to kya hota hai? <!-- A-000-28 -->
29. Sorted array (10 lakh items) mein ek number dhoondhna hai — kaunsa algorithm, kitni complexity? <!-- A-000-29 -->
30. *CTO chai rakh ke aage jhukta hai:* "Humari service pehle slow hui, phir OutOfMemory se mar gayi. Heap dump mein ek `static Map` mila jis mein croron entries thin. Kya hua tha, aur ilaaj kya?" <!-- A-000-30 -->

---

## 🤝 Interview khatam

Panel haath milata hai. Recruiter bahar chhorne aati hai: *"Feedback do din mein."*

Sach yeh hai: aaj ke tum is loop ke kuch rounds mein phislo ge — **aur yehi poore saal ka point hai.** Jab main grade karunga, har ghalat jawab tumhara zaati syllabus banega. Aur ek waada likh ke rakh lo: **Day 365 pe Zarr ka yehi loop, yehi 30 sawal, cold — aur us din tum panel ko apna khud ka banaya hua bank bhi dikhaoge.** Hiring committee ka jawab us din kuch aur hoga.

## 🏦 Ek aakhri kaam: apne bank ka naam

Interview ke raste ghar aate hue tum sochte ho: *"Naukri to yeh log denge ya nahi denge — lekin bank to main khud bana sakta hoon."* Yehi is saal ka asli project hai. Isko naam do jo repo, packages, aur Demo Day tak chale. (Chingari: *Bunyaad Bank, Qila Pay, Daryaa Bank* — magar naam TUMHARA hoga.)

---

## ✍️ Mere Jawab

> Har number ke aage jawab likho. Nahi aata to "nahi pata" — khali line se behtar hai.

```
R1-1:
R1-2:
R1-3:
R1-4:
R1-5:
R1-6:
R2-7:
R2-8:
R2-9:
R2-10:
R2-11:
R2-12:
R3-13:
R3-14:
R3-15:
R3-16:
R3-17:
R3-18:
R4-19:
R4-20:
R4-21:
R4-22:
R4-23:
R4-24:
R5-25:
R5-26:
R5-27:
R5-28:
R5-29:
R5-30:
```

**Bank ka naam:**

**Time laga:** ___ min • **Energy (1–5):** ___ • **Closed book raha? (haan/nahi):** ___

---

<details>
<summary>🔑 Answer Key — SIRF interview dene ke BAAD kholo</summary>

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

*Word count: ~1,300 (Day 0 special) • System ke mechanics (Airlock, ledger, keywords) ke liye: README.md — 2 minute ka parhna, jab dil kare.*

**Interview de diya? Ab:** `git add -A && git commit -m "day-000 done"` — phir `AGLA DIN` likhna. Main hiring-committee report dunga (round-wise), aur Day 1 shuru hoga: **"Compiler se CPU tak."**
