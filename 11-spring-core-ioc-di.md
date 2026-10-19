# 11 — Spring Core: IoC Container, Dependency Injection & Beans

## 1. The problem Spring solves

```java
public class OrderService {
    private final OrderRepository repo = new JdbcOrderRepository(new HikariDataSource(...));
    private final PaymentGateway gateway = new StripeGateway("api-key");
    private final EmailSender email = new SmtpEmailSender("smtp.host");
}
```
Problems: tightly coupled to concrete classes, configuration hard-coded, impossible to unit test with mocks, every class wires its own dependencies, lifecycle (connection pools) unmanaged.

**Solution — Inversion of Control (IoC)**: objects don't create their dependencies; an external **container** creates and injects them. **Dependency Injection (DI)** is the technique Spring uses to achieve IoC.

```java
@Service
public class OrderService {
    private final OrderRepository repo;
    private final PaymentGateway gateway;
    public OrderService(OrderRepository repo, PaymentGateway gateway) {
        this.repo = repo; this.gateway = gateway;
    }
}
```
Now `OrderService` depends only on abstractions, and tests can pass mocks.

## 2. Spring ecosystem map
| Layer | Pieces |
|-------|--------|
| Spring Framework (6.x / 7.x) | Core container (beans, context, SpEL), AOP, data access (JDBC, transactions, ORM), web (MVC, WebFlux), test, messaging |
| Spring Boot | Auto-configuration, starters, embedded server, Actuator |
| Spring projects | Data (JPA, MongoDB, Redis, R2DBC…), Security, Cloud, Batch, Integration, Kafka, AMQP, Session, Authorization Server, GraphQL, Modulith, AI |

## 3. The IoC container

### 3.1 BeanFactory vs ApplicationContext
| | BeanFactory | ApplicationContext |
|--|--|--|
| Basic DI | ✔ | ✔ |
| Bean init | Lazy | **Eager** singletons at startup |
| Events | ✘ | ✔ `ApplicationEventPublisher` |
| i18n | ✘ | ✔ `MessageSource` |
| Resource loading, Environment, profiles | limited | ✔ |
| Auto BeanPostProcessor registration | manual | ✔ |
| Use | Rare/embedded | **Always** |

Implementations: `AnnotationConfigApplicationContext` (plain Java), `AnnotationConfigServletWebServerApplicationContext` (Spring Boot MVC), `ReactiveWebServerApplicationContext` (WebFlux).

### 3.2 Container startup (simplified)
1. **Read bean definitions** — component scanning, `@Configuration` classes, `@Import`, auto-configurations → `BeanDefinition` objects (class, scope, dependencies, init methods...).
2. **BeanFactoryPostProcessors** run — may modify definitions (e.g., `PropertySourcesPlaceholderConfigurer` resolves `${...}`; `ConfigurationClassPostProcessor` processes `@Configuration`).
3. **Register BeanPostProcessors**.
4. **Instantiate singletons** (non-lazy): create → inject → initialize → post-process (proxies created here).
5. Publish `ContextRefreshedEvent`; Boot then publishes `ApplicationStartedEvent` and `ApplicationReadyEvent`.

## 4. Defining beans — three ways

### 4.1 Component scanning (your own classes)
```java
@Component     // generic
@Service       // business logic layer
@Repository    // persistence layer — also enables exception translation to DataAccessException
@Controller    // MVC controller (returns views)
@RestController// @Controller + @ResponseBody
@Configuration // source of @Bean definitions (also a @Component)
```
Functionally all are `@Component`; stereotypes communicate intent and enable layer-specific features (AOP pointcuts, exception translation).

```java
@Configuration
@ComponentScan(basePackages = "com.example.shop")       // Boot's @SpringBootApplication does this for its package
public class AppConfig {}
```
Default bean name = class name with lowercase first letter (`orderService`). Override: `@Service("orders")`.

### 4.2 `@Bean` methods (third-party classes / custom construction)
```java
@Configuration
public class InfraConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder().addModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS).build();
    }

    @Bean(initMethod = "start", destroyMethod = "stop")
    public SomeClient someClient(ObjectMapper mapper, @Value("${client.url}") String url) {
        return new SomeClient(url, mapper);      // dependencies via method params
    }
}
```

