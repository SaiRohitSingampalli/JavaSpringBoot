# 31 — Interview Revision Sheet, Scenario Questions & Practice Projects

> Every topic file ends with its own Q&A. This file adds **scenario-based questions** (what interviewers ask 2–6 YOE backend devs), a **one-page rapid revision**, and **projects** to build so the notes turn into skills.

## Part A — Rapid revision (one-liners)

### Java
- JVM = class loader + runtime areas (heap, stack, metaspace) + execution engine (interpreter, JIT, GC).
- String immutable → pool, security, thread safety, cached hash. Compare with `equals`.
- `equals`/`hashCode` contract; records generate both.
- HashMap: array of buckets, hash spreading, `equals` on collision, treeify at 8 (cap ≥ 64), resize at 0.75 load.
- ArrayList (array, O(1) get) > LinkedList almost always. ArrayDeque for stack/queue.
- ConcurrentHashMap: CAS + per-bin locks, no nulls, atomic `compute/merge`.
- Checked vs unchecked; try-with-resources; don't swallow; preserve cause.
- Streams lazy, single-use; `map` vs `flatMap`; `groupingBy` + downstream collectors.
- Optional for return types only; `orElseGet` for lazy defaults.
- `volatile` = visibility; `synchronized` = visibility + mutual exclusion; atomics = lock-free CAS.
- Executors: bounded queues + rejection policy in prod; CompletableFuture with custom executor for I/O.
- Virtual threads (21): cheap blocking threads for I/O; don't pool them; limit downstream with semaphores/pools.
- Java 17 baseline features: records, sealed, pattern matching, text blocks, switch expressions.

### Spring core / Boot
- IoC container creates & wires beans; prefer constructor injection.
- Resolution: type → `@Qualifier` → `@Primary` → name.
- Singleton default; beans must be stateless. Prototype in singleton → `ObjectProvider`.
- Lifecycle: construct → inject → aware → `@PostConstruct` → BPP after (proxies) → `@PreDestroy`.
- AOP proxies power `@Transactional`, `@Async`, `@Cacheable`, `@PreAuthorize` → self-invocation & private methods bypass them.
- `@SpringBootApplication` = config + auto-config + component scan.
- Auto-config: `AutoConfiguration.imports` + `@Conditional*`; your beans win (`@ConditionalOnMissingBean`); debug with `--debug`.
- Config precedence: CLI > sys props > env vars > profile files > application.yml; use `@ConfigurationProperties`.

### Web / data / security
- DispatcherServlet → HandlerMapping → HandlerAdapter → converters → controller → advice for errors.
- DTOs + `@Valid`; `@RestControllerAdvice` + ProblemDetail; correct status codes; 201 + Location.
- JPA: persistence context, dirty checking, LAZY everywhere, N+1 → join fetch/entity graph/batch size/DTO projections; OSIV off; `@Version`.
- `@Transactional`: service layer, rollback on unchecked only, REQUIRED vs REQUIRES_NEW, readOnly for queries, keep transactions short.
- Security 6: `SecurityFilterChain` beans, stateless JWT resource server, BCrypt, `@EnableMethodSecurity`, 401 vs 403, CSRF only for cookie auth.

### Microservices
- DB per service; Saga + Outbox + idempotent consumers; avoid 2PC.
- Gateway, discovery (Eureka/K8s DNS), config (Config Server/ConfigMaps), resilience (timeouts, retry, CB, bulkhead), observability (Micrometer, OTel, Prometheus, Grafana), containers (Docker), orchestration (K8s).
- Circuit breaker: CLOSED → OPEN → HALF_OPEN.
- Kafka ordering per partition by key; consumer groups; at-least-once + idempotency; DLT.

## Part B — Scenario-based questions (with model answers)

**1. Your API p99 latency jumped from 200ms to 3s after a release. How do you investigate?**
Check dashboards (which endpoints, since when, error rate), correlate with deploy, look at traces for slow spans (DB? downstream?), inspect DB metrics (slow query log, connection pool pending), compare SQL counts (N+1), check GC/CPU/thread pool saturation; mitigate (rollback/feature flag), then fix root cause and add a regression test/alert.

**2. `HikariPool - Connection is not available, request timed out after 30000ms` in production.**
Pool exhausted: long transactions (remote calls inside `@Transactional`), connection leaks (unclosed in custom JDBC), slow queries/locks, traffic spike, pool too small vs concurrency (virtual threads!), OSIV holding connections. Look at `hikaricp.connections.pending/active`, enable `leak-detection-threshold`, thread dumps, DB `pg_stat_activity`. Fix design, tune pool, add timeouts.

**3. Two users buy the last item simultaneously and both succeed. Fix it.**
Race condition (lost update). Options: atomic update `UPDATE stock SET qty = qty - 1 WHERE sku = ? AND qty > 0` and check affected rows; optimistic locking with `@Version` + retry/409; pessimistic `SELECT ... FOR UPDATE`; Redis atomic `DECR` for flash sales; DB check constraint `qty >= 0` as last line of defence.

