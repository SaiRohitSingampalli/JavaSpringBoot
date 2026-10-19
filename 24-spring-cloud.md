# 24 — Spring Cloud: Building Microservices with Spring Boot 3

> Hands-on reference following the classic course project: **limits-service**, **config-server**, **currency-exchange-service**, **currency-conversion-service**, **naming-server (Eureka)**, **api-gateway**, plus Resilience4j and tracing.

## 1. Spring Cloud overview
| Concern | Spring Cloud project | Alternatives |
|---------|---------------------|-------------|
| Centralized config | **Spring Cloud Config** (+ Bus for refresh) | Kubernetes ConfigMaps/Secrets, Consul, Vault, AWS AppConfig |
| Service discovery | **Netflix Eureka** | Consul, Zookeeper, Kubernetes DNS |
| Client-side load balancing | **Spring Cloud LoadBalancer** (Ribbon retired) | K8s Service, service mesh |
| Declarative HTTP client | **OpenFeign** (feature-complete) | Spring HTTP Interfaces (`@HttpExchange`) |
| API Gateway | **Spring Cloud Gateway** (reactive & MVC variants) | Kong, NGINX, Envoy, cloud gateways |
| Fault tolerance | **Spring Cloud CircuitBreaker** + **Resilience4j** (Hystrix retired) | Service mesh, Spring Framework 7 resilience |
| Tracing | **Micrometer Tracing** (Sleuth retired in Boot 3) + Zipkin/OTel | |
| Messaging abstraction | **Spring Cloud Stream** | Spring Kafka / AMQP directly |
| Functions/serverless | Spring Cloud Function | |
| Contract testing | Spring Cloud Contract | Pact |
| Kubernetes integration | Spring Cloud Kubernetes | |
| Secrets | Spring Cloud Vault | |

### Release trains & compatibility
Spring Cloud versions are calendar-based release trains, each matching a Boot generation:
| Release train | Spring Boot |
|---------------|-------------|
| 2022.0.x (Kilburn) | 3.0 |
| 2023.0.x (Leyton) | 3.2 / 3.3 |
| 2024.0.x (Moorgate) | 3.4 |
| 2025.0.x (Northfields) | 3.5 |
| 2025.1.x (Oakwood) | 4.0 |
Always import the BOM:
```xml
<properties><spring-cloud.version>2025.0.0</spring-cloud.version></properties>
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```
Mismatched versions are the #1 cause of weird startup errors in course projects.

## 2. Ports used in the course-style project
| Service | Port(s) |
|---------|---------|
| limits-service | 8080, 8081 |
| spring-cloud-config-server | 8888 |
| currency-exchange-service | 8000, 8001, 8002 |
| currency-conversion-service | 8100, 8101 |
| naming-server (Eureka) | 8761 |
| api-gateway | 8765 |
| Zipkin | 9411 |

---

## 3. Spring Cloud Config Server

### 3.1 Why
Many services × many environments (dev/qa/stage/prod) → config duplicated in each jar. Changing a value requires rebuild/redeploy. Config Server serves configuration from a **Git repo** (versioned, auditable) over HTTP.

### 3.2 Server
Dependency: `spring-cloud-config-server`.
```java
@SpringBootApplication
@EnableConfigServer
public class SpringCloudConfigServerApplication {
    public static void main(String[] args) { SpringApplication.run(SpringCloudConfigServerApplication.class, args); }
}
```
```properties
spring.application.name=spring-cloud-config-server
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/acme/config-repo
# local repo: file:///Users/me/git-config-repo
spring.cloud.config.server.git.default-label=main
# spring.cloud.config.server.git.search-paths={application}
# private repo: username/password (token) or SSH key properties
```
Other backends: native filesystem (`spring.profiles.active=native`), Vault, JDBC, S3, AWS Secrets Manager/Parameter Store.

