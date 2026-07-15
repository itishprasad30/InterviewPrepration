# Spring Boot Interview Prep — Bean, DI & Core Concepts

## Part 1: Bean & Dependency Injection — The Foundation

### What is a Bean?

A **Bean** is just a normal Java object — but instead of you creating it with `new`, **Spring creates it, configures it, and manages its entire lifecycle** (creation → dependency injection → initialization → destruction).

```java
@Service
public class PayrollService {
    // Spring creates this object and puts it in a container called
    // the "ApplicationContext". You never write `new PayrollService()`.
}
```

Think of Spring's container as a big warehouse. Every class annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, or defined via `@Bean` gets registered as an item in that warehouse. Whenever some other class needs it, Spring hands it over — that's the "bean."

### What is Dependency Injection (DI)?

**Dependency Injection** = instead of a class creating its own dependencies, those dependencies are "injected" (handed to it) from outside.

**Without DI (bad — tight coupling):**
```java
public class PayrollService {
    private EmployeeRepository repo = new EmployeeRepository(); 
    // PayrollService is now stuck with THIS exact implementation.
    // Hard to test, hard to swap, hard to mock.
}
```

**With DI (good — loose coupling):**
```java
@Service
public class PayrollService {
    private final EmployeeRepository repo;

    @Autowired
    public PayrollService(EmployeeRepository repo) {
        this.repo = repo; // Spring hands this in from outside
    }
}
```

### Why does DI matter?

1. **Loose coupling** — `PayrollService` doesn't care *how* `EmployeeRepository` is built, just that it implements the contract.
2. **Testability** — in unit tests, you can inject a mock `EmployeeRepository` instead of a real DB-backed one.
3. **Single Responsibility** — object creation logic is centralized in Spring, not scattered everywhere.
4. **Easy swapping** — change implementation (e.g., switch from JPA repo to a Mongo repo) without touching the consuming class.

### How does Spring do it? (IoC — Inversion of Control)

Normally *your code* controls object creation ("I decide when to create X"). With Spring, that control is **inverted** — the *framework* decides when and how objects are created and wired together. DI is the mechanism; **IoC is the principle** behind it. The `ApplicationContext` is the IoC container that does this.

**Flow:**
1. Spring Boot starts → scans packages (`@ComponentScan`, implicit in `@SpringBootApplication`)
2. Finds all `@Component`/`@Service`/`@Repository`/`@Controller`/`@Bean` definitions
3. Builds a dependency graph
4. Creates beans in the correct order, injecting dependencies as needed
5. Beans are now ready to use, sitting in the `ApplicationContext`

---

## Part 2: 30+ Spring Boot Interview Questions & Answers

### Core Concepts

**1. What is Spring Boot, and how is it different from Spring?**
Spring Boot is built on top of the Spring Framework. Spring requires a lot of manual XML/Java configuration; Spring Boot auto-configures things based on what's on the classpath (auto-configuration), ships an embedded server (Tomcat/Jetty), and provides "starter" dependencies so you don't manage versions manually.

**2. What is auto-configuration?**
Spring Boot looks at the JARs on your classpath and automatically configures beans it thinks you need. E.g., if `spring-boot-starter-web` is present, it auto-configures an embedded Tomcat and `DispatcherServlet`. Driven by `@EnableAutoConfiguration` and `spring.factories` / `AutoConfiguration.imports`.

**3. What are Spring Boot starters?**
Curated dependency bundles (e.g., `spring-boot-starter-web`, `spring-boot-starter-data-jpa`) that pull in compatible versions of everything needed for that feature, avoiding version conflicts.

**4. What is the difference between `@Component`, `@Service`, `@Repository`, `@Controller`?**
All are stereotypes registered as beans via component scanning. `@Repository` additionally translates DB-specific exceptions into Spring's `DataAccessException`. `@Service` and `@Controller` are semantic markers for readability — functionally close to `@Component`.

### Beans & DI

**5. What is a Bean?**
An object whose lifecycle (creation, wiring, destruction) is managed by the Spring IoC container instead of by your own code.

**6. What is the ApplicationContext?**
The Spring IoC container — holds all bean definitions, creates beans, wires dependencies, manages their lifecycle. `BeanFactory` is the base interface; `ApplicationContext` is a richer version with event publishing, internationalization, etc.

**7. What are the types of Dependency Injection?**
- **Constructor injection** (preferred) — dependencies passed via constructor, allows `final` fields, mandatory dependencies enforced at compile/creation time.
- **Setter injection** — for optional dependencies.
- **Field injection** (`@Autowired` on a field directly) — easiest to write but hardest to test/mock; generally discouraged.

**8. Why is constructor injection preferred over field injection?**
- Fields can be `final` → immutability
- Dependencies are explicit and mandatory — object can't be constructed in an incomplete state
- Easier to unit test (just call the constructor with mocks, no need for reflection or Spring context)
- Helps catch circular dependencies early (fails at startup vs. silently working via reflection)

**9. What is `@Qualifier` used for?**
When two beans of the same type exist, `@Autowired` doesn't know which one to inject. `@Qualifier("beanName")` disambiguates.

**10. What is `@Primary`?**
Marks one bean as the default choice when multiple candidates exist, so `@Autowired` doesn't need `@Qualifier` every time.

**11. What are Bean Scopes?**
- `singleton` (default) — one instance per Spring container
- `prototype` — new instance every time it's requested
- `request` / `session` — web-specific, one per HTTP request/session

**12. Explain the Bean lifecycle.**
Instantiate → populate properties (DI) → `@PostConstruct` (or `InitializingBean.afterPropertiesSet()`) → bean ready for use → `@PreDestroy` (or `DisposableBean.destroy()`) on shutdown.

