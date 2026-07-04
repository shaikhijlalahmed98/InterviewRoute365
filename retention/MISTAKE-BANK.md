# ❌ MISTAKE BANK — Har Ghalti, Verbatim

> Har ghalat quiz/exam jawab yahan aata hai: sawal, ghalat choice, sahi jawab, aur **trap ka naam** (yeh option kyun lubhata tha). Gate exams ~20% sawal yahin se uthate hain. Do lagatar sahi re-tests ke baad entry "retired" mark hoti hai (delete kabhi nahi).

| # | Day | Atom | Sawal (mukhtasar) | Meri ghalti | Sahi + kyun | Trap ka naam | Status |
|---|-----|------|-------------------|-------------|-------------|--------------|--------|
| 1 | D0 | A-000-01 | `Integer` 127/128 pe `==` ka output | "TRUE TRUE" | `true false` — −128..127 cached objects, `==` reference compare; 128 pe naye objects | Chhota number cache | active |
| 2 | D0 | A-000-03 | Same-class call pe `@Transactional` kyun nahi laga | "transaction sirf us method pe lagti hai" | Spring proxy sirf BAHAR se aane wali call intercept karta hai; `this.method()` proxy bypass | Proxy ghost | active |
