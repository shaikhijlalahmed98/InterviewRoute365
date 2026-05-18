# 🎯 Quiz Format Standard (For All Future Daily Quizzes)

**Established**: Day 4 onwards
**Purpose**: Mentor ne yeh format follow karna hai har quiz file mein.

---

## 📐 Mandatory Sections (50 MCQs total)

| Section | Count | Type | Stars |
|---------|-------|------|-------|
| **A. Code Output Prediction** | 10 | "Yeh code chala — output kya?" | Mostly ⭐⭐ + few ⭐⭐⭐ |
| **B. Bug Spotting** | 8 | "Is code mein kya galat hai?" | ⭐⭐ to ⭐⭐⭐ |
| **C. Dry-Run Tracing** | 10 | Step-by-step execution trace, predict final state | Mostly ⭐⭐⭐ |
| **D. Scenario-Based Decisions** | 10 | Real production situation, choose right action | ⭐⭐ to ⭐⭐⭐ |
| **E. Concept Mastery** | 12 | Direct conceptual MCQs (foundational) | ⭐ to ⭐⭐ |

---

## ⭐ Difficulty Stars

Every question marked:
- **⭐** Easy — direct concept recall, no thinking
- **⭐⭐** Medium — requires application, code reading
- **⭐⭐⭐** Hard — multi-step reasoning, tricky edge case, dry-run

**Distribution target per quiz**:
- ⭐ Easy: 8-10 questions (16-20%)
- ⭐⭐ Medium: 25-30 questions (50-60%)
- ⭐⭐⭐ Hard: 12-15 questions (24-30%)

---

## 💻 Code Block Requirements

### Section A (Output Prediction) — Examples

**Good code analysis question**:
```java
SecureRandom random = new SecureRandom();
byte[] tokenBytes = new byte[32];
random.nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
System.out.println(token.length());
```
*Tests: understanding of Base64 encoding math (32 bytes → 43 chars)*

**Bad question** (too generic):
```
What does Base64.encode return?
```
*Doesn't test real understanding.*

### Section B (Bug Spotting) — Real Production Bugs

Always include:
- Realistic code (multi-line, looks like real codebase)
- The bug is subtle (not obvious typo)
- Options include the bug + 3 plausible distractors

### Section C (Dry-Run) — Step Tables

Use this format:
```
Initial state: <state>

T+0ms:   <action>
T+10ms:  <action>
T+20ms:  <action>

Question: What's the final state? / Which request wins?
```

### Section D (Scenarios) — Production Stories

Format:
```
Production issue: <symptom user observes>
Setup: <relevant architecture context, 3-5 lines>
What's the most likely cause / right action?
```

---

## 🛠️ Code Coverage Requirements

Each quiz MUST include code from these stacks (when lesson covers them):

| Stack | Code Examples Expected |
|-------|------------------------|
| Java/Spring | Entity classes, Service methods, annotation usage |
| .NET/C# | EF Core configurations, async methods, attributes |
| SQL | Schema, SELECT/UPDATE/INSERT, indexes, transactions |
| Angular | Components, RxJS pipes, Reactive Forms, services |
| OOP | Class hierarchies, polymorphic dispatches, interface impls |

---

## 🎨 Engagement Patterns (Avoid Boring)

### ❌ Boring MCQ Anti-Pattern:
```
Q. What is dependency injection?
A) Adding new features
B) A design pattern where dependencies are provided
C) Database connection
D) Compile error
```

### ✅ Engaging MCQ Pattern:
```
Q. Yeh service mein design issue spot karo:
public class UserService {
    private EmailSender emailSender = new EmailSender("smtp.com", 587, "pass");
    public void register(User u) {
        emailSender.send(u.getEmail(), "Welcome");
    }
}
A) Email service slow
B) Hard-coded dependency — untestable + tight coupling; should inject EmailSender via constructor
C) Email content wrong
D) Code is fine
```

### Patterns to use:
- **Concrete numbers** ("at T+10ms, what happens?")
- **Real company context** (Daraz, Stripe, Netflix scenarios)
- **Before/after code comparisons**
- **Multi-step traces** (force user to follow execution)
- **Edge case probing** (concurrent, crash, timeout scenarios)

---

## 📊 Answer Key Format

Always hidden in `<details>` block. Per-section explanations:

```markdown
<details>
<summary>⚠️ Click to reveal — complete all 50 first!</summary>

### Section A: Code Output Prediction
1. **B (43)** — 32 bytes Base64 URL-encoded without padding = 43 chars (32 × 4/3 = 42.67 → 43)
2. **B** — BCrypt embeds random salt; h1 ≠ h2 but matches() works for both
...

</details>
```

**Each answer explanation**: 1-2 sentences with **why** the answer is correct (not just letter).

---

## 🔢 Numbering & Structure

```markdown
### Q[N]. ⭐⭐ [Question text — practical, contextual]
```code block if needed```
A) [Plausible distractor]
B) [Correct answer with enough detail]
C) [Plausible distractor]
D) [Plausible distractor]

**Your Answer**: ___

---
```

- Bold question number
- Difficulty stars after number
- Code block immediately follows when relevant
- Options with realistic plausibility
- `**Your Answer**: ___` slot for user
- `---` separator between questions

---

## 📈 Quality Self-Check (Before Pushing Quiz)

Mentor's checklist:

- [ ] 50 MCQs total (or 30+ for smaller-scope days)
- [ ] All 5 sections present (Code Output, Bug Spot, Dry-Run, Scenarios, Concept)
- [ ] Difficulty distribution roughly matches target
- [ ] At least 60% questions have code blocks
- [ ] At least 30% questions are ⭐⭐⭐ hard
- [ ] No "trivia" questions that test memorization without understanding
- [ ] All correct answers have 1-2 sentence explanation in answer key
- [ ] Scenarios use Pakistani context where natural (Daraz, Careem)
- [ ] Stack distribution roughly matches what lesson covered

---

## 🆕 Future Enhancements (Day 31+)

When design patterns enter (Day 31+), add new question type:
- **Section F: Pattern Recognition** — "Yeh code konsa pattern use kar raha?"

When system design enters Expert (Day 91+), add:
- **Section G: Architecture Diagram Trace** — given diagram, what's the bottleneck?

---

## 🎯 Day 4 = Reference Implementation

See `quizzes/day-004-password-reset.md` as the **canonical example** of this format.

All future quizzes (Day 5 onwards) follow this standard.