### 3.3 Repository layout (naming = `{application}-{profile}.properties|yml`)
```
config-repo/
├── application.yml                  shared by all services
├── limits-service.properties        limits-service, any profile
├── limits-service-dev.properties
├── limits-service-qa.properties
└── currency-exchange-service.yml
```
```properties
# limits-service-dev.properties
limits-service.minimum=1
limits-service.maximum=111
```
### 3.4 Endpoints
`GET http://localhost:8888/{application}/{profile}[/{label}]` → JSON with property sources in priority order.
e.g. `/limits-service/default`, `/limits-service/dev`, `/limits-service/qa/main`. Also `/{label}/{application}-{profile}.yml` returns raw files.

### 3.5 Client
Dependency: `spring-cloud-starter-config`.
```properties
spring.application.name=limits-service          # must match file names in the repo
spring.config.import=optional:configserver:http://localhost:8888
spring.profiles.active=dev
spring.cloud.config.profile=qa                  # override profile used for config only
spring.cloud.config.fail-fast=true              # fail startup if server unreachable (prod)
```
```java
@ConfigurationProperties("limits-service")
public record LimitsConfiguration(int minimum, int maximum) {}

@RestController
@RequiredArgsConstructor
public class LimitsController {
    private final LimitsConfiguration config;
    @GetMapping("/limits")
    public Limits retrieveLimits() { return new Limits(config.minimum(), config.maximum()); }
}
```
Priority: profile-specific remote > remote default > local `application.properties`.

### 3.6 Refreshing configuration without restart
- `@RefreshScope` beans (or `@ConfigurationProperties` beans which rebind) + `POST /actuator/refresh` (expose the endpoint) → re-fetches config.
- **Spring Cloud Bus** (`spring-cloud-starter-bus-amqp` or `-kafka`): `POST /actuator/busrefresh` on one instance broadcasts refresh to all; can be triggered by Git webhooks via `spring-cloud-config-monitor`.

### 3.7 Secrets
- Encrypt values in Git: `{cipher}AQB...` with `encrypt.key` (symmetric) or a keystore (asymmetric); `POST /encrypt`, `/decrypt` on the server.
- Better: HashiCorp Vault backend / cloud secret managers.
- Secure the Config Server itself (basic auth/OAuth2, network policy).

---

## 4. Currency Exchange & Conversion services (the running example)

### 4.1 currency-exchange-service
Dependencies: web, data-jpa, h2 (or Postgres), actuator, devtools, config client (optional), eureka client.
```java
@Entity
public class CurrencyExchange {
    @Id private Long id;
    @Column(name = "currency_from") private String from;
    @Column(name = "currency_to") private String to;
    private BigDecimal conversionMultiple;
    private String environment;          // which instance served it — shows load balancing
    // constructors, getters, setters
}

public interface CurrencyExchangeRepository extends JpaRepository<CurrencyExchange, Long> {
    Optional<CurrencyExchange> findByFromAndTo(String from, String to);
}

@RestController
@RequiredArgsConstructor
public class CurrencyExchangeController {
    private final CurrencyExchangeRepository repository;
    private final Environment environment;

    @GetMapping("/currency-exchange/from/{from}/to/{to}")
    public CurrencyExchange retrieveExchangeValue(@PathVariable String from, @PathVariable String to) {
        CurrencyExchange ce = repository.findByFromAndTo(from, to)
            .orElseThrow(() -> new RuntimeException("Unable to find data for " + from + " to " + to));
        ce.setEnvironment(environment.getProperty("local.server.port"));
        return ce;
    }
}
```
`data.sql`:
```sql
insert into currency_exchange (id, currency_from, currency_to, conversion_multiple, environment)
values (10001, 'USD', 'INR', 65, ''), (10002, 'EUR', 'INR', 75, ''), (10003, 'AUD', 'INR', 25, '');
```
```properties
spring.application.name=currency-exchange
server.port=8000
spring.jpa.show-sql=true
spring.datasource.url=jdbc:h2:mem:testdb
spring.h2.console.enabled=true
spring.jpa.defer-datasource-initialization=true
```
Run multiple instances: IntelliJ run config with `-Dserver.port=8001`, or `java -jar app.jar --server.port=8002`.

