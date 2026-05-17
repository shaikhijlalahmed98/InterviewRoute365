# 📝 Quizzes — Self-Test MCQs (Auto-Evaluated)

**Purpose**: Test your understanding after reading the lesson. **Mentor auto-evaluates every day** — no need to ask, no need to paste answers in chat.

---

## 📂 Structure

```
quizzes/
├── README.md                          ← you are here
├── day-001-user-registration.md
├── day-002-user-login-jwt.md
├── day-003-user-profile-update.md
├── day-004-password-reset.md
└── day-XXX-topic.md (one per lesson)
```

## 📐 Each Quiz File Contains

1. **30-50 MCQs** across all stacks (Java, .NET, SQL, Angular, System Design, OOP)
2. **`**Your Answer**: ___`** slot per question (you replace `___` with your letter)
3. **Hidden answer key** at the bottom (collapsed `<details>` block)
4. **Evaluation section** (auto-added by mentor when you fill answers)

---

## 🎯 How To Use — Updated Workflow

### Step 1: Read the lesson
First go through `lessons/day-XXX-*.md` (45-60 min)

### Step 2: Fill the quiz
Open `quizzes/day-XXX-*.md` and replace `___` with your letter:

```markdown
### Q5. CSPRNG ka full form kya hai?
A) Common Standard PRNG
B) Cryptographically Secure Pseudo-Random Number Generator
C) Computer-Safe PRNG
D) Cyclic Secure PRNG

**Your Answer**: B   ← replace ___ with letter
```

### Step 3: Commit + push your changes
Just save your edits to the quiz file and push to GitHub (or edit directly on GitHub web UI). That's it.

### Step 4: Auto-evaluation (mentor does this)
Every day during routine, mentor:
1. **Scans all quiz files** in this folder
2. Detects files where **at least one `___` has been replaced with a letter**
3. Compares against correct answers
4. **Appends `## 📊 Evaluation — <date>` section** to the same file
5. Updates `progress.md` with score
6. Mentions it in daily chat summary

**No paste in chat needed.** Mentor checks automatically.

---

## ⏱️ Flexible Timing

- Fill quiz **same day** as lesson (most effective for memory)
- Fill on **random days** when time permits (mentor checks daily, will catch your fill)
- **Multiple attempts**: re-fill answers → mentor re-evaluates next routine, adds new dated section
- **Skip questions**: leave `___` for ones you don't know — counted as wrong (or skipped)

---

## 📊 Evaluation Format (What Mentor Adds)

After detecting your filled answers, mentor appends this section to the quiz file:

```markdown
---

## 📊 Evaluation — 2026-05-18

**Score**: 38/50 (76%)
**Status**: ✅ Solid — small gaps

### ❌ Wrong Answers (12)

| Q# | Your Answer | Correct | Why |
|----|-------------|---------|-----|
| 5 | A | C | Hashing protects against DB breach, not faster lookup. See Foundation Concept 2. |
| 12 | D | B | EF Core uses snapshot diff, not analytics. Re-read Stack 2 §3. |
| ...

### 🎯 Weak Area Diagnosis
- **SQL locking**: 4/12 wrong → re-read [Stack 3 SQL](../lessons/day-004-password-reset.md)
- **Angular RxJS**: 3/7 wrong → re-read [Stack 4](../lessons/day-004-password-reset.md)

### 📚 Recommended Action
1. Re-read SQL section (15 min)
2. Re-read Angular RxJS section (10 min)
3. Re-take quiz after 2 days for spaced repetition

**Evaluated by**: Claude AI mentor (auto-routine)
```

---

## 🏆 Scoring Targets

| Score | Meaning | Action |
|-------|---------|--------|
| 90-100% | ⭐ Mastered — interview ready | Move on confidently |
| 70-89% | ✅ Solid — small gaps | Review wrong answers |
| 50-69% | ⚠️ Re-read weak sections | Re-study + retake |
| <50% | 🔴 Full re-study needed | Re-read entire lesson |

---

## 🔁 Re-Attempts

Want to retake a quiz?

1. Replace your previous answers with new ones (or reset `___`)
2. Push to GitHub
3. Mentor's next routine detects new attempt → adds new dated `## 📊 Evaluation — <new date>` section
4. Old evaluation stays in file (track improvement over time)

Multiple evaluation sections = your spaced-repetition history.

---

## 🛡️ How Mentor Detects "Quiz Filled"

Mentor checks each quiz file for:
- Lines matching `**Your Answer**: A` or `**Your Answer**: B` or `**Your Answer**: C` or `**Your Answer**: D`
- If **at least one** such answer exists → file is "filled" → evaluate
- If all answers still `___` → skip (no evaluation needed)

Smart detection: re-evaluates only when **new answers** appear (compares with last evaluation timestamp).

---

## 📅 Daily Cadence

| Day | Quiz Creation | Quiz Evaluation |
|-----|---------------|-----------------|
| Day X (new lesson) | Mentor creates `quizzes/day-X-*.md` | — |
| Day X+N (you fill) | — | Mentor's next routine evaluates, appends section |

**You never tell mentor "evaluate karo"** — it's automatic part of daily routine.
