# 19 — Testing Spring Boot Applications

## 1. Testing pyramid
```
          ▲  E2E / UI tests        few, slow, brittle
         ▲▲▲ Integration tests     DB, HTTP, messaging (Testcontainers)
       ▲▲▲▲▲ Slice tests           @WebMvcTest, @DataJpaTest
    ▲▲▲▲▲▲▲▲ Unit tests            many, fast, no Spring
```
Plus: **contract tests** between services, performance/load tests, security tests, mutation testing (PIT).

`spring-boot-starter-test` includes: **JUnit 5 (Jupiter)**, **Mockito**, **AssertJ**, Hamcrest, Spring Test & Boot Test, JSONassert, JsonPath, Awaitility, XMLUnit.

## 2. JUnit 5 essentials
```java
@DisplayName("Price calculator")
class PriceCalculatorTest {

    private PriceCalculator calc;

    @BeforeAll static void initAll() {}          // once per class (static)
    @BeforeEach void setUp() { calc = new PriceCalculator(); }
    @AfterEach void tearDown() {}
    @AfterAll static void tearDownAll() {}

    @Test
    void appliesTenPercentDiscountAbove1000() {
        // Arrange - Act - Assert (Given - When - Then)
        BigDecimal result = calc.finalPrice(new BigDecimal("2000"));
        assertThat(result).isEqualByComparingTo("1800");
    }

    @Test
    void rejectsNegativePrice() {
        assertThatThrownBy(() -> calc.finalPrice(new BigDecimal("-1")))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("negative");
        // JUnit style: IllegalArgumentException ex = assertThrows(IllegalArgumentException.class, () -> ...);
    }

    @ParameterizedTest(name = "{0} → {1}")
    @CsvSource({ "100, 100", "1000, 1000", "1001, 900.90" })
    void discountTable(BigDecimal input, BigDecimal expected) {
        assertThat(calc.finalPrice(input)).isEqualByComparingTo(expected);
    }

    @ParameterizedTest @ValueSource(strings = {"", " ", "\t"})
    void blankNamesInvalid(String name) { ... }

    @ParameterizedTest @MethodSource("orders")
    void totals(Order o, BigDecimal expected) { ... }
    static Stream<Arguments> orders() { return Stream.of(Arguments.of(order1, new BigDecimal("10"))); }

    @Nested class WhenCartIsEmpty { @Test void totalIsZero() {} }

    @Test @Disabled("flaky — JIRA-123") void skipped() {}
    @Test @Timeout(2) void fast() {}
    @RepeatedTest(5) void repeated() {}
    @Test @Tag("slow") void slowTest() {}
    @Test @EnabledOnOs(OS.LINUX) void linuxOnly() {}
}
```
Assertions: JUnit `assertEquals(expected, actual)`, `assertTrue`, `assertAll(...)` (grouped), `assertIterableEquals`, `assertTimeout`. **AssertJ** (fluent, preferred): `assertThat(list).hasSize(3).extracting(User::name).containsExactly("a","b","c")`, `assertThat(opt).isPresent().get().extracting("id").isEqualTo(1L)`, `assertThat(map).containsEntry("k", 1)`.

Test naming: `methodName_condition_expectedResult` or descriptive sentences with `@DisplayName`.

