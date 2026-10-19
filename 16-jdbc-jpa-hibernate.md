# 16 — Data Access: JDBC, JPA & Hibernate In Depth

## 1. The data access stack
```
Your code
  ├── Spring Data JPA repositories  (interfaces → generated queries)
  │        │
  │        ▼
  ├── JPA API (jakarta.persistence: EntityManager, @Entity, JPQL)   ← specification
  │        │
  │        ▼
  ├── Hibernate ORM 6.x  (JPA implementation)  [alternatives: EclipseLink]
  │        │
  ├── Spring JDBC (JdbcTemplate / JdbcClient)  ← simpler, no ORM
  │        │
  ▼        ▼
JDBC API (java.sql: Connection, PreparedStatement, ResultSet)
  │
Connection pool (HikariCP — Boot default)
  │
JDBC driver (PostgreSQL, MySQL, Oracle...) → Database
```

## 2. Plain JDBC (understand what everything else hides)
```java
String sql = "SELECT id, name, email FROM users WHERE status = ?";
try (Connection con = dataSource.getConnection();
     PreparedStatement ps = con.prepareStatement(sql)) {
    ps.setString(1, "ACTIVE");
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            users.add(new User(rs.getLong("id"), rs.getString("name"), rs.getString("email")));
        }
    }
} catch (SQLException e) { throw new DataAccessResourceFailureException("query failed", e); }

// Transactions
con.setAutoCommit(false);
try { ...; con.commit(); } catch (SQLException e) { con.rollback(); }
```
**Always `PreparedStatement`** — prevents SQL injection (parameters are sent separately from SQL text) and allows statement caching. Never concatenate user input into SQL.
Batching: `ps.addBatch()` … `ps.executeBatch()`.

## 3. Connection pooling — HikariCP
Opening a DB connection takes ~ms (TCP + TLS + auth). Pools keep connections open and lend them out.
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: shop
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10          # default 10
      minimum-idle: 10               # Hikari recommends fixed-size pool
      connection-timeout: 30000      # ms waiting for a connection before exception
      idle-timeout: 600000
      max-lifetime: 1800000          # < DB/firewall timeout
      leak-detection-threshold: 20000
      pool-name: shop-pool
```
**Sizing**: bigger is not better. A starting formula: `connections = (core_count × 2) + effective_spindle_count`. Total connections across all service instances must stay below the DB's `max_connections`. Symptoms of exhaustion: `SQLTransientConnectionException: Connection is not available, request timed out` → long transactions, slow queries, leaks, or N+1 storms. With virtual threads, pool size becomes the real concurrency limit.

## 4. Spring JDBC
Removes boilerplate (open/close, exception translation to `DataAccessException`, transaction participation).

### JdbcTemplate
```java
@Repository
@RequiredArgsConstructor
public class ProductJdbcRepository {
    private final JdbcTemplate jdbc;
    private final NamedParameterJdbcTemplate named;

    private static final RowMapper<Product> MAPPER = (rs, rowNum) ->
        new Product(rs.getLong("id"), rs.getString("name"), rs.getBigDecimal("price"));

    public List<Product> findAll() { return jdbc.query("SELECT * FROM product", MAPPER); }

    public Optional<Product> findById(long id) {
        return jdbc.query("SELECT * FROM product WHERE id = ?", MAPPER, id).stream().findFirst();
    }

    public int count() { return jdbc.queryForObject("SELECT COUNT(*) FROM product", Integer.class); }

    public long insert(Product p) {
        KeyHolder kh = new GeneratedKeyHolder();
        named.update("INSERT INTO product(name, price) VALUES (:name, :price)",
            new MapSqlParameterSource().addValue("name", p.name()).addValue("price", p.price()), kh, new String[]{"id"});
        return kh.getKey().longValue();
    }

    public void batchUpdatePrices(List<Product> products) {
        jdbc.batchUpdate("UPDATE product SET price = ? WHERE id = ?", products, 500,
            (ps, p) -> { ps.setBigDecimal(1, p.price()); ps.setLong(2, p.id()); });
    }
}
```

### JdbcClient (Spring 6.1 / Boot 3.2) — modern fluent API
```java
@Repository
@RequiredArgsConstructor
class CourseRepository {
    private final JdbcClient jdbc;

