# 20 — Caching, Async Processing, Scheduling & Batch

## Part A — Caching

### 1. Why cache?
Reduce latency and load on DBs/downstream services for data that's read often and changes rarely. "There are only two hard things in computer science: cache invalidation and naming things."

### 2. Caching patterns
| Pattern | How | Notes |
|---------|-----|-------|
| **Cache-aside** (lazy loading) | App reads cache → miss → read DB → put in cache | Most common; what `@Cacheable` does |
| Read-through | Cache itself loads from DB on miss | Cache provider integration |
| **Write-through** | Write to cache and DB synchronously | Consistent, slower writes |
| Write-behind (write-back) | Write cache, DB updated asynchronously | Fast, risk of loss |
| Refresh-ahead | Refresh hot keys before expiry | |

**Eviction policies**: LRU (least recently used), LFU, FIFO, TTL (time-to-live), size-based. Caffeine uses W-TinyLFU (excellent hit rates).

**Problems**:
- **Stale data** → TTLs, evict on write, events.
- **Cache stampede / thundering herd** — many requests miss simultaneously and hammer the DB → `@Cacheable(sync = true)`, locking, early refresh, jittered TTLs.
- **Cache penetration** — repeated lookups for non-existent keys → cache nulls/negative results briefly, bloom filters.
- **Cache avalanche** — many keys expire together → randomized TTL.
- **Consistency across instances** — local caches diverge → distributed cache (Redis) or invalidation messages.

### 3. Spring Cache abstraction
```java
@SpringBootApplication
@EnableCaching
public class ShopApplication {}

@Service
@RequiredArgsConstructor
public class ProductService {

    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public ProductResponse get(Long id) { return repo.findById(id).map(mapper::toResponse).orElse(null); }

    @Cacheable(value = "productsByCategory", key = "#category + ':' + #page", condition = "#page < 5")
    public List<ProductResponse> byCategory(String category, int page) { ... }

    @CachePut(value = "products", key = "#result.id()")          // always runs; updates cache
    public ProductResponse update(Long id, UpdateRequest r) { ... }

    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) { ... }

    @CacheEvict(value = "productsByCategory", allEntries = true)
    public void refreshCategories() {}

    @Caching(evict = { @CacheEvict(value = "products", key = "#id"),
                       @CacheEvict(value = "productsByCategory", allEntries = true) })
    public void archive(Long id) {}

    @Cacheable(value = "exchangeRates", sync = true)            // only one thread computes on miss
    public Rate rate(String pair) { ... }
}
```
- Default key generator: `SimpleKeyGenerator` (method params). Custom `KeyGenerator` bean possible.
- Proxy-based (AOP) → self-invocation doesn't use cache; private methods not cached.
- Cache values should be **immutable DTOs** (not JPA entities — lazy proxies, detached state) and **Serializable/JSON-friendly** for Redis.

### 4. Providers
| Provider | Type | Config |
|----------|------|--------|
| `ConcurrentMapCacheManager` | Local map, no eviction | Default if no provider (dev only) |
| **Caffeine** | Local, high-performance, TTL & size | `spring.cache.caffeine.spec=maximumSize=10000,expireAfterWrite=10m` |
| **Redis** | Distributed | `spring.cache.type=redis`, `spring.cache.redis.time-to-live=10m` |
| Ehcache 3 / Hazelcast / Infinispan (JCache) | Local/clustered | |

```java
@Bean
RedisCacheManagerBuilderCustomizer redisCaches() {
    return builder -> builder
        .withCacheConfiguration("products", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())))
        .withCacheConfiguration("exchangeRates", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofSeconds(30)));
}
```
**Multi-level**: L1 Caffeine (per instance, microseconds) + L2 Redis (shared, ~1ms).

