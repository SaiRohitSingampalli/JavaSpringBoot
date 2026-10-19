# 22 — Reactive Programming & Spring WebFlux

## 1. Why reactive?
Traditional **thread-per-request** (Spring MVC on Tomcat): each request holds a thread while it waits for DB/HTTP I/O. 200 threads → ~200 concurrent in-flight requests; more threads = more memory and context switching.

**Reactive / non-blocking**: a few event-loop threads (≈ CPU cores) handle many connections; when I/O is pending, the thread is released and resumes via callbacks when data arrives. Gains: high concurrency with few resources, built-in **backpressure**, streaming.

**Since Java 21 virtual threads** give similar scalability for blocking code with a simpler model — so choose reactive mainly for streaming, backpressure-heavy pipelines, gateways (Spring Cloud Gateway is reactive), or teams already invested in it.

## 2. Reactive Streams specification
Four interfaces (also in `java.util.concurrent.Flow`):
```java
Publisher<T>    { void subscribe(Subscriber<? super T> s); }
Subscriber<T>   { onSubscribe(Subscription s); onNext(T t); onError(Throwable t); onComplete(); }
Subscription    { request(long n); cancel(); }       // n = backpressure demand
Processor<T,R>  extends Subscriber<T>, Publisher<R>
```
Signals: `onSubscribe → onNext* → (onError | onComplete)`.
**Backpressure**: subscriber requests how many items it can handle, so a fast producer doesn't overwhelm a slow consumer.

**Nothing happens until you subscribe** (cold publishers are lazy).

## 3. Project Reactor
Spring's reactive library.
- **`Mono<T>`** — 0 or 1 element (like an async Optional / CompletableFuture).
- **`Flux<T>`** — 0..N elements (async stream).

### 3.1 Creating
```java
Mono.just("x"); Mono.empty(); Mono.error(new NotFoundException());
Mono.fromCallable(() -> blockingCall());          // lazy
Mono.defer(() -> Mono.just(Instant.now()));       // lazy re-evaluation per subscriber
Flux.just(1, 2, 3); Flux.fromIterable(list); Flux.range(1, 10);
Flux.interval(Duration.ofSeconds(1));             // ticks
Flux.generate(...) / Flux.create(sink -> ...)     // programmatic
```

### 3.2 Operators (most used)
| Category | Operators |
|----------|-----------|
| Transform | `map` (sync 1→1), `flatMap` (async 1→Publisher, interleaved), `concatMap` (ordered), `flatMapSequential`, `switchMap`, `cast` |
| Filter | `filter`, `take(n)`, `skip`, `distinct`, `takeWhile`, `next()` |
| Combine | `zip`/`zipWith` (pair up), `merge` (interleave), `concat` (sequential), `then()` (ignore result, continue), `thenMany` |
| Aggregate | `collectList`, `collectMap`, `reduce`, `count` |
| Errors | `onErrorReturn`, `onErrorResume`, `onErrorMap`, `retry(n)`, `retryWhen(Retry.backoff(3, Duration.ofMillis(100)))`, `timeout` |
| Side effects | `doOnNext`, `doOnError`, `doOnSubscribe`, `doFinally`, `log()` |
| Empty handling | `switchIfEmpty`, `defaultIfEmpty` |
| Time | `delayElements`, `buffer`, `window`, `sample` |
| Backpressure | `limitRate`, `onBackpressureBuffer`, `onBackpressureDrop`, `onBackpressureLatest` |
| Caching | `cache()`, `share()` (hot) |

```java
Mono<OrderDetails> details(Long id) {
    return orderRepo.findById(id)                                         // Mono<Order>
        .switchIfEmpty(Mono.error(new OrderNotFoundException(id)))
        .flatMap(order -> Mono.zip(
                customerClient.get(order.customerId()),                 // parallel calls
                paymentClient.status(order.id()))
            .map(t -> new OrderDetails(order, t.getT1(), t.getT2())))
        .timeout(Duration.ofSeconds(2))
        .retryWhen(Retry.backoff(2, Duration.ofMillis(200)).filter(this::isTransient))
        .onErrorMap(TimeoutException.class, e -> new ServiceUnavailableException("timeout"));
}
```
**map vs flatMap**: `map` for synchronous transforms; `flatMap` when the function returns a `Mono/Flux` (async call). Using `map` with an async call gives `Mono<Mono<T>>`.

### 3.3 Schedulers (threading)
Reactor is concurrency-agnostic: operators run on whatever thread emits.
- `Schedulers.parallel()` — CPU-bound work (cores).
- `Schedulers.boundedElastic()` — **wrapping blocking calls**.
- `publishOn(scheduler)` — switches thread for **downstream** operators.
- `subscribeOn(scheduler)` — chooses thread for the **subscription/source**.
```java
Mono.fromCallable(() -> legacyJdbcDao.find(id)).subscribeOn(Schedulers.boundedElastic());
```
**Golden rule: never block an event-loop thread** (`block()`, `Thread.sleep`, JDBC, blocking HTTP). Detect with **BlockHound**.

### 3.4 Hot vs cold
- **Cold**: each subscriber triggers its own execution from the start (HTTP call, DB query).
- **Hot**: emits regardless of subscribers; late subscribers miss earlier data (`Sinks.many().multicast()`, `share()`, UI events, price ticks).