    List<Course> findByAuthor(String author) {
        return jdbc.sql("SELECT id, name, author FROM course WHERE author = :author")
                   .param("author", author)
                   .query(Course.class)            // maps columns to record components / bean properties
                   .list();
    }
    Optional<Course> findById(long id) {
        return jdbc.sql("SELECT * FROM course WHERE id = ?").param(id).query(Course.class).optional();
    }
    int insert(Course c) {
        return jdbc.sql("INSERT INTO course(id, name, author) VALUES (:id, :name, :author)")
                   .paramSource(c).update();       // binds record/bean properties by name
    }
}
```
**When to use JDBC over JPA**: reporting/analytics queries, bulk operations, complex SQL (window functions, CTEs), performance-critical paths, simple services without rich domain models. Also consider **jOOQ** (type-safe SQL DSL) and Spring Data JDBC (simpler aggregate-oriented ORM, no lazy loading/caching).

## 5. ORM, JPA and Hibernate
**Object-Relational Mapping** bridges the **impedance mismatch** between objects (inheritance, references, collections, identity) and tables (rows, foreign keys, joins).

- **JPA** (Jakarta Persistence 3.1/3.2): specification — annotations, `EntityManager`, JPQL, Criteria API.
- **Hibernate 6.x**: implementation (Boot 3 default) + extra features (`@Formula`, `@BatchSize`, filters, `@NaturalId`, statistics).
- Code against JPA APIs; use Hibernate-specific features consciously.

## 6. Entity mapping
```java
@Entity
@Table(name = "orders",
       indexes = @Index(name = "idx_orders_customer", columnList = "customer_id"),
       uniqueConstraints = @UniqueConstraint(columnNames = "order_number"))
@Getter @Setter @NoArgsConstructor(access = AccessLevel.PROTECTED)     // JPA needs a no-arg constructor
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
    private Long id;

    @Column(name = "order_number", nullable = false, length = 30, updatable = false)
    private String orderNumber;

    @Enumerated(EnumType.STRING)                 // NEVER ORDINAL
    @Column(nullable = false, length = 20)
    private OrderStatus status = OrderStatus.NEW;

    @Column(precision = 12, scale = 2)
    private BigDecimal total;

    @Embedded
    private Address shippingAddress;             // value object stored in same table

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Version                                      // optimistic locking
    private Long version;

    @CreationTimestamp private Instant createdAt; // Hibernate; or Spring Data @CreatedDate auditing
    @UpdateTimestamp   private Instant updatedAt;

    @Transient private BigDecimal tempDiscount;    // not persisted

    @Lob private String notes;                     // CLOB/TEXT

    @Convert(converter = MoneyConverter.class)     // custom type conversion
    private Money shippingFee;

    // Helper methods keep both sides of a bidirectional association in sync
    public void addItem(OrderItem item) { items.add(item); item.setOrder(this); }
    public void removeItem(OrderItem item) { items.remove(item); item.setOrder(null); }
}

@Embeddable
public record Address(String street, String city, String zip) {}   // records allowed as embeddables (Hibernate 6.2+)
```

### Primary key generation strategies
| Strategy | How | Notes |
|----------|-----|-------|
| `IDENTITY` | DB auto-increment column | Simple; **disables JDBC insert batching** in Hibernate (needs id immediately) |
| `SEQUENCE` | DB sequence; `allocationSize` lets Hibernate fetch ids in blocks (pooled optimizer) | **Best performance** on Postgres/Oracle |
| `TABLE` | Simulated sequence in a table | Slow, avoid |
| `AUTO` | Provider chooses | Hibernate 6: SEQUENCE where supported |
| `UUID` (`@GeneratedValue` + `UUID` type / `@UuidGenerator`) | App-generated | Good for distributed systems; prefer time-ordered UUIDv7 for index locality |

Composite keys: `@EmbeddedId` or `@IdClass`.
Natural ids: `@NaturalId` (Hibernate) for business keys like ISBN.

## 7. Relationships
| Mapping | Default fetch | Owning side | FK location |
|---------|--------------|-------------|------------|
| `@ManyToOne` | **EAGER** ⚠ | Always the owner | This table |
| `@OneToMany` | LAZY | Inverse if `mappedBy` | Child table |
| `@OneToOne` | **EAGER** ⚠ | Side with `@JoinColumn` | Owner's table |
| `@ManyToMany` | LAZY | Side without `mappedBy` | Join table |

**Rule: make everything LAZY** (`@ManyToOne(fetch = LAZY)`, `@OneToOne(fetch = LAZY)`) and fetch what each use case needs explicitly.

### Owning vs inverse side
- The **owning side** controls the FK column; only changes on the owning side are persisted.
- `mappedBy` marks the **inverse** side — "the FK is managed by the field `order` on the other entity".
- In bidirectional one-to-many, the `@ManyToOne` child side is the owner. Always update both sides (helper methods).

### OneToMany — unidirectional pitfall
```java
@OneToMany private List<Item> items;   // without @JoinColumn → Hibernate creates a JOIN TABLE + extra statements
@OneToMany @JoinColumn(name = "order_id") private List<Item> items;   // better if unidirectional
```
Best: bidirectional with `mappedBy`, or just a `@ManyToOne` from child and query children when needed.

### ManyToMany
```java
@ManyToMany
@JoinTable(name = "course_student",
    joinColumns = @JoinColumn(name = "course_id"),
    inverseJoinColumns = @JoinColumn(name = "student_id"))