### 5. Redis beyond caching (backend dev should know)
- Data structures: String, Hash, List, Set, Sorted Set (leaderboards), Stream, HyperLogLog, Bitmap, Geo.
- Uses: sessions (Spring Session), rate limiting counters (`INCR` + `EXPIRE`), distributed locks (`SET key val NX PX 30000`, Redisson / ShedLock), pub/sub, queues, idempotency keys, dedup.
- Single-threaded command execution → atomic ops; Lua scripts for multi-step atomicity.
- Persistence: RDB snapshots, AOF; HA via Sentinel or Redis Cluster (sharding by hash slots).
```java
@Autowired StringRedisTemplate redis;
redis.opsForValue().set("otp:" + phone, code, Duration.ofMinutes(5));
Boolean locked = redis.opsForValue().setIfAbsent("lock:order:" + id, instanceId, Duration.ofSeconds(30));
redis.opsForZSet().incrementScore("leaderboard", userId, points);
```

## Part B — Asynchronous Execution

### 6. `@Async`
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "mailExecutor")
    ThreadPoolTaskExecutor mailExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(4);
        ex.setMaxPoolSize(16);
        ex.setQueueCapacity(500);
        ex.setThreadNamePrefix("mail-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.setTaskDecorator(new MdcTaskDecorator());          // propagate MDC/trace/security context
        ex.setWaitForTasksToCompleteOnShutdown(true);
        ex.setAwaitTerminationSeconds(30);
        ex.initialize();
        return ex;
    }
}

@Service
public class NotificationService {
    @Async("mailExecutor")
    public void sendWelcome(String email) { ... }                       // fire-and-forget

    @Async
    public CompletableFuture<Report> buildReport(Long id) {             // caller gets a future
        return CompletableFuture.completedFuture(generate(id));
    }
}
```
- Default executor in Boot: auto-configured `ThreadPoolTaskExecutor` (`spring.task.execution.pool.*`), or virtual threads when `spring.threads.virtual.enabled=true`.
- Exceptions in `void` async methods are lost unless you set an `AsyncUncaughtExceptionHandler` (implement `AsyncConfigurer`).
- Proxy rules apply (self-invocation, public methods).
- No transaction/security/MDC context is carried automatically to the new thread.
- `@Async` is in-memory: if the app crashes, queued tasks are lost → use a message broker or job store for important work.

## Part C — Scheduling

### 7. `@Scheduled`
```java
@Configuration
@EnableScheduling
class SchedulingConfig {}

@Component
@Slf4j
public class Jobs {

    @Scheduled(fixedRate = 60_000)                  // every 60s from start of previous run
    public void refreshRates() {}

    @Scheduled(fixedDelayString = "PT30S", initialDelay = 10_000)   // 30s after previous finished
    public void pollInbox() {}

    @Scheduled(cron = "0 0 2 * * MON-FRI", zone = "Asia/Kolkata")  // 02:00 on weekdays
    public void nightlyReport() {}

    @Scheduled(cron = "${jobs.cleanup.cron:0 */15 * * * *}")       // configurable
    public void cleanup() {}
}
```
**Spring cron format has 6 fields**: `second minute hour day-of-month month day-of-week`. Macros: `@hourly`, `@daily`, `@weekly`, `@monthly`.
Examples: `0 0 * * * *` (every hour), `0 */5 * * * *` (every 5 min), `0 30 9 1 * *` (09:30 on the 1st), `0 0 0 L * *` (last day of month).

- Default scheduler is **single-threaded** → a slow job delays others. Set `spring.task.scheduling.pool.size=5`.
- Exceptions are logged and the schedule continues.
- **Multiple instances**: every instance runs the job! Use **ShedLock** (DB/Redis lock), Quartz in clustered mode, Kubernetes CronJobs, or a dedicated scheduler service.
```java
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "nightlyReport", lockAtMostFor = "PT30M", lockAtLeastFor = "PT1M")
public void nightlyReport() {}
```
- **Quartz** (`spring-boot-starter-quartz`): persistent jobs, misfire handling, clustering, dynamic schedules.

## Part D — Batch processing

### 8. Spring Batch overview
For large-volume, restartable, chunk-oriented processing (ETL, end-of-day settlements, CSV imports).
```
Job ── Step 1 (chunk: ItemReader → ItemProcessor → ItemWriter, commit every N items)
    └─ Step 2 (Tasklet: single task, e.g., cleanup)
