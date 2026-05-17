# 📝 Quizzes — Self-Test MCQs

**Purpose**: Test your understanding after reading the lesson. 50 comprehensive MCQs per day. Mentor evaluates and grades.

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

1. **50 MCQs** (or 30-40 for smaller lessons) across all stacks
2. **Section breakdown**: Java | .NET | SQL | Angular | System Design | OOP
3. **"Your Answer: ___"** slot per question (you fill in)
4. **Hidden answer key** at the bottom (collapsed/spoiler)

## 🎯 How To Use

### Step 1: Read the lesson
First go through `lessons/day-XXX-*.md` (45-60 min)

### Step 2: Take the quiz
Open `quizzes/day-XXX-*.md`. Fill your answers next to each question:
```markdown
### Q5. CSPRNG ka full form kya hai?
A) Common Standard PRNG
B) Cryptographically Secure Pseudo-Random Number Generator
C) Computer-Safe PRNG
D) Cyclic Secure PRNG

**Your Answer**: B
```

### Step 3: Submit to mentor
Two options:
- **Option A**: Just paste answers in chat → `"Day 4 quiz: 1.B 2.A 3.C 4.D 5.B ..."`
- **Option B**: Commit your edited quiz file → tell mentor `"Day 4 quiz evaluate karo"`

### Step 4: Get evaluation
Mentor responds with:
- ✅ Correct count + ❌ Wrong count
- Per-question feedback for wrong answers
- Weak area diagnosis: "Tum SQL locking pe weak ho, Day 4 ka §SQL re-read karo"
- Score: X/50

### Flexible Timing
- Fill quiz **same day** as lesson (most effective)
- Fill on **random days** when time permits (mentor doesn't care when)
- Multiple attempts allowed (re-attempt after weak-area review)

## 🏆 Suggested Scoring Targets

| Score | Meaning |
|-------|---------|
| 45-50 | ⭐ Mastered — interview ready |
| 35-44 | ✅ Solid — small gaps |
| 25-34 | ⚠️ Re-read weak sections |
| <25 | 🔴 Full re-study needed |

## 📊 Quiz Score Tracker

Track your scores in `progress.md` under the "Quiz Score" column.

## 🔄 Auto-Update

Mentor creates one quiz file daily, alongside lesson push. No manual work for you to set up.