### 3.5 Context
ThreadLocal doesn't work (threads hop). Reactor `Context` carries values (security context, trace ids) along the chain: `contextWrite(Context.of("tenant", t))`, `Mono.deferContextual(ctx -> ...)`. Micrometer context-propagation bridges MDC/tracing automatically (`Hooks.enableAutomaticContextPropagation()`).

### 3.6 Testing
```java
StepVerifier.create(service.details(1L))
    .assertNext(d -> assertThat(d.customer().name()).isEqualTo("Ravi"))
    .verifyComplete();
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(1)).take(2))
    .thenAwait(Duration.ofHours(2)).expectNextCount(2).verifyComplete();
```

## 4. Spring WebFlux
`spring-boot-starter-webflux` → Netty server by default (also Tomcat/Jetty/Undertow via Servlet 3.1 non-blocking I/O).

### 4.1 Annotated controllers (familiar style)
```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {
    private final ProductRepository repo;          // ReactiveCrudRepository (R2DBC/Mongo)

    @GetMapping
    public Flux<Product> all() { return repo.findAll(); }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<Product>> one(@PathVariable Long id) {
        return repo.findById(id).map(ResponseEntity::ok).defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<Product> create(@Valid @RequestBody Mono<Product> body) { return body.flatMap(repo::save); }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<PriceTick> stream() { return priceService.ticks(); }      // Server-Sent Events
}
```

### 4.2 Functional endpoints
```java
@Bean
RouterFunction<ServerResponse> routes(ProductHandler h) {
    return RouterFunctions.route()
        .GET("/api/products", h::all)
        .GET("/api/products/{id}", h::one)
        .POST("/api/products", h::create)
        .build();
}
@Component class ProductHandler {
    Mono<ServerResponse> one(ServerRequest req) {
        return repo.findById(Long.valueOf(req.pathVariable("id")))
            .flatMap(p -> ServerResponse.ok().bodyValue(p))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
}
```

### 4.3 WebClient
```java
WebClient client = WebClient.builder().baseUrl("http://inventory").build();
Mono<Stock> stock = client.get().uri("/stock/{sku}", sku)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, r -> Mono.error(new SkuNotFoundException(sku)))
    .bodyToMono(Stock.class)
    .timeout(Duration.ofSeconds(2));
```
WebClient can also be used in MVC apps (call `.block()` only outside event loops — in MVC that's acceptable on request threads, though RestClient is simpler there).

### 4.4 Reactive data access
- **R2DBC** (Postgres, MySQL, SQL Server, H2) + Spring Data R2DBC — no lazy loading/relationships like JPA.
- Reactive MongoDB, Cassandra, Redis (Lettuce), Elasticsearch.
- **JPA/JDBC are blocking** — don't use them on event loops (wrap with boundedElastic if unavoidable).
- Transactions: `@Transactional` with `ReactiveTransactionManager` / `TransactionalOperator`.

### 4.5 Security, errors, testing
- `@EnableWebFluxSecurity`, `SecurityWebFilterChain` with `ServerHttpSecurity`; `ReactiveSecurityContextHolder`.
- Errors: `@ControllerAdvice` works; also `WebExceptionHandler`.
- `@WebFluxTest` + `WebTestClient`:
```java
webTestClient.get().uri("/api/products/1").exchange()
    .expectStatus().isOk().expectBody(Product.class).value(p -> assertThat(p.name()).isEqualTo("Pen"));
```

## 5. MVC vs WebFlux vs MVC + virtual threads
| | Spring MVC (platform threads) | Spring MVC + virtual threads | Spring WebFlux |
|--|--|--|--|
| Programming model | Imperative, blocking | Imperative, blocking | Reactive, non-blocking |
| Concurrency | Limited by thread pool | Very high for I/O | Very high |
| Learning curve | Low | Low | **High** (debugging, stack traces, operators) |
| Libraries | JDBC/JPA, everything | Everything | Need reactive drivers (R2DBC, WebClient) |
| Streaming/backpressure | Limited | Limited | Excellent |
| Best for | Most CRUD services | Most I/O-heavy services today | Gateways, streaming, high fan-out aggregation, SSE |

Don't mix: one blocking call in a reactive chain can stall an event loop and throttle the whole service.

## 6. Interview Q&A
**Q. Mono vs Flux?** Mono emits 0–1 items; Flux emits 0–N.

**Q. What is backpressure?** Consumer-controlled flow: subscriber requests n items so producers don't overwhelm it.

**Q. map vs flatMap in Reactor?** map applies a synchronous function; flatMap subscribes to the inner publisher returned by the function (async), merging results.

**Q. publishOn vs subscribeOn?** publishOn changes the thread for operators after it; subscribeOn affects where the source/subscription runs (position-independent).

**Q. What happens if you block in WebFlux?** The event-loop thread is stuck, reducing throughput dramatically; offload to boundedElastic or use non-blocking drivers.

**Q. Is WebFlux faster than MVC?** Not per request; it scales better under high concurrency with I/O waits using fewer threads. Virtual threads now give MVC similar scalability for many workloads.

**Q. Cold vs hot publisher?** Cold restarts per subscriber; hot emits independent of subscribers (shared stream).