## 3. Mockito
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository repo;
    @Mock PaymentGateway gateway;
    @Mock ApplicationEventPublisher events;
    @InjectMocks OrderService service;           // constructor injection of mocks
    @Captor ArgumentCaptor<Order> orderCaptor;

    @Test
    void placesOrderAndChargesCard() {
        // stubbing
        when(gateway.charge(any(Money.class), eq("tok_123"))).thenReturn(new Receipt("TXN-1"));
        when(repo.save(any(Order.class))).thenAnswer(inv -> { Order o = inv.getArgument(0); o.setId(10L); return o; });

        Order result = service.place(new OrderRequest("tok_123", List.of(item)));

        assertThat(result.getId()).isEqualTo(10L);
        verify(gateway, times(1)).charge(any(), eq("tok_123"));
        verify(repo).save(orderCaptor.capture());
        assertThat(orderCaptor.getValue().getStatus()).isEqualTo(OrderStatus.PAID);
        verify(events).publishEvent(any(OrderPlaced.class));
        verifyNoMoreInteractions(gateway);
    }

    @Test
    void paymentFailureDoesNotSave() {
        when(gateway.charge(any(), any())).thenThrow(new PaymentDeclinedException("declined"));
        assertThatThrownBy(() -> service.place(request)).isInstanceOf(PaymentDeclinedException.class);
        verify(repo, never()).save(any());
    }
}
```
| API | Purpose |
|-----|---------|
| `when(...).thenReturn/thenThrow/thenAnswer` | Stub |
| `doReturn/doThrow/doNothing().when(mock).method()` | Stub void methods / spies |
| `verify(mock, times(n) / never() / atLeastOnce())` | Interaction checks |
| `ArgumentCaptor` | Capture arguments for assertions |
| Matchers: `any()`, `anyLong()`, `eq()`, `argThat(...)`, `isNull()` | All-or-none matchers per call |
| `@Spy` / `spy(obj)` | Partial mock — real methods unless stubbed |
| `mockStatic(Clock.class)` | Static mocking (inline mock maker, default since Mockito 5) |
| BDD: `given(...).willReturn(...)`, `then(mock).should()` | BDDMockito |
| `lenient()` | Allow unused stubs (strict stubs fail on them by default) |

Guidelines: mock **collaborators you own**, not value objects; don't mock what you don't own deeply (wrap third-party APIs); prefer state verification over excessive `verify`; inject a `Clock` instead of calling `Instant.now()` so time is testable.

## 4. Spring Boot test slices
Load only the part of the context you need → faster tests.

| Annotation | Loads | Typical use |
|-----------|-------|-------------|
| `@WebMvcTest(Controller.class)` | MVC infra, controllers, advice, filters, Jackson, security | Controller tests with MockMvc; services mocked |
| `@WebFluxTest` | WebFlux equivalent | WebTestClient |
| `@DataJpaTest` | JPA, repositories, embedded DB / Testcontainers, `TestEntityManager`; **each test transactional & rolled back** | Repository/query tests |
| `@JdbcTest`, `@DataJdbcTest` | JDBC | |
| `@DataMongoTest`, `@DataRedisTest` | | |
| `@JsonTest` | Jackson, `JacksonTester` | Serialization format |
| `@RestClientTest` | RestTemplate/RestClient builder + `MockRestServiceServer` | HTTP client tests |
| `@SpringBootTest` | **Whole application context** | Integration tests |

### Controller test with MockMvc
```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired MockMvc mvc;
    @Autowired ObjectMapper json;
    @MockitoBean ProductService service;          // Boot 3.4+ (previously @MockBean, now deprecated)

    @Test
    @WithMockUser
    void returnsProduct() throws Exception {
        when(service.get(1L)).thenReturn(new ProductResponse(1L, "Pen", new BigDecimal("10.00"), "office", null));

        mvc.perform(get("/api/v1/products/1").accept(MediaType.APPLICATION_JSON))
           .andExpect(status().isOk())
           .andExpect(content().contentType(MediaType.APPLICATION_JSON))
           .andExpect(jsonPath("$.name").value("Pen"))
           .andExpect(jsonPath("$.price").value(10.00))
           .andDo(print());
    }

    @Test
    @WithMockUser
    void createValidatesBody() throws Exception {
        var invalid = new CreateProductRequest("", null, "office", null);
        mvc.perform(post("/api/v1/products").with(csrf())
                .contentType(MediaType.APPLICATION_JSON).content(json.writeValueAsString(invalid)))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.errors").isArray());
        verifyNoInteractions(service);
    }

    @Test
    @WithMockUser
    void createReturns201WithLocation() throws Exception {
        when(service.create(any())).thenReturn(new ProductResponse(5L, "Pen", BigDecimal.TEN, "office", null));
        mvc.perform(post("/api/v1/products").with(csrf()).contentType(APPLICATION_JSON)
                .content("""
                    {"name":"Pen","price":10,"category":"office"}
                    """))
           .andExpect(status().isCreated())
           .andExpect(header().string("Location", endsWith("/api/v1/products/5")));
    }
}
```
Boot 3.4 also offers `MockMvcTester` (AssertJ-style): `assertThat(mvc.get().uri("/products/1")).hasStatusOk().bodyJson().extractingPath("$.name").isEqualTo("Pen");`

### Repository test
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)   // use Testcontainers DB instead of H2
@Import(TestcontainersConfig.class)
class ProductRepositoryTest {
    @Autowired ProductRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void findsCheapProductsInCategory() {
        em.persist(new Product("Pen", new BigDecimal("5"), "office"));
        em.persist(new Product("Desk", new BigDecimal("500"), "office"));
        em.flush(); em.clear();

        assertThat(repo.findByCategoryAndPriceLessThan("office", new BigDecimal("100")))
            .extracting(Product::getName).containsExactly("Pen");
    }
}
```

