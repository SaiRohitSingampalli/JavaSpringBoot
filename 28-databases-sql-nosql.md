# 28 — Databases for Backend Developers: SQL, Indexing, Transactions, NoSQL

## 1. Relational fundamentals
- **Table** (relation), **row** (tuple), **column** (attribute), **schema**.
- **Primary key**: unique, not null row identifier (surrogate `id` vs natural key).
- **Foreign key**: references another table's key → referential integrity (`ON DELETE CASCADE / SET NULL / RESTRICT`).
- **Constraints**: `NOT NULL`, `UNIQUE`, `CHECK (price > 0)`, `DEFAULT`, FK.
- Popular RDBMS: **PostgreSQL** (feature-rich, default choice for many), MySQL/MariaDB, Oracle, SQL Server; cloud: Aurora, Cloud SQL, AlloyDB, Azure SQL.

## 2. SQL you must be fluent in
```sql
-- DDL
CREATE TABLE customer (
  id          BIGSERIAL PRIMARY KEY,
  email       VARCHAR(255) NOT NULL UNIQUE,
  name        VARCHAR(100) NOT NULL,
  city        VARCHAR(100),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE orders (
  id           BIGSERIAL PRIMARY KEY,
  customer_id  BIGINT NOT NULL REFERENCES customer(id),
  status       VARCHAR(20) NOT NULL CHECK (status IN ('NEW','PAID','SHIPPED','CANCELLED')),
  total        NUMERIC(12,2) NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_customer_created ON orders(customer_id, created_at DESC);
ALTER TABLE orders ADD COLUMN coupon_code VARCHAR(30);

-- DML
INSERT INTO customer(email, name) VALUES ('a@x.com', 'Asha') RETURNING id;
UPDATE orders SET status = 'PAID' WHERE id = 10 AND status = 'NEW';
DELETE FROM orders WHERE created_at < now() - interval '2 years';
INSERT INTO stock(sku, qty) VALUES ('P1', 5)
  ON CONFLICT (sku) DO UPDATE SET qty = stock.qty + EXCLUDED.qty;        -- upsert (Postgres)

-- Querying
SELECT c.name, COUNT(o.id) AS orders, SUM(o.total) AS revenue
FROM customer c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status <> 'CANCELLED'
WHERE c.created_at >= DATE '2026-01-01'
GROUP BY c.name
HAVING SUM(o.total) > 1000
ORDER BY revenue DESC
LIMIT 10 OFFSET 0;
```
**Logical order of execution**: `FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET`. (That's why you can't use a SELECT alias in WHERE.)

### Joins
| Join | Returns |
|------|---------|
| INNER | Matching rows in both |
| LEFT (OUTER) | All left rows + matches (NULLs if none) |
| RIGHT | All right rows + matches |
| FULL OUTER | All rows from both |
| CROSS | Cartesian product |
| SELF | Table joined with itself (employee–manager) |
| Anti-join | `LEFT JOIN ... WHERE right.id IS NULL` or `NOT EXISTS` |
| Semi-join | `EXISTS` / `IN` |

### Subqueries, CTEs, window functions
```sql
-- Correlated subquery
SELECT * FROM employee e WHERE salary > (SELECT AVG(salary) FROM employee WHERE dept_id = e.dept_id);

-- CTE
WITH monthly AS (
  SELECT date_trunc('month', created_at) AS month, SUM(total) AS revenue
  FROM orders GROUP BY 1
)
SELECT month, revenue, revenue - LAG(revenue) OVER (ORDER BY month) AS growth FROM monthly;

-- Window functions: ranking within groups
SELECT * FROM (
  SELECT e.*, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
  FROM employee e
) t WHERE rnk <= 3;                      -- top 3 salaries per department

-- Running total
SELECT id, total, SUM(total) OVER (PARTITION BY customer_id ORDER BY created_at) AS running FROM orders;

-- Recursive CTE (hierarchy)
WITH RECURSIVE org AS (
  SELECT id, name, manager_id, 1 AS level FROM employee WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, org.level + 1 FROM employee e JOIN org ON e.manager_id = org.id
)
SELECT * FROM org;
```
`ROW_NUMBER` (unique 1,2,3), `RANK` (1,1,3), `DENSE_RANK` (1,1,2), `LAG/LEAD`, `NTILE`, `FIRST_VALUE`.

### Classic interview queries
```sql
-- 2nd highest salary
SELECT MAX(salary) FROM employee WHERE salary < (SELECT MAX(salary) FROM employee);
SELECT DISTINCT salary FROM employee ORDER BY salary DESC OFFSET 1 LIMIT 1;

-- Duplicate emails
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Delete duplicates keeping lowest id (Postgres)
DELETE FROM users a USING users b WHERE a.email = b.email AND a.id > b.id;

-- Customers with no orders
SELECT c.* FROM customer c WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- Employees earning more than their managers
SELECT e.name FROM employee e JOIN employee m ON e.manager_id = m.id WHERE e.salary > m.salary;
```
`UNION` (dedup) vs `UNION ALL` (keeps duplicates, faster). `WHERE` filters rows; `HAVING` filters groups. `COUNT(*)` counts rows; `COUNT(col)` skips NULLs. `NULL = NULL` is unknown → use `IS NULL`; `NOT IN` with a NULL in the list returns nothing — prefer `NOT EXISTS`.

