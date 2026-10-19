# 29 — Backend Fundamentals & System Design

## Part A — Networking & HTTP fundamentals

### 1. What happens when you type a URL and press Enter?
1. Browser parses URL; checks HSTS/caches.
2. **DNS** resolution: browser cache → OS → resolver → root → TLD → authoritative → IP (A/AAAA record; CNAME chains).
3. **TCP** 3-way handshake (SYN, SYN-ACK, ACK) — or QUIC/UDP for HTTP/3.
4. **TLS** handshake (certificate verification, key exchange → symmetric session keys). TLS 1.3 = 1 round trip.
5. HTTP request sent → passes CDN / load balancer / API gateway / reverse proxy → app server → DB/cache.
6. Response with status, headers, body; browser renders, fetches assets (often from CDN), caches per headers.

### 2. OSI vs TCP/IP (brief)
L7 Application (HTTP, gRPC, DNS) · L4 Transport (TCP, UDP) · L3 Network (IP) · L2 Data link · L1 Physical. Load balancers operate at **L4** (TCP connections) or **L7** (HTTP-aware routing by path/header).

### 3. TCP vs UDP
| TCP | UDP |
|-----|-----|
| Connection-oriented, reliable, ordered, flow & congestion control | Connectionless, no guarantees, low overhead |
| HTTP/1.1, HTTP/2, DB protocols | DNS, video/voice streaming, gaming, QUIC (HTTP/3) |

### 4. HTTP versions
| Version | Key features |
|---------|-------------|
| HTTP/1.1 | Persistent connections (keep-alive), one request at a time per connection (head-of-line blocking) |
| HTTP/2 | Binary framing, **multiplexing** many streams over one TCP connection, header compression (HPACK), server push (deprecated) |
| HTTP/3 | Over **QUIC** (UDP): no TCP head-of-line blocking, faster handshakes, connection migration |

### 5. Important headers
`Content-Type`, `Accept`, `Authorization`, `Cache-Control`, `ETag`/`If-None-Match`, `If-Match`, `Location`, `Set-Cookie` (HttpOnly, Secure, SameSite), `Access-Control-Allow-*` (CORS), `X-Forwarded-For`/`Forwarded`, `traceparent`, `Retry-After`, `Idempotency-Key`, `Content-Encoding: gzip/br`.

