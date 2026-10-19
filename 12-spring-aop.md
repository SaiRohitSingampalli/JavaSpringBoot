# 12 — Spring AOP (Aspect-Oriented Programming)

## 1. Why AOP?
**Cross-cutting concerns** are needed in many layers but aren't business logic: logging, auditing, security checks, transactions, caching, metrics, retries, exception translation.

Without AOP:
```java
public Order place(OrderRequest r) {
    long start = System.currentTimeMillis();
    log.info("place called with {}", r);
    securityCheck();
    TransactionStatus tx = txManager.getTransaction(def);
    try {
        Order o = ... // 3 lines of real logic
        txManager.commit(tx);
        return o;
    } catch (Exception e) { txManager.rollback(tx); throw e; }
    finally { metrics.record(System.currentTimeMillis() - start); }
}
```
With AOP the method contains only business logic; the concerns are declared once in **aspects** and applied wherever needed.

## 2. Terminology
| Term | Meaning | Example |
|------|---------|---------|
| **Aspect** | Module containing a cross-cutting concern | `LoggingAspect` |
| **Join point** | A point during execution where advice can apply. In Spring AOP: **method execution only** | `OrderService.place()` call |
| **Advice** | Action taken at a join point | log before the method |
| **Pointcut** | Expression selecting join points | `execution(* com.shop.service.*.*(..))` |
| **Target object** | The original bean being advised | `OrderService` instance |
| **Proxy** | Object created by AOP wrapping the target | CGLIB subclass of OrderService |
| **Weaving** | Linking aspects with targets | Spring: runtime via proxies |
| **Introduction** | Add new interfaces/methods to a bean | `@DeclareParents` (rare) |

## 3. Advice types
| Advice | Runs | Can stop execution / change result? |
|--------|------|------------------------------------|
| `@Before` | Before method | Only by throwing |
| `@AfterReturning` | After normal return; can access return value | Can't replace result (can mutate object) |
| `@AfterThrowing` | After exception thrown | Can't swallow; can translate by throwing another |
| `@After` (finally) | After method regardless of outcome | No |
| `@Around` | Wraps method: you call `proceed()` | **Yes** — full control (skip, retry, change args/result) |

Use the least powerful advice that works; `@Around` is most flexible but easiest to get wrong (forgetting `proceed()` or its return value).

## 4. Setup
Spring Boot: add `spring-boot-starter-aop` (AspectJ weaver — note: in Boot 4 the starter is renamed `spring-boot-starter-aspectj`). Auto-enables `@EnableAspectJAutoProxy`.

## 5. Complete example
```java
@Aspect
@Component
@Slf4j
public class ApplicationAspects {

    // ---------- Reusable pointcuts ----------
    @Pointcut("within(com.shop..service..*)")
    public void serviceLayer() {}

    @Pointcut("within(com.shop..web..*)")
    public void webLayer() {}

    @Pointcut("@annotation(com.shop.common.TrackTime)")
    public void trackTimeAnnotation() {}

    // ---------- Before ----------
    @Before("serviceLayer()")
    public void logEntry(JoinPoint jp) {
        log.debug("→ {}.{}({})", jp.getSignature().getDeclaringTypeName(),
                  jp.getSignature().getName(), Arrays.toString(jp.getArgs()));
    }

    // ---------- AfterReturning ----------
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logExit(JoinPoint jp, Object result) {
        log.debug("← {} returned {}", jp.getSignature().toShortString(), result);
    }

    // ---------- AfterThrowing ----------
    @AfterThrowing(pointcut = "serviceLayer() || webLayer()", throwing = "ex")
    public void logException(JoinPoint jp, Exception ex) {
        log.error("✖ {} threw {}", jp.getSignature().toShortString(), ex.getMessage());
    }

    // ---------- After (finally) ----------
    @After("execution(* com.shop..*Repository.*(..))")
    public void afterRepo(JoinPoint jp) { /* metrics */ }

    // ---------- Around ----------
    @Around("trackTimeAnnotation()")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();                     // MUST call and return the result
        } finally {
            log.info("{} took {} ms", pjp.getSignature().toShortString(),
                     (System.nanoTime() - start) / 1_000_000);
        }
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface TrackTime {}
```

### Binding arguments & annotations
```java
@Before("serviceLayer() && args(orderId, ..)")
public void check(Long orderId) { ... }

@Around("@annotation(audit)")
public Object audit(ProceedingJoinPoint pjp, Audited audit) throws Throwable {
    String action = audit.action();           // read annotation attributes
    ...
}
```

## 6. Pointcut expression language (AspectJ)

### `execution` — the most common
```
execution( modifiers? return-type declaring-type? method-name(params) throws? )

execution(public * *(..))                          any public method
execution(* com.shop.service.OrderService.*(..))   any method in OrderService
execution(* com.shop.service.*.*(..))              any method of any class in package service
execution(* com.shop..*.*(..))                     any method in com.shop or sub-packages
execution(* get*(..))                              methods starting with get
execution(* *(Long, ..))                           first param Long
execution(String com.shop..*.find*(*))             returns String, exactly one param
```
Wildcards: `*` one element, `..` any number of params or sub-packages.

### Other designators
| Designator | Matches |
|-----------|---------|
| `within(com.shop.service..*)` | All methods of types in package |
| `@within(org.springframework.stereotype.Service)` | Types annotated `@Service` |
| `@annotation(com.shop.TrackTime)` | Methods annotated |
| `args(java.lang.String, ..)` | Runtime argument types |
| `@args(...)` | Args whose types carry annotation |
| `this(Type)` | Proxy is instance of Type |
| `target(Type)` | Target object is instance of Type |
| `bean(*Service)` | Spring-only: bean name pattern |

