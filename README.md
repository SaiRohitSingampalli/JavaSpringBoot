# Java Backend Developer Notes — Java · Spring Boot 3 · Microservices

Detailed, course-style notes covering what a Java backend developer is expected to know: core and modern Java, the Spring Framework and Spring Boot 3, data access, security, testing, messaging, microservices with Spring Cloud, Docker, Kubernetes, databases, system design and DevOps.

Each file explains **concepts and the reasoning behind them**, shows **working code**, calls out **pitfalls**, and ends with **interview questions with answers**.

## Contents

### Part 1 — Java
| # | Topic | Highlights |
|---|-------|-----------|
| 01 | [JVM internals, memory & GC](01-jvm-internals.md) | Class loading, heap/stack/metaspace, JIT, GC algorithms & collectors, OOM troubleshooting, JVM flags & tools |
| 02 | [Java language basics](02-java-basics.md) | Types, overflow, BigDecimal, operators, switch (all forms), methods, strings in depth, wrappers, arrays, java.time |
| 03 | [OOP in depth](03-oop-in-depth.md) | Four pillars, overriding rules, abstract vs interface, equals/hashCode, enums, records, sealed types, immutability, SOLID |
| 04 | [Exception handling](04-exceptions.md) | Hierarchy, checked vs unchecked, try-with-resources, custom exception design, best practices |
| 05 | [Collections deep dive](05-collections.md) | Choosing collections, Big-O, HashMap internals, ConcurrentHashMap, iterators, comparators, coding patterns |
| 06 | [Generics](06-generics.md) | Bounded types, wildcards & PECS, type erasure, super type tokens |
| 07 | [Functional programming & Streams](07-functional-streams.md) | Lambdas, functional interfaces, streams, collectors, parallel streams, Optional, 12 classic stream problems |
| 08 | [Concurrency & multithreading](08-concurrency.md) | JMM, synchronized/volatile/atomics/locks, executors, CompletableFuture, synchronizers, ThreadLocal, virtual threads |
| 09 | [Modern Java 8 → 25](09-modern-java-features.md) | Feature-by-version guide, migration notes |
| 10 | [Design patterns](10-design-patterns.md) | GoF patterns with code and where Spring uses each |

### Part 2 — Spring & Spring Boot
| # | Topic | Highlights |
|---|-------|-----------|
| 11 | [Spring core: IoC, DI, beans](11-spring-core-ioc-di.md) | Container, bean definitions, injection types, resolution rules, scopes, lifecycle, profiles, events, SpEL |
| 12 | [Spring AOP](12-spring-aop.md) | Advice types, pointcut language, proxies, self-invocation trap, AOP-based Spring features |
| 13 | [Spring Boot internals & configuration](13-spring-boot-internals-config.md) | Starters, auto-configuration internals, startup sequence, property precedence, `@ConfigurationProperties`, logging, packaging |
| 14 | [Spring MVC & REST APIs](14-spring-mvc-rest.md) | REST principles, status codes, DispatcherServlet flow, controllers, DTOs, Jackson, pagination, versioning, filters/interceptors, RestClient, OpenAPI |
| 15 | [Validation & exception handling](15-validation-exception-handling.md) | Bean Validation, groups, custom validators, `@RestControllerAdvice`, ProblemDetail |
| 16 | [JDBC, JPA & Hibernate](16-jdbc-jpa-hibernate.md) | Hikari, JdbcClient, entity mapping, relationships, persistence context, N+1, locking, performance, Flyway |
| 17 | [Spring Data JPA & transactions](17-spring-data-transactions.md) | Repositories, derived queries, projections, Specifications, `@Transactional` propagation/isolation/pitfalls |
| 18 | [Spring Security 6](18-spring-security.md) | Filter chain, authentication flow, SecurityFilterChain config, JWT, OAuth2/OIDC, method security, CSRF/CORS, OWASP |
| 19 | [Testing](19-testing.md) | JUnit 5, Mockito, test slices, MockMvc, Testcontainers, WireMock, ArchUnit |
| 20 | [Caching, async, scheduling, batch](20-caching-async-scheduling-events.md) | Cache patterns, `@Cacheable`, Redis, `@Async`, `@Scheduled` + ShedLock, Spring Batch, retry |
| 21 | [Messaging: Kafka & RabbitMQ](21-messaging-kafka-rabbitmq.md) | Kafka internals, Spring Kafka, DLT, delivery semantics, RabbitMQ exchanges, Spring Cloud Stream |
| 22 | [Reactive & WebFlux](22-reactive-webflux.md) | Reactive Streams, Mono/Flux operators, schedulers, WebFlux, WebClient, MVC vs WebFlux vs virtual threads |

### Part 3 — Microservices, cloud & beyond
| # | Topic | Highlights |
|---|-------|-----------|
| 23 | [Microservices architecture & patterns](23-microservices-architecture-patterns.md) | When/why, DDD boundaries, communication, Saga, Outbox, CQRS, resilience, security, observability |
| 24 | [Spring Cloud hands-on](24-spring-cloud.md) | Config Server, Feign, Eureka, LoadBalancer, Gateway, Resilience4j, Micrometer Tracing + Zipkin, Compose |
| 25 | [Observability & Actuator](25-observability-actuator.md) | Actuator endpoints, health/probes, structured logging, Micrometer metrics, Prometheus/Grafana, SLOs |
| 26 | [Docker](26-docker.md) | Concepts, commands, Dockerfiles for Spring Boot, buildpacks, JVM in containers, Compose |
| 27 | [Kubernetes](27-kubernetes.md) | Architecture, objects, full manifests for a Boot app, probes, rollouts, Helm/GitOps, troubleshooting |
| 28 | [Databases: SQL & NoSQL](28-databases-sql-nosql.md) | SQL incl. window functions, indexing & EXPLAIN, isolation/locking, scaling, NoSQL families, CAP |
| 29 | [Backend fundamentals & system design](29-system-design-backend-fundamentals.md) | HTTP/TLS/networking, API styles, scaling & reliability, classic design problems, LLD |
| 30 | [Build tools, Git, CI/CD, cloud](30-build-tools-git-cicd.md) | Maven/Gradle, Git workflows, pipelines, GitHub Actions, cloud service map |
| 31 | [Interview revision & projects](31-interview-revision-and-projects.md) | Rapid revision, 22 scenario questions with answers, coding checklist, 5 practice projects, 12-week plan |

## How to use these notes
1. Read in order the first time; later jump to topics as needed.
2. Type out and run the code (JShell for Java snippets, start.spring.io for Spring projects).
3. Build the practice projects in file 31 — that's what turns notes into skill.
4. Before interviews, use each file's Q&A section and the rapid revision sheet.

## Baseline versions
- Java 21 LTS (Java 17 minimum for Spring Boot 3; Java 25 is the newest LTS)
- Spring Boot 3.x / Spring Framework 6.x (Boot 4 / Framework 7 changes are noted where relevant)
- Jakarta EE namespaces (`jakarta.*`)
- Spring Cloud 2024.x/2025.x release trains
- Hibernate 6.x, JUnit 5, Mockito 5, Testcontainers

Versions move fast. When a detail matters (exact property names, plugin versions), check the official docs: docs.spring.io, dev.java, kafka.apache.org, kubernetes.io.