## 3. Normalization
| Form | Rule |
|------|------|
| 1NF | Atomic values, no repeating groups |
| 2NF | 1NF + every non-key column depends on the **whole** composite key |
| 3NF | 2NF + no transitive dependency (non-key depends only on the key) |
| BCNF | Every determinant is a candidate key |
Normalize to avoid update anomalies; **denormalize deliberately** for read performance (reporting tables, cached counts, JSON columns, read models).

## 4. Indexes ⭐
### 4.1 B-tree (default)
Balanced tree keeping keys sorted → O(log n) lookups, range scans, ORDER BY support. Leaf entries point to rows (heap tuple id in Postgres; clustered PK in InnoDB).

### 4.2 Types
| Index | Use |
|-------|-----|
| B-tree | Equality, ranges, sorting — default |
| Hash | Equality only |
| **Composite** (multi-column) | `(customer_id, created_at)` — order matters |
| **Unique** | Enforce uniqueness |
| **Covering** (`INCLUDE (total)`) | Index-only scans |
| **Partial** (`WHERE status = 'NEW'`) | Index subset of rows |
| Expression (`lower(email)`) | Functions in WHERE |
| GIN | JSONB, arrays, full-text search |
| GiST / BRIN | Geo, ranges / huge append-only tables (time series) |
| Clustered (InnoDB PK, SQL Server) | Table stored in index order |

### 4.3 Composite index & leftmost prefix rule
Index `(a, b, c)` helps queries filtering `a`, `a,b`, `a,b,c`; not `b` alone or `c` alone. Put equality columns first, then range/sort columns.

### 4.4 When indexes aren't used
- Functions on the column: `WHERE lower(email) = ?` (needs expression index), `WHERE date(created_at) = ?`.
- Leading wildcard: `LIKE '%abc'`.
- Implicit type casts, `OR` across different columns (sometimes), low selectivity (`status` with 3 values on huge table — planner may prefer seq scan), stale statistics.