**`@Configuration` vs `@Component` with `@Bean` (full vs lite mode)**
```java
@Configuration
class Cfg {
    @Bean A a() { return new A(b()); }   // b() intercepted by CGLIB proxy → returns the singleton B
    @Bean B b() { return new B(); }
}
```
In `@Configuration` classes, Spring subclasses the config with CGLIB so inter-bean method calls return the container-managed singleton. In "lite" mode (`@Component` or `@Configuration(proxyBeanMethods = false)`) calling `b()` creates a **new** B each time. Boot's auto-configs use `proxyBeanMethods = false` for startup speed — then inject dependencies as method parameters instead of calling methods.

### 4.3 Functional / programmatic registration
```java
context.registerBean(MyService.class, () -> new MyService(...));
```
Also: `BeanDefinitionRegistryPostProcessor`, `ImportBeanDefinitionRegistrar`, and Spring Framework 7's `BeanRegistrar`. XML config (`<bean id=... class=...>`) still works but is legacy.

## 5. Dependency injection types

### 5.1 Constructor injection — recommended
```java
@Service
@RequiredArgsConstructor                 // Lombok generates the constructor for final fields
public class InvoiceService {
    private final InvoiceRepository repo;
    private final TaxCalculator tax;
}
```
- `@Autowired` optional with a single constructor (Spring 4.3+).
- Dependencies are `final` → immutable, thread-safe.
- Object can't exist in a half-initialized state.
- Easy unit tests: `new InvoiceService(mockRepo, mockTax)`.
- Too many constructor params = code smell (class does too much).

### 5.2 Setter injection — optional dependencies
```java
@Autowired(required = false)
public void setMetrics(MetricsClient metrics) { this.metrics = metrics; }
```

### 5.3 Field injection — avoid in production code
```java
@Autowired private InvoiceRepository repo;
```
Can't be final, hides dependencies, needs reflection/Spring for tests, allows circular dependencies silently. Acceptable in test classes.

### 5.4 Optional dependencies
```java
public MyService(Optional<Cache> cache) { }
public MyService(ObjectProvider<Cache> cacheProvider) { cacheProvider.getIfAvailable(); }   // lazy, optional, handles multiple
public MyService(@Nullable Cache cache) { }
```

## 6. Autowiring resolution — how Spring picks a bean
1. **By type** — find beans assignable to the parameter type.
2. One match → inject it.
3. Multiple → check **`@Qualifier`**.
4. Then check **`@Primary`**.
5. Then fall back to **parameter/field name** matching a bean name.
6. Still ambiguous → `NoUniqueBeanDefinitionException`. None → `NoSuchBeanDefinitionException` (unless optional).

```java
public interface PaymentGateway {}
@Component @Primary class RazorpayGateway implements PaymentGateway {}
@Component("stripe") class StripeGateway implements PaymentGateway {}

public CheckoutService(PaymentGateway gateway) {}                      // Razorpay (@Primary)
public RefundService(@Qualifier("stripe") PaymentGateway gateway) {}   // Stripe
public Router(List<PaymentGateway> all) {}                             // both, ordered by @Order
public Router(Map<String, PaymentGateway> byName) {}                   // {"razorpayGateway":..., "stripe":...}
```
Custom qualifier annotations:
```java
@Qualifier @Retention(RUNTIME) @Target({FIELD, PARAMETER, TYPE, METHOD})
public @interface Domestic {}
@Component @Domestic class RazorpayGateway implements PaymentGateway {}
public Checkout(@Domestic PaymentGateway g) {}
```
`@Order(1)` / `Ordered` controls the order of beans in injected lists; `@Priority` also works.

**Generics as qualifiers**: `Repository<User>` and `Repository<Order>` beans can be injected by generic type.

