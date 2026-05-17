# 💬 Chat Response Template (Daily Format)

This is the **fixed format** for daily chat responses. Mentor follows this every day.

---

## 📌 Day X/365 — Routine Complete ✅

### 🛠️ Aaj Ka Kaam Ka Log (Step By Step)

Sequential list of what was done today:

**Phase A — Quiz Auto-Evaluation (FIRST step every routine)**
1. **Scan all `quizzes/*.md` files** — detect any with filled answers (`**Your Answer**: A/B/C/D`)
2. For each filled quiz NOT yet evaluated (or with new answers since last eval):
   - Grade against correct answer key
   - Append `## 📊 Evaluation — <date>` section to the file
   - Update `progress.md` Score column
   - Mention in chat summary: *"Day X quiz fill mila tha — evaluated: 38/50"*
3. Push evaluation updates

**Phase B — New Day Lesson**
4. Day detect kiya (GitHub se confirm)
5. Curriculum lookup (scenario + OOP focus)
6. Lesson content draft kiya (~50-100 KB) → push to `lessons/`
7. Revision keys file banayi (~3-4 KB) → push to `revision/`
8. Quiz file banayi (50 MCQs with hidden answer key) → push to `quizzes/`
9. `_STACK_INDEX.md` updated with new row
10. `progress.md` updated with new row (lesson + revision + quiz links)
11. Cheatsheets updated if applicable (memory-hooks, oop-principles, design-patterns)
12. Email draft banayi
13. Yeh chat summary banayi

### 📊 Evaluation Mention In Chat Summary

If any quizzes were evaluated today, mention briefly in chat:

```
## 📊 Quiz Evaluations Done Today

- ✅ Day 4 quiz: 42/50 (84%) — solid! See evaluation in file.
- ⚠️ Day 2 quiz: 22/30 (73%) — weak on JWT structure. Re-read §Stack-1.
```

### 🚀 GitHub Pe Kya Push Kiya
- **File**: `lessons/day-XXX-topic-slug.md` (e.g. `day-005-product-listing.md`)
- **Size**: XX KB | **Commit**: `<sha>`
- **Link**: [GitHub pe lesson kholo](<url>)

**Naming rule**:
- `day-XXX` = 3-digit zero-padded day number (sorts correctly to day-999)
- `-topic-slug` = kebab-case short topic name (no level, no date)
- Example: `day-005-product-listing.md`, `day-051-saga-pattern.md`

### 📧 Email
- **Status**: Gmail draft ID `<id>`
- **Action**: Gmail → Drafts kholke send karo

### 🎯 Aaj Ka Topic
**<Scenario> + <OOP/Pattern Focus>**

### 🗺️ Lecture Kaise Approach Karein (45-60 min)

**Phase 1 — Setup (5 min)**: scenario timeline padho
**Phase 2 — Attacks/Challenges (10 min)**: 4 gotchas story-style
**Phase 3 — Foundations (20 min)** ⭐ sabse important: foundation concepts ek-ek karke
**Phase 4 — Stacks (15 min)**: .NET → SQL → Angular → System Design
**Phase 5 — OOP/Pattern + Interview Drill (10 min)**: cross-questions bolke practice

### 💡 Aaj Ke 3 Memory Hooks
- **<ACRONYM>** = <meaning>
- **"<Roman Urdu phrase>"** = <concept>
- **<emoji metaphor>** = <concept>

### 🎤 Aaj Ka Interview Question
*"<main question>"*

Full answer structure GitHub pe.

### 🔗 Kal (Day X+1)
**<next topic> + <next OOP focus>**

---

## ⚙️ Rules

1. **No lamba detail yahan** — woh lesson file mein hai
2. **Scannable bullets only** — user 30 seconds mein read kar sake
3. **Hyperlinks for navigation** — GitHub link sabse top
4. **End with next-day hint** — momentum maintain
5. **Roman Urdu + English mix** — friendly bara-bhai tone
