# ☕🌐 Spring vs .NET Cheatsheet

Side-by-side reference. Use when switching context between Java and .NET interviews.

---

## Dependency Injection

| Aspect | Spring | .NET Core |
|--------|--------|-----------|
| Register class | `@Service`, `@Component`, `@Repository` | `builder.Services.AddScoped<I, T>()` |
| Inject | Constructor (preferred) or `@Autowired` | Constructor only |
| Scopes | `singleton` (default), `prototype`, `request`, `session` | `Singleton`, `Scoped`, `Transient` |
| Default web scope | `singleton` | `Scoped` (per HTTP request) |
| Configuration | Annotations + `@Configuration` classes | `Program.cs` fluent API |

---

## Transactions

| Aspect | Spring | .NET |
|--------|--------|------|
| Declarative | `@Transactional` annotation | Not annotation-based; explicit |
| Programmatic | `TransactionTemplate` | `using var tx = await db.Database.BeginTransactionAsync()` |
| Auto rollback | On unchecked exceptions | Manual `tx.RollbackAsync()` or scope dispose |
| Mechanism | AOP proxy wraps method | Manual using-block lifecycle |

---

## ORM

| Aspect | Spring (Hibernate/JPA) | .NET (EF Core) |
|--------|------------------------|----------------|
| Entity marker | `@Entity` | `[Table]` (often implicit) |
| Repository | Extend `JpaRepository<T, ID>` | `DbSet<T>` on `DbContext` |
| Query builder | JPQL / Criteria API | LINQ |
| Eager loading | `@OneToMany(fetch=FetchType.EAGER)` | `.Include(x => x.Children)` |
| Change tracking | Hibernate dirty checking | EF Change Tracker (snapshot-based) |
| Optimistic lock | `@Version` Long field | `[Timestamp]` byte[] field |
| Bulk update | `@Modifying` + JPQL | `ExecuteUpdateAsync` (EF 7+) |

---

## Async / Concurrency

| Aspect | Spring (Java) | .NET (C#) |
|--------|---------------|-----------|
| Async method | `@Async` on method, returns `CompletableFuture<T>` | `async Task<T>` |
| Await | `.thenApply()`, `.thenCompose()` | `await` keyword |
| Thread pool | `TaskExecutor` (configurable) | `ThreadPool` (managed) |
| Cancellation | `CompletableFuture.cancel()` | `CancellationToken` |
| Parallel ops | `CompletableFuture.allOf(...)` | `Task.WhenAll(...)` |

---

## Security

| Aspect | Spring Security | ASP.NET Core Identity |
|--------|-----------------|----------------------|
| Password hashing | `BCryptPasswordEncoder` | `IPasswordHasher<TUser>` (PBKDF2 default) |
| JWT library | `jjwt` (`Jwts.builder()`) | `System.IdentityModel.Tokens.Jwt` |
| Auth context | `SecurityContextHolder` (ThreadLocal) | `HttpContext.User` (ClaimsPrincipal) |
| Per-request interception | `OncePerRequestFilter` | Middleware (`JwtBearerHandler`) |
| Authorization annotation | `@PreAuthorize("hasRole('ADMIN')")` | `[Authorize(Roles = "Admin")]` |
| CSRF protection | `csrf()` config (default on for non-GET) | `[ValidateAntiForgeryToken]` |

---

## Cryptography

| Operation | Java | C#/.NET |
|-----------|------|---------|
| Secure random | `SecureRandom.nextBytes()` | `RandomNumberGenerator.Fill()` |
| SHA-256 hash | `MessageDigest.getInstance("SHA-256")` | `SHA256.Create()` |
| BCrypt | `BCryptPasswordEncoder` | `BCrypt.Net-Next` package |
| HMAC | `Mac.getInstance("HmacSHA256")` | `HMACSHA256` |
| AES | `Cipher.getInstance("AES/GCM/NoPadding")` | `AesGcm` (built-in) |

---

## HTTP / Web API

| Aspect | Spring Boot | ASP.NET Core |
|--------|-------------|--------------|
| Controller | `@RestController` + `@RequestMapping` | `[ApiController]` + `[Route]` |
| GET endpoint | `@GetMapping("/users/{id}")` | `[HttpGet("users/{id}")]` |
| Request body | `@RequestBody DTO dto` | `[FromBody] DTO dto` |
| Validation | `@Valid` + Bean Validation (JSR-380) | `[FromBody]` + DataAnnotations or FluentValidation |
| Response status | `ResponseEntity.status(409).body(...)` | `return Conflict(...)` |
| Exception handling | `@ControllerAdvice` + `@ExceptionHandler` | Middleware or `[ProducesResponseType]` |

---

## Configuration

| Aspect | Spring | .NET |
|--------|--------|------|
| Property file | `application.yml` / `.properties` | `appsettings.json` |
| Environment-specific | `application-prod.yml` | `appsettings.Production.json` |
| Inject value | `@Value("${app.jwt.secret}")` | `IConfiguration["Jwt:Secret"]` or `IOptions<T>` |
| Secret management | Spring Cloud Config / Vault | Azure Key Vault / `dotnet user-secrets` (dev) |

---

## Common Identifier Types

| Concept | Java | C#/.NET |
|---------|------|---------|
| UUID | `UUID.randomUUID()` | `Guid.NewGuid()` |
| Timestamp UTC | `Instant.now()` | `DateTime.UtcNow` |
| BigInteger ID | `Long` | `long` |

---

## Testing

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Framework | JUnit 5 + Mockito | xUnit / NUnit + Moq |
| Mocking | `@Mock` + `when(...)` | `var mock = new Mock<T>(); mock.Setup(...)` |
| Integration | `@SpringBootTest` | `WebApplicationFactory<T>` |
| Database test | H2 in-memory + `@DataJpaTest` | EF Core InMemory provider |
| Assertion | `assertThat(x).isEqualTo(y)` (AssertJ) | `Assert.Equal(y, x)` (xUnit) |

---

## Key Conceptual Differences

1. **Property naming**: Java `getX()/setX()` methods vs C# auto-properties `public X { get; set; }`
2. **Override**: Java `@Override` optional but recommended; C# `override` keyword **mandatory**
3. **Virtual methods**: Java methods virtual by default; C# methods sealed unless `virtual`
4. **Nullable types**: Java `Optional<T>`; C# `T?` (nullable reference types in C# 8+)
5. **String formatting**: Java `String.format("%s", x)`; C# `$"{x}"` or `string.Format("{0}", x)`
6. **Collections**: Java `List<T>` interface, `ArrayList<T>` impl; C# `List<T>` is the impl, `IList<T>` interface