## 7. Circular dependencies
```java
@Service class A { A(B b) {} }
@Service class B { B(A a) {} }
// → BeanCurrentlyInCreationException
```
Spring Boot 2.6+ **prohibits circular references by default** (`spring.main.allow-circular-references=false`).
Fixes (in order of preference):
1. **Redesign** — extract shared logic into a third bean, or use events.
2. `@Lazy` on one constructor param (injects a lazy proxy).
3. Setter/field injection (Spring resolves via early references — the old three-level singleton cache).

## 8. Bean scopes
| Scope | Lifetime | Notes |
|-------|----------|-------|
| `singleton` | One per container (default) | Must be stateless/thread-safe |
| `prototype` | New object per lookup/injection | Spring does NOT call destroy callbacks |
| `request` | One per HTTP request | Web-aware context |
| `session` | One per HTTP session | |
| `application` | One per `ServletContext` | |
| `websocket` | One per WebSocket session | |
| custom | e.g., `refresh` scope in Spring Cloud (`@RefreshScope`), thread scope | |

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { private String correlationId; }
// or @RequestScope (proxy by default)
```

### Injecting a shorter-lived bean into a singleton
A singleton is created once, so a prototype/request bean injected into it is resolved **once** — you'd always get the same instance. Solutions:
```java
// 1. Scoped proxy (proxyMode above) — proxy looks up the current instance on each call
// 2. ObjectProvider
@Service class ReportService {
    private final ObjectProvider<ReportBuilder> builders;
    void run() { ReportBuilder b = builders.getObject(); }   // new prototype each time
}
// 3. @Lookup method injection
@Component abstract class Shop { @Lookup protected abstract Cart newCart(); }
// 4. Provider<T> (jakarta.inject)
```

## 9. Lazy initialization
- Singletons are eagerly created at startup → configuration errors show up immediately.
- `@Lazy` on a bean → created on first use. On an injection point → inject a lazy-resolution proxy.
- Global: `spring.main.lazy-initialization=true` (faster startup, errors surface later; first requests slower).

## 10. Bean lifecycle in detail
```
1. Instantiation (constructor)
2. Populate properties (setter/field injection)
3. Aware interfaces: BeanNameAware.setBeanName → BeanClassLoaderAware → BeanFactoryAware
   → (ApplicationContextAware, EnvironmentAware, ... via ApplicationContextAwareProcessor)
4. BeanPostProcessor.postProcessBeforeInitialization   (@PostConstruct is invoked here by CommonAnnotationBeanPostProcessor)
5. InitializingBean.afterPropertiesSet()
6. Custom init method (@Bean(initMethod = "..."))
7. BeanPostProcessor.postProcessAfterInitialization    ← AOP proxies are created here!
8. Bean ready — in use
9. On context close: @PreDestroy → DisposableBean.destroy() → custom destroyMethod
```
```java
@Component
public class CacheWarmer implements InitializingBean, DisposableBean {
    @PostConstruct void postConstruct() { log.info("1. @PostConstruct"); }
    @Override public void afterPropertiesSet() { log.info("2. afterPropertiesSet"); }
    @PreDestroy void preDestroy() { log.info("3. @PreDestroy"); }
    @Override public void destroy() { log.info("4. destroy"); }
}
```
Prefer `@PostConstruct`/`@PreDestroy` (jakarta.annotation) — no coupling to Spring interfaces. For work after the **whole app** is ready (e.g., warm caches that call other beans), use `@EventListener(ApplicationReadyEvent.class)`, `ApplicationRunner`, or `CommandLineRunner`. `SmartLifecycle` for start/stop ordering (message listeners, schedulers).

### Extension points
| Interface | When | Typical use |
|-----------|------|-------------|
| `BeanFactoryPostProcessor` | After definitions loaded, before beans created | Modify bean definitions, resolve placeholders |
| `BeanDefinitionRegistryPostProcessor` | Before above | Register extra bean definitions dynamically |
| `BeanPostProcessor` | Around each bean's initialization | Wrap beans in proxies, validate, inject custom annotations |
| `ApplicationListener` / `@EventListener` | On events | React to startup, custom events |
| `FactoryBean<T>` | Bean that produces another object | Complex object creation (e.g., `LocalContainerEntityManagerFactoryBean`) |
| `ImportSelector` / `ImportBeanDefinitionRegistrar` | Config processing | Powering `@Enable*` annotations |

## 11. Environment, properties & `@Value`
```java
@Value("${app.page-size:20}") int pageSize;                 // with default
@Value("${app.allowed-origins}") List<String> origins;        // comma-separated → list
@Value("#{${app.limits}}") Map<String, Integer> limits;       // SpEL map literal
@Value("#{systemProperties['user.home']}") String home;       // SpEL
@Value("#{T(java.lang.Math).random() * 100}") double rnd;
@Value("classpath:templates/mail.html") Resource template;