JobRepository (DB tables) stores executions → restart from failure point
```
```java
@Bean
Step importStep(JobRepository repo, PlatformTransactionManager tx,
                FlatFileItemReader<CustomerCsv> reader, CustomerProcessor processor, JpaItemWriter<Customer> writer) {
    return new StepBuilder("importCustomers", repo)
        .<CustomerCsv, Customer>chunk(500, tx)
        .reader(reader).processor(processor).writer(writer)
        .faultTolerant().skip(ValidationException.class).skipLimit(100).retry(TransientDataAccessException.class).retryLimit(3)
        .build();
}
@Bean Job importJob(JobRepository repo, Step importStep) {
    return new JobBuilder("importJob", repo).start(importStep).build();
}
```
Features: skip/retry, restartability, partitioning & multi-threaded steps for scaling, listeners, readers/writers for CSV, JDBC, JPA, Kafka, JSON.

## Part E — Other useful Spring features

### 9. Retry
```java
@EnableRetry
@Retryable(retryFor = TransientException.class, maxAttempts = 3, backoff = @Backoff(delay = 500, multiplier = 2))
public Rate fetch() {}
@Recover public Rate fallback(TransientException e) { return Rate.cached(); }
```
(Spring Retry; Spring Framework 7 adds core `@Retryable` + `@EnableResilientMethods`; in microservices Resilience4j is common.) Only retry **idempotent**, **transient** failures, with backoff + jitter.

### 10. Sending email
```yaml
spring.mail: { host: smtp.example.com, port: 587, username: ..., password: ..., properties.mail.smtp.starttls.enable: true }
```
```java
MimeMessage msg = mailSender.createMimeMessage();
MimeMessageHelper h = new MimeMessageHelper(msg, true, "UTF-8");
h.setTo(to); h.setSubject("Welcome"); h.setText(htmlBody, true);
mailSender.send(msg);
```
Send after commit (transactional events) and asynchronously.

### 11. WebSockets (real-time)
`spring-boot-starter-websocket`, STOMP over WebSocket with `@MessageMapping` + `SimpMessagingTemplate.convertAndSend("/topic/prices", price)`; scale out with a broker relay (RabbitMQ). Alternatives: SSE for server→client only.

### 12. File storage
Don't store large files in DB; use object storage (S3/GCS/Azure Blob/MinIO), store metadata + key in DB, use pre-signed URLs for direct upload/download. Limit size: `spring.servlet.multipart.max-file-size=10MB`, validate content types, scan for malware.

## Interview Q&A
**Q. How does `@Cacheable` work?** AOP proxy builds a key from args, checks the cache; on hit returns cached value without invoking the method; on miss invokes and stores the result.

**Q. `@Cacheable` vs `@CachePut` vs `@CacheEvict`?** Read-through cache; always execute & update; remove entries.

**Q. Local vs distributed cache?** Local (Caffeine): fastest, per-instance, can be inconsistent across instances. Distributed (Redis): shared/consistent, network hop.

**Q. How do you avoid a scheduled job running on all instances?** ShedLock / distributed lock, Quartz clustering, K8s CronJob, leader election.

**Q. Why might `@Async` not work?** Missing `@EnableAsync`, self-invocation, non-public method, calling from the same class, or the method being invoked on a non-proxied object.

**Q. `fixedRate` vs `fixedDelay`?** fixedRate measures from start of previous execution; fixedDelay from its completion.

**Q. What's a cache stampede and how to prevent it?** Many concurrent misses recompute the same value; use sync/locking, early refresh, jittered TTLs.