**4. A payment request times out on the client, who retries — the customer is charged twice. How do you prevent it?**
Idempotency key per payment attempt (header), stored with request hash and result (unique constraint); repeated key returns stored result; propagate idempotency key to the payment provider; webhook dedup by event id.

**5. `@Transactional` method saved data even though an exception was thrown.**
Checked exception (default commit) → `rollbackFor`; exception caught and swallowed; method called via self-invocation (no transaction at all — each repository call committed itself); method not public; different thread.

**6. LazyInitializationException when returning an entity from a controller.**
Serialization touches lazy association after the transaction ended (OSIV off). Fetch needed associations in the service (join fetch/entity graph) and map to DTO inside the transaction. Don't switch to EAGER.

**7. Service B is down and Service A's threads are all stuck, taking A down too.**
Cascading failure: add connect/read timeouts, circuit breaker with fallback, bulkhead (separate pool/semaphore for B), retries with backoff only for idempotent calls, async/queue-based integration where possible, readiness unaffected by B (don't fail liveness).

**8. How do you update an order in Service A and reliably notify Service B?**
Transactional outbox: write order + outbox event in one local transaction; relay/Debezium publishes to Kafka; B consumes idempotently. Avoid "save then publish" dual writes.

**9. Saga failure: payment succeeded but inventory reservation failed.**
Compensate: refund payment, mark order CANCELLED; ensure compensations are idempotent and retried; track saga state (orchestrator state machine or event log); alert on stuck sagas.

**10. You need a scheduled job, but the app runs on 5 pods.**
ShedLock with DB/Redis lock, Quartz cluster mode, or move it to a Kubernetes CronJob / dedicated worker. Make the job idempotent anyway.

**11. Memory keeps growing and pods get OOMKilled.**
Distinguish heap OOM vs container OOMKill (exit 137). Check heap metrics after GC; heap dump (MAT) for leaks (unbounded caches, ThreadLocals, static maps); non-heap (metaspace, threads, direct buffers) if heap is fine; set `MaxRAMPercentage` and realistic limits.

**12. How do you design a REST API for a resource with partial updates and concurrent editors?**
PATCH with JSON Merge Patch or explicit update DTO; ETag/`@Version` with `If-Match` → 412/409 on conflicts; validation; return updated representation.

**13. How would you add caching to a product catalog read API?**
Cache-aside with `@Cacheable` on service (DTOs), Caffeine L1 + Redis L2 for multiple instances, TTL + evict on update (`@CacheEvict`/events), protect against stampede (`sync=true`), measure hit rate; HTTP caching/ETag/CDN for public data.

**14. Kafka consumer is lagging behind more and more.**
Check processing time per message (slow DB/HTTP), partitions vs consumer count (scale consumers up to partition count), batch processing, increase partitions (plan key distribution), async processing within ordering constraints, avoid long rebalances (`max.poll.interval.ms`), check DLT/poison pills and errors causing retries.

**15. How do you secure service-to-service calls?**
OAuth2 client credentials (JWT with audience/scopes) validated by each service, or mTLS via service mesh; network policies; no shared static API keys in code; short-lived credentials; least privilege.

**16. How do you migrate a monolith to microservices without big-bang?**
Strangler fig: gateway in front, extract one bounded context at a time, sync data via CDC/events, anti-corruption layer, move traffic gradually, remove old code; modular monolith first to discover boundaries.

**17. How do you do zero-downtime deployments with DB schema changes?**
Rolling updates + backward-compatible migrations (expand/contract), feature flags, readiness probes, graceful shutdown; never drop/rename columns used by the running version.

**18. Your endpoint returns 403 for a valid admin user.**
Role prefix mismatch (`hasRole("ADMIN")` expects `ROLE_ADMIN` authority; JWT roles mapped to `SCOPE_` by default), matcher order (`anyRequest()` before specific rule), CSRF on POST (returns 403 in Spring Security!), method security expression; enable security TRACE logging.

**19. How do you handle file uploads of 2GB?**
Don't stream through the app server memory — pre-signed URL direct to object storage (multipart upload), then process asynchronously via event; if through app, stream (`InputStream`), raise multipart limits, avoid `getBytes()`.

**20. How do you make an endpoint that triggers a 10-minute report?**
Return `202 Accepted` with job id/location; process asynchronously (queue/worker, Spring Batch); status endpoint or webhook/SSE/email when done; store result in object storage.

**21. Explain a challenging bug you fixed. (Behavioural)**
Use STAR: Situation, Task, Action (diagnosis with metrics/traces/logs, hypothesis, fix, test), Result (numbers: latency −80%, incidents eliminated), plus what you changed in process.

**22. Explain your current project architecture.**
Prepare a 2-minute story: business domain, services & responsibilities, tech stack (Java 21, Spring Boot 3, Postgres, Kafka, Redis, K8s), request flow, data flow/events, your contributions, scale numbers (RPS, data size), challenges & trade-offs, monitoring, CI/CD.

## Part C — Coding round checklist for Java backend roles
- Strings/arrays/hashmaps/two pointers/sliding window/stack/queue/binary search/sorting/BFS-DFS/basic DP — LeetCode easy–medium.
- Java-specific: stream API problems, collections usage, implementing LRU cache, thread-safe singleton, producer–consumer with BlockingQueue, print odd/even with two threads, custom immutable class, equals/hashCode, custom exception, comparator chains.
- Spring live coding: build a CRUD REST API with validation, exception handling, JPA, pagination and tests in 45–60 minutes.
- SQL: joins, group by/having, window functions (top N per group), Nth highest salary.

Two-thread odd/even (classic):
```java
public class OddEven {
    private int n = 1; private final int max = 10;
    public synchronized void print(boolean odd) throws InterruptedException {
        while (n <= max) {
            while (n <= max && (n % 2 == 1) != odd) wait();
            if (n <= max) { System.out.println(Thread.currentThread().getName() + ": " + n++); notifyAll(); }
        }
    }
    public static void main(String[] args) {
        OddEven oe = new OddEven();
        new Thread(() -> { try { oe.print(true); } catch (InterruptedException ignored) {} }, "odd").start();
        new Thread(() -> { try { oe.print(false); } catch (InterruptedException ignored) {} }, "even").start();
    }
}
```

## Part D — Practice projects (build these to own the material)

### Project 1 — Course-style foundations (Java + Spring core)
- JShell exercises for every Java topic; small console apps: bank account (OOP, exceptions), student grades (collections + streams), concurrent file downloader (ExecutorService/CompletableFuture/virtual threads).
- Plain Spring (no Boot) "gaming app": interface + multiple implementations, `@Primary`/`@Qualifier`, scopes, lifecycle callbacks.

### Project 2 — Social media REST API (Spring Boot 3)
- Users & posts: CRUD, `@Valid`, global exception handling with ProblemDetail, HATEOAS link, content negotiation (XML), i18n, versioning (4 strategies), dynamic filtering, OpenAPI docs.
- JPA with H2 → Postgres (Docker), one-to-many User → Posts, Flyway migrations.
- Spring Security: basic auth → JWT resource server; method security.
- Tests: unit (Mockito), `@WebMvcTest`, `@DataJpaTest` with Testcontainers.
- Actuator + Prometheus + Grafana via Docker Compose.

### Project 3 — Todo / Task manager full stack (optional frontend)
- React/Angular frontend with JWT login, Spring Boot backend, CORS, pagination, sorting, search with Specifications.
- Deploy: Docker image → AWS Elastic Beanstalk/App Runner or a K8s cluster.

### Project 4 — Currency microservices (Spring Cloud)
- limits-service + Config Server (Git backed, profiles).
- currency-exchange (JPA) + currency-conversion (RestClient → Feign).
- Eureka naming server; run 2–3 exchange instances; observe load balancing.
- Spring Cloud Gateway: discovery routes, custom routes, rewrite path, global logging filter.
- Resilience4j: retry, circuit breaker, rate limiter, bulkhead; observe states via actuator.
- Micrometer Tracing + Zipkin; trace a request gateway → conversion → exchange.
- Docker images via buildpacks; Docker Compose for everything.
- Kubernetes: Deployments/Services for each, remove Eureka (K8s DNS), ConfigMaps, probes, HPA, rolling update & rollback.

### Project 5 — Event-driven e-commerce (advanced, résumé-worthy)
- Services: catalog, cart (Redis), order, payment (mock gateway, idempotency keys), inventory, notification.
- Kafka events with outbox (Debezium or polling), saga for checkout with compensations, DLT handling, idempotent consumers.
- Gateway with OAuth2 (Keycloak) + JWT resource servers; client credentials between services.
- Observability: OTel → Tempo/Zipkin, Prometheus/Grafana dashboards & alerts, structured JSON logs to Loki.
- CI/CD: GitHub Actions → image scan → push → Argo CD to a kind/minikube cluster; Testcontainers integration tests; contract tests.
- Load test with k6/Gatling; tune pool sizes; try virtual threads; document results.

## Part E — Suggested 12-week study plan
| Week | Focus | Files |
|------|-------|-------|
| 1 | Java basics, OOP | 02, 03 |
| 2 | Exceptions, collections, generics | 04, 05, 06 |
| 3 | Functional & streams, modern Java | 07, 09 |
| 4 | Concurrency, JVM | 08, 01 |
| 5 | Design patterns, Spring core, AOP | 10, 11, 12 |
| 6 | Spring Boot internals, REST, validation | 13, 14, 15 |
| 7 | JPA/Hibernate, Spring Data, transactions | 16, 17, 28 |
| 8 | Security, testing | 18, 19 |
| 9 | Caching/async/scheduling, messaging | 20, 21 |
| 10 | Microservices patterns, Spring Cloud | 23, 24 |
| 11 | Observability, Docker, Kubernetes, CI/CD | 25, 26, 27, 30 |
| 12 | System design, reactive overview, mock interviews | 29, 22, 31 |
Build projects 1–4 alongside; finish with project 5.