**13. What is circular dependency and how does Spring handle it?**
Bean A needs Bean B, Bean B needs Bean A. With **setter/field injection**, Spring can resolve this using early-reference caching (three-tier cache). With **constructor injection**, it **fails at startup** with `BeanCurrentlyInCreationException` — which is actually a good thing, it forces you to redesign (e.g., extract shared logic to a third bean).

**14. What's the difference between `@Autowired` and `@Resource`?**
`@Autowired` (Spring) resolves by type first, then name. `@Resource` (JSR-250/Jakarta) resolves by name first, then type.

### REST & Web Layer

**15. Difference between `@Controller` and `@RestController`?**
`@Controller` returns view names (for server-rendered pages). `@RestController` = `@Controller` + `@ResponseBody`, so return values are serialized directly to JSON/XML in the response body.

**16. Difference between `@RequestParam` and `@PathVariable`?**
`@RequestParam` reads query string params (`?status=active`). `@PathVariable` reads a value from the URI path itself (`/leaves/{id}`).

**17. What does `@RequestBody` do?**
Deserializes the incoming JSON/XML request payload into a Java object using `HttpMessageConverter` (Jackson by default).

**18. How do you handle validation in a REST API?**
`@Valid` on the `@RequestBody` parameter + Bean Validation annotations (`@NotNull`, `@Size`, `@Email`, etc.) on the DTO fields. Errors are captured as `MethodArgumentNotValidException`, typically handled globally.

**19. How do you handle exceptions globally in Spring Boot?**
`@ControllerAdvice` + `@ExceptionHandler(SpecificException.class)` — centralizes error handling instead of try-catch in every controller.

**20. What is `ResponseEntity` and why use it?**
Wraps the response body along with HTTP status code and headers, giving full control over the HTTP response instead of just returning a POJO.

### Data Layer

**21. What is `@Transactional` and what does it do?**
Wraps a method in a database transaction. If a `RuntimeException` (unchecked) is thrown, the transaction rolls back automatically. Checked exceptions do **not** trigger rollback by default — you'd need `@Transactional(rollbackFor = Exception.class)`.

**22. What is transaction propagation? Name a couple of types.**
Defines how a transactional method behaves when called from another transactional method.
- `REQUIRED` (default) — join existing transaction, or create one if none exists
- `REQUIRES_NEW` — always starts a new transaction, suspending the current one
- `NESTED` — starts a nested transaction (savepoint) within the existing one

**23. What's the N+1 query problem and how do you avoid it?**
Occurs when fetching a list of entities triggers one query for the list + N additional queries for each entity's related/lazy-loaded data. Fixed using JOIN FETCH, `@EntityGraph`, or batch fetching.

**24. Difference between `@Entity` and a DTO?**
`@Entity` maps directly to a DB table and is part of the persistence layer. A DTO (Data Transfer Object) is a plain object used to transfer data between layers (e.g., API responses) without exposing internal entity/DB structure.

### Security & Config

**25. What is `@ConfigurationProperties` vs `@Value`?**
`@Value("${key}")` injects a single property. `@ConfigurationProperties(prefix="x")` binds a whole block of related properties to a POJO — cleaner for grouped config (e.g., rate-limit settings: per-day, per-hour, per-minute all in one class).

**26. How do Spring profiles work?**
`@Profile("dev")` on a bean/config class activates it only when that profile is active (`spring.profiles.active=dev`). Used for environment-specific beans (e.g., different DB configs for dev/prod).

**27. How does Spring Boot handle externalized configuration?**
Reads from `application.properties`/`application.yml`, environment variables, command-line args, in a defined priority order (command-line > env vars > application.yml > defaults).

### Microservices-Adjacent (relevant to your Humine work)

**28. How would you implement rate limiting in a Spring Boot service?**
Typically a filter/interceptor (or aspect) that tracks request counts per key (user/IP) in a store (in-memory map, Redis, or DB) with a sliding/fixed window, rejecting requests once a threshold is hit — similar to what you built for the notification service (1000/day with tiered hourly/minute caps).

**29. How do you ensure idempotency in an API (e.g., avoiding duplicate form submissions)?**
Track a unique idempotency key (e.g., request ID, or a hash of submission content) in the DB with a unique constraint; reject or return the cached result if the same key is seen again — this is basically the pattern behind fixing your double-submit bug.

**30. What is IDOR and how do you prevent it in Spring Boot?**
Insecure Direct Object Reference — when an endpoint uses a client-supplied ID (e.g., `/documents/{id}`) without verifying the requester actually owns/has access to that resource. Prevented by checking the authenticated user's ownership/permissions server-side before returning data, not just relying on the ID being "hard to guess."

**31. Bonus — What is `@Async` used for?**
Runs a method in a separate thread (via a `TaskExecutor`) instead of blocking the calling thread — useful for things like sending emails or notifications without holding up the main request.

**32. Bonus — What is Spring Boot Actuator?**
A set of production-ready endpoints (`/actuator/health`, `/actuator/metrics`, etc.) for monitoring and managing a running application.

---

## Quick tie-back to your Humine experience
When answering these, anchor with your real stories:
- **Constructor injection / testability** → your service layer refactors
- **`@Transactional` + rollback** → LOP calculation logic
- **Idempotency** → double-submit bug fix on policy acknowledgment forms
- **IDOR** → salary document download fix
- **Rate limiting + `@Async`-style patterns** → notification service with tiered caps

Want a mock Q&A round where I ask these one at a time and you answer out loud (text), and I give feedback like a real interviewer?
