# 🃏 REVIEW LEDGER — Spaced Repetition ka Dimagh

> **Usool (COURSE_ENGINE.md §7):** Har atom ek row. Intervals (day-numbers mein): Box 1→+1, 2→+3, 3→+7, 4→+16, 5→+35, 6→+75, 7→+150 (retired-but-sampled). Sahi → Box+1. Ghalat → Box = max(1, Box−2), Lapses+1, kal warm-up mein pakka slot. Mentor har grading ke baad yeh table update karta hai; roz ka warm-up `Due ≤ current day` se top-3 uthata hai (sab se overdue pehle, phir sab se zyada lapses).

| ID | Atom (ek line) | Born | Box | Last | Due | Lapses | Hook (learner ka likha) |
|----|----------------|------|-----|------|-----|--------|--------------------------|
| A-000-01 | Integer −128..127 cached; `==` reference compare, `.equals()` value | D0 | 1 | D0 | D1 | 0 | — |
| A-000-02 | Java pass-by-value: reference ki COPY jati hai — mutate dikhta hai, reassign nahi | D0 | 1 | D0 | D1 | 0 | — |
| A-000-03 | `@Transactional` self-invocation = proxy bypass — bahar se call aaye tabhi lagta hai | D0 | 1 | D0 | D1 | 0 | — |
| A-000-04 | String immutable — loop mein `+=` O(n²); ilaaj StringBuilder | D0 | 1 | D0 | D1 | 0 | — |
| A-000-05 | equals override to hashCode bhi — warna HashMap mein cheez kho jati hai | D0 | 1 | D0 | D1 | 0 | — |
| A-000-07 | C# struct = value type: assignment poori COPY banata hai | D0 | 1 | D0 | D1 | 0 | — |
| A-000-08 | `async void` await nahi ho sakta, exception unobserved — sirf event handlers | D0 | 1 | D0 | D1 | 0 | — |
| A-000-09 | Paisa kabhi double nahi — C# `decimal` (exact base-10) | D0 | 1 | D0 | D1 | 0 | — |
| A-000-10 | LINQ deferred: query enumerate hone PE chalti hai, likhne pe nahi | D0 | 1 | D0 | D1 | 0 | — |
| A-000-11 | UseAuthentication (pehchan) pehle, UseAuthorization (ijazat) baad — tarteeb farz | D0 | 1 | D0 | D1 | 0 | — |
| A-000-12 | Boxing: value type heap object mein lipat-ta hai — allocation + GC cost | D0 | 1 | D0 | D1 | 0 | — |
| A-000-13 | Check-then-insert = TOCTOU race; pakka ilaaj sirf DB UNIQUE constraint | D0 | 1 | D0 | D1 | 0 | — |
| A-000-14 | Clustered index = khud table (ek hi); nonclustered = alag structure + pointer | D0 | 1 | D0 | D1 | 0 | — |
| A-000-15 | Sargability: column pe function = index bekar = scan | D0 | 1 | D0 | D1 | 0 | — |
| A-000-16 | Durability = WAL: COMMIT se pehle log disk pe; recovery log se replay | D0 | 1 | D0 | D1 | 0 | — |
| A-000-17 | Paison ka column DECIMAL(19,4) — FLOAT binary approximation, audit fail | D0 | 1 | D0 | D1 | 0 | — |
| A-000-18 | SYSUTCDATETIME (UTC) — GETDATE local/DST pe audit ka bharosa torta hai | D0 | 1 | D0 | D1 | 0 | — |
| A-000-19 | 401 = pehchan nahi; 403 = pehchan hai, ijazat nahi | D0 | 1 | D0 | D1 | 0 | — |
| A-000-20 | POST retry-safe banane ko idempotency key + UNIQUE index + stored response | D0 | 1 | D0 | D1 | 0 | — |
| A-000-21 | localStorage XSS-readable — tokens HttpOnly+Secure+SameSite cookie mein | D0 | 1 | D0 | D1 | 0 | — |
| A-000-22 | PUT idempotent by contract (aakhri haalat wohi); POST nahi | D0 | 1 | D0 | D1 | 0 | — |
| A-000-23 | State-changing GET kabhi nahi; secret query string mein kabhi nahi (logs/referrer/scanners) | D0 | 1 | D0 | D1 | 0 | — |
| A-000-24 | Signal = trackable container — Angular ko theek pata hai kya badla (zones ka ilaaj) | D0 | 1 | D0 | D1 | 0 | — |
| A-000-25 | Locals + references stack pe; objects heap pe | D0 | 1 | D0 | D1 | 0 | — |
| A-000-26 | Generational hypothesis: zyada tar objects jawani mein marte hain | D0 | 1 | D0 | D1 | 0 | — |
| A-000-27 | Two Sum O(n): HashMap mein complement dhoondo — O(n²) nested loop bach gaya | D0 | 1 | D0 | D1 | 0 | — |
| A-000-28 | Hash → bucket index O(1); collision pe chain (Java 8+: tree agar lambi ho) | D0 | 1 | D0 | D1 | 0 | — |
| A-000-29 | Sorted array + binary search = O(log n) — 10 lakh ≈ 20 muqablay | D0 | 1 | D0 | D1 | 0 | — |
| A-000-30 | Unbounded static cache = leak → GC thrash → OOM; ilaaj bounded (LRU/TTL) ya Redis | D0 | 1 | D0 | D1 | 0 | — |

> A-000-06 (@Transactional unchecked rollback) Day 0 pe SAHI tha — ledger mein nahi daala; W14 (proxies/transactions week) mein naya atom ban ke aayega.