### 4.2 currency-conversion-service
```java
public record CurrencyConversion(Long id, String from, String to, BigDecimal quantity,
                                 BigDecimal conversionMultiple, BigDecimal totalCalculatedAmount, String environment) {}
```
**Version 1 — RestTemplate/RestClient (manual, hard-coded URL)**
```java
@GetMapping("/currency-conversion/from/{from}/to/{to}/quantity/{quantity}")
public CurrencyConversion convert(@PathVariable String from, @PathVariable String to, @PathVariable BigDecimal quantity) {
    CurrencyConversion c = restClient.get()
        .uri("http://localhost:8000/currency-exchange/from/{from}/to/{to}", from, to)
        .retrieve().body(CurrencyConversion.class);
    return new CurrencyConversion(c.id(), from, to, quantity, c.conversionMultiple(),
            quantity.multiply(c.conversionMultiple()), c.environment() + " rest client");
}
```
Problems: boilerplate, hard-coded host/port, no load balancing.

**Version 2 — OpenFeign**
Dependency: `spring-cloud-starter-openfeign`.
```java
@SpringBootApplication
@EnableFeignClients
public class CurrencyConversionServiceApplication {}

@FeignClient(name = "currency-exchange")      // logical service name → resolved via Eureka + LoadBalancer
// before Eureka: @FeignClient(name = "currency-exchange", url = "localhost:8000")
public interface CurrencyExchangeProxy {
    @GetMapping("/currency-exchange/from/{from}/to/{to}")
    CurrencyConversion retrieveExchangeValue(@PathVariable("from") String from, @PathVariable("to") String to);
}

@GetMapping("/currency-conversion-feign/from/{from}/to/{to}/quantity/{quantity}")
public CurrencyConversion convertFeign(@PathVariable String from, @PathVariable String to, @PathVariable BigDecimal quantity) {
    CurrencyConversion c = proxy.retrieveExchangeValue(from, to);
    return new CurrencyConversion(c.id(), from, to, quantity, c.conversionMultiple(),
            quantity.multiply(c.conversionMultiple()), c.environment() + " feign");
}
```
Feign configuration:
```yaml
spring.cloud.openfeign.client.config:
  default:            { connectTimeout: 2000, readTimeout: 5000, loggerLevel: basic }
  currency-exchange:  { readTimeout: 3000 }
spring.cloud.openfeign.circuitbreaker.enabled: true     # wrap Feign calls with CircuitBreaker
```
Extras: `@FeignClient(fallback = ...)` / `fallbackFactory`, `RequestInterceptor` (add auth headers), `ErrorDecoder` (map 404 → custom exception), `Logger.Level`.

**Modern alternative — HTTP interface with load balancing**
```java
@Bean @LoadBalanced RestClient.Builder lbRestClientBuilder() { return RestClient.builder(); }
@Bean CurrencyExchangeClient client(RestClient.Builder lbRestClientBuilder) {
    RestClient rc = lbRestClientBuilder.baseUrl("http://currency-exchange").build();
    return HttpServiceProxyFactory.builderFor(RestClientAdapter.create(rc)).build().createClient(CurrencyExchangeClient.class);
}
```

---

## 5. Service discovery — Eureka Naming Server

### 5.1 Concept
```
currency-exchange (8000) ─register/heartbeat─►┐
currency-exchange (8001) ─register/heartbeat─►├─► Eureka Server (registry)
currency-conversion      ─fetch registry──────┘        │
       │ LoadBalancer picks one instance ◄─────────────┘
       └──► http://192.168.1.5:8001/currency-exchange/...
```
### 5.2 Server
Dependency: `spring-cloud-starter-netflix-eureka-server`.
```java
@SpringBootApplication
@EnableEurekaServer
public class NamingServerApplication {}
```
```properties
spring.application.name=naming-server
server.port=8761
eureka.client.register-with-eureka=false     # don't register itself
eureka.client.fetch-registry=false
```
Dashboard: http://localhost:8761 — shows registered instances.

