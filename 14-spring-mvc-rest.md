# 14 — Spring MVC & Building REST APIs

## 1. REST fundamentals (what interviewers expect)
**REST** (Representational State Transfer) — architectural style by Roy Fielding:
1. **Client–server** separation.
2. **Stateless** — each request contains all info (auth token); no server session.
3. **Cacheable** — responses declare cacheability.
4. **Uniform interface** — resources identified by URIs, manipulated via representations (JSON), self-descriptive messages, HATEOAS.
5. **Layered system** — gateways, proxies, load balancers transparent.
6. Code on demand (optional).

**Richardson Maturity Model**: L0 single endpoint (RPC/SOAP) → L1 resources → L2 HTTP verbs + status codes (most "REST" APIs) → L3 hypermedia (HATEOAS).

### HTTP methods
| Method | Purpose | Safe | Idempotent | Body |
|--------|---------|:-:|:-:|:-:|
| GET | Read | ✔ | ✔ | No |
| POST | Create / non-idempotent action | ✘ | ✘ | Yes |
| PUT | Replace whole resource (or create at known URI) | ✘ | ✔ | Yes |
| PATCH | Partial update | ✘ | ✘ (can be) | Yes |
| DELETE | Remove | ✘ | ✔ | Usually no |
| HEAD | Headers only | ✔ | ✔ | No |
| OPTIONS | Allowed methods / CORS preflight | ✔ | ✔ | No |

**Idempotent** = same request repeated N times has the same effect as once (important for retries). Make POST idempotent with an `Idempotency-Key` header stored server-side.

### Status codes you must know
| Code | Meaning | When |
|------|---------|------|
| 200 OK | Success with body | GET, PUT, PATCH |
| 201 Created | Resource created + `Location` header | POST |
| 202 Accepted | Accepted for async processing | Long jobs |
| 204 No Content | Success, no body | DELETE, PUT |
| 301/302/307/308 | Redirects | |
| 304 Not Modified | Conditional GET cache hit (ETag) | |
| 400 Bad Request | Malformed / validation failure | |
| 401 Unauthorized | Not authenticated (missing/invalid token) | |
| 403 Forbidden | Authenticated but not allowed | |
| 404 Not Found | Resource doesn't exist | |
| 405 Method Not Allowed | Wrong verb | |
| 409 Conflict | State conflict (duplicate email, version conflict) | |
| 412 Precondition Failed | If-Match ETag mismatch | |
| 415 Unsupported Media Type | Wrong Content-Type | |
| 422 Unprocessable Content | Semantically invalid | |
| 429 Too Many Requests | Rate limited (+ `Retry-After`) | |
| 500 Internal Server Error | Unhandled error | |
| 502 / 503 / 504 | Bad gateway / unavailable / gateway timeout | Downstream issues |

### URI design best practices
```
GET    /api/v1/customers                    list (with ?page=0&size=20&sort=name,asc&status=ACTIVE)
GET    /api/v1/customers/42                 one
POST   /api/v1/customers                    create
PUT    /api/v1/customers/42                 replace
PATCH  /api/v1/customers/42                 partial update
DELETE /api/v1/customers/42                 delete
GET    /api/v1/customers/42/orders          sub-resource
POST   /api/v1/orders/17/cancel             action that isn't CRUD (verb sub-resource is acceptable)
```
Plural nouns, lowercase, hyphens (`/order-items`), no verbs for CRUD, no trailing slash, version in path or header, filtering/sorting/pagination via query params.

## 2. Spring MVC architecture

```
HTTP request
   │
   ▼
Servlet container (Tomcat) → Filter chain (security, CORS, logging...)
   │
   ▼
DispatcherServlet  (Front Controller)
   │ 1. HandlerMapping (RequestMappingHandlerMapping) → finds controller method + interceptors
   │ 2. HandlerInterceptor.preHandle()
   │ 3. HandlerAdapter (RequestMappingHandlerAdapter)
   │       ├─ HandlerMethodArgumentResolvers  (@PathVariable, @RequestBody → HttpMessageConverter → Jackson)
   │       ├─ invoke controller method
   │       └─ HandlerMethodReturnValueHandlers (@ResponseBody → HttpMessageConverter → JSON)
   │ 4. HandlerInterceptor.postHandle()
   │ 5. (MVC views only) ViewResolver → View render
   │ 6. On exception → HandlerExceptionResolver (@ExceptionHandler / @ControllerAdvice)
   │ 7. HandlerInterceptor.afterCompletion()
   ▼
HTTP response
```
Spring Boot auto-configures DispatcherServlet mapped to `/`, Jackson `ObjectMapper`, message converters, static resources, error handling (`/error` → `BasicErrorController`).