private Set<Student> students = new HashSet<>();     // Set, not List (List → delete-all-reinsert on changes)
```
If the link has extra columns (enrolled_at, grade) → model it as its own entity `Enrollment` with two `@ManyToOne`.

### Cascade types
`PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL`. Cascade from **parent to child** in a composition (Order → OrderItems). Never cascade `REMOVE` on `@ManyToOne`/`@ManyToMany` (deleting an order would delete the customer!).
`orphanRemoval = true` → removing a child from the collection deletes the row.

### Inheritance strategies
| Strategy | Tables | Pros | Cons |
|----------|--------|------|------|
| `SINGLE_TABLE` (default) | One table + discriminator column | Fast, polymorphic queries simple | Many nullable columns |
| `JOINED` | Parent table + table per subclass | Normalized | Joins for every query |
| `TABLE_PER_CLASS` | Table per concrete class | No joins for concrete queries | Polymorphic queries use UNION |
| `@MappedSuperclass` | No table for parent; fields copied | Share common columns (id, audit fields) | Not queryable polymorphically |

## 8. Persistence context & entity lifecycle

The **persistence context** = first-level cache + unit of work, bound to a transaction (in Spring, one `EntityManager` per transaction).

```
         new Order()                  em.persist()
TRANSIENT ─────────────────────────────────► MANAGED ◄──── em.find() / query
                                              │  ▲
                         em.detach()/clear()  │  │ em.merge() returns a managed copy
                         tx/context ends      ▼  │
                                             DETACHED
                                              
MANAGED ──em.remove()──► REMOVED ──flush/commit──► DELETE
```
| State | Tracked? | In DB? |
|-------|----------|--------|
| Transient | No | No |
| Managed (persistent) | Yes — changes auto-detected | Yes (after flush) |
| Detached | No | Yes |
| Removed | Scheduled for delete | Until flush |

### Key behaviours
- **Identity guarantee**: within one persistence context, `em.find(Order.class, 1)` twice returns the **same object** (first-level cache) — no second SELECT.
- **Dirty checking**: at flush, Hibernate compares managed entities with their loaded snapshot and issues UPDATEs automatically. You don't need `save()` for managed entities inside a transaction:
  ```java
  @Transactional
  public void rename(Long id, String name) {
      Product p = repo.findById(id).orElseThrow();
      p.setName(name);            // UPDATE issued at commit — no save() needed
  }
  ```
- **Write-behind**: SQL is queued and sent at **flush** — before commit, before a JPQL query that touches affected tables (`FlushMode.AUTO`), or on `em.flush()`.
- `persist` vs `merge`: persist makes a new instance managed; merge copies state of a detached object onto a managed one and returns it (the argument stays detached). Spring Data `save()` calls `persist` if the entity is new (id null / version null), else `merge`.
- `getReference()` / `repo.getReferenceById()` returns an uninitialized proxy — useful to set a FK without loading the row.

### EntityManager API
```java
em.persist(e); em.merge(e); em.remove(e); em.find(Order.class, id);
em.getReference(Order.class, id); em.flush(); em.clear(); em.detach(e); em.refresh(e);
em.createQuery("select o from Order o where o.status = :s", Order.class).setParameter("s", NEW).getResultList();
em.createNativeQuery("select * from orders", Order.class);
em.lock(order, LockModeType.PESSIMISTIC_WRITE);
```

## 9. Fetching strategies & the N+1 problem ⭐

```java
List<Order> orders = orderRepo.findAll();            // 1 query
for (Order o : orders) {
    o.getCustomer().getName();                       // +1 query per order (lazy proxy init) → N+1
}
```
With EAGER it's no better — Hibernate still issues extra selects for each association after a JPQL query.

**Detect**: enable SQL logging (`org.hibernate.SQL=DEBUG`), Hibernate statistics (`spring.jpa.properties.hibernate.generate_statistics=true`), tests asserting query counts (datasource-proxy, Hypersistence Utils), APM tools.

**Fixes**
```java
// 1. JOIN FETCH in JPQL
@Query("select o from Order o join fetch o.customer where o.status = :status")
List<Order> findWithCustomer(@Param("status") OrderStatus status);

