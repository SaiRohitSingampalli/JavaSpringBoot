# 10 — Design Patterns (with where Spring uses them)

> Interviewers often ask: "Which design patterns have you used?" and "Which patterns does Spring use internally?" Know the intent, a code sketch, and a real example for each.

## Creational patterns

### 1. Singleton
**Intent**: exactly one instance, global access point.
```java
// Best: enum (thread-safe, serialization- and reflection-safe)
public enum AppConfig { INSTANCE; public String get(String k) { ... } }

// Lazy holder idiom
public class IdGenerator {
    private IdGenerator() {}
    private static class Holder { static final IdGenerator INSTANCE = new IdGenerator(); }
    public static IdGenerator getInstance() { return Holder.INSTANCE; }
}
```
Other variants: eager static field, synchronized method (slow), double-checked locking with `volatile`.
Ways to break a singleton: reflection (call private constructor), serialization (implement `readResolve`), cloning, multiple class loaders. Enum prevents all.
**In Spring**: default bean scope is singleton — one instance **per ApplicationContext** (not per JVM), managed by the container. Prefer Spring singletons over hand-written ones (testable, injectable).

### 2. Factory Method
**Intent**: let subclasses / a method decide which concrete class to instantiate.
```java
public interface Notifier { void send(String msg); }
public class NotifierFactory {
    public static Notifier create(Channel c) {
        return switch (c) {
            case EMAIL -> new EmailNotifier();
            case SMS -> new SmsNotifier();
            case PUSH -> new PushNotifier();
        };
    }
}
```
**In JDK/Spring**: `Calendar.getInstance()`, `List.of()`, `LoggerFactory.getLogger()`, Spring's `FactoryBean<T>`, `@Bean` methods are factory methods, `BeanFactory.getBean()`.

**Spring-idiomatic factory** (no switch, open/closed):
```java
@Component
public class NotifierRegistry {
    private final Map<Channel, Notifier> byChannel;
    public NotifierRegistry(List<Notifier> notifiers) {      // Spring injects all implementations
        byChannel = notifiers.stream().collect(toMap(Notifier::channel, n -> n));
    }
    public Notifier get(Channel c) { return byChannel.get(c); }
}
```

### 3. Abstract Factory
**Intent**: create families of related objects without specifying concrete classes.
E.g., `UiFactory` → `WindowsUiFactory` creates `WinButton` + `WinCheckbox`; `MacUiFactory` creates Mac versions. JDK: `DocumentBuilderFactory`.

### 4. Builder
**Intent**: construct complex objects step by step, readable, immutable result.
```java
HttpRequest req = HttpRequest.newBuilder().uri(uri).header("A", "b").timeout(Duration.ofSeconds(5)).build();
```
**In Spring/libs**: `UriComponentsBuilder`, `ResponseEntity.ok().header(...).body(...)`, `RestClient.builder()`, `WebClient.builder()`, `MockMvcRequestBuilders`, `SecurityFilterChain` via `HttpSecurity`, Lombok `@Builder`, `StringBuilder`.

### 5. Prototype
**Intent**: create new objects by copying an existing one.
**In Spring**: `@Scope("prototype")` — a new bean instance each time it's requested (concept differs slightly; not cloning).

## Structural patterns

### 6. Adapter
**Intent**: convert one interface to another clients expect.
```java
public interface PaymentGateway { Receipt pay(Money m); }
public class LegacyBankAdapter implements PaymentGateway {
    private final LegacyBankApi legacy;
    public Receipt pay(Money m) {
        LegacyResult r = legacy.doTransfer(m.amount().doubleValue(), m.currency());
        return new Receipt(r.getRef());
    }
}
```
**In Spring**: `HandlerAdapter` in Spring MVC (adapts different handler types to DispatcherServlet), `JpaVendorAdapter`, `MessageListenerAdapter`. JDK: `Arrays.asList`, `InputStreamReader` (byte → char).