### 4.5 Costs
Indexes speed reads but slow writes (every insert/update maintains them) and use storage. Index FK columns (Postgres doesn't automatically), columns in frequent WHERE/JOIN/ORDER BY; drop unused indexes.

### 4.6 EXPLAIN
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
```
Look for: `Seq Scan` on big tables, `Index Scan`/`Index Only Scan`, `Bitmap Heap Scan`, join types (Nested Loop / Hash Join / Merge Join), estimated vs actual rows (bad estimates → `ANALYZE`), sorts spilling to disk.

## 5. Transactions, isolation & locking (DB side)
- ACID; isolation levels & anomalies (see Spring transactions notes).
- **MVCC** (Postgres, Oracle, InnoDB): readers see a snapshot; writers create new row versions → readers don't block writers. Postgres needs **VACUUM** to clean dead tuples (autovacuum).
- Row locks: `SELECT ... FOR UPDATE` (exclusive), `FOR SHARE`; `SKIP LOCKED` → job queues in SQL; `NOWAIT`.
- **Deadlocks**: DB detects and aborts one transaction → app should retry; prevent by locking rows in consistent order and keeping transactions short.
- Lost update prevention: optimistic version column, `SELECT FOR UPDATE`, atomic `UPDATE ... SET qty = qty - 1 WHERE qty > 0`.
- Long-running transactions hold locks/snapshots → bloat & contention.

## 6. Scaling relational databases
| Technique | Notes |
|-----------|-------|
| Query & index tuning | First and cheapest |
| Connection pooling | HikariCP; PgBouncer in front of Postgres |
| Caching | Redis/Caffeine in front |
| **Read replicas** | Async replication; route reads; beware **replication lag** (read-your-writes issues) |
| Vertical scaling | Bigger machine |
| **Partitioning** | Split a big table by range/list/hash inside one DB (e.g., orders by month) — pruning & easier archiving |
| **Sharding** | Split data across multiple DBs by shard key (customer_id) — cross-shard queries/transactions are hard; consider Citus, Vitess, CockroachDB/YugabyteDB |
| Archiving | Move cold data out |
| CQRS read models / search engines | Elasticsearch/OpenSearch for search |

## 7. Database migrations
- Version-controlled (Flyway `V1__...sql`, Liquibase changelogs), applied automatically at startup or in CI/CD.
- Zero-downtime **expand/contract**: add new nullable column → deploy code writing both → backfill → switch reads → remove old column in a later release. Never rename/drop columns in the same deploy that stops using them.
- Large table changes: create indexes `CONCURRENTLY` (Postgres), batch backfills.

## 8. NoSQL families
| Type | Examples | Data model | Good for |
|------|----------|-----------|----------|
| **Key-value** | Redis, DynamoDB, Memcached | key → value | Caching, sessions, counters, rate limits |
| **Document** | MongoDB, Couchbase, Firestore | JSON/BSON documents | Flexible schemas, catalogs, content, aggregates read together |
| **Wide-column** | Cassandra, ScyllaDB, HBase, Bigtable | Rows with dynamic columns, partition key + clustering key | Massive write throughput, time series, IoT, messaging timelines |
| **Graph** | Neo4j, Neptune | Nodes + relationships | Social networks, recommendations, fraud detection |
| **Search** | Elasticsearch, OpenSearch, Solr | Inverted index | Full-text search, log analytics |
| **Time series** | InfluxDB, TimescaleDB, Prometheus | Timestamped metrics | Monitoring, IoT |
| **Vector** | pgvector, Pinecone, Weaviate, Milvus | Embeddings | Semantic search, RAG/AI features |

### SQL vs NoSQL — choosing
| Choose relational when | Choose NoSQL when |
|------------------------|-------------------|
| Complex relations & joins, ad-hoc queries | Access patterns known & simple (by key) |
| Strong consistency & multi-row transactions | Massive scale / write throughput beyond one node |
| Evolving reporting needs | Flexible/varied schema per record |
| Most business apps (default!) | Specific use cases: cache, search, graph, time series |
Postgres also handles JSONB, full-text search, geo (PostGIS) and vectors (pgvector) — often enough without adding another store.

### MongoDB essentials (Spring Data MongoDB)
```java
@Document("products")
public record Product(@Id String id, String name, BigDecimal price, List<String> tags, Map<String, Object> attributes) {}
public interface ProductRepo extends MongoRepository<Product, String> { List<Product> findByTagsContaining(String tag); }
mongoTemplate.find(Query.query(Criteria.where("price").lt(100)).with(Sort.by("price")), Product.class);
```
Model by **access patterns**: embed data read together (order with items), reference when unbounded or shared. Indexes matter just as much. Multi-document transactions exist (replica sets) but design to avoid them.

### Cassandra essentials
Query-first modelling: one table per query; partition key determines node placement; clustering columns sort within partition; no joins; tunable consistency (`ONE`, `QUORUM`, `ALL`); writes are extremely fast (LSM trees).

## 9. CAP & PACELC
- **CAP**: in a network **Partition**, a distributed system must choose **Consistency** (reject/stall requests) or **Availability** (serve possibly stale data). CP examples: ZooKeeper/etcd, HBase, MongoDB (default majority writes); AP examples: Cassandra, DynamoDB (tunable), Eureka.
- **PACELC**: if Partition → A or C; Else (normal operation) → trade-off between **Latency** and **Consistency**.
- **Consistency models**: strong (linearizable), read-your-writes, monotonic reads, causal, eventual.
- **BASE**: Basically Available, Soft state, Eventually consistent (vs ACID).

## 10. Redis data structures (quick reference)
| Structure | Commands | Use |
|-----------|----------|-----|
| String | `GET/SET/INCR/EXPIRE/SETNX` | Cache, counters, locks |
| Hash | `HSET/HGET/HGETALL` | Objects |
| List | `LPUSH/RPOP/BLPOP` | Queues |
| Set | `SADD/SISMEMBER/SINTER` | Unique items, tags |
| Sorted Set | `ZADD/ZRANGE/ZREVRANK` | Leaderboards, delayed jobs, rate limiting windows |
| Stream | `XADD/XREADGROUP` | Lightweight event log with consumer groups |
| HyperLogLog, Bitmap, Geo | | Unique counts, flags, location |

## 11. Interview Q&A
**Q. What is an index and how does a B-tree index work?** Sorted balanced tree structure mapping keys to row locations; lookups and ranges in O(log n).

**Q. Clustered vs non-clustered index?** Clustered determines physical row order (one per table, e.g., InnoDB PK); non-clustered is a separate structure pointing to rows.

**Q. Why might a query not use an index?** Functions on columns, leading wildcard, low selectivity, type mismatch, outdated statistics, wrong column order in composite index.

**Q. Explain normalization and when to denormalize.** Remove redundancy to avoid anomalies (1NF–3NF); denormalize for read performance with controlled duplication.

**Q. WHERE vs HAVING?** Row filter before grouping vs group filter after aggregation.

**Q. INNER vs LEFT JOIN?** Only matches vs all left rows with optional matches.

**Q. How do you find the Nth highest salary?** `DENSE_RANK()` window function or `ORDER BY ... OFFSET n-1 LIMIT 1` on distinct values.

**Q. What is a deadlock and how to avoid it?** Circular wait between transactions holding locks; consistent lock order, short transactions, retries.

**Q. SQL vs NoSQL?** (Table above.) Default to relational; use NoSQL for specific access patterns or scale.

**Q. Explain CAP theorem.** During a partition you choose consistency or availability; in practice it's a spectrum (tunable consistency).

**Q. Replication vs sharding?** Replication copies the same data (availability, read scaling); sharding splits data across nodes (write/storage scaling).

**Q. What is replication lag and how does it affect apps?** Replicas behind primary → stale reads; route read-after-write to primary or use session consistency.
