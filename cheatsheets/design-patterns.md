# 🎨 Design Patterns Cheatsheet (GoF + Modern)

Quick reference. Covered in lessons Day 31-90.

---

## Creational Patterns (Day 31-40)

### Singleton
**Purpose**: One instance, global access.
**Use case**: Logger, config, DB connection pool.
**⚠️ Warning**: Anti-pattern when overused (hidden dependencies, untestable).
**Java**: Spring's `@Singleton` (default scope) handles this for you.

### Factory Method
**Purpose**: Hide object creation behind a method.
**Use case**: Creating polymorphic objects without exposing concrete classes.
```java
NotificationSender sender = NotifierFactory.create(channel);
```

### Abstract Factory
**Purpose**: Factory for related families of objects.
**Use case**: Cross-platform UI (`WindowsButtonFactory`, `MacButtonFactory`).

### Builder
**Purpose**: Construct complex objects step-by-step.
**Use case**: Objects with many optional params.
```java
User user = new User.Builder()
    .name("Ahmed")
    .email("...")
    .build();
```

### Prototype
**Purpose**: Clone existing objects instead of creating new.
**Use case**: Expensive-to-create objects.

---

## Structural Patterns (Day 41-60)

### Adapter
**Purpose**: Make incompatible interfaces work together.
**Use case**: Integrating third-party library with your existing code.

### Decorator
**Purpose**: Add behavior to objects without modifying their class.
**Use case**: Logging, caching, transactions (Spring AOP uses this!).
```java
new LoggingDecorator(new CachingDecorator(realService));
```

### Facade
**Purpose**: Simplify complex subsystem with one entry point.
**Use case**: API gateway, complex library wrappers.

### Proxy
**Purpose**: Stand-in for another object, add control.
**Use case**: Spring's `@Transactional` proxy, lazy loading.
**Critical**: Spring uses CGLib/JDK proxies for `@Transactional`, `@Cacheable`, etc.

### Composite
**Purpose**: Treat individual objects and compositions uniformly.
**Use case**: File system (file + folder both implement `Node`), UI trees.

---

## Behavioral Patterns (Day 61-80)

### Strategy ⭐
**Purpose**: Algorithm interchangeable at runtime via polymorphism.
**Use case**: Payment methods, notification channels, sort algorithms.
```java
interface PaymentStrategy { void pay(Order o); }
class StripeStrategy implements PaymentStrategy { ... }
class JazzCashStrategy implements PaymentStrategy { ... }
```
**Where seen in Day 4**: `NotificationSender` with Email/SMS/Push.

### Observer (Pub-Sub)
**Purpose**: Notify multiple listeners when state changes.
**Use case**: Event systems, UI listeners.
**Roman Urdu**: "AZAAN PATTERN" — ek bola, sab ne suna, sab apna kaam karte hain.

### Chain of Responsibility
**Purpose**: Pass request through chain of handlers.
**Use case**: Spring Security filter chain, middleware pipelines.

### Command
**Purpose**: Encapsulate request as object.
**Use case**: Undo/redo, queueable operations.

### Template Method
**Purpose**: Base class defines skeleton, subclass fills in steps.
**Use case**: Spring's `JdbcTemplate.query(sql, rowMapper)`.

### State
**Purpose**: Object behavior changes based on internal state.
**Use case**: Order workflow (Pending → Paid → Shipped → Delivered).

---

## Anti-Patterns (Day 81-90)

### God Object / God Class
**Symptom**: One class doing 20 things.
**Fix**: Single Responsibility Principle.

### Spaghetti Code
**Symptom**: Tangled control flow, no structure.
**Fix**: Refactor with clear boundaries.

### Singleton Overuse
**Symptom**: Everything is global.
**Fix**: Dependency Injection.

### Premature Optimization
**Symptom**: Complex code to optimize hot path that isn't actually hot.
**Fix**: Profile first, optimize what's actually slow.

### Magic Numbers
**Symptom**: `if (status == 7)` — what is 7?
**Fix**: Constants or enums.

### Hard Coding
**Symptom**: Config in source code.
**Fix**: Environment variables, config files.

### Golden Hammer
**Symptom**: Using one tool for everything ("everything is a microservice", "use Kafka for X").
**Fix**: Right tool for the problem.

---

## Modern Patterns (Beyond GoF)

### Repository
**Purpose**: Abstract data access behind a collection-like interface.
**Use case**: Spring Data JPA's `JpaRepository`, EF Core's `DbContext`.

### Unit of Work
**Purpose**: Track changes, commit as single transaction.
**Use case**: EF Core's `SaveChangesAsync()`, Hibernate's session.

### DTO (Data Transfer Object)
**Purpose**: Carry data between layers without exposing domain models.
**Use case**: API request/response, decoupling internal model from public API.

### CQRS (Command Query Responsibility Segregation)
**Purpose**: Separate read model from write model.
**Use case**: High-read systems, event sourcing.

### Saga
**Purpose**: Manage distributed transactions across microservices.
**Use case**: Multi-step orders (reserve inventory → charge card → ship).

### Circuit Breaker
**Purpose**: Stop calling failing service to give it recovery time.
**Use case**: Resilient microservices (Resilience4j, Polly).

---

## When To Use What — Quick Decision Tree

| Need | Pattern |
|------|---------|
| Multiple implementations, runtime switch | **Strategy** |
| Notify many on change | **Observer** |
| Chain of handlers | **Chain of Responsibility** |
| Complex object construction | **Builder** |
| Hide complex subsystem | **Facade** |
| Add behavior without modification | **Decorator** |
| Object as request | **Command** |
| Cross-platform families | **Abstract Factory** |
| Step-by-step algorithm with variants | **Template Method** |
| State-dependent behavior | **State** |
| Wrap incompatible interface | **Adapter** |
| Control access to object | **Proxy** |
| Track changes in business transaction | **Unit of Work** |
| Abstract data access | **Repository** |
| Decouple read/write models | **CQRS** |
| Distributed transaction | **Saga** |
| Resilient service calls | **Circuit Breaker** |
