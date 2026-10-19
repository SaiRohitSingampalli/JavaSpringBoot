# 25 — Observability: Actuator, Logging, Metrics, Tracing

> "You can't fix what you can't see." Production readiness = health checks + metrics + structured logs + traces + alerts.

## 1. Spring Boot Actuator
Dependency: `spring-boot-starter-actuator`. Base path `/actuator`.

| Endpoint | Purpose |
|----------|---------|
| `health` | UP/DOWN with component details (db, disk, redis, kafka, circuitbreakers…); liveness/readiness groups |
| `info` | App info (build, git, custom `info.*` properties, java, os) |
| `metrics` | List & query Micrometer metrics: `/actuator/metrics/http.server.requests?tag=uri:/api/orders` |
| `prometheus` | Metrics in Prometheus format (needs `micrometer-registry-prometheus`) |
| `loggers` | View/change log levels at runtime |
| `env` | Property sources (values masked by default in Boot 3) |
| `configprops` | `@ConfigurationProperties` beans (masked) |
| `beans` | All beans |
| `conditions` | Auto-configuration report |
| `mappings` | All request mappings |
| `threaddump`, `heapdump` | Diagnostics |
| `scheduledtasks`, `caches`, `flyway`, `liquibase`, `httpexchanges` (needs repository bean), `startup`, `sbom`, `shutdown` (disabled by default) | |

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers   # default exposes only health (and info in older versions)
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized        # never | always | when-authorized
      probes.enabled: true                  # auto on Kubernetes
      group:
        readiness.include: readinessState,db,redis
  server.port: 9090                         # separate management port (not exposed publicly)
  info:
    env.enabled: true
    git.mode: full
