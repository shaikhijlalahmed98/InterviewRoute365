# 📚 Curriculum V2 — 365 Din ka Naqsha

> Padhne ka tareeqa: `COURSE_ENGINE.md` usool batata hai, yeh file **kya kab** batati hai. Day numbers completion pe barhte hain (calendar pe nahi). Har level ka darwaza **Mastery Gate** kholta hai, calendar nahi.
>
> **Format note (2026-07-05, learner ka faisla):** Har weekday = **EK problem × 5 stacks** (V1 pattern — engine §6). Neeche ke week/day breakdowns ab **concept-checklists** hain — mentor unhe problems ke andar bunta hai, alag-alag "type" wale dinon mein nahi. Daily drill optional (tracked); **Sunday comprehensive quiz compulsory** (hafte ka gate).

## Saal ka Arc — 7 Levels

| # | Level | Days | Weeks | Ek line mein pehchan |
|---|-------|------|-------|----------------------|
| 1 | 🪨 **Bunyaad** (Foundations Reforged) | 1–42 | 1–6 | "Main jaanta hoon machine ke andar kya hota hai" |
| 2 | 🏦 **Data ka Ustaad** | 43–84 | 7–12 | "Paisa data hai; main data kabhi corrupt nahi hone dunga" |
| 3 | ⚙️ **Backend Karigar** | 85–140 | 13–20 | "Framework mere liye jaadu nahi, machinery hai" |
| 4 | 🖥️ **Frontend Engineer** | 141–189 | 21–27 | "UI bhi engineering hai, sajawat nahi" |
| 5 | 🕸️ **Distributed Duniya** | 190–252 | 28–36 | "Network fail hota hai; mera system phir bhi sahi rehta hai" |
| 6 | 🏛️ **Meymaar** (The Architect) | 253–322 | 37–46 | "Main trade-offs mein sochta hoon, tools mein nahi" |
| 7 | 🎯 **The Finisher** (Interview Peak) | 323–365 | 47–52 | "Ab main perform karta hoon" |

Jama check: 42+42+56+49+63+70+43 = **365** ✓

---

## Level 1 — 🪨 Bunyaad (Days 1–42, W1–6)

**Exit competency:** Bina poochhe, English mein bata sake: `javac`/`csc` se machine code tak kya hota hai; stack vs heap; dono runtimes ka GC; value vs reference semantics (Java/.NET ka gehra tareen farq); string/equality/collection internals; HTTP pehle usoolon se; SQL Server row kaise store karta hai (pages, B-trees); ACID; aur kyun UNIQUE constraint app-level check ko hamesha harata hai.

**Kyun yeh tarteeb:** V1 ne Day 2 pe JWT parhaya tha — deewarein banne se pehle chhat. 2.2-GPA wala foundation gap pehle bharta hai — tez, tareekh ke saath, "what is a variable" jaisi tauheen ke baghair.

**Haftawar themes:** W1 runtime & memory (JVM vs CLR) • W2 SQL Server disk se upar tak • W3 HTTP & REST (**Wall Week** — halka, Day 15 = 30-min, Day 16 = Record Ceremony) • W4 collections & generics internals • W5 exceptions/errors + logging + Git fluency • W6 testing fundamentals (JUnit5/xUnit) + clean code + **Gate week**.

**Gate 1:** Sat practical: bounded LRU cache DONO languages mein, tests ke saath • Sun 40-MCQ + English viva: *"Walk me through what happens when a request hits your API and how memory is managed while serving it."*

**Capstone:** repo scaffold, docker-compose (SQL Server), Spring Boot monolith walking skeleton, `customers` schema asli constraints ke saath, pehla `POST /customers` (Java) + .NET mirror kata. **ADR-001: Modular monolith first** — learner khud likhta hai.

### Week 1 — "Code se Machine tak"
*Scenario: Apne bank ka naya onboarding service prod mein pehle slow hua, phir OutOfMemory se crash. Post-mortem hum khud karenge.*

