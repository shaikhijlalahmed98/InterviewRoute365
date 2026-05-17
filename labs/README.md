# 🧪 Labs — Hands-On Code

Yahan tum actual code likho jo lesson concepts ko internalize kare. Reading theory aur building working code mein bohot difference hai.

## Convention

**Folder naming**: `day-XXX-topic-stack/`

Examples:
- `day-001-user-signup-springboot/`
- `day-004-password-reset-dotnet/`
- `day-021-rbac-fullstack/` (Angular + Spring)

## What Goes In A Lab Folder

```
day-XXX-topic-stack/
├── README.md          ← what you built, how to run
├── src/               ← actual code
├── docker-compose.yml ← if multi-service
└── notes.md           ← lessons learned, bugs faced
```

## Lab Priority

Not every day needs a lab. Build labs when:
- ✅ Concept is **practice-heavy** (locking, async, security flows)
- ✅ You want **interview demo material** ("I built this in 1 hour")
- ✅ You'll **showcase on GitHub portfolio**
- ❌ Skip for pure theory days (system design discussions, anti-patterns)

## Suggested Lab Stacks

| Day Range | Recommended Stack |
|-----------|-------------------|
| 1-20 (Beginner) | Spring Boot + H2 (in-memory DB, fast iteration) |
| 21-50 (Intermediate) | Spring Boot + MySQL + Angular full-stack |
| 51-90 (Advanced) | .NET 8 + SQL Server + EF Core (variety) |
| 91+ (Expert+) | Docker compose multi-service systems |

## Quality Bar

- Code runs (provide instructions)
- README explains what was built
- Commit messages clear
- No secrets in code (use env vars)
