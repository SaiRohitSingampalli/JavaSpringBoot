# 13 — Spring Boot 3: Internals, Auto-configuration & Configuration

## 1. What Spring Boot is (and isn't)
Spring Boot is an **opinionated layer on top of the Spring Framework** that lets you build production-ready apps with minimal configuration. It is **not** a different framework: everything is still Spring beans.

| Without Boot | With Boot |
|--------------|-----------|
| Pick & align dozens of dependency versions | **Starters** + dependency management BOM |
| Configure DispatcherServlet, DataSource, EntityManagerFactory, TransactionManager, Jackson… by hand | **Auto-configuration** |
| Build WAR, deploy to external Tomcat | **Embedded server**, `java -jar` |
| Custom health/metrics endpoints | **Actuator** |
| Many XML/Java config files | `application.yml` + sensible defaults |

## 2. Spring Boot 3.x facts (and what's next)
- Spring Framework **6.x**, Java **17+**, Jakarta EE 9/10 (`jakarta.*`).
- GraalVM native image support with AOT processing.
- Observability via **Micrometer Observation API** + Micrometer Tracing (Sleuth retired).
- `ProblemDetail` (RFC 9457) error responses.
- 3.1: `@ServiceConnection` + Testcontainers at dev time, Docker Compose support (`spring-boot-docker-compose`).
- 3.2: **virtual threads** (`spring.threads.virtual.enabled`), `RestClient`, `JdbcClient`, CRaC support, SSL bundles.
- 3.3: CDS support, Micrometer improvements, SBOM actuator endpoint.
- 3.4: structured logging (ECS, Logstash, GELF), `@MockitoBean`/`@MockitoSpyBean`, `MockMvcTester` (AssertJ).
- 3.5: last 3.x minor line.
- **Spring Boot 4.0 / Spring Framework 7** (released Nov 2025): Java 17 baseline (25 recommended), Jakarta EE 11, JSpecify null-safety, API versioning in MVC/WebFlux, HTTP interface client auto-config, built-in resilience (`@Retryable`, `@ConcurrencyLimit`), modularized auto-configuration jars, Jackson 3. Concepts in these notes carry over.

## 3. Creating a project
**start.spring.io** (or IntelliJ / `spring init` CLI): choose Maven/Gradle, Java version, Boot version, group/artifact, dependencies (Web, JPA, Validation, Security, Actuator, DevTools, Lombok, driver).

### Maven pom essentials
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.6</version>          <!-- manages versions of ~hundreds of libraries -->
</parent>

<properties><java.version>21</java.version></properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>   <!-- no version needed -->
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>   <!-- builds executable fat jar, build-image -->
        </plugin>
    </plugins>
</build>
```
The parent gives: dependency versions (`spring-boot-dependencies` BOM), Java version, UTF-8, resource filtering, plugin config. If you have your own corporate parent, import the BOM in `<dependencyManagement>` instead.

### Common starters
| Starter | Brings |
|---------|--------|
| `spring-boot-starter` | Core, logging (Logback), YAML |
| `-web` | Spring MVC, embedded Tomcat, Jackson, validation-less |
| `-webflux` | Reactive web, Netty |
| `-validation` | Hibernate Validator |
| `-data-jpa` | Hibernate, Spring Data JPA, HikariCP |
| `-jdbc` | JdbcTemplate/JdbcClient, HikariCP |
| `-data-redis`, `-data-mongodb`, `-data-elasticsearch` | Respective clients |
| `-security`, `-oauth2-resource-server`, `-oauth2-client` | Spring Security |
| `-actuator` | Health, metrics, info endpoints, Micrometer |
| `-cache` | Cache abstraction |
| `-amqp`, `spring-kafka` | Messaging |
| `-mail`, `-thymeleaf`, `-websocket`, `-batch`, `-quartz`, `-graphql` | |
| `-test` | JUnit 5, Mockito, AssertJ, Spring Test, JSONassert, JsonPath, Awaitility |
| `spring-boot-devtools` | Auto-restart, LiveReload (dev only) |
| `spring-boot-configuration-processor` | Metadata for IDE autocomplete of your properties |

## 4. `@SpringBootApplication` unpacked
```java
@SpringBootApplication
public class ShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopApplication.class, args);
    }
}
```
=
```java
@SpringBootConfiguration      // a @Configuration
@EnableAutoConfiguration      // import auto-configuration classes
@ComponentScan                // scan this package and sub-packages
```
Place the main class in the **root package** (`com.example.shop`) so everything below it is scanned. Classes in `com.example.other` won't be found.

Exclusions: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` or `spring.autoconfigure.exclude=...`.