- **Day 0 (pehla din, koi bhi weekday):** **"Zarr Loop"** — simulated interview at fictional Karachi fintech (5 rounds, 30-Q diagnostic, interactive app; ghalat jawab = pehle ledger atoms) + capstone ka naam. Day 365 pe yehi loop cold retake.
- **Day 1 (Mon):** *Compiler se CPU tak.* Source → bytecode/IL → JIT → machine code. Tareekh: 1995 Java "write once, run anywhere" (C/C++ porting ka azaab), 2000 .NET (J++ muqadma → Anders Hejlsberg → C#). Tiered compilation. 2-second pehli request ka raaz (JIT warm-up; AOT/GraalVM ek-line preview).
- **Day 2 (Tue, JAVA):** *Java memory ki sachai.* Stack frames, heap, "pass-by-value... of references" (screen-killer sawal hamesha ke liye hal), autoboxing traps (`Integer` cache −128..127 ka `==` ambush), static kahan rehta hai. OOM ka Java mujrim: static Map cache.
- **Day 3 (Wed, .NET):** *Value types — Java se sab se bara farq.* struct vs class (copy vs reference), boxing cost, string immutability dono taraf, struct chupke se heap pe kab jata hai. Contrast cards shuru.
- **Day 4 (Thu, DATA-flex):** *GC dono duniyaon mein + hamara OOM hal.* Generational hypothesis, G1/ZGC vs .NET Server GC, stop-the-world. Heap dump parhna (VisualVM / dotnet-counters). Do classic leak shapes.
- **Day 5 (Fri):** *Request aati hai to hota kya hai?* Thread pool, thread-per-request; slow-phir-dead ka anatomy (GC thrash → pauses → pool starvation → timeout cascade). Failure Museum #1: **Knight Capital 2012** ($440M, 45 min, deployment flag). DS&A #1: Big-O naap ke (string concat loop vs StringBuilder, timed) + Two Sum (Java).
- **Day 6 (Sat, LAB 1):** *Bank ki bunyaad.* Repo + docker-compose + Spring Boot skeleton (modules: customer/account/ledger khali) + health endpoints + **jaan boojh ke static-cache leak likho, VisualVM mein marte dekho, theek karo** + .NET minimal API mirror + ADR-001. Planted bug: SQL Server ka password complexity policy fail — container chupchaap exit (kyun?).
- **Day 7 (Sun):** Weekly Exam #1 (25-Q). Mock: **story mining #1** — mentor 8 sawal poochta hai uske asli GRC/banking kaam pe → STAR skeleton → 90-second English "Tell me about yourself" script → record.

### Week 2 — "Data ki Bunyaad: SQL Server disk se upar tak"
*Scenario: Onboarding mein duplicate CNIC ghus gaya (do parallel requests), aur customer-search 8 second leti hai.*

- **Day 8 (Mon):** Pages (8KB), heap vs clustered index, B-tree pehle usoolon se. Tareekh: Codd 1970 — pehle kya tha (IMS/CODASYL pointer-chasing ka azaab), declarative kyun jeeta.
- **Day 9 (Tue):** Indexes: 8s → 80ms. Nonclustered, key lookups, covering (INCLUDE), execution plan parhna (seek vs scan), sargability (`WHERE LTRIM(cnic)=` index ka qatil).
- **Day 10 (Wed):** Constraints: app-level check hamesha haarta hai. TOCTOU race timeline, UNIQUE hi asli ilaaj, NVARCHAR vs VARCHAR niyat se, dono stacks mein constraint violation pakarna (contrast cards).
- **Day 11 (Thu):** Transactions I: ACID asli zindagi mein. WAL (COMMIT tez magar crash-safe kyun — System R ki tareekh), THROW not RAISERROR, SYSUTCDATETIME discipline.
- **Day 12 (Fri):** Connection pooling (HikariCP vs ADO.NET — pool exhaustion = classic outage). Failure Museum #2: **RBS 2012 batch collapse** (6.5M log bank se bahar). DS&A #2: hash maps under the hood ("DB ne tree kyun chuna, hash kyun nahi? — range queries!") + Group Anagrams (C#).
- **Day 13 (Sat, LAB 2):** Customers schema + Flyway migration + EF Core mirror + **duplicate-CNIC race ko do parallel sessions se reproduce karo, UNIQUE lagao, race ko marte dekho**. Planted bug: ek migration mein GETDATE() + VARCHAR name column.
- **Day 14 (Sun):** Exam #2 (Week-1 atoms +7 pe wapas). Mock: English technical delivery — *"Walk me through what happens when you run a Java program"* + *"Why a unique constraint instead of checking in code?"* — record, 2 fixes.

### Week 3 — "HTTP aur REST: web ki zubaan" (WALL WEEK — jahan V1 mara tha, wahan V2 jashn manata hai)
*Scenario: Mobile team ne API integrate ki — sab toot gaya: ghalat status codes, CORS block, retry pe DUPLICATE customer.*

- **Day 15 (Mon — 30-MIN LIGHT DAY):** HTTP anatomy (raw request text mein parho), verbs ke waaday (safe/idempotent table), 5 zaroori status codes (200/201/400/401-vs-403/409), statelessness. Tareekh: Berners-Lee 1991, cookie ka janam (Netscape 1994). File ka aakhri jumla: *"Kal record tootega."*
- **Day 16 (Tue — RECORD-BREAK DAY):** Pehle **Wall Ceremony** (10 min): 15 dinon ka score-trend retro + record ka jashn. Phir compact lesson: DispatcherServlet lifecycle, @Valid, @RestControllerAdvice error contract.
- **Day 17 (Wed):** ASP.NET Core middleware pipeline (tarteeb ka mental model), minimal APIs vs controllers, ProblemDetails (RFC 7807). Mapping table: filters/interceptors vs middleware.
- **Day 18 (Thu):** DTO vs entity (over-posting/mass-assignment — banking bug class), Jackson vs System.Text.Json defaults, API versioning ek paragraph.
- **Day 19 (Fri):** **Idempotency — duplicate customer ka qatil.** Idempotency keys (client key + unique index + stored response). Currency rule war-story: state-changing GET kabhi nahi, secrets query string mein kabhi nahi (V1 ke apne Day-14 ka `GET /verify?token=` yahan theek hota hai — apna purana code audit karna sab se senior aadat hai). Failure Museum #3: **Bangladesh Bank 2016** ($81M, SWIFT, authZ). DS&A #3: two pointers (halka — Wall Week ki riayat).
- **Day 20 (Sat, LAB 3):** Asli `POST /customers` Java mein: validation + ProblemDetails + **idempotency-key filter** (Week-2 ka unique index kaam aya) + integration test: same request do dafa = EK customer. .NET mirror. Planted bug: validation failure pe 200-with-error-body.
- **Day 21 (Sun):** Exam #3 (spacing ab nazar aati hai: W1 atoms +16 ke qareeb, W2 +7 pe). Mock: **pehla mini system design (20 min, guided):** *"Design the onboarding API for 10k requests/day with idempotent submission"* → `interview/DESIGN-DOCS/DD-001.md`. Retro: **21 din — zindagi ki sab se lambi streak, likh ke.**

*(W4–W6: collections/generics internals • exceptions + logging + Git • testing + clean code → Gate 1. Har hafta isi Week-1–3 pattern pe mentor detail karega.)*

---

## Level 2 — 🏦 Data ka Ustaad (Days 43–84, W7–12)

**Exit competency:** Paison ka schema design (DECIMAL, kabhi float nahi — banks ne 1960s mein kyun seekha); har isolation level us anomaly ke saath jo woh rokta hai (+ SQL Server ka RCSI twist); blocking/deadlock diagnosis; query ke liye index + plan se saboot; JPA/Hibernate AUR EF Core (N+1, change tracking, lazy traps); **double-entry bookkeeping** (Pacioli 1494 — 500 saal purana pattern kyun zinda hai) — balance kabhi UPDATE nahi hota.

**Themes:** W7 transactions & isolation deep • W8 indexing & tuning • W9 ORM sach (JPA vs EF Core aamne-samne) • W10 double-entry + money types • W11 window functions, temporal tables, audit patterns • W12 migrations + Gate week.

**Gate 2:** slow + deadlocking transfer procedure diya jayega — isolation + indexing + THROW/idempotent-retry se theek karo (2-session race script ke against saboot) • viva: *"Explain isolation levels to a junior using a bank-transfer example."*

**Capstone:** **THE LEDGER** — taaj ka heera. `journal_entries` + `postings` (append-only, immutable), invariant SUM(debits)=SUM(credits) + nightly trial-balance job, accounts module, internal transfer via postings. Har interview mein yehi demo hoga.

## Level 3 — ⚙️ Backend Karigar (Days 85–140, W13–20)

**Exit competency:** DI containers + lifetimes (Spring context vs .NET host); AOP proxies — self-invocation `@Transactional` kyun torta hai; **transaction kabhi SMTP/network I/O ke upar nahi — Outbox default hai**; auth currency-rules ke mutabiq (jjwt 0.12+, ≥256-bit, JwtBearer, HttpOnly+SameSite; localStorage sirf flagged trade-off); async/concurrency (CompletableFuture vs async/await, virtual threads); testing pyramid (Testcontainers, WebApplicationFactory); resilience basics; idempotent APIs.

**Themes:** W13 DI internals • W14 AOP/proxies + `@Transactional` ka sach • W15 authN/Z (OAuth2/OIDC map + tareekh) • W16 validation + error contracts + versioning • W17 async & concurrency • W18 outbox + notifications + idempotency • W19 testing deep • W20 resilience + observability basics + Gate.

**Gate 3:** vulnerable API dono stacks mein secure karo (planted: localStorage token, GET-with-secret, SMTP-in-transaction) + **pehla full 45-min design mock:** *"Design the money-transfer API"* + viva.

**Capstone:** auth done right (web = HttpOnly cookie session; service-to-service = JWT), payments module (idempotency keys), **transactional outbox + email**, audit-event capture har state change pe.

## Level 4 — 🖥️ Frontend Engineer (Days 141–189, W21–27)

**Exit competency:** Angular 17+ sirf modern idioms — standalone, signals (signal/computed/effect — aur effect kab NAHI), `@if/@for/@defer`, functional interceptors/guards, `inject()`, typed forms; RxJS kahan genuinely jeetta hai (streams over time) vs signals (state); change detection under the hood (zones ki tareekh → zoneless); XSS/CSRF/CSP — tokens localStorage mein kyun nahi. Interviewer ko samjha sake ke Angular modules+zones se standalone+signals pe KYUN gaya.

**Themes:** W21 mental model reset (+tareekh) • W22 components + signals architecture • W23 typed forms + KYC form UX • W24 HTTP layer + interceptors + security • W25 state architecture (signal stores vs NgRx trade-off) • W26 performance (@defer, virtual scroll) + accessibility • W27 testing + Gate.

**Gate 4:** spec se account-dashboard feature (signals, @defer, typed form, interceptor) + viva: *"Explain signals vs RxJS to a React developer."*

**Capstone:** DO Angular apps — `admin-app` (customer search, audit viewer, maker-checker queue) + `customer-app` (login, accounts, transfer with idempotent double-submit-proof UX).

## Level 5 — 🕸️ Distributed Duniya (Days 190–252, W28–36)

**Exit competency:** Monolith KAB torna hai (aur kab nahi — strangler-fig ki kahani); bounded contexts; Kafka fundamentals (partitions, consumer groups, offsets, exactly-once ka sach); outbox→relay; **sagas vs 2PC (tareekh: XA kyun mara)**; idempotent consumers + DLQ; Redis patterns (cache-aside, stampede protection); Docker/K8s basics; distributed tracing; CAP/PACELC asaan Urdu mein, banking misaalon se. **Excursions:** Go (goroutines vs virtual threads vs async/await — rate-limiter banake mehsoos karo), Python (ops tooling).

**Gate 5:** diye gaye saga mein compensation bug hai jo paisa kho deta hai — dhoondo, theek karo, duplicate-delivery ke against idempotency saabit karo + 45-min design: *"Inter-bank transfer with an unreliable partner-bank API"* + viva.

**Capstone:** **THE SPLIT** — Payments service .NET 8 mein extract (strangler), Kafka backbone, saga (compensation ke saath), KYC screening service (.NET, async workflow), Redis balance read-model, tracing Java↔.NET. ADRs: why-split-now, saga-not-2PC, polyglot policy.

## Level 6 — 🏛️ Meymaar (Days 253–322, W37–46)

**Exit competency:** Poora 45-min open-ended system design senior ki tarah (requirements → estimates → API → data → design → deep-dive → failure modes → evolution); event-driven maturity + schema evolution; CQRS jahan jaayez (aur push-back jahan nahi); security architecture (threat modeling); multi-tenancy; **GRC/compliance architecture** (uska apna maidan, architecture level pe: immutable audit, maker-checker, SoD, regulatory reporting); fraud basics; performance engineering (k6/Gatling); ADR likhna aur difaa karna.

**Gate 6:** **Architecture Review Board** — poora capstone C4 + ADR pack mein pesh karo; mentor = hostile review board, 15 cross-questions (English, timed) + ek **unseen non-banking domain** ka full design (transfer ka saboot).

**Capstone:** fraud rules service (.NET) + Python scoring excursion, GRC reporting module (maker-checker + regulatory extracts), statement generation, CQRS read models (jahan reject kiya wahan ADR "where-we-said-no"), load-test report before/after.

## Level 7 — 🎯 The Finisher (Days 323–365, W47–52)

Weekdays = 60% consolidation (poora ledger ek aur chakkar) + 40% gap-filling (mock feedback se). Weekends = **full simulated onsite loops** (40-min DS&A + 45-min design + 30-min behavioral, back-to-back, English, recorded). CV/LinkedIn/portfolio finalize. 12 STAR stories polished.

**Final (Days 363–365):** Day 363–364: aakhri simulated loop + 100-Q comprehensive exam (ledger saabit karega ke Day-1 ka jawab Day-364 pe bhi aata hai — system ne bhoolna namumkin banaya) + **Zarr Day-0 loop ka cold retake — is dafa jeet ke**. **Day 365: DEMO DAY + "Defend Your Bank"** — 15-min English video walkthrough + 90-min viva jisme mentor-as-CTO har ADR pe hamla karta hai. Pass = graduation.

---

## Capstone Roadmap — Quarters

| Quarter | Theme | Architecture state |
|---|---|---|
| Q1 (D1–91) | **Bunyaad** — monolith + THE LEDGER | Modular monolith, enforced boundaries, trial balance green |
| Q2 (D92–182) | **Mazbooti** — auth, outbox, dono Angular apps | Hardened, secured, observable monolith |
| Q3 (D183–273) | **Taqseem** — the split | .NET Payments + Kafka + saga + KYC + Redis; polyglot event-driven |
| Q4 (D274–365) | **Kamaal** — fraud, GRC reporting, polish | Production-shaped platform + portfolio kit + Demo Day |

**Day 365 pe sach-much keh sakega:** *"I designed and built a cross-stack (Java + .NET + Angular 17 + SQL Server) event-driven banking platform from scratch — double-entry ledger, KYC with maker-checker, saga-based payments over Kafka, GRC audit reporting, fraud scoring — and evolved it from a modular monolith with a documented ADR at every step."*

---

## DS&A Pattern Ladder (Fridays + monthly screens)

W1–4 arrays/hashing/Big-O → W5–8 two pointers + sliding window → W9–12 stack/queue + binary search → W13–16 linked lists → W17–20 trees/BST → W21–24 recursion/BFS/DFS → W25–28 heaps → W29–32 graphs (sagas ke dependency-thinking wale hafton mein topological sort) → W33–36 intervals/greedy → W37–40 1-D DP → W41–44 2-D DP-lite + tries → W45–52 mixed timed company-style sets.

Volume: ~52 fresh + ~26 spaced re-solves + 24 screen problems ≈ **100+ encounters** — senior screens ke liye kaafi, LeetCode-lifestyle ke baghair. Language haftawar Java↔C#.

## Sunday Mock Rotation (4-hafta cycle)

**A:** DS&A screen (2 problems, timed, English think-aloud) • **B:** System design (ramp: 20-min guided → 45-min recorded) • **C:** Behavioral STAR (mining/rehearsal + recording) • **D:** Mixed mini-loop. Gate weekends rotation ko override karte hain.