Combine with `&&`, `||`, `!`.

## 7. How Spring AOP works — proxies
At startup, `AnnotationAwareAspectJAutoProxyCreator` (a **BeanPostProcessor**) checks every bean; if any pointcut matches, it returns a **proxy** instead of the raw bean. Other beans get the proxy injected.

```
Caller ──► Proxy ──[advice chain: tx, security, logging...]──► Target bean method
```

| Proxy type | When | Mechanism | Limitations |
|-----------|------|-----------|-------------|
| **JDK dynamic proxy** | Target implements interfaces and `proxyTargetClass=false` | `java.lang.reflect.Proxy`, implements same interfaces | Only interface methods; inject by interface type |
| **CGLIB** | Default in Spring Boot (`spring.aop.proxy-target-class=true`) | Generates a subclass at runtime | Can't advise `final` classes/methods; private methods not proxied |

### The self-invocation trap ⚠️
```java
@Service
public class OrderService {
    public void placeAll(List<Order> orders) {
        orders.forEach(this::place);           // `this` is the TARGET, not the proxy!
    }
    @Transactional
    public void place(Order o) { ... }         // transaction NOT applied when called from placeAll
}
```
The call never passes through the proxy. Same for `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize`.
**Fixes**: move the method to another bean (best), inject self lazily (`@Lazy OrderService self`), use `TransactionTemplate`, or use AspectJ compile/load-time weaving.

Other proxy gotchas:
- `private`, `final`, and `static` methods aren't advised.
- Advice on non-public methods: CGLIB can intercept `protected`/package methods in Spring 6, but `@Transactional` historically required public — stick to public service methods.
- Calls inside constructors / `@PostConstruct` run before the proxy is used by callers.

## 8. Spring AOP vs AspectJ
| | Spring AOP | AspectJ |
|--|--|--|
| Weaving | Runtime proxies | Compile-time, post-compile, or load-time |
| Join points | Method execution on Spring beans | Methods, constructors, field access, static init… |
| Self-invocation | Not intercepted | Intercepted |
| Performance | Small proxy overhead | Faster at runtime |
| Setup | Zero-config | Needs `ajc` compiler or agent |
| Use | 95% of apps | Advanced cases |

Spring AOP uses AspectJ's **annotations and pointcut language**, but not its weaver.

## 9. Ordering multiple aspects
```java
@Aspect @Component @Order(1) class SecurityAspect {}   // lower = higher precedence = outermost
@Aspect @Component @Order(2) class LoggingAspect {}
```
Highest-precedence aspect runs **first on the way in** and **last on the way out**. Spring's own: transaction advisor default order is `Ordered.LOWEST_PRECEDENCE` (configurable via `@EnableTransactionManagement(order=...)`).

## 10. Features built on AOP in Spring
| Feature | Annotation |
|---------|-----------|
| Declarative transactions | `@Transactional` |
| Caching | `@Cacheable`, `@CacheEvict`, `@CachePut` |
| Async execution | `@Async` |
| Method security | `@PreAuthorize`, `@PostAuthorize`, `@Secured` |
| Retry | `@Retryable` (Spring Retry; Spring Framework 7 core resilience) |
| Method validation | `@Validated` on class + constraints on params |
| Exception translation | `@Repository` → `PersistenceExceptionTranslationPostProcessor` |
| Resilience4j | `@CircuitBreaker`, `@Retry`, `@RateLimiter` |
| Micrometer | `@Timed`, `@Counted`, `@Observed` |

## 11. Practical aspects you'll write in real projects
```java
// Audit log for annotated methods
@Around("@annotation(auditable)")
public Object audit(ProceedingJoinPoint pjp, Auditable auditable) throws Throwable {
    String user = SecurityContextHolder.getContext().getAuthentication().getName();
    Object result = pjp.proceed();
    auditRepo.save(new AuditEntry(user, auditable.action(), Instant.now()));
    return result;
}

// Mask sensitive arguments in logs
// Enforce tenant isolation: verify tenantId argument == current tenant
// Simple retry around external clients (prefer Resilience4j in production)
// Feature flags: skip method if flag disabled
```

## 12. Interview Q&A
**Q. What is AOP and why use it?** Modularizing cross-cutting concerns (logging, tx, security) separately from business logic for cleaner, DRY code.

**Q. Join point vs pointcut vs advice?** Join point: where (a method execution). Pointcut: expression selecting join points. Advice: what code runs and when.

**Q. Types of advice?** Before, AfterReturning, AfterThrowing, After (finally), Around.

**Q. How does Spring implement AOP?** Runtime proxies created by a BeanPostProcessor — JDK dynamic proxies for interfaces or CGLIB subclasses (Boot default).

**Q. Why doesn't `@Transactional` work when calling a method from the same class?** Self-invocation bypasses the proxy, so advice isn't applied.

**Q. Can you advise private or final methods?** No with Spring AOP proxies; yes with AspectJ weaving.

**Q. JDK proxy vs CGLIB?** JDK: interface-based, requires interfaces. CGLIB: subclass-based, works on classes, can't handle final.

**Q. Spring AOP vs AspectJ?** Spring AOP: proxy-based, method execution only, simple. AspectJ: full weaving, all join points, more complex.