**Content negotiation**: `Accept` header decides output format (JSON by default; XML if `jackson-dataformat-xml` present). `Content-Type` header decides how to read the request body.

## 3. A complete, production-style REST controller

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Validated                                   // enables validation of @PathVariable/@RequestParam constraints
public class ProductController {

    private final ProductService service;

    // GET /api/v1/products?category=books&page=0&size=20&sort=price,desc
    @GetMapping
    public Page<ProductResponse> list(@RequestParam(required = false) String category,
                                      @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        return service.search(category, pageable);
    }

    @GetMapping("/{id}")
    public ProductResponse get(@PathVariable @Positive Long id) {
        return service.get(id);
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody CreateProductRequest req,
                                                  UriComponentsBuilder uriBuilder) {
        ProductResponse created = service.create(req);
        URI location = uriBuilder.path("/api/v1/products/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ProductResponse replace(@PathVariable Long id, @Valid @RequestBody CreateProductRequest req) {
        return service.replace(id, req);
    }

    @PatchMapping("/{id}")
    public ProductResponse patch(@PathVariable Long id, @RequestBody Map<String, Object> changes) {
        return service.patch(id, changes);       // or JSON Merge Patch / dedicated UpdateRequest DTO with nullable fields
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        service.delete(id);
    }

    @PostMapping(path = "/{id}/image", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public void upload(@PathVariable Long id, @RequestParam("file") MultipartFile file) throws IOException {
        service.storeImage(id, file.getOriginalFilename(), file.getBytes());
    }

    @GetMapping(path = "/export", produces = "text/csv")
    public ResponseEntity<Resource> export() {
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=products.csv")
            .body(new ByteArrayResource(service.exportCsv()));
    }
}
```

### DTOs
```java
public record CreateProductRequest(
        @NotBlank @Size(max = 120) String name,
        @NotNull @DecimalMin("0.01") BigDecimal price,
        @NotBlank String category,
        @Size(max = 2000) String description) {}

public record ProductResponse(Long id, String name, BigDecimal price, String category, Instant createdAt) {}
```
**Why DTOs, not entities, in APIs?** Prevent over-posting (client setting `id`/`role`), avoid lazy-loading exceptions & infinite recursion in JSON, decouple API contract from DB schema, hide internal fields, different shapes for create/update/read.

**Mapping**: manual static methods, or **MapStruct** (compile-time generated, fast):
```java
@Mapper(componentModel = "spring")
public interface ProductMapper {
    ProductResponse toResponse(Product p);
    Product toEntity(CreateProductRequest r);
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void update(@MappingTarget Product p, UpdateProductRequest r);
}
```

## 4. Request mapping annotations in detail
| Annotation | Binds |
|-----------|-------|
| `@RequestMapping(path, method, params, headers, consumes, produces)` | General mapping |
| `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` | Shortcuts |
| `@PathVariable("id")` | URI template var `/users/{id}`; regex `{id:\\d+}` |
| `@RequestParam(name, required, defaultValue)` | Query/form params; `List<String> tags` for `?tags=a&tags=b` |
| `@RequestBody` | Body deserialized by HttpMessageConverter |
| `@RequestHeader("X-Request-Id")` | Header; `@RequestHeader HttpHeaders all` |
| `@CookieValue("SESSION")` | Cookie |
| `@MatrixVariable` | `/cars;color=red` (rare) |
| `@ModelAttribute` | Bind query/form params to an object |
| `@RequestPart` | Multipart part (JSON + file) |
| `@SessionAttribute`, `@RequestAttribute` | Session/request attributes |
| `HttpServletRequest/Response`, `Principal`, `@AuthenticationPrincipal`, `Locale`, `Pageable`, `Sort` | Injected automatically |

Binding query params into an object (great for search filters):
```java
public record ProductFilter(String category, BigDecimal minPrice, BigDecimal maxPrice, Boolean inStock) {}
@GetMapping("/search")
public List<ProductResponse> search(@ModelAttribute ProductFilter filter) { ... }   // @ModelAttribute optional
```

### Return types
| Return | Result |
|--------|--------|
| POJO / record / collection | Serialized to JSON, 200 |
| `ResponseEntity<T>` | Full control of status, headers, body |
| `void` + `@ResponseStatus` | Status only |
| `ProblemDetail` | RFC 9457 error |
| `Optional<T>` | Not special in MVC — map to 404 yourself |
| `Resource`, `byte[]`, `StreamingResponseBody` | Files/streams |
| `CompletableFuture<T>`, `DeferredResult<T>`, `Callable<T>` | Async — frees the servlet thread |
| `SseEmitter`, `ResponseBodyEmitter` | Server-sent events/streaming |

```java
return ResponseEntity.status(HttpStatus.ACCEPTED).header("X-Job-Id", jobId).build();
return ResponseEntity.ok().eTag(version).cacheControl(CacheControl.maxAge(Duration.ofMinutes(5))).body(dto);
return ResponseEntity.notFound().build();
```

## 5. Jackson (JSON) essentials
```java
public record UserResponse(
    Long id,
    @JsonProperty("full_name") String name,          // rename
    @JsonIgnore String passwordHash,                  // never serialize
    @JsonFormat(shape = STRING, pattern = "yyyy-MM-dd") LocalDate dob,
    @JsonInclude(JsonInclude.Include.NON_NULL) String nickname) {}
```
```yaml
spring:
  jackson:
    default-property-inclusion: non_null
    serialization.write-dates-as-timestamps: false   # ISO-8601 dates
    deserialization.fail-on-unknown-properties: false   # Boot default already false
    property-naming-strategy: SNAKE_CASE
```
Customize with `Jackson2ObjectMapperBuilderCustomizer` bean (Boot 4 / Jackson 3: `JsonMapperBuilderCustomizer`).
Other annotations: `@JsonCreator`, `@JsonValue`, `@JsonAlias`, `@JsonIgnoreProperties(ignoreUnknown=true)`, `@JsonView`, `@JsonManagedReference/@JsonBackReference` (bidirectional entities — but use DTOs instead), `@JsonTypeInfo/@JsonSubTypes` (polymorphism), `@JsonUnwrapped`.

Dynamic filtering (Ranga's course example):
```java
@JsonFilter("SomeBeanFilter") public class SomeBean { String f1, f2, f3; }
MappingJacksonValue value = new MappingJacksonValue(bean);
value.setFilters(new SimpleFilterProvider().addFilter("SomeBeanFilter",
        SimpleBeanPropertyFilter.filterOutAllExcept("f1", "f2")));
return value;
```

## 6. Pagination & sorting
```java
@GetMapping
public Page<ProductResponse> list(Pageable pageable) { return repo.findAll(pageable).map(mapper::toResponse); }
// GET /products?page=2&size=10&sort=price,desc&sort=name
```
For stable JSON, return a custom page DTO or enable `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)` (Spring Data 3.3+) — serializing `PageImpl` directly is discouraged.
```java
public record PageResponse<T>(List<T> content, int page, int size, long totalElements, int totalPages) {
    public static <T> PageResponse<T> of(Page<T> p) {
        return new PageResponse<>(p.getContent(), p.getNumber(), p.getSize(), p.getTotalElements(), p.getTotalPages());
    }
}
```
**Offset vs keyset (cursor) pagination**: offset (`LIMIT 20 OFFSET 100000`) gets slow on deep pages and is unstable with inserts; keyset (`WHERE id > :lastId ORDER BY id LIMIT 20`) is fast and stable → use for infinite scroll/large tables (Spring Data `ScrollPosition`/`Window`).

## 7. API versioning strategies
| Strategy | Example | Pros/cons |
|---------|---------|-----------|
| URI path | `/api/v1/users` | Simple, cacheable, visible; "pollutes" URI |
| Query param | `/users?version=1` | Easy; less clean |
| Custom header | `X-API-VERSION: 1` | Clean URIs; harder to test in browser |
| Media type | `Accept: application/vnd.shop.v1+json` | Most RESTful; complex |

```java
@GetMapping(path = "/person", headers = "X-API-VERSION=2")
@GetMapping(path = "/person", produces = "application/vnd.company.app-v2+json")
@GetMapping(path = "/person", params = "version=2")
```
Spring Framework 7 / Boot 4 adds first-class support: `@GetMapping(path = "/person", version = "2")` with configurable resolver (header, path, param, media type).

## 8. HATEOAS
```java
@GetMapping("/{id}")
public EntityModel<UserResponse> get(@PathVariable Long id) {
    EntityModel<UserResponse> model = EntityModel.of(service.get(id));
    model.add(linkTo(methodOn(UserController.class).get(id)).withSelfRel());
    model.add(linkTo(methodOn(UserController.class).list(null)).withRel("all-users"));
    return model;
}
```
`spring-boot-starter-hateoas` → HAL format (`_links`). Rarely used in internal microservice APIs; good to know.

## 9. Interceptors & Filters
| | Servlet `Filter` | Spring `HandlerInterceptor` |
|--|--|--|
| Level | Servlet container, before DispatcherServlet | Inside Spring MVC, around handler |
| Knows the handler method? | No | Yes |
| Hooks | `doFilter` | `preHandle`, `postHandle`, `afterCompletion` |
| Use | Security, CORS, request logging, compression, correlation ids | Auth checks per handler, locale, timing, audit |

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String id = Optional.ofNullable(req.getHeader("X-Correlation-Id")).orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", id);
        res.setHeader("X-Correlation-Id", id);
        try { chain.doFilter(req, res); }
        finally { MDC.remove("correlationId"); }
    }
}

@Component
public class TimingInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("start", System.nanoTime());
        return true;                    // false = stop processing
    }
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        long ms = (System.nanoTime() - (long) req.getAttribute("start")) / 1_000_000;
        log.info("{} {} -> {} in {}ms", req.getMethod(), req.getRequestURI(), res.getStatus(), ms);
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    private final TimingInterceptor timing;
    public void addInterceptors(InterceptorRegistry r) { r.addInterceptor(timing).addPathPatterns("/api/**"); }
    public void addCorsMappings(CorsRegistry r) {
        r.addMapping("/api/**").allowedOrigins("https://shop.example.com")
         .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE").allowCredentials(true).maxAge(3600);
    }
    public void configurePathMatch(PathMatchConfigurer c) { c.addPathPrefix("/api", HandlerTypePredicate.forAnnotation(RestController.class)); }
}
```
Register a filter for specific URLs: `FilterRegistrationBean<MyFilter>`.

## 10. CORS
Browsers block cross-origin requests (different scheme/host/port) unless the server sends `Access-Control-Allow-Origin` etc. Non-simple requests trigger a **preflight** `OPTIONS`. Configure via `@CrossOrigin` on controllers, `WebMvcConfigurer.addCorsMappings`, or (with Spring Security) `http.cors(...)` + a `CorsConfigurationSource` bean — security must be configured too, otherwise preflights get 401.

## 11. Async & streaming endpoints
```java
@GetMapping("/report")
public CompletableFuture<Report> report() { return reportService.buildAsync(); }   // servlet thread freed

@GetMapping(path = "/prices/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter stream() {
    SseEmitter emitter = new SseEmitter(60_000L);
    executor.execute(() -> {
        try { for (int i = 0; i < 10; i++) { emitter.send(price()); Thread.sleep(1000); } emitter.complete(); }
        catch (Exception e) { emitter.completeWithError(e); }
    });
    return emitter;
}
```
With virtual threads enabled, plain blocking controllers usually scale enough without async return types.

## 12. Calling other HTTP APIs from Spring
| Client | Style | Status |
|--------|-------|--------|
| `RestTemplate` | Synchronous, template | Maintenance mode — still common in legacy code |
| `WebClient` | Reactive, non-blocking (works in MVC too) | Active |
| **`RestClient`** (6.1 / Boot 3.2) | Synchronous fluent API | **Recommended for blocking apps** |
| HTTP Interface (`@HttpExchange`) | Declarative interface → proxy | Active (Feign-like, built-in) |
| OpenFeign | Declarative (Spring Cloud) | Feature-complete/maintenance |

```java
@Configuration
class ClientsConfig {
    @Bean
    RestClient inventoryRestClient(RestClient.Builder builder) {          // inject Boot's builder → metrics/tracing
        return builder.baseUrl("http://inventory:8080")
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .requestFactory(ClientHttpRequestFactories.get(ClientHttpRequestFactorySettings.DEFAULTS
                .withConnectTimeout(Duration.ofSeconds(2)).withReadTimeout(Duration.ofSeconds(5))))
            .build();
    }

    @Bean
    InventoryClient inventoryClient(RestClient inventoryRestClient) {
        return HttpServiceProxyFactory.builderFor(RestClientAdapter.create(inventoryRestClient))
                .build().createClient(InventoryClient.class);
    }
}

// Fluent usage
StockResponse stock = restClient.get().uri("/stock/{sku}", sku)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> { throw new SkuNotFoundException(sku); })
    .body(StockResponse.class);

List<Product> list = restClient.get().uri("/products").retrieve()
    .body(new ParameterizedTypeReference<List<Product>>() {});

ResponseEntity<Void> r = restClient.post().uri("/reserve").contentType(APPLICATION_JSON)
    .body(new ReserveRequest(sku, qty)).retrieve().toBodilessEntity();

// Declarative HTTP interface
@HttpExchange("/stock")
public interface InventoryClient {
    @GetExchange("/{sku}") StockResponse get(@PathVariable String sku);
    @PostExchange("/reserve") void reserve(@RequestBody ReserveRequest r);
}
```
**Always set connect and read timeouts** — default can be infinite, and a slow downstream will exhaust your thread pool.

## 13. API documentation — OpenAPI / Swagger
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.8.x</version>
</dependency>
```
- JSON spec: `/v3/api-docs`, UI: `/swagger-ui.html`.
```java
@Operation(summary = "Get product by id", description = "Returns a single product")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Found"),
    @ApiResponse(responseCode = "404", description = "Not found", content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
})
@GetMapping("/{id}") public ProductResponse get(@Parameter(description = "Product id") @PathVariable Long id) { ... }