### 6. HTTPS/TLS essentials
Asymmetric crypto (certificates, CA chain) to authenticate the server and agree on symmetric keys; symmetric encryption for data. **mTLS**: client also presents a certificate (service-to-service). Certificates managed by cert-manager/ACME (Let's Encrypt) or cloud certificate managers.

### 7. API styles compared
| | REST | gRPC | GraphQL | WebSocket | SSE | Webhooks |
|--|--|--|--|--|--|--|
| Transport | HTTP/1.1–2 | HTTP/2 | HTTP | TCP (upgrade from HTTP) | HTTP | HTTP callbacks |
| Format | JSON | Protobuf (binary) | JSON | Any | Text events | JSON |
| Strength | Simple, cacheable, universal | Fast, typed contracts, streaming | Flexible queries, one round trip | Bidirectional real-time | Server→client streaming | Event notification to external systems |
| Use | Public APIs | Internal service calls | BFF for varied clients | Chat, games, live collaboration | Live feeds, notifications | Payment/status callbacks |

**Long polling vs WebSockets vs SSE** — SSE is simplest for one-way updates; WebSockets for bidirectional; long polling as fallback.

### 8. Authentication mechanisms overview
Session cookies, JWT bearer tokens, OAuth2/OIDC, API keys (server-to-server, identify clients), mTLS, HMAC request signing (webhooks: verify signature header), SAML (enterprise SSO), passkeys/WebAuthn.

## Part B — System design toolkit

### 9. How to approach a system design interview
1. **Clarify requirements** — functional (features) and non-functional (scale, latency, availability, consistency, durability, security, cost).
2. **Estimate** — users, QPS (read/write), storage, bandwidth. (1 day ≈ 86,400 s ≈ 10⁵; 1M requests/day ≈ 12 QPS; peak ≈ 2–3× average.)
3. **API design** — main endpoints / events.
4. **Data model** — entities, storage choice, partitioning keys.
5. **High-level design** — boxes: clients, LB, gateway, services, caches, DBs, queues, CDN, object storage.
6. **Deep dives** — bottlenecks, scaling, consistency, failure handling, hot keys.
7. **Trade-offs & extensions** — monitoring, security, cost, evolution.

### 10. Back-of-the-envelope numbers
| Operation | Approx. latency |
|-----------|----------------|
| L1 cache | 1 ns |
| Main memory | 100 ns |
| SSD random read | ~100 µs |
| Read 1 MB sequentially from memory | ~10 µs (fast) |
| Network round trip same datacenter | 0.5 ms |
| Redis GET (same DC) | < 1 ms |
| Simple indexed DB query | 1–10 ms |
| Cross-region round trip | 50–150 ms |
Storage: char ≈ 1 byte (ASCII), UUID 16 bytes (36 as text), timestamp 8 bytes. 1 KB × 1M rows ≈ 1 GB.
Availability: 99.9% ≈ 8.8 h downtime/year; 99.99% ≈ 53 min/year; 99.999% ≈ 5 min/year.

### 11. Building blocks
| Component | Purpose | Examples |
|-----------|---------|----------|
| **DNS** | Name → IP, geo-routing, failover | Route 53, Cloud DNS |
| **CDN** | Cache static/edge content near users | CloudFront, Cloudflare, Akamai |
| **Load balancer** | Distribute traffic, health checks, TLS termination | ALB/NLB, NGINX, HAProxy, Envoy |
| **API gateway** | Auth, rate limit, routing | SCG, Kong, AWS API GW |
| **Stateless app servers** | Horizontal scaling | Spring Boot on K8s |
| **Cache** | Reduce latency/load | Redis, Memcached, Caffeine |
| **Relational DB** | Transactions, relations | Postgres, MySQL |
| **NoSQL** | Scale/flexibility | DynamoDB, Cassandra, MongoDB |
| **Object storage** | Files, images, backups | S3, GCS, Azure Blob |
| **Message queue / stream** | Async, decoupling, buffering | Kafka, RabbitMQ, SQS |
| **Search** | Full-text, faceting | Elasticsearch/OpenSearch |
| **Workers / batch** | Background processing | Consumers, Spring Batch, K8s Jobs |
| **Monitoring** | Metrics, logs, traces | Prometheus, Grafana, ELK |

**Load balancing algorithms**: round robin, weighted round robin, least connections, least response time, IP hash / consistent hashing (sticky).

### 12. Scaling concepts
- **Vertical** (bigger machine) vs **horizontal** (more machines). Horizontal requires **stateless** services (sessions in Redis/JWT, files in object storage).
- **Caching layers**: browser → CDN → gateway → app (local) → distributed cache → DB buffer cache.
- **Database scaling**: indexes → read replicas → caching → partitioning → sharding → CQRS/specialized stores.
- **Consistent hashing**: map keys and nodes onto a ring; adding/removing a node remaps only ~1/N of keys; virtual nodes smooth distribution — used in distributed caches, Cassandra, DynamoDB.
- **Async processing**: queue heavy work (emails, reports, video processing) and respond `202 Accepted`.
- **Rate limiting algorithms**: token bucket (bursty allowed), leaky bucket (smooth), fixed window counter (boundary spikes), sliding window log/counter (accurate). Distributed limits via Redis (`INCR` + TTL or Lua).
- **Hot spots**: hot keys/partitions (celebrity problem) → key salting, replication of hot data, caching.
- **Backpressure & load shedding**: bounded queues, 429/503, prioritize critical traffic.
- **Idempotency** everywhere retries happen.

### 13. Reliability concepts
- Redundancy (N+1), multi-AZ, multi-region (active-passive / active-active).
- **RPO** (max data loss) & **RTO** (max downtime) drive backup/DR strategy.
- Health checks, auto-healing, graceful degradation, circuit breakers, retries with jitter, timeouts.
- **Leader election & consensus** (Raft, Paxos — etcd, ZooKeeper, Kafka KRaft).
- **Distributed locks** (Redis Redlock caveats, ZooKeeper/etcd, DB advisory locks) — use fencing tokens for safety.
- **Unique ID generation**: UUIDv4 (random), UUIDv7 / ULID (time-ordered, index-friendly), Twitter **Snowflake** (timestamp + machine id + sequence), DB sequences/ranges.
- **Clock issues**: don't rely on wall clocks for ordering across machines (use logical clocks/versions).

### 14. Classic design problems — key ideas
**URL shortener**
- API: `POST /urls {longUrl}` → short code; `GET /{code}` → 301/302 redirect.
- Code generation: base62 of a unique id (counter/Snowflake) → 7 chars ≈ 3.5 trillion; or hash + collision check.
- Storage: key-value (code → url), heavy reads → Redis cache + CDN; analytics via async events (Kafka → click counts).
- 301 (cacheable, fewer hits) vs 302 (track every click).

**Rate limiter**
- Token bucket per user/API key in Redis (Lua for atomicity), at gateway; return 429 + `X-RateLimit-*` headers; local + global limits.

**Notification system**
- Producers publish events → notification service → per-channel queues (email/SMS/push) → workers with provider adapters, retries, DLQ; user preferences & templates; idempotency to avoid duplicates; rate limits per user.

**E-commerce order/checkout**
- Services: catalog, cart (Redis), order, payment (idempotency keys, webhooks from gateway), inventory (reservation with TTL, atomic decrement / optimistic locking), shipping, notification.
- Saga for order → payment → inventory; outbox for events; read models for order history; search via Elasticsearch.
- Flash sale: queue/token gate, pre-warmed cache, inventory in Redis with atomic `DECR`, then persist asynchronously.

**Chat application**
- WebSockets via gateway; connection registry (which server holds a user's socket) in Redis; messages persisted (Cassandra: partition by conversation id, cluster by time); pub/sub to fan out across servers; delivery/read receipts; push notifications for offline users.

**News feed**
- Fan-out on write (push to followers' feeds in cache) vs fan-out on read (pull at read time) — hybrid for celebrities. Feed stored as sorted sets of post ids; ranking service.

**File upload service**
- Pre-signed URLs for direct upload to object storage; metadata in DB; virus scan & processing asynchronously (events); CDN for downloads; multipart/chunked uploads for large files.

**Payment system**
- Strong consistency, double-entry ledger, idempotency keys, exactly-once semantics via dedup, reconciliation jobs, PCI-DSS (tokenize cards — never store PAN), webhook signature verification, retry with care (never double charge).

## Part C — Low-level design (LLD) expectations
- Clean OOP: SOLID, design patterns, interfaces, immutability, clear class responsibilities.
- Common LLD questions: parking lot, elevator system, library management, splitwise, BookMyShow (seat locking with TTL), rate limiter class, LRU cache, logger, vending machine (State pattern), snake & ladder, chess.
- Approach: clarify requirements → identify entities & relationships → define interfaces/classes → key flows (sequence) → concurrency concerns (locking seats, atomic counters) → extensibility (Strategy for pricing, Observer for notifications).

```java
// Example: thread-safe token bucket rate limiter (LLD)
public class TokenBucket {
    private final long capacity; private final double refillPerNano;
    private double tokens; private long lastRefill;
    public TokenBucket(long capacity, long refillPerSecond) {
        this.capacity = capacity; this.refillPerNano = refillPerSecond / 1e9;
        this.tokens = capacity; this.lastRefill = System.nanoTime();
    }
    public synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - lastRefill) * refillPerNano);
        lastRefill = now;
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;
    }
}
```

## Part D — Interview Q&A
**Q. How would you scale a Spring Boot service to handle 10× traffic?** Make it stateless, scale horizontally behind a LB (HPA), add caching, tune DB (indexes, pool size, replicas), move heavy work async, add rate limiting, profile hot paths, consider virtual threads, CDN for static content.

**Q. Horizontal vs vertical scaling?** More nodes vs bigger node; horizontal needs statelessness and gives better resilience.

**Q. How do you design for high availability?** Redundancy across zones, health checks & failover, no single points of failure, replication, graceful degradation, tested backups/DR.

**Q. What is consistent hashing?** Hash ring placement minimizing remapping when nodes change.

**Q. How do you prevent duplicate payments?** Idempotency keys stored with results, unique constraints, state machine transitions, dedup on webhook events.

**Q. Explain eventual consistency with an example.** Order service marks order PAID; search index updated seconds later via event — reads may briefly show old state.

**Q. How does a CDN improve performance?** Serves cached content from edge locations close to users, reducing latency and origin load.

**Q. REST vs gRPC?** JSON over HTTP, human-readable, universal vs binary Protobuf over HTTP/2, faster, strongly typed, streaming — great for internal calls.
