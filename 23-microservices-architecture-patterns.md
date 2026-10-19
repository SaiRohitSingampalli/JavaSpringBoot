# 23 — Microservices Architecture & Patterns

## 1. Monolith → Microservices
**Monolith**: one deployable unit containing all features, usually one database.
**Modular monolith**: one deployable, but strict internal module boundaries (Spring Modulith) — a great default for new products.
**Microservices**: suite of small, **independently deployable** services, each owning a business capability and its data, communicating over the network.

| | Monolith | Microservices |
|--|--|--|
| Deployment | All-or-nothing | Independent per service |
| Scaling | Whole app | Per service |
| Data | Shared DB, ACID transactions | DB per service, eventual consistency |
| Tech stack | One | Polyglot possible |
| Team structure | Shared codebase | Team per service ("you build it, you run it") |
| Failure | Bug can take down all | Isolated (if designed for it) |
| Complexity | In the code | In the network/operations |
| Testing | Simpler end-to-end | Contract tests, harder E2E |
| Latency | In-process calls | Network calls |

### When microservices make sense
- Multiple teams stepping on each other in one codebase.
- Parts with very different scaling/availability/release needs.
- Mature DevOps: CI/CD, containers, observability, on-call.

### When NOT to
- Small team / early-stage product / unclear domain boundaries.
- No automation or monitoring.
- **Distributed monolith** anti-pattern: services that must be deployed together, share a DB, or call each other synchronously in long chains — all the costs, none of the benefits.

### Fallacies of distributed computing
The network is reliable; latency is zero; bandwidth is infinite; the network is secure; topology doesn't change; there is one administrator; transport cost is zero; the network is homogeneous. → Design for failure: timeouts, retries, circuit breakers, idempotency.

## 2. Designing service boundaries — DDD essentials
- **Domain**: the business problem space; split into **subdomains** (core, supporting, generic).
- **Bounded context**: explicit boundary inside which a model and its **ubiquitous language** are consistent. "Product" in Catalog (description, images) ≠ "Product" in Inventory (sku, quantity) ≠ in Shipping (weight, dimensions). Bounded contexts are natural service candidates.
- **Context mapping**: relationships between contexts — Customer/Supplier, Conformist, **Anti-Corruption Layer** (translate external models), Shared Kernel, Open Host Service / Published Language.
- **Aggregate**: cluster of entities treated as one unit for consistency; one **aggregate root** (Order with OrderItems); transactions shouldn't span aggregates; reference other aggregates by id.
- **Entities** (identity) vs **Value Objects** (immutable, compared by value: Money, Address).
- **Domain events**: something meaningful that happened (`OrderPlaced`).
- **Event Storming**: workshop technique to discover events, commands, aggregates and contexts.

Decomposition strategies: by **business capability** (Orders, Payments, Catalog, Shipping, Customers), by subdomain, strangler fig migration from a monolith.

Size heuristic: a service should be owned by one team, independently deployable, and cohesive — "small enough" not "as small as possible". Avoid nano-services.

## 3. Communication

### 3.1 Synchronous
| Style | Notes |
|-------|-------|
| **REST/HTTP + JSON** | Universal, simple, cacheable |
| **gRPC** | HTTP/2 + Protobuf; fast, strongly-typed contracts, streaming; great for internal service-to-service; less browser-friendly |
| **GraphQL** | Client chooses fields; good for BFFs/aggregation (Spring for GraphQL) |

Downside: temporal coupling, latency adds up, cascading failures → keep sync chains short; use resilience patterns.

### 3.2 Asynchronous
Message brokers (Kafka, RabbitMQ) — events and commands. Loose coupling, resilience, eventual consistency. See messaging notes.

### 3.3 Choreography vs orchestration
- **Choreography**: services react to each other's events; no central brain. Loose coupling but flow is implicit and hard to trace.
- **Orchestration**: a coordinator (Saga orchestrator, workflow engine like Temporal/Camunda/Step Functions) tells services what to do. Explicit flow, easier monitoring; coordinator is extra component.

## 4. Data management

### 4.1 Database per service
Each service owns its schema; others access data **only through its API/events**. Enables independent evolution and polyglot persistence (Postgres for orders, Elasticsearch for search, Redis for sessions, Cassandra for timelines).
Shared database = hidden coupling (schema change breaks other teams).

### 4.2 Distributed transactions — the Saga pattern ⭐
A business transaction spanning services is a **sequence of local transactions**; each publishes an event/message triggering the next. On failure, run **compensating transactions** to undo previous steps (semantic undo — "refund", not rollback).

Example — place order:
```
1. Order Service:     create order (PENDING)            → OrderCreated
2. Payment Service:   charge customer                   → PaymentCompleted  | PaymentFailed
3. Inventory Service: reserve stock                     → StockReserved     | StockUnavailable
4. Order Service:     mark CONFIRMED

Failure at step 3 (StockUnavailable):
   Payment Service: refund (compensation)
   Order Service:   mark CANCELLED (compensation)
```
| | Choreography saga | Orchestration saga |
|--|--|--|
| Coordination | Events between services | Central orchestrator sends commands |
| Coupling | Lower | Services coupled to orchestrator API |
| Visibility | Hard to see whole flow | Clear state machine |
| Good for | 2–4 simple steps | Complex flows, many steps |