## 5. What happens inside `SpringApplication.run()`
1. Create `SpringApplication`: deduce app type (SERVLET / REACTIVE / NONE) from classpath, load `ApplicationContextInitializer`s and `ApplicationListener`s from `META-INF/spring.factories`.
2. Start `SpringApplicationRunListeners` → `ApplicationStartingEvent`.
3. Prepare **Environment**: load property sources (command line, env vars, `application.yml`, profiles) → `ApplicationEnvironmentPreparedEvent`.
4. Print banner.
5. Create the right `ApplicationContext` (`AnnotationConfigServletWebServerApplicationContext` for MVC).
6. Prepare context: apply initializers, register main class as a bean source → `ApplicationContextInitializedEvent`, `ApplicationPreparedEvent`.
7. **Refresh context**: process configuration classes, auto-configurations, component scan → create beans → `onRefresh()` **creates & starts embedded Tomcat** → `ContextRefreshedEvent`.
8. `ApplicationStartedEvent` → call `CommandLineRunner` / `ApplicationRunner` beans → `ApplicationReadyEvent` (app is ready to serve; readiness = ACCEPTING_TRAFFIC).
9. On failure → `ApplicationFailedEvent` + `FailureAnalyzer` prints a friendly message ("Port 8080 was already in use…").

```java
@Component
class StartupRunner implements ApplicationRunner {
    public void run(ApplicationArguments args) { log.info("Options: {}", args.getOptionNames()); }
}
```

## 6. Auto-configuration — how the magic works
1. `@EnableAutoConfiguration` → `AutoConfigurationImportSelector`.
2. Reads candidate classes from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in every jar (Boot 2.7+; older: `spring.factories`). `spring-boot-autoconfigure` lists ~150 (DataSource, JPA, MVC, Jackson, Security…).
3. Each class is `@AutoConfiguration` with **conditions**; only those whose conditions pass contribute beans.
4. They run **after** your own configuration, and most beans use `@ConditionalOnMissingBean` → **your beans win** ("back off").

Example — simplified `DataSourceAutoConfiguration` idea:
```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })   // JDBC on classpath?
@EnableConfigurationProperties(DataSourceProperties.class)             // binds spring.datasource.*
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(DataSource.class)                         // only if you didn't define one
    @ConditionalOnProperty(name = "spring.datasource.url")
    DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }
}
```

### Condition annotations
| Annotation | Matches when |
|-----------|-------------|
| `@ConditionalOnClass` / `OnMissingClass` | Class (not) on classpath |
| `@ConditionalOnBean` / `OnMissingBean` | Bean (not) already defined |
| `@ConditionalOnProperty(prefix, name, havingValue, matchIfMissing)` | Property value |
| `@ConditionalOnResource` | Resource exists |
| `@ConditionalOnWebApplication(type = SERVLET)` / `OnNotWebApplication` | App type |
| `@ConditionalOnExpression` | SpEL true |
| `@ConditionalOnJava` | Java version |
| `@ConditionalOnSingleCandidate` | Exactly one bean of type |
| `@ConditionalOnCloudPlatform(KUBERNETES)` | Running on platform |
| `@ConditionalOnThreading(VIRTUAL)` | Virtual threads enabled |