info.app: { name: shop, team: payments }
```
**Security**: never expose `env`, `heapdump`, `beans`, `shutdown` publicly; separate port or require ADMIN role.

### Custom health indicator
```java
@Component
public class PaymentGatewayHealth implements HealthIndicator {
    public Health health() {
        try {
            gateway.ping();
            return Health.up().withDetail("latencyMs", 42).build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```
**Careful**: don't put external dependencies in the **liveness** check — if the DB is down, restarting your pods won't help (restart storm). Liveness = "is this process healthy/deadlocked?"; Readiness = "can it serve traffic now?" (DB reachable, warm-up done).

### Custom endpoint
```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {
    @ReadOperation public Map<String, Boolean> features() { return flags.all(); }
    @WriteOperation public void toggle(@Selector String name, boolean enabled) { flags.set(name, enabled); }
}
```

## 2. Logging (production practices)
- SLF4J API; Logback default; Log4j2 optional.
- **Structured JSON logs** (Boot 3.4+): `logging.structured.format.console=ecs` (or `logstash`, `gelf`) — machine-parseable for ELK/Loki/Datadog.
- **MDC** (Mapped Diagnostic Context) for correlation: traceId/spanId added automatically by Micrometer Tracing; add userId/tenantId/orderId in filters.
- Levels: ERROR (needs attention), WARN (unexpected but handled), INFO (business milestones, startup), DEBUG/TRACE (dev diagnostics).
- Log **once** per error at the boundary with stack trace; include context ids; **no secrets/PII** (passwords, tokens, card numbers, Aadhaar/PAN, emails as per policy) — mask them.
- Use parameterized messages (`log.info("Order {} paid", id)`), guard expensive debug building with `log.isDebugEnabled()` or lambdas (`log.atDebug().addArgument(() -> expensive()).log(...)`).
- Async appenders for throughput; log rotation; centralized shipping (Fluent Bit / Filebeat / OTel Collector) from stdout in containers.
- Audit logs (who did what when) are separate from application logs with stronger retention/integrity.

## 3. Metrics with Micrometer
Micrometer = "SLF4J for metrics": vendor-neutral API with registries for Prometheus, Datadog, New Relic, CloudWatch, OTLP, etc.

### Meter types
| Type | Use | Example |
|------|-----|---------|
| **Counter** | Monotonic count | orders placed, errors |
| **Gauge** | Current value sampled | queue size, active sessions, cache size |
| **Timer** | Latency + count (+ percentiles/histograms) | method duration, HTTP calls |
| **DistributionSummary** | Distribution of non-time values | payload sizes, order amounts |
| LongTaskTimer | In-flight long tasks | batch jobs |

```java
@Service
public class OrderMetrics {
    private final Counter ordersPlaced;
    private final Timer paymentTimer;

    public OrderMetrics(MeterRegistry registry, OrderQueue queue) {
        ordersPlaced = Counter.builder("shop.orders.placed").description("Orders placed")
            .tag("channel", "web").register(registry);
        paymentTimer = Timer.builder("shop.payment.duration").publishPercentileHistogram().register(registry);
        Gauge.builder("shop.orders.queue.size", queue, OrderQueue::size).register(registry);
    }
    public void placed() { ordersPlaced.increment(); }
    public <T> T timePayment(Supplier<T> s) { return paymentTimer.record(s); }
}

@Timed(value = "shop.checkout", histogram = true)    // needs TimedAspect bean / observation annotations enabled
public Receipt checkout(Cart c) { ... }
```
**Tags / labels** give dimensions (uri, status, method, outcome). Avoid **high cardinality** tags (user ids, order ids, raw URLs) — they explode time series and memory.

Auto-collected metrics: `http.server.requests` (uri, method, status, outcome), `jvm.memory.*`, `jvm.gc.*`, `jvm.threads.*`, `process.cpu.usage`, `system.cpu.usage`, `hikaricp.connections.*` (active, pending, timeout), `tomcat.*`, `spring.data.repository.invocations`, `cache.*`, `kafka.*`, `resilience4j.*`, `executor.*`, `logback.events`.

### Observation API (Boot 3)
One instrumentation → metrics + traces + logs:
```java
Observation.createNotStarted("order.place", observationRegistry)
    .lowCardinalityKeyValue("channel", "web")
    .highCardinalityKeyValue("orderId", id.toString())     // only on spans, not metric tags
    .observe(() -> placeOrder(id));

@Observed(name = "inventory.reserve", contextualName = "reserve-stock")   // with ObservedAspect bean
public void reserve(...) {}
```

### Prometheus + Grafana
- Prometheus **scrapes** `/actuator/prometheus` (pull model) every 15–30s and stores time series.
- PromQL examples:
  ```
  rate(http_server_requests_seconds_count{uri="/api/orders",status=~"5.."}[5m])                    # 5xx per second
  histogram_quantile(0.99, sum by (le) (rate(http_server_requests_seconds_bucket{uri="/api/orders"}[5m])))   # p99 latency
  hikaricp_connections_pending > 0
  sum(rate(shop_orders_placed_total[1h]))
  ```
- Grafana dashboards (JVM Micrometer dashboard, Spring Boot dashboard), **Alertmanager** for alerts.
- Percentiles: prefer histograms (`publishPercentileHistogram`) so percentiles can be aggregated across instances.

### What to monitor (Spring service)
- **RED**: request rate, error rate, duration (p50/p95/p99) per endpoint.
- JVM: heap after GC, GC pause time, threads, CPU.
- DB pool: active/pending connections, acquisition time.
- Downstream calls: latency, errors, circuit breaker state.
- Messaging: consumer lag, DLT count, processing time.
- Business KPIs: orders/min, payment success rate.
- Saturation: thread pool queue sizes, CPU throttling in containers.

## 4. Distributed tracing (recap)
- Trace = tree of spans across services; context propagated by W3C `traceparent` header.
- Micrometer Tracing + OpenTelemetry/Brave → Zipkin, Jaeger, Grafana Tempo, Datadog, etc.
- Sampling: 100% in dev; probabilistic or tail-based sampling in prod (keep all errors/slow traces via collector).
- **Exemplars** link a metric data point to a trace id (Grafana).
- OpenTelemetry Java agent: zero-code auto-instrumentation alternative; Spring Boot 4 adds a `spring-boot-starter-opentelemetry`.

## 5. Alerting & SRE basics
- **SLI** (measured indicator: % requests < 300ms), **SLO** (target: 99.9% over 30 days), **SLA** (contractual).
- **Error budget** = 100% − SLO; drives release velocity decisions.
- Alert on user-facing symptoms (SLO burn rate, error rate, latency), not every CPU spike.
- Runbooks per alert; on-call rotation; blameless postmortems.
- Incident flow: detect → triage (dashboards, traces, logs) → mitigate (rollback, scale, feature flag off) → root cause → follow-ups.

## 6. Debugging a slow endpoint — a practical walkthrough
1. Grafana: p99 latency for `/api/orders` jumped at 10:05; error rate normal; deploy happened at 10:00.
2. Traces: slow traces show 2s in `SELECT ... FROM order_items` span, repeated 50× per request → **N+1** introduced by new field in DTO.
3. Logs (by traceId): confirm queries.
4. Fix: `@EntityGraph`/join fetch; add test asserting query count; roll forward.
5. Postmortem: add query-count check to CI, alert on DB time per request.

## 7. Interview Q&A
**Q. What is Spring Boot Actuator?** Production-ready endpoints for health, metrics, info, env, loggers, thread/heap dumps, integrated with Micrometer.

**Q. Liveness vs readiness?** Liveness: should the container be restarted? Readiness: should it receive traffic now?

**Q. How do you expose metrics to Prometheus?** Add `micrometer-registry-prometheus`, expose `prometheus` endpoint; Prometheus scrapes it.

**Q. Counter vs Gauge vs Timer?** Monotonic count; instantaneous value; duration + count distribution.

**Q. What is high cardinality and why avoid it in metrics?** Tags with many unique values create huge numbers of series → memory/cost explosion; put such data in traces/logs instead.

**Q. How do you correlate logs across services?** Propagated trace ids in MDC, structured logs, central log store searchable by traceId.

**Q. What are the three pillars of observability?** Logs, metrics, traces (plus profiling and events).