Saga considerations: lack of isolation (other transactions see intermediate states) → countermeasures: semantic locks (status PENDING), commutative updates, reread values, versioning. Steps should be idempotent and retryable; design pivot transactions (point of no return) carefully.

### 4.3 Transactional Outbox ⭐
**Dual-write problem**: `db.save(order); kafka.send(event);` — if the app crashes between them, DB and broker disagree.
**Solution**: in the **same local DB transaction**, insert the event into an `outbox` table. A separate relay publishes outbox rows to the broker:
- **Polling publisher**: scheduled job reads unpublished rows, sends, marks sent.
- **Transaction log tailing (CDC)**: Debezium reads the DB WAL/binlog and streams outbox rows to Kafka.
Guarantees at-least-once publishing → consumers idempotent. Mirror pattern on the consumer side: **Inbox** (dedup table of processed message ids).

### 4.4 CQRS (Command Query Responsibility Segregation)
Separate write model (commands, normalized, validated) from read model(s) (denormalized views optimized for queries), often updated asynchronously via events. Use when read and write patterns differ greatly (e.g., order history search across services). Adds eventual consistency & complexity.

### 4.5 Event Sourcing
Store the **sequence of events** as the source of truth instead of current state (`AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn`); current state = replay/fold of events (+ snapshots). Benefits: full audit trail, temporal queries, rebuild projections. Costs: complexity, event schema evolution, eventual consistency. Tools: Axon Framework, EventStoreDB, Kafka as log.

### 4.6 Querying across services
- **API Composition**: an aggregator (gateway/BFF/service) calls multiple services and joins results in memory.
- **CQRS read model**: subscribe to events and build a local denormalized view.
- **Data replication** of reference data via events (each service keeps the subset it needs).

## 5. Infrastructure patterns
| Pattern | Problem → solution | Tools |
|---------|-------------------|-------|
| **API Gateway** | Single entry point: routing, auth, rate limiting, SSL termination, CORS, aggregation | Spring Cloud Gateway, Kong, NGINX, Envoy, AWS API Gateway |
| **Backend for Frontend (BFF)** | Separate gateway per client type (web, mobile) | Spring Cloud Gateway / GraphQL |
| **Service Discovery** | Find dynamic instance locations | Eureka, Consul, Kubernetes DNS |
| Client-side vs server-side discovery | Client queries registry & load-balances (Eureka + LoadBalancer) vs load balancer queries registry (K8s Service, AWS ALB) | |
| **Externalized Configuration** | Config separate from code, per environment | Spring Cloud Config, K8s ConfigMaps/Secrets, Vault |
| **Circuit Breaker** | Stop calling failing dependency | Resilience4j |
| **Retry with backoff + jitter**, **Timeout**, **Bulkhead**, **Rate limiter**, **Fallback** | Resilience | Resilience4j |
| **Health Check API** | Liveness/readiness for orchestrators | Actuator |
| **Distributed Tracing** | Follow a request across services | Micrometer Tracing + OpenTelemetry, Zipkin/Jaeger/Tempo |
| **Log Aggregation** | Central searchable logs | ELK/EFK, Loki, Splunk, CloudWatch |
| **Application Metrics** | Dashboards & alerts | Micrometer → Prometheus → Grafana |
| **Sidecar** / **Service Mesh** | mTLS, retries, traffic shifting outside app code | Istio, Linkerd, Envoy |
| **Strangler Fig** | Incrementally replace a monolith behind a facade/gateway | |
| **Anti-Corruption Layer** | Translate between your model and legacy/external models | |
| **Consumer-Driven Contracts** | Catch breaking API changes before deploy | Pact, Spring Cloud Contract |
| **Idempotency Key** | Safe retries of POST | Redis/DB store |
| **Competing Consumers** | Scale processing of a queue | Kafka groups, Rabbit queues |
| **Deployment patterns** | Rolling, **blue-green**, **canary**, feature flags, shadow traffic | K8s, Argo Rollouts, Flagger, LaunchDarkly/Unleash |

## 6. Resilience in depth
Why: in a call chain A → B → C, if C is slow, B's threads wait, then A's threads wait → whole system collapses (**cascading failure**).

1. **Timeouts** on every remote call (connect + read). Budget: downstream timeout < upstream timeout.
2. **Retries** only for transient, idempotent operations; exponential backoff + jitter; limit attempts; beware **retry storms** (retries at each layer multiply).
3. **Circuit breaker**: CLOSED (normal) → too many failures/slow calls → OPEN (fail fast, fallback) → after wait → HALF_OPEN (test a few calls) → CLOSED or OPEN.
4. **Bulkhead**: isolate resources per dependency (separate thread pools/semaphores) so one slow dependency can't consume all threads.
5. **Rate limiting / load shedding**: protect services from overload; return 429/503 early.
6. **Fallbacks**: cached data, default values, degraded functionality ("recommendations unavailable").
7. **Asynchronous decoupling** via queues.
8. **Graceful degradation & chaos engineering** (Chaos Monkey for Spring Boot, Gremlin, Litmus) to verify.

