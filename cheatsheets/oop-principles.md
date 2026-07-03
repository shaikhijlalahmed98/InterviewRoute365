# 🏛️ OOP Principles Cheatsheet

Condensed reference for OOP-heavy interviews.

---

## The 4 Pillars

### 1. Encapsulation
**Definition**: Data + methods bundled in a class. Internal state hidden, accessed only through controlled public interface.

**Why**: Validation in one place, implementation freedom, thread safety, easier refactoring.

**Mnemonic**: "DABBA" = Data Access By Backed Access (ATM machine analogy)

**Java**: `private` fields + getters/setters or Lombok `@Getter/@Setter`
**C#**: `private` fields + properties (`public X { get; private set; }`)

**Bad**: `public String password;` — anyone can set anything
**Good**: `setPassword(raw)` method internally hashes + validates

---

### 2. Abstraction
**Definition**: Show only essential features, hide complexity. Interface tells WHAT, implementation tells HOW.

**Why**: Decouple consumers from implementations, easy to swap, easier testing.

**Mnemonic**: "Naqsha aur Naqshanavees" — naqsha (interface) dikhta hai, naqshanavees (impl) chhupa hota hai.

**Java**: `interface` keyword, or `abstract class`
**C#**: Same — `interface`, `abstract class`

**Example**: `List<T>` interface, `ArrayList` / `LinkedList` implementations — consumer doesn't care which.

---

### 3. Inheritance
**Definition**: Child class inherits properties + methods from parent. "IS-A" relationship.

**Why**: Code reuse, polymorphism enabler, hierarchical modeling.

**Mnemonic**: "Khandani Wirasat" — baap ke gun bachhon ko.

**Java**: `class Dog extends Animal`
**C#**: `class Dog : Animal`

**Liskov rule**: Subclass must honor parent's contract — if `Animal.eat()` doesn't throw, `Dog.eat()` shouldn't either.

**⚠️ Warning**: Prefer **composition over inheritance**. Deep hierarchies = maintenance nightmare. Use inheritance ONLY when child genuinely IS-A parent (Dog IS-A Animal ✓; Order IS-A List ✗).

---

### 4. Polymorphism
**Definition**: Same method call, different behavior based on actual object type.

**Mnemonic**: "BARTAN BADLO, KHAANA WOHI" — bartan (concrete class) badle, khaana (interface) wohi.

**2 Types**:

| Type | When Decided | Example |
|------|--------------|---------|
| **Overloading** (compile-time) | Compiler picks based on arg types | `print(int)`, `print(String)` in same class |
| **Overriding** (runtime) | JVM/CLR picks based on actual object | Child redefines parent's method |

**Mechanism — Vtable** (Virtual Function Table):
- Every class has its own vtable (array of function pointers)
- Object holds pointer to its class's vtable
- Method call → follow object's class pointer → vtable → method pointer → execute

**JIT Devirtualization**: If runtime sees only one impl, compiler can skip vtable lookup (monomorphic optimization).

---

## SOLID Principles

### S — Single Responsibility
**Mnemonic**: "EK BANDA, EK KAAM" (one class, one responsibility, one reason to change)

**Test**: Can you summarize this class in one sentence without "and"?

### O — Open/Closed
**Mnemonic**: "EXTEND HAAN, MODIFY NAA"

Open for extension (subclass, new implementation), closed for modification (existing code untouched).

**Achieved via**: Polymorphism + Strategy Pattern.

### L — Liskov Substitution
**Mnemonic**: "BACHA APNE BAAP SE BURAI NA KARE"

If S is subtype of T, S can replace T without breaking. Subclass cannot violate parent's contract (no narrower returns, no broader exceptions, no extra preconditions).

### I — Interface Segregation
**Mnemonic**: "BHARI INTERFACE MAT BHEJO"

Many small focused interfaces beat one fat interface. Clients shouldn't depend on methods they don't use.

### D — Dependency Inversion
**Mnemonic**: "ABSTRACTION PE DEPEND KARO, CONCRETE PE NAHI"

High-level modules depend on abstractions. Abstractions don't depend on details. Realized via DI containers.

---

## DRY, KISS, YAGNI

### DRY — Don't Repeat Yourself
Same logic in multiple places = bug magnet. Extract to function/class.

### KISS — Keep It Simple, Stupid
Simplest solution that works > clever solution.

### YAGNI — You Aren't Gonna Need It
Don't build for hypothetical future. Build for today's requirement. Extract abstraction when **second** use case appears, not first.

---

## Composition vs Inheritance

**Inheritance**: "IS-A" — Dog IS-A Animal
**Composition**: "HAS-A" — Car HAS-A Engine

**Rule of thumb**: 90% of the time, composition is better.

**Why inheritance fails at scale**:
- Tight coupling (parent changes → all children break)
- Fragile base class problem
- Diamond inheritance issues (multiple parents)
- Hard to change parent (breaks all subclasses)

**When inheritance IS right**:
- True is-a relationship
- Framework-required (Spring's `JpaRepository`, .NET's `Controller`)
- Marker types

---

## Aggregation vs Composition vs Association

| Relationship | Lifetime | Example |
|--------------|----------|---------|
| **Association** | Independent | Student ↔ Course |
| **Aggregation** | Independent but related | University → Departments |
| **Composition** | Dependent (child dies with parent) | House → Rooms |