### 5.3 Clients
Dependency: `spring-cloud-starter-netflix-eureka-client` (auto-registers when on classpath).
```properties
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.prefer-ip-address=true
# eureka.instance.lease-renewal-interval-in-seconds=30 (heartbeat), lease-expiration-duration-in-seconds=90
```
Registry details: instances send heartbeats every 30s; removed after 90s without one. Clients cache the registry (refresh every 30s) → resilient if Eureka briefly goes down. **Self-preservation mode**: if many heartbeats are lost at once (network partition), Eureka stops evicting instances. Run multiple Eureka servers (peer replication) in production. Eureka is **AP** (availability over consistency).

### 5.4 Load balancing
Spring Cloud LoadBalancer (in eureka-client starter) — default **round-robin**; also random, or custom `ReactorServiceInstanceLoadBalancer`; zone preference, health-check based, same-instance preference (sticky). Feign & `@LoadBalanced` clients use it automatically. Hit `currency-conversion-feign` repeatedly and watch `environment` alternate between 8000/8001.

---

## 6. API Gateway — Spring Cloud Gateway

### 6.1 Responsibilities
Single entry point: routing, authentication/authorization (OAuth2 resource server / token relay), rate limiting, CORS, request/response header manipulation, path rewriting, retries & circuit breakers, logging/metrics/tracing, canary routing, request size limits.

Built on **Spring WebFlux + Netty** (non-blocking). Since 2023.0 there's also **Gateway Server MVC** (servlet-based). In 2025 trains the artifacts were renamed: `spring-cloud-starter-gateway-server-webflux` / `-server-webmvc`, with properties under `spring.cloud.gateway.server.webflux.*` (old names deprecated). Concepts below are the same.

### 6.2 Core concepts
- **Route**: id + destination URI + predicates + filters.
- **Predicate**: matches requests — `Path`, `Method`, `Header`, `Query`, `Host`, `Cookie`, `After/Before/Between` (time), `Weight` (canary), `RemoteAddr`.
- **Filter**: modifies request/response — `AddRequestHeader`, `RewritePath`, `StripPrefix`, `PrefixPath`, `SetStatus`, `RequestRateLimiter`, `CircuitBreaker`, `Retry`, `TokenRelay`, `RequestSize`, `DedupeResponseHeader`, custom filters.

### 6.3 Discovery-based automatic routing
```properties
spring.application.name=api-gateway
server.port=8765
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
spring.cloud.gateway.discovery.locator.enabled=true
spring.cloud.gateway.discovery.locator.lower-case-service-id=true
```
→ `http://localhost:8765/currency-exchange/currency-exchange/from/USD/to/INR` (service id as first path segment).