## 7. Security in microservices
- **Edge authentication** at the gateway (OAuth2/OIDC with an IdP like Keycloak); services act as **resource servers** validating JWTs themselves (zero trust — don't blindly trust internal traffic).
- **Token relay**: gateway forwards the user's access token downstream; or **token exchange** for narrower scopes.
- **Service-to-service**: OAuth2 client credentials or **mTLS** (service mesh).
- Secrets management (Vault, cloud secret managers, K8s secrets + encryption at rest).
- Least privilege network policies.

## 8. Observability (three pillars + more)
- **Logs**: structured JSON with traceId/spanId, centralized.
- **Metrics**: RED (Rate, Errors, Duration) for services; USE (Utilization, Saturation, Errors) for resources; golden signals (latency, traffic, errors, saturation).
- **Traces**: end-to-end request flow with spans; find the slow hop.
- **Correlation**: same traceId in logs & traces → jump between them.
- **SLI/SLO/SLA**: indicator (p99 latency), objective (99.9% < 300ms), agreement (contract with penalties); error budgets.
- Alerting on symptoms (SLO burn rate), not just causes.

## 9. Deployment & operations
- Containerize (Docker), orchestrate (Kubernetes), CI/CD pipelines per service, GitOps (Argo CD/Flux).
- **Twelve-Factor App**: 1 codebase, 2 explicit dependencies, 3 config in env, 4 backing services as attached resources, 5 build/release/run separation, 6 stateless processes, 7 port binding, 8 scale via processes, 9 disposability (fast startup, graceful shutdown), 10 dev/prod parity, 11 logs as event streams, 12 admin processes as one-off tasks.
- Versioning & backward compatibility of APIs and events (tolerant reader, additive changes, expand/contract DB migrations).
- Platform engineering: internal developer platforms, golden paths, service templates.

## 10. Typical Spring-based microservice reference architecture
```
                 ┌──────────────┐
 Web/Mobile ───► │  API Gateway  │──── OAuth2 (Keycloak) ── validate JWT, rate limit
                 └──────┬───────┘
          ┌─────────────┼───────────────┬───────────────┐
          ▼             ▼               ▼               ▼
     Order Svc     Payment Svc     Inventory Svc   Notification Svc
     (Postgres)    (Postgres)      (Postgres)      (consumes events)
          │  outbox ─► Kafka ◄── events ──┘               ▲
          └─────────────────────────────────────────────────┘
 Cross-cutting: Config (Config Server / ConfigMaps), Discovery (Eureka / K8s DNS),
 Resilience4j, Micrometer → Prometheus/Grafana, OTel traces → Tempo/Zipkin, Loki/ELK logs,
 Docker images → Kubernetes (Helm/Argo CD), CI/CD (GitHub Actions/Jenkins)
```

## 11. Migration from monolith — Strangler Fig steps
1. Put a gateway/proxy in front of the monolith.
2. Pick a well-bounded capability with high change rate (e.g., notifications).
3. Build the new service; sync data (events/CDC); route the relevant endpoints to it.
4. Remove the code from the monolith; repeat. Keep the monolith shrinking, not growing.

## 12. Interview Q&A
**Q. What are microservices and their pros/cons?** (See §1.) Independent deployability and scaling vs operational complexity, distributed data, network failures.

**Q. How do you decide service boundaries?** DDD bounded contexts / business capabilities, team ownership, data cohesion, change frequency; avoid chatty, tightly coupled splits.

**Q. How do you maintain data consistency across services?** Sagas with compensations, outbox for reliable events, idempotent consumers, eventual consistency; avoid 2PC.

**Q. Choreography vs orchestration?** Event-driven implicit flow vs central coordinator with explicit flow.

**Q. What is the outbox pattern?** Persist the event in an outbox table within the same transaction as the state change; a relay/CDC publishes it — solves dual writes.

**Q. What is CQRS? When to use it?** Separate read/write models; for complex domains with very different read requirements or high read scale.

**Q. How do you handle a downstream service being slow?** Timeouts, circuit breaker, bulkhead, fallback, caching, async processing.

**Q. API Gateway responsibilities?** Routing, auth, rate limiting, TLS termination, request aggregation, CORS, logging/metrics, canary routing.

**Q. How do you trace a request across services?** Distributed tracing (W3C `traceparent` header propagation via Micrometer Tracing/OpenTelemetry), traceId in logs.

**Q. What is a distributed monolith?** Services that are tightly coupled (shared DB, synchronized deployments, sync call chains) — worst of both worlds.

**Q. Blue-green vs canary deployment?** Blue-green switches all traffic between two full environments; canary shifts a small percentage gradually while monitoring.