@Autowired Environment env;  env.getProperty("app.page-size", Integer.class, 20);  env.getActiveProfiles();

@PropertySource("classpath:custom.properties")   // plain Spring; Boot loads application.* automatically
```
For groups of properties use type-safe `@ConfigurationProperties` (see Boot notes).

### SpEL (Spring Expression Language) cheat sheet
`#{...}` expressions: `#{bean.property}`, `#{bean.method()}`, `#{T(Class).staticMethod()}`, operators `+ - and or not ?: ?.` (safe navigation), collection selection `#{list.?[age > 18]}`, projection `#{list.![name]}`. Used in `@Value`, `@PreAuthorize`, `@Cacheable(key = "#id")`, `@ConditionalOnExpression`.

## 12. Profiles
```java
@Configuration
@Profile("prod")
class ProdConfig { @Bean DataSource ds() { ... } }

@Bean @Profile("!prod") DataSource h2() { ... }            // not prod
@Profile({"dev", "test"})                                    // either
@Profile("cloud & !aws")                                     // expressions
```
Activate: `spring.profiles.active=prod` (properties), `-Dspring.profiles.active=prod`, `SPRING_PROFILES_ACTIVE=prod` env var, `@ActiveProfiles("test")` in tests. `spring.profiles.default` when none active. Boot groups: `spring.profiles.group.prod=db,metrics`.

## 13. Conditional beans
```java
@Bean
@Conditional(OnLinuxCondition.class)
public FileWatcher watcher() { ... }

class OnLinuxCondition implements Condition {
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return ctx.getEnvironment().getProperty("os.name").contains("Linux");
    }
}
```
Spring Boot provides ready-made: `@ConditionalOnProperty`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnBean`, `@ConditionalOnWebApplication`, `@ConditionalOnExpression`, `@ConditionalOnResource`, `@ConditionalOnCloudPlatform`, `@ConditionalOnThreading`.

## 14. Application events
```java
// Publish
public record UserRegistered(Long userId, String email) {}
@Service @RequiredArgsConstructor
class UserService {
    private final ApplicationEventPublisher events;
    @Transactional
    public void register(User u) {
        repo.save(u);
        events.publishEvent(new UserRegistered(u.getId(), u.getEmail()));
    }
}

// Listen
@Component
class WelcomeMailer {
    @EventListener                                  // synchronous, same thread & transaction
    void onRegistered(UserRegistered e) { ... }

    @Async @EventListener                           // async (needs @EnableAsync)
    void audit(UserRegistered e) { ... }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)   // only if tx commits
    void sendMail(UserRegistered e) { mail.send(e.email()); }

    @EventListener(condition = "#e.email.endsWith('@corp.com')")
    void corpUser(UserRegistered e) { ... }
}
```
Built-in events: `ContextRefreshedEvent`, `ContextClosedEvent`, Boot's `ApplicationStartingEvent`, `ApplicationEnvironmentPreparedEvent`, `ApplicationReadyEvent`, `ApplicationFailedEvent`, `AvailabilityChangeEvent`.
Events decouple modules inside a monolith (Spring Modulith builds on this) — but they're in-memory; lost on crash → use outbox + broker for reliability.

## 15. Resources & i18n
```java
@Autowired ResourceLoader loader;
Resource r = loader.getResource("classpath:data/countries.json");   // also file:, https:
try (InputStream in = r.getInputStream()) { ... }