// 2. Entity graph
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus status);

// 3. Batch fetching — load lazy associations for many parents in one IN (...) query
spring.jpa.properties.hibernate.default_batch_fetch_size: 50
// or @BatchSize(size = 50) on the collection/class

// 4. DTO projection — select exactly the columns needed (often the best for reads)
@Query("select new com.shop.dto.OrderSummary(o.id, o.total, c.name) from Order o join o.customer c")
List<OrderSummary> summaries();

// 5. @Fetch(FetchMode.SUBSELECT) for collections
```
**Pitfalls**: `JOIN FETCH` of a collection + pagination → Hibernate paginates **in memory** (warning `HHH90003004` / `firstResult/maxResults specified with collection fetch`) — fetch ids page first, then fetch entities with collections by ids. Fetching two `List` collections at once → `MultipleBagFetchException` (use `Set` or separate queries). Hibernate 6 de-duplicates parent rows automatically (no need for `distinct`).

### LazyInitializationException
`could not initialize proxy - no Session` → accessing a lazy association after the transaction/persistence context closed (e.g., in the controller or during JSON serialization).
Fixes: fetch what you need inside the transactional service method (join fetch / entity graph), map to DTOs within the transaction. **Don't** "fix" with EAGER or `hibernate.enable_lazy_load_no_trans`.

**Open Session in View (OSIV)**: Spring Boot enables `spring.jpa.open-in-view=true` by default (and logs a warning) — keeps the EntityManager open for the whole web request so lazy loading works in controllers/views. Hides N+1 problems and holds DB connections longer → **set it to false** in REST APIs.

## 10. Queries
### JPQL
Object-oriented query language over entities, not tables:
```sql
select o from Order o where o.customer.email = :email and o.total > :min order by o.createdAt desc
select c.city, count(c) from Customer c group by c.city having count(c) > 10
update Product p set p.price = p.price * 1.1 where p.category = :cat      -- bulk (bypasses persistence context!)
```
### Criteria API — type-safe dynamic queries (verbose); Spring Data **Specifications** wrap it.
### Native SQL — `@Query(nativeQuery = true)`; for DB-specific features.

### Projections
- Entity: managed, tracked (overhead) — use when you will modify.
- DTO (constructor expression, interface/record projections in Spring Data): read-only, fast — **use for read APIs**.
- Tuple / `Object[]`.

## 11. Locking & concurrency control
### Optimistic locking (`@Version`) — default choice
```java
@Version private Long version;
```
`UPDATE product SET name=?, version=6 WHERE id=? AND version=5` — if 0 rows updated → `OptimisticLockException` → Spring `ObjectOptimisticLockingFailureException` → return 409 or retry. No DB locks held; great for low-contention web apps ("lost update" prevention).

### Pessimistic locking
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)                 // SELECT ... FOR UPDATE
@Query("select a from Account a where a.id = :id")
Optional<Account> findForUpdate(@Param("id") Long id);

@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
```
Use for high contention critical sections (inventory decrement, balance transfer). Lock rows in a consistent order to avoid deadlocks. Alternative: atomic SQL `UPDATE stock SET qty = qty - :n WHERE sku = :sku AND qty >= :n`.

## 12. Caching
- **First-level cache**: persistence context, per transaction, always on.
- **Second-level cache (L2)**: per `EntityManagerFactory`, shared across transactions/sessions; opt-in (`@Cacheable` JPA annotation + `@Cache(usage = READ_WRITE)`, provider like Ehcache/Infinispan/Caffeine via JCache). For read-mostly reference data. In multi-instance deployments you need a distributed/invalidating cache.
- **Query cache**: caches query result ids; rarely worth it.

## 13. Performance checklist for JPA
- [ ] All associations LAZY; fetch per use case
- [ ] Watch for N+1 (logs/tests)
- [ ] `open-in-view: false`
- [ ] DTO projections for read endpoints
- [ ] SEQUENCE ids with `allocationSize` + JDBC batching:
  ```yaml
  spring.jpa.properties.hibernate:
    jdbc.batch_size: 50
    order_inserts: true
    order_updates: true
  ```