### 6.4 Custom routes (Java DSL)
```java
@Configuration
public class ApiGatewayConfiguration {
    @Bean
    public RouteLocator gatewayRouter(RouteLocatorBuilder builder) {
        return builder.routes()
            .route(p -> p.path("/get")
                .filters(f -> f.addRequestHeader("MyHeader", "MyURI").addRequestParameter("Param", "MyValue"))
                .uri("http://httpbin.org:80"))
            .route(p -> p.path("/currency-exchange/**")
                .uri("lb://currency-exchange"))                     // lb:// = load-balanced via discovery
            .route(p -> p.path("/currency-conversion/**")
                .uri("lb://currency-conversion"))
            .route(p -> p.path("/currency-conversion-feign/**")
                .uri("lb://currency-conversion"))
            .route(p -> p.path("/currency-conversion-new/**")
                .filters(f -> f.rewritePath("/currency-conversion-new/(?<segment>.*)",
                                            "/currency-conversion-feign/${segment}"))
                .uri("lb://currency-conversion"))
            .build();
    }
}
```
### 6.5 YAML routes
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args: { name: ordersCB, fallbackUri: forward:/fallback/orders }
            - name: Retry
              args: { retries: 3, statuses: BAD_GATEWAY, methods: GET, backoff: { firstBackoff: 50ms, maxBackoff: 500ms } }
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10      # tokens per second
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
        - id: canary
          uri: lb://order-service-v2
          predicates: [ Path=/api/orders/**, Weight=orders, 10 ]   # 10% traffic
```
```java
@Bean KeyResolver userKeyResolver() {
    return exchange -> Mono.justOrEmpty(exchange.getRequest().getHeaders().getFirst("X-User-Id")).defaultIfEmpty("anonymous");
}
```
### 6.6 Global filter
```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("Path of the request received -> {}", exchange.getRequest().getPath());
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> log.info("Response status -> {}", exchange.getResponse().getStatusCode())));
    }
    @Override public int getOrder() { return -1; }
}
```
### 6.7 Security at the gateway
```java
@Bean
SecurityWebFilterChain gatewaySecurity(ServerHttpSecurity http) {
    return http.authorizeExchange(ex -> ex.pathMatchers("/actuator/health").permitAll().anyExchange().authenticated())
               .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
               .csrf(ServerHttpSecurity.CsrfSpec::disable)
               .build();
}
```
For browser apps: gateway as OAuth2 **client** (`oauth2Login`) + `TokenRelay` filter (BFF pattern — tokens never reach the browser).

---

## 7. Resilience4j

### 7.1 Setup
Dependencies: `spring-boot-starter-aop` + `resilience4j-spring-boot3` (annotations), or `spring-cloud-starter-circuitbreaker-resilience4j` (Spring Cloud abstraction), + `spring-boot-starter-actuator` for health/metrics.

### 7.2 Retry
```java
@RestController
public class CircuitBreakerController {
    private final Logger logger = LoggerFactory.getLogger(CircuitBreakerController.class);

    @GetMapping("/sample-api")
    @Retry(name = "sample-api", fallbackMethod = "hardcodedResponse")
    public String sampleApi() {
        logger.info("Sample api call received");
        return restClient.get().uri("http://localhost:8080/some-dummy-url").retrieve().body(String.class);
    }

    public String hardcodedResponse(Exception ex) { return "fallback-response"; }   // same return type + exception param
}
```
```properties
resilience4j.retry.instances.sample-api.max-attempts=5
resilience4j.retry.instances.sample-api.wait-duration=1s
resilience4j.retry.instances.sample-api.enable-exponential-backoff=true
resilience4j.retry.instances.sample-api.retry-exceptions=java.io.IOException,org.springframework.web.client.ResourceAccessException
```

### 7.3 Circuit Breaker
```java
@GetMapping("/sample-api")
@CircuitBreaker(name = "default", fallbackMethod = "hardcodedResponse")
public String sampleApi() { ... }
```
```yaml
resilience4j.circuitbreaker:
  configs:
    default:
      sliding-window-type: COUNT_BASED          # or TIME_BASED
      sliding-window-size: 10
      minimum-number-of-calls: 5
      failure-rate-threshold: 50                # % failures to OPEN
      slow-call-rate-threshold: 80
      slow-call-duration-threshold: 2s
      wait-duration-in-open-state: 10s          # then HALF_OPEN
      permitted-number-of-calls-in-half-open-state: 3
      automatic-transition-from-open-to-half-open-enabled: true
      record-exceptions: [java.io.IOException, java.util.concurrent.TimeoutException]
      ignore-exceptions: [com.shop.BusinessException]
  instances:
    default: { base-config: default }
management.health.circuitbreakers.enabled: true
```
States:
```
CLOSED ──failure rate ≥ threshold──► OPEN ──wait-duration──► HALF_OPEN
  ▲                                                             │
  └───────────── trial calls succeed ──────────────────────────┤
                                       trial calls fail ───────► OPEN
(also DISABLED, FORCED_OPEN, METRICS_ONLY for ops control)
```
Test it: `watch -n 0.1 curl http://localhost:8000/sample-api` → after failures, calls stop reaching the method and the fallback returns instantly.

### 7.4 Rate limiter & bulkhead & time limiter
```java
@RateLimiter(name = "default")      // e.g., 2 calls per 10s
@Bulkhead(name = "sample-api")      // max concurrent calls
@TimeLimiter(name = "slow")         // for CompletableFuture-returning methods
```
```properties
resilience4j.ratelimiter.instances.default.limit-for-period=2
resilience4j.ratelimiter.instances.default.limit-refresh-period=10s
resilience4j.ratelimiter.instances.default.timeout-duration=0
resilience4j.bulkhead.instances.sample-api.max-concurrent-calls=10
resilience4j.bulkhead.instances.sample-api.max-wait-duration=0
resilience4j.thread-pool-bulkhead.instances.reports.max-thread-pool-size=4
resilience4j.timelimiter.instances.slow.timeout-duration=2s
```
**Order of decorators** (outermost → innermost by default): Retry → CircuitBreaker → RateLimiter → TimeLimiter → Bulkhead → method. So each retry attempt goes through the circuit breaker.

Programmatic use:
```java
Supplier<String> decorated = Decorators.ofSupplier(() -> client.call())
    .withCircuitBreaker(cbRegistry.circuitBreaker("cb"))
    .withRetry(retryRegistry.retry("r"))
    .withFallback(List.of(CallNotPermittedException.class), e -> "fallback")
    .decorate();
```
Monitoring: `/actuator/circuitbreakers`, `/actuator/circuitbreakerevents`, Micrometer metrics `resilience4j_circuitbreaker_state`.

---

## 8. Distributed tracing (Boot 3)
Each request gets a **trace id**; each hop/operation is a **span** (with parent span id). Propagated through HTTP headers (W3C `traceparent` by default, or B3) and messaging headers.

Dependencies (each service):
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId></dependency>
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-observation</artifactId></dependency>
<!-- choose ONE bridge -->
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-zipkin</artifactId></dependency>
<!-- or: micrometer-tracing-bridge-brave + zipkin-reporter-brave -->
<!-- Feign: --><dependency><groupId>io.github.openfeign</groupId><artifactId>feign-micrometer</artifactId></dependency>
```
```properties
management.tracing.sampling.probability=1.0           # 100% in dev, e.g. 0.1 in prod
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
```
Run Zipkin: `docker run -p 9411:9411 openzipkin/zipkin:latest` → UI http://localhost:9411 → search by service or trace id, see the timeline across gateway → conversion → exchange.

Gotchas: create `RestTemplate`/`RestClient`/`WebClient` from the auto-configured **builder** (instrumented); `new RestTemplate()` won't propagate trace headers. For Kafka: enable observation (`spring.kafka.template.observation-enabled=true`, `spring.kafka.listener.observation-enabled=true`). Custom spans: `@Observed(name = "order.place")` or `Observation.createNotStarted(...)`.

Modern stack: OTLP exporter → OpenTelemetry Collector → Grafana Tempo/Jaeger; logs to Loki with traceId; metrics to Prometheus; Grafana links all three.

---

## 9. Putting it together with Docker Compose
```yaml
services:
  naming-server:
    image: acme/mmv3-naming-server:0.0.1-SNAPSHOT
    mem_limit: 700m
    ports: ["8761:8761"]
    networks: [currency-network]

  currency-exchange:
    image: acme/mmv3-currency-exchange-service:0.0.1-SNAPSHOT
    mem_limit: 700m
    ports: ["8000:8000"]
    networks: [currency-network]
    depends_on: [naming-server]
    environment:
      EUREKA.CLIENT.SERVICEURL.DEFAULTZONE: http://naming-server:8761/eureka
      MANAGEMENT.ZIPKIN.TRACING.ENDPOINT: http://zipkin-server:9411/api/v2/spans

  currency-conversion:
    image: acme/mmv3-currency-conversion-service:0.0.1-SNAPSHOT
    mem_limit: 700m
    ports: ["8100:8100"]
    networks: [currency-network]
    depends_on: [naming-server]
    environment:
      EUREKA.CLIENT.SERVICEURL.DEFAULTZONE: http://naming-server:8761/eureka
      MANAGEMENT.ZIPKIN.TRACING.ENDPOINT: http://zipkin-server:9411/api/v2/spans

  api-gateway:
    image: acme/mmv3-api-gateway:0.0.1-SNAPSHOT
    mem_limit: 700m
    ports: ["8765:8765"]
    networks: [currency-network]
    depends_on: [naming-server]
    environment:
      EUREKA.CLIENT.SERVICEURL.DEFAULTZONE: http://naming-server:8761/eureka
      MANAGEMENT.ZIPKIN.TRACING.ENDPOINT: http://zipkin-server:9411/api/v2/spans

  zipkin-server:
    image: openzipkin/zipkin:latest
    mem_limit: 300m
    ports: ["9411:9411"]
    networks: [currency-network]
    restart: always

networks:
  currency-network:
```
Build images: `mvn spring-boot:build-image -DskipTests` with `<image><name>acme/mmv3-${project.artifactId}:${project.version}</name></image>`.

## 10. Spring Cloud on Kubernetes — what changes
| Concern | Spring Cloud (VMs/Compose) | Kubernetes-native |
|---------|---------------------------|-------------------|
| Discovery | Eureka | K8s Service DNS (`http://currency-exchange:8000`) |
| Load balancing | Spring Cloud LoadBalancer | kube-proxy / Service |
| Config | Config Server | ConfigMaps / Secrets (env vars or `configtree:` mounts), Spring Cloud Kubernetes reload |
| Gateway | Spring Cloud Gateway | Ingress / Gateway API (or keep SCG for app-level logic) |
| Resilience | Resilience4j | Resilience4j and/or service mesh (Istio retries, timeouts, outlier detection) |
| Health | Actuator | Actuator liveness/readiness probes |

Feign without Eureka on K8s:
```java
@FeignClient(name = "currency-exchange", url = "${CURRENCY_EXCHANGE_SERVICE_HOST:http://localhost}:8000")
```
(K8s injects `CURRENCY_EXCHANGE_SERVICE_HOST` env var for Services in the same namespace; or use DNS name directly.)

## 11. Interview Q&A
**Q. What is Spring Cloud Config and how does a client get properties?** A server serving versioned config from Git; clients import `configserver:` URL and fetch `{application}/{profile}` at startup; refresh via actuator or Bus.

**Q. How does Eureka work?** Services register and heartbeat; clients fetch & cache the registry; LoadBalancer picks an instance; stale instances evicted after missed heartbeats; self-preservation during partitions.

**Q. Client-side vs server-side load balancing?** Client picks instance from registry (Spring Cloud LoadBalancer) vs a separate load balancer/proxy (K8s Service, NGINX, ALB) routes requests.

**Q. What does Feign do?** Generates an HTTP client implementation from an annotated interface; integrates with discovery, load balancing, circuit breakers.

**Q. Why an API gateway and how does Spring Cloud Gateway route?** Single entry point with cross-cutting concerns; routes = predicates + filters + destination URI (`lb://service-id` for discovery).

**Q. Explain circuit breaker states and configuration.** CLOSED/OPEN/HALF_OPEN with failure-rate threshold, sliding window, wait duration, half-open trial calls.

**Q. Retry vs circuit breaker — how do they interact?** Retry handles transient errors with repeated attempts; CB stops calls when failures persist. Retry wraps CB by default so repeated failures quickly open the circuit.

**Q. What replaced Sleuth in Boot 3?** Micrometer Tracing (Observation API) with Brave or OpenTelemetry bridges, exporting to Zipkin/OTLP.

**Q. Do you need Eureka and Config Server on Kubernetes?** Usually not — K8s provides DNS-based discovery, load balancing and ConfigMaps/Secrets.