### 7. Decorator
**Intent**: add behaviour to objects dynamically by wrapping them, same interface.
```java
public class CachingProductService implements ProductService {
    private final ProductService delegate;
    private final Map<Long, Product> cache = new ConcurrentHashMap<>();
    public Product get(long id) { return cache.computeIfAbsent(id, delegate::get); }
}
```
**JDK**: `BufferedInputStream(new FileInputStream(..))`, `Collections.unmodifiableList`. **Spring**: `TransactionAwareDataSourceProxy`, `ContentCachingRequestWrapper`, `HttpServletRequestWrapper`, `BeanPostProcessor`-based wrapping.

### 8. Proxy
**Intent**: a surrogate controlling access to another object (lazy loading, security, remote, logging, transactions).
**In Spring — the most important one**: AOP proxies implement `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable`. JDK dynamic proxies (interface-based) or CGLIB/ByteBuddy subclass proxies. Hibernate lazy-loading proxies for entities. Feign clients are proxies of your interface. Spring Data repositories are proxies too.

```java
// JDK dynamic proxy
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(), new Class<?>[]{UserService.class},
    (p, method, args) -> {
        long t = System.nanoTime();
        try { return method.invoke(realService, args); }
        finally { log.info("{} took {}ns", method.getName(), System.nanoTime() - t); }
    });
```
Decorator vs Proxy: structurally similar; decorator adds features chosen by the client, proxy controls access and is often created by a framework.

### 9. Facade
**Intent**: simple interface over a complex subsystem.
`OrderFacade.placeOrder()` hides inventory, payment, shipping, notification calls. **In Spring**: `JdbcTemplate` is a facade over raw JDBC; SLF4J is a facade over logging frameworks.

### 10. Composite
**Intent**: treat individual objects and compositions uniformly (tree structures).
Menu → MenuItem / SubMenu; file system. **Spring**: `CompositeCacheManager`, `CompositePropertySource`.

### 11. Bridge, Flyweight (know definitions)
- **Bridge**: decouple abstraction from implementation so both vary independently (JDBC API ↔ drivers).
- **Flyweight**: share common state among many fine-grained objects (`Integer.valueOf` cache, String pool).

## Behavioural patterns

### 12. Strategy
**Intent**: family of interchangeable algorithms selected at runtime.
```java
public interface DiscountStrategy { BigDecimal apply(BigDecimal total); }
@Component("FESTIVE") class FestiveDiscount implements DiscountStrategy { ... }
@Component("LOYALTY") class LoyaltyDiscount implements DiscountStrategy { ... }

@Service
public class CheckoutService {
    private final Map<String, DiscountStrategy> strategies;   // Spring injects bean-name → bean map
    public CheckoutService(Map<String, DiscountStrategy> strategies) { this.strategies = strategies; }
    public BigDecimal total(Cart cart, String type) { return strategies.get(type).apply(cart.total()); }
}
```
**JDK**: `Comparator`. **Spring**: `PasswordEncoder`, `Resource` loading strategies, `CacheManager`, `TaskExecutor`, `PlatformTransactionManager` implementations.

### 13. Template Method
**Intent**: define algorithm skeleton in a base class; subclasses fill in steps.
**Spring**: `JdbcTemplate`, `RestTemplate`, `JmsTemplate`, `KafkaTemplate`, `TransactionTemplate` (callback flavour: template handles resource acquisition, error handling, cleanup; you supply the varying part as a lambda). `AbstractController`, `OncePerRequestFilter.doFilterInternal`.

### 14. Observer (Publish–Subscribe)
**Intent**: when one object changes, dependents are notified.
```java
public record OrderPlacedEvent(Long orderId) {}

@Service class OrderService {
    private final ApplicationEventPublisher publisher;
    void place(Order o) { save(o); publisher.publishEvent(new OrderPlacedEvent(o.getId())); }
}
@Component class EmailListener {
    @EventListener void on(OrderPlacedEvent e) { sendMail(e.orderId()); }
}
```
**Spring**: `ApplicationEvent`, `@EventListener`, `@TransactionalEventListener`. At scale → message brokers (Kafka).