@Bean OpenAPI api() {
    return new OpenAPI().info(new Info().title("Shop API").version("v1"))
        .components(new Components().addSecuritySchemes("bearer",
            new SecurityScheme().type(SecurityScheme.Type.HTTP).scheme("bearer").bearerFormat("JWT")));
}
```
API-first alternative: write the OpenAPI YAML first and generate interfaces with `openapi-generator-maven-plugin`.

## 14. Caching HTTP responses — ETags
```java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> get(@PathVariable Long id, WebRequest request) {
    ProductResponse p = service.get(id);
    String etag = "\"" + p.version() + "\"";
    if (request.checkNotModified(etag)) return null;     // 304
    return ResponseEntity.ok().eTag(etag).body(p);
}
```
Or `ShallowEtagHeaderFilter` (hashes the body; saves bandwidth, not server work). Optimistic concurrency for updates: client sends `If-Match: "3"`; mismatch → 412.

## 15. Rate limiting (overview)
Typically at API Gateway (Spring Cloud Gateway + Redis), or in-app with **Bucket4j** / Resilience4j RateLimiter. Return 429 + `Retry-After`. Algorithms: token bucket, leaky bucket, fixed window, sliding window.

## 16. i18n of messages
```properties
# messages.properties / messages_fr.properties
good.morning.message=Good Morning
```
```java
@GetMapping("/hello-i18n")
public String hello() {
    return messageSource.getMessage("good.morning.message", null, "Default", LocaleContextHolder.getLocale());
}
```
Locale from `Accept-Language` header via `AcceptHeaderLocaleResolver` (Boot default).

## 17. REST API checklist (production)
- [ ] Consistent resource naming, versioning
- [ ] DTOs + validation on every input
- [ ] Global exception handling with consistent error format (ProblemDetail)
- [ ] Correct status codes + `Location` on create
- [ ] Pagination & max page size
- [ ] Authentication/authorization, CORS
- [ ] Timeouts and retries on outbound calls
- [ ] Idempotency for payments/creates
- [ ] OpenAPI docs
- [ ] Request/response logging without PII, correlation IDs
- [ ] Rate limiting, request size limits (`spring.servlet.multipart.max-file-size`)
- [ ] Health checks, metrics
- [ ] Contract & integration tests

## 18. Interview Q&A
**Q. Explain the Spring MVC request flow.** Filters → DispatcherServlet → HandlerMapping → interceptors preHandle → HandlerAdapter (argument resolvers, message converters) → controller → return value handlers / message converters → postHandle → afterCompletion; exceptions via HandlerExceptionResolvers.

**Q. `@Controller` vs `@RestController`?** RestController = Controller + `@ResponseBody` on every method, so return values are written to the body instead of resolved as view names.

**Q. `@RequestParam` vs `@PathVariable`?** Query string (`?q=x`, optional filters) vs part of the URI path (resource identity).

**Q. PUT vs PATCH vs POST?** PUT replaces (idempotent), PATCH partially updates, POST creates or triggers non-idempotent actions.

**Q. How do you return 201 with a Location header?** `ResponseEntity.created(uri).body(dto)` using `ServletUriComponentsBuilder` / `UriComponentsBuilder`.

**Q. Filter vs Interceptor?** Filter: servlet-level, before Spring MVC, all requests. Interceptor: Spring MVC-level, knows the handler, can run after the handler.

**Q. How does Spring convert objects to JSON?** `HttpMessageConverter`s; `MappingJackson2HttpMessageConverter` uses Jackson's ObjectMapper, chosen by `Accept`/`Content-Type`.

**Q. What is idempotency and why is it important?** Repeating a request has the same effect — enables safe retries after timeouts; implement with idempotency keys for POST.

**Q. RestTemplate vs WebClient vs RestClient?** RestTemplate is legacy synchronous; WebClient is reactive non-blocking; RestClient is the modern synchronous fluent client built on the same infrastructure.

**Q. How do you version a REST API?** URI, request param, header, or media type versioning — pick one consistently; URI is most common.