### Debugging auto-configuration
- `debug=true` (or `--debug`) → **Condition Evaluation Report** in logs: *Positive matches*, *Negative matches*, *Exclusions*.
- Actuator `/actuator/conditions`, `/actuator/beans`, `/actuator/configprops`.

### Writing your own starter (common in companies)
```
acme-audit-spring-boot-autoconfigure/
   src/main/java/.../AuditAutoConfiguration.java
   src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
        → com.acme.audit.AuditAutoConfiguration
acme-audit-spring-boot-starter/   (pom with dependencies on the autoconfigure module + libs)
```
```java
@AutoConfiguration
@ConditionalOnClass(AuditClient.class)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    AuditClient auditClient(AuditProperties p) { return new AuditClient(p.getUrl()); }
}
```
Naming: third-party starters are `xyz-spring-boot-starter`; official are `spring-boot-starter-xyz`.

## 7. Configuration files & externalized configuration

### 7.1 application.properties vs application.yml
```properties
server.port=8081
spring.datasource.url=jdbc:postgresql://localhost:5432/shop
app.features.new-checkout=true
app.allowed-origins=http://a.com,http://b.com
```
```yaml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: ${DB_USER:shop}          # env var with default
    password: ${DB_PASSWORD}
app:
  features:
    new-checkout: true
  allowed-origins:
    - http://a.com
    - http://b.com
---                                     # multi-document YAML
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### 7.2 Property source precedence (high → low, abbreviated)
1. Devtools global settings (`~/.config/spring-boot`)
2. `@TestPropertySource`, `@SpringBootTest(properties=...)`
3. **Command-line arguments** `--server.port=9000`
4. `SPRING_APPLICATION_JSON`
5. Servlet config/context params, JNDI
6. Java system properties `-Dserver.port=9000`
7. **OS environment variables** `SERVER_PORT=9000`
8. `random.*` values
9. **Profile-specific** files outside jar (`./config/application-prod.yml`) → outside jar → inside jar
10. `application.yml` outside jar → **inside jar**
11. `@PropertySource` on config classes
12. Default properties (`SpringApplication.setDefaultProperties`)

File locations searched: `classpath:/`, `classpath:/config/`, `./`, `./config/`, `./config/*/`. Additional: `spring.config.import=optional:file:./secrets.yml`, `configserver:`, `configtree:` (Kubernetes mounted secrets).

### 7.3 Relaxed binding
`spring.datasource.max-pool-size` ↔ `spring.datasource.maxPoolSize` ↔ `SPRING_DATASOURCE_MAXPOOLSIZE` (env var: uppercase, `.`→`_`, remove `-`). Lists via env: `APP_ALLOWEDORIGINS_0_=...`. Use kebab-case in files.

### 7.4 `@ConfigurationProperties` (type-safe config) ✔ preferred
```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
        @NotBlank String provider,
        @NotNull URI baseUrl,
        @DefaultValue("5s") Duration timeout,          // "5s", "500ms", "PT5S"
        @DefaultValue("3") @Min(1) int maxRetries,
        DataSize maxPayload,                            // "10MB"
        Map<String, String> headers,
        Retry retry) {
    public record Retry(@DefaultValue("200ms") Duration backoff) {}
}

@SpringBootApplication
@ConfigurationPropertiesScan            // or @EnableConfigurationProperties(PaymentProperties.class)
public class ShopApplication {}
```
```yaml
app:
  payment:
    provider: razorpay
    base-url: https://api.razorpay.com
    timeout: 3s
    max-payload: 5MB
    headers: { X-Client: shop }
    retry:
      backoff: 1s
```
Inject: `public PaymentClient(PaymentProperties props)`. Add `spring-boot-configuration-processor` for IDE autocomplete.

| | `@Value` | `@ConfigurationProperties` |
|--|--|--|
| Grouping | One property each | Hierarchical object |
| Relaxed binding | Limited | Full |
| Validation | No | `@Validated` + JSR-380 |
| SpEL | Yes | No |
| Type conversion (Duration, DataSize, lists, maps) | Basic | Rich |
| Immutability (records) | — | Yes |

### 7.5 Profiles in Boot
- Files: `application-dev.yml`, `application-prod.yml` override `application.yml`.
- Activate: `spring.profiles.active=dev,metrics` or `SPRING_PROFILES_ACTIVE=prod`.
- Groups: `spring.profiles.group.production=prod,postgres,monitoring`.
- Tests: `@ActiveProfiles("test")`.
- Don't put secrets in `application-prod.yml` in git — use env vars, Vault, Kubernetes secrets, cloud secret managers.

### 7.6 Random values & placeholders
```yaml
app.instance-id: ${random.uuid}
server.port: ${PORT:8080}
app.description: ${spring.application.name} is running
```

## 8. Embedded web server
- Default **Tomcat**; switch by excluding `spring-boot-starter-tomcat` and adding `-jetty` or `-undertow`. WebFlux uses **Netty**.
```yaml
server:
  port: 8080
  servlet.context-path: /api
  shutdown: graceful                       # finish in-flight requests on SIGTERM
  tomcat:
    threads.max: 200                       # default 200 worker threads
    threads.min-spare: 10
    accept-count: 100                      # queue when all threads busy
    max-connections: 8192
    connection-timeout: 20s
  compression.enabled: true
  http2.enabled: true
  ssl:                                     # or SSL bundles (spring.ssl.bundle.*)
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASS}
spring.lifecycle.timeout-per-shutdown-phase: 30s
```
Customize programmatically: `WebServerFactoryCustomizer<TomcatServletWebServerFactory>`.
Port `0` = random port (useful in tests/multiple instances).

### Packaging
- **Fat (uber) jar**: `BOOT-INF/classes`, `BOOT-INF/lib/*.jar`, `org/springframework/boot/loader` → launched by `JarLauncher` with a custom class loader that reads nested jars.
- **Layered jars** for Docker (dependencies, spring-boot-loader, snapshot-dependencies, application) — `java -Djarmode=tools -jar app.jar extract --layers`.
- WAR deployment: extend `SpringBootServletInitializer`, packaging `war`, mark Tomcat `provided`.

## 9. Logging
- Default: **SLF4J API + Logback**, console output, INFO level. Log4j2 available via starter.
```yaml
logging:
  level:
    root: INFO
    com.example.shop: DEBUG
    org.hibernate.SQL: DEBUG                       # SQL statements
    org.hibernate.orm.jdbc.bind: TRACE             # bound parameters (Hibernate 6)
    org.springframework.web: DEBUG
  file.name: logs/app.log
  logback.rollingpolicy.max-file-size: 10MB
  pattern.console: "%d %5p [%X{traceId:-}] %logger{36} - %msg%n"
  structured.format.console: ecs                    # Boot 3.4+: JSON logs (ecs | logstash | gelf)
logging.group.web: org.springframework.web, org.springframework.http
```
- Customize fully with `logback-spring.xml` (supports `<springProfile name="prod">`).
- Change levels at runtime: `POST /actuator/loggers/com.example.shop {"configuredLevel":"DEBUG"}`.
- Best practices: parameterized logging `log.info("Order {} placed", id)` (no string concat), never log secrets/PII, use MDC for correlation ids, JSON logs in production for ELK/Loki, appropriate levels (ERROR = needs action, WARN = unexpected but handled, INFO = business events, DEBUG = diagnostics).

## 10. DevTools
- Auto **restart** when classpath changes (two class loaders: base for libs, restart for your code → fast).
- LiveReload for templates, disables template caching, dev-friendly property defaults.
- Automatically disabled in packaged jars (`java -jar`). Mark `<optional>true</optional>` / `developmentOnly`.

## 11. Docker Compose & Testcontainers at dev time (3.1+)
- Add `spring-boot-docker-compose`, put a `compose.yaml` in the project root → Boot runs `docker compose up` at startup and auto-configures connection details (Postgres, Redis, Kafka...).
- Or run the app from a test class with Testcontainers: `SpringApplication.from(ShopApplication::main).with(TestcontainersConfig.class).run(args)` using `@ServiceConnection` beans.

## 12. Useful Boot features recap
| Feature | How |
|---------|-----|
| Banner | `banner.txt` in resources, `spring.main.banner-mode=off` |
| Startup time tracking | `spring.application.name`, `BufferingApplicationStartup` + `/actuator/startup` |
| Graceful shutdown | `server.shutdown=graceful` |
| Virtual threads | `spring.threads.virtual.enabled=true` (Java 21+) |
| Lazy init | `spring.main.lazy-initialization=true` |
| Build info | `spring-boot-maven-plugin` `build-info` goal → `/actuator/info` |
| Git info | `git-commit-id-maven-plugin` → `/actuator/info` |
| SSL bundles | `spring.ssl.bundle.jks.*` used by server and clients |
| Native image | `-Pnative native:compile` (GraalVM) |
| CDS / AOT cache | faster startup on JVM |

## 13. Typical project structure (layered)
```
com.example.shop
├── ShopApplication.java
├── config/            SecurityConfig, OpenApiConfig, WebConfig, properties records
├── common/            exceptions, utils, base classes
├── order/             ← package-by-feature (recommended for larger apps)
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   ├── Order.java (entity)
│   ├── dto/ OrderRequest, OrderResponse
│   └── OrderMapper.java
├── customer/
└── payment/
src/main/resources
├── application.yml, application-dev.yml, application-prod.yml
├── db/migration/V1__init.sql        (Flyway)
└── static/, templates/
```
Package-by-feature keeps related code together and enables package-private encapsulation (Spring Modulith can verify module boundaries).

Layer responsibilities:
- **Controller**: HTTP mapping, input validation, DTO ↔ service; no business logic.
- **Service**: business rules, transactions, orchestration.
- **Repository**: data access only.
- **Entity/Domain**: state + domain behaviour.
- **DTO/Mapper**: API contracts decoupled from entities (MapStruct for mapping).

## 14. Interview Q&A
**Q. What are the main features of Spring Boot?** Auto-configuration, starters, embedded server, externalized configuration, Actuator, opinionated defaults, devtools, testing support, native/AOT support.

**Q. How does auto-configuration work?** `@EnableAutoConfiguration` imports classes listed in `AutoConfiguration.imports`; each is guarded by `@Conditional*` annotations (class present, missing bean, property set); user beans take precedence via `@ConditionalOnMissingBean`.

**Q. How to disable a specific auto-configuration?** `exclude` attribute on `@SpringBootApplication` or `spring.autoconfigure.exclude`.

**Q. What is a starter?** A dependency descriptor aggregating compatible libraries for a feature, plus its auto-configuration.

**Q. Order of property sources?** Command line > system properties > env vars > profile-specific files > application.yml > defaults (outside jar beats inside).

**Q. `@Value` vs `@ConfigurationProperties`?** Single values with SpEL vs type-safe, validated, hierarchical binding.

**Q. How do you change the server port / use random port?** `server.port=9090`; `server.port=0` for random.

**Q. What happens when you run `SpringApplication.run`?** Determine app type, prepare environment, create context, load beans & auto-configs, start embedded server, run runners, publish ready event.

**Q. How to run code after startup?** `CommandLineRunner`/`ApplicationRunner` or `@EventListener(ApplicationReadyEvent.class)`.

**Q. How does Boot pick Tomcat?** `ServletWebServerFactoryAutoConfiguration` with `@ConditionalOnClass(Tomcat.class)` — Tomcat is on the classpath through `spring-boot-starter-web`.

**Q. Difference between Spring and Spring Boot?** Spring = framework (DI, MVC, data...). Boot = opinionated setup on top: auto-config, starters, embedded server, production features.