### 15. Chain of Responsibility
**Intent**: pass a request along a chain of handlers; each decides to handle or pass on.
**Spring/Servlet**: `Filter` chain (`chain.doFilter`), Spring Security's `SecurityFilterChain` (~15 filters), `HandlerInterceptor` chain, Spring Cloud Gateway filters.

### 16. Command
**Intent**: encapsulate a request as an object (queue, log, undo). `Runnable`/`Callable` submitted to executors; message objects in CQRS.

### 17. Iterator
`Iterator`/`Iterable` in collections.

### 18. State
**Intent**: object changes behaviour when its internal state changes. Order status transitions (enum with per-constant behaviour), Spring Statemachine.

### 19. Mediator, Memento, Visitor, Interpreter (definitions)
- **Mediator**: central object coordinates interactions (DispatcherServlet mediates between handlers, views, resolvers).
- **Memento**: capture & restore state (undo).
- **Visitor**: add operations to object structures without changing classes (now often replaced by sealed types + pattern switch).
- **Interpreter**: evaluate a language (Spring Expression Language — SpEL).

## Enterprise / architectural patterns used daily
| Pattern | Description | Spring example |
|---------|-------------|----------------|
| **Dependency Injection / IoC** | Container supplies dependencies | The Spring container |
| **MVC** | Model–View–Controller separation | Spring MVC |
| **Front Controller** | One handler receives all requests | `DispatcherServlet` |
| **Repository** | Collection-like abstraction over persistence | Spring Data repositories |
| **DAO** | Data access object encapsulating queries | `@Repository` classes with JdbcTemplate |
| **Service Layer** | Business logic boundary, transactions | `@Service` |
| **DTO** | Data transfer objects between layers/APIs | records |
| **Unit of Work** | Track changes, commit together | JPA persistence context |
| **Identity Map** | One object per DB row in a session | Hibernate 1st-level cache |
| **Lazy Load** | Load data on access | Hibernate proxies |
| **Circuit Breaker / Retry / Bulkhead** | Resilience | Resilience4j |
| **API Gateway, Saga, CQRS, Outbox, Event Sourcing** | Microservice patterns | see microservices notes |

## Patterns used inside Spring — summary answer
"Spring uses **Singleton** & **Prototype** scopes, **Factory** (`BeanFactory`, `FactoryBean`), **Proxy** for AOP (`@Transactional`, `@Async`, `@Cacheable`), **Template Method** (`JdbcTemplate`, `RestTemplate`), **Front Controller** (`DispatcherServlet`), **Observer** (application events), **Strategy** (`PasswordEncoder`, resource loaders, transaction managers), **Adapter** (`HandlerAdapter`), **Decorator** (request wrappers, bean post-processing), **Chain of Responsibility** (filter chain/interceptors), **Builder** (`ResponseEntity`, `UriComponentsBuilder`) and of course **Dependency Injection**."

## Anti-patterns to recognize
- **God class** — one class does everything.
- **Anemic domain model** — entities with only getters/setters, all logic in services (debated, common in Spring CRUD apps).
- **Service locator** — `ctx.getBean()` everywhere instead of DI.
- **Golden hammer**, **premature optimization**, **copy-paste programming**, **magic numbers/strings**, **distributed monolith** (microservices that must deploy together).

## Interview Q&A
**Q. How would you implement a thread-safe singleton?** Enum, or lazy holder idiom; double-checked locking needs volatile.

**Q. Strategy vs State?** Both delegate to interchangeable objects. Strategy is chosen by the client; State changes internally as the object transitions.

**Q. Decorator vs Proxy vs Adapter?** Decorator adds behaviour (same interface). Proxy controls access (same interface). Adapter converts to a different interface.

**Q. Factory vs Abstract Factory?** Factory creates one product type via a method; Abstract Factory creates families of related products.

**Q. Which pattern does `@Transactional` use?** Proxy (AOP) — a proxy wraps the bean, opens a transaction before the method and commits/rolls back after.