@Autowired MessageSource messages;   // messages.properties, messages_hi.properties
messages.getMessage("greeting", new Object[]{name}, Locale.forLanguageTag("hi"));
```

## 16. Other core annotations
| Annotation | Purpose |
|-----------|---------|
| `@Import(OtherConfig.class)` | Pull in other configurations |
| `@ImportResource("classpath:legacy.xml")` | Load XML config |
| `@DependsOn("flyway")` | Force creation order |
| `@Order` / `@Priority` | Ordering in lists, aspects, filters |
| `@Lazy` | Lazy init |
| `@Primary` / `@Qualifier` | Disambiguation |
| `@Scope`, `@RequestScope`, `@SessionScope` | Scopes |
| `@Value`, `@PropertySource` | Properties |
| `@Profile`, `@Conditional` | Conditional registration |
| `@EventListener` | Events |
| `@EnableAsync`/`@Async`, `@EnableScheduling`/`@Scheduled`, `@EnableCaching`, `@EnableTransactionManagement`, `@EnableAspectJAutoProxy` | Enable features (Boot auto-enables most) |
| `@Inject`, `@Named` (jakarta.inject, JSR-330) | Standard alternatives to `@Autowired`, `@Component` |
| `@Resource` (jakarta.annotation) | Inject by name |

## 17. Plain Spring (no Boot) example — understand what Boot hides
```java
@Configuration
@ComponentScan
public class GameApp {
    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(GameApp.class)) {
            ctx.getBean(GameRunner.class).run();
            System.out.println(ctx.getBeanDefinitionCount());
            Arrays.stream(ctx.getBeanDefinitionNames()).forEach(System.out::println);
        }   // close() triggers @PreDestroy
    }
}
interface GamingConsole { void up(); }
@Component @Primary class MarioGame implements GamingConsole { public void up() { System.out.println("Jump"); } }
@Component class PacmanGame implements GamingConsole { public void up() { System.out.println("Up"); } }
@Component record GameRunner(GamingConsole game) { void run() { game.up(); } }
```

## 18. Interview Q&A
**Q. What is IoC? What is DI?** IoC: control of object creation/lifecycle moves from code to a container. DI: the container passes dependencies in (constructor/setter/field) — the mechanism used to implement IoC.

**Q. Why constructor injection?** Immutability, required dependencies explicit, no partially built objects, tests without Spring, detects circular deps early.

**Q. `@Component` vs `@Bean`?** `@Component` on your class + scanning; `@Bean` on a method in a config class — for third-party classes or custom creation logic.

**Q. `@Component` vs `@Service` vs `@Repository`?** All register beans; `@Repository` adds persistence exception translation; stereotypes clarify layers and allow targeted AOP.

**Q. How does Spring resolve which bean to inject?** By type → `@Qualifier` → `@Primary` → name match → error.

**Q. Are singleton beans thread-safe?** Not automatically — Spring doesn't synchronize anything. They're safe if stateless (no mutable instance fields).

**Q. Prototype bean inside singleton — what happens?** Injected once; use `ObjectProvider`, `@Lookup` or scoped proxy for fresh instances.

**Q. Explain the bean lifecycle.** Instantiate → inject → Aware callbacks → BPP before (`@PostConstruct`) → `afterPropertiesSet` → init-method → BPP after (proxies) → use → `@PreDestroy` → `destroy` → destroy-method.

**Q. What is a BeanPostProcessor vs BeanFactoryPostProcessor?** BFPP modifies bean definitions before instantiation; BPP modifies/wraps bean instances during initialization.

**Q. How are circular dependencies handled?** Constructor cycles fail; field/setter cycles were resolved via early singleton references, but Boot disallows them by default — refactor.

**Q. What is `@Configuration(proxyBeanMethods = false)`?** Lite mode: no CGLIB proxy; inter-`@Bean` method calls create new instances; faster startup.

**Q. How do you run code at startup?** `CommandLineRunner`/`ApplicationRunner`, `@EventListener(ApplicationReadyEvent.class)`, `@PostConstruct` for bean-local init.