- [ ] Bulk updates via JPQL/SQL, not loading thousands of entities
- [ ] For large batch processing: `em.flush(); em.clear();` every N records, or `StatelessSession`, or stream results
- [ ] Pagination for lists; `Slice` when you don't need total count (avoids `COUNT(*)`)
- [ ] Indexes on FK and filter columns (Hibernate doesn't create FK indexes for you on Postgres)
- [ ] `@Transactional(readOnly = true)` for queries (Hibernate skips dirty checking; can route to replicas)
- [ ] Avoid `ddl-auto=update` in production — use **Flyway/Liquibase**

## 14. Schema management
```yaml
spring.jpa.hibernate.ddl-auto: validate     # none | validate | update | create | create-drop
```
| Value | Use |
|-------|-----|
| create-drop | Tests / demos (default for embedded H2) |
| create / update | Local prototyping only |
| validate | Production with migrations — fail fast if mapping ≠ schema |
| none | Production |

**Flyway**: `src/main/resources/db/migration/V1__create_orders.sql`, `V2__add_status_index.sql` — applied in order, tracked in `flyway_schema_history`, immutable once applied. **Liquibase**: changelogs in YAML/XML/SQL with rollback support. Write backward-compatible migrations for zero-downtime deploys (expand → migrate → contract).

`schema.sql` / `data.sql` (Spring SQL init) for simple cases: `spring.sql.init.mode=always`; with JPA set `spring.jpa.defer-datasource-initialization=true` to run data.sql after Hibernate creates tables (H2 demo apps in courses).

## 15. Auditing
```java
@EnableJpaAuditing
@Configuration class JpaConfig {
    @Bean AuditorAware<String> auditor() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication()).map(Authentication::getName);
    }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate @Column(updatable = false) private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy @Column(updatable = false) private String createdBy;
    @LastModifiedBy private String updatedBy;
}
```
Full history: Hibernate **Envers** (`@Audited`) keeps revision tables.

JPA lifecycle callbacks: `@PrePersist`, `@PostPersist`, `@PreUpdate`, `@PostUpdate`, `@PreRemove`, `@PostRemove`, `@PostLoad`.

## 16. Soft delete
```java
@SQLDelete(sql = "UPDATE product SET deleted = true WHERE id = ? AND version = ?")
@SQLRestriction("deleted = false")          // Hibernate 6.3+ (replaces @Where)
// Hibernate 6.4+: @SoftDelete annotation
```

## 17. equals/hashCode for entities
- Don't use Lombok `@Data`/`@EqualsAndHashCode` on entities (includes lazy collections → loads them, recursion, broken when id changes).
- Don't put collections in `toString()` (`@ToString.Exclude`).
- Pattern:
```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Order other)) return false;
    return id != null && id.equals(other.getId());
}
@Override public int hashCode() { return getClass().hashCode(); }   // constant: stable across persist
```
Or use an immutable natural/business key.

## 18. H2 for learning (Ranga-style setup)
```yaml
spring:
  datasource.url: jdbc:h2:mem:testdb
  h2.console.enabled: true          # /h2-console
  jpa:
    show-sql: true
    defer-datasource-initialization: true
```
Use Testcontainers with the real DB engine for integration tests — H2's SQL dialect differs.

## 19. Interview Q&A
**Q. JPA vs Hibernate vs Spring Data JPA?** JPA = spec; Hibernate = implementation; Spring Data JPA = repository abstraction generating JPA code.

**Q. What is the persistence context?** A set of managed entity instances (first-level cache + unit of work) tied to an EntityManager/transaction, guaranteeing identity and enabling dirty checking.

**Q. Entity states?** Transient, managed, detached, removed.

**Q. `persist` vs `merge`? `save` in Spring Data?** persist makes a new object managed; merge copies a detached object's state into a managed instance and returns it. `save` chooses based on `isNew`.

**Q. What is the N+1 problem and how do you fix it?** 1 query for parents + N for lazy children. Fix: join fetch, entity graphs, batch fetching, DTO projections.

**Q. FetchType LAZY vs EAGER? Defaults?** ToOne = EAGER, ToMany = LAZY. Prefer LAZY everywhere.

**Q. What causes LazyInitializationException?** Accessing an uninitialized proxy after the session closes. Fetch inside the transaction or use DTOs.

**Q. Optimistic vs pessimistic locking?** Optimistic: `@Version` check at update, no locks, fails on conflict. Pessimistic: DB row locks (`SELECT FOR UPDATE`) blocking others.

**Q. Why is `IDENTITY` bad for batching?** Hibernate must execute each insert immediately to obtain the generated id.

**Q. What is dirty checking?** At flush Hibernate compares managed entities to snapshots and generates UPDATEs automatically.

**Q. What is OSIV and why disable it?** Keeps the session open during view rendering; hides N+1 and holds connections longer.

**Q. `ddl-auto` in production?** `validate` or `none` with Flyway/Liquibase migrations.

**Q. How do you handle bulk inserts efficiently?** SEQUENCE ids + `jdbc.batch_size` + `order_inserts`, flush/clear in chunks, or JdbcTemplate batchUpdate / COPY.