### Full integration test
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderFlowIT {

    @Container @ServiceConnection                        // Boot 3.1+: auto-configures spring.datasource.*
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("apache/kafka:3.8.0"));

    @Autowired TestRestTemplate rest;                    // or WebTestClient / RestClient on the random port
    @Autowired OrderRepository orders;

    @Test
    void placeOrderEndToEnd() {
        ResponseEntity<OrderResponse> res = rest.postForEntity("/api/v1/orders", request, OrderResponse.class);
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        await().atMost(Duration.ofSeconds(10))
               .untilAsserted(() -> assertThat(orders.findById(res.getBody().id())).get()
                   .extracting(Order::getStatus).isEqualTo(OrderStatus.CONFIRMED));
    }
}
```
Older approach (pre-3.1): `@DynamicPropertySource static void props(DynamicPropertyRegistry r) { r.add("spring.datasource.url", postgres::getJdbcUrl); }`.

**Context caching**: Spring caches application contexts across tests with the same configuration. Every distinct combination of `@MockitoBean`s, properties, profiles creates a new context → slow suites. Keep test configurations consistent; use a shared base class.

## 5. Testing HTTP clients
```java
@RestClientTest(InventoryClient.class)
class InventoryClientTest {
    @Autowired InventoryClient client;
    @Autowired MockRestServiceServer server;

    @Test void getsStock() {
        server.expect(requestTo("/stock/SKU1")).andRespond(withSuccess("{\"qty\":5}", APPLICATION_JSON));
        assertThat(client.stock("SKU1").qty()).isEqualTo(5);
    }
}
```
Or **WireMock** (`wiremock-spring-boot`) for realistic stubbed HTTP servers including faults and delays — great for testing timeouts/circuit breakers.

## 6. Other test tools
| Tool | Use |
|------|-----|
| **Testcontainers** | Real Postgres/MySQL/Kafka/Redis/RabbitMQ/Keycloak in Docker for tests |
| **WireMock** | Stub external HTTP APIs |
| **Awaitility** | Assert async results eventually |
| **ArchUnit** | Architecture rules (controllers don't call repositories, no cycles) |
| **Spring Modulith tests** | Verify module boundaries, test modules in isolation |
| **Spring Cloud Contract / Pact** | Consumer-driven contract testing |
| **REST Assured** | Fluent HTTP API testing |
| **JaCoCo** | Coverage reports |
| **PIT** | Mutation testing |
| **Gatling / k6 / JMeter** | Load tests |
| **Instancio / EasyRandom** | Generate test data |

ArchUnit example:
```java
@AnalyzeClasses(packages = "com.shop")
class ArchitectureTest {
    @ArchTest static final ArchRule controllersDontUseRepos =
        noClasses().that().resideInAPackage("..web..").should().dependOnClassesThat().resideInAPackage("..repository..");
}
```

## 7. Testing configuration helpers
- `@ActiveProfiles("test")`, `src/test/resources/application-test.yml`.
- `@TestPropertySource(properties = "feature.x=true")`, `@SpringBootTest(properties = ...)`.
- `@TestConfiguration` — extra beans only for tests (`@Import` it).
- `@DirtiesContext` — rebuild context (slow, avoid).
- `@Sql("/data/orders.sql")` — run SQL before a test.
- `@Transactional` on test methods → rollback after each test (not for RANDOM_PORT tests — server runs in other threads).
- `OutputCaptureExtension` — assert log output.
- `@AutoConfigureMockMvc` in `@SpringBootTest` for MockMvc against the full context.

## 8. Good test practices
- **F.I.R.S.T.**: Fast, Independent, Repeatable, Self-validating, Timely.
- One behaviour per test; clear Arrange/Act/Assert.
- Test behaviour, not implementation details.
- Use real dependencies where cheap (Testcontainers) instead of over-mocking.
- Deterministic: inject `Clock`, fixed random seeds, no sleeps (Awaitility).
- Keep test data builders (`aProduct().withPrice(10).build()`).
- Run fast unit/slice tests on every commit; integration tests in CI (Maven: Surefire for `*Test`, Failsafe for `*IT`).
- Coverage is a signal, not a goal (~70–80% on business logic is common).

## 9. TDD in brief
Red → Green → Refactor: write a failing test, minimal code to pass, then clean up. Improves design (testable units, DI) and confidence.

## 10. Interview Q&A
**Q. `@Mock` vs `@MockitoBean` (`@MockBean`)?** `@Mock` = plain Mockito mock (no Spring). `@MockitoBean` replaces/adds a bean in the Spring test context.

**Q. `@WebMvcTest` vs `@SpringBootTest`?** Slice loading only MVC components (fast, mock services) vs the full application context (integration).

**Q. How do you test repositories?** `@DataJpaTest` with Testcontainers (real DB) or H2; `TestEntityManager` to set up data; flush/clear before asserting.

**Q. Why Testcontainers over H2?** Same DB engine/dialect/features as production → catches real SQL/migration issues.

**Q. How do you test async/event-driven code?** Awaitility to poll for expected state; Testcontainers Kafka; capture events with test listeners.

**Q. Mock vs Spy vs Stub?** Stub returns canned answers; Mock also verifies interactions; Spy wraps a real object, calling real methods unless stubbed.

**Q. How do you test security rules?** MockMvc with `@WithMockUser`, `jwt()`, `csrf()`; assert 401/403/200.

**Q. What is a contract test?** Verifies provider and consumer agree on API shape (Spring Cloud Contract / Pact), preventing integration breakage between microservices.
