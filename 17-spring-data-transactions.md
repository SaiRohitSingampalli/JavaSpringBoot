# 17 — Spring Data JPA & Transaction Management

## Part A — Spring Data JPA

### 1. Repository hierarchy
```
Repository<T, ID>                          marker
 └─ CrudRepository<T, ID>                  save, saveAll, findById, existsById, findAll, count, deleteById, delete...
     └─ ListCrudRepository                 same but returns List instead of Iterable (Spring Data 3)
 └─ PagingAndSortingRepository<T, ID>      findAll(Sort), findAll(Pageable)  (no longer extends Crud in v3)
     └─ ListPagingAndSortingRepository
 └─ JpaRepository<T, ID>                   flush, saveAndFlush, saveAllAndFlush, deleteAllInBatch,
                                           getReferenceById, findAll(Example)...
 + JpaSpecificationExecutor<T>             findAll(Specification), count(spec)...
 + QueryByExampleExecutor<T>
```
At startup Spring creates a **proxy** implementing your interface, backed by `SimpleJpaRepository`. Methods are `@Transactional` by default (readOnly for finders).

```java
public interface ProductRepository extends JpaRepository<Product, Long>, JpaSpecificationExecutor<Product> {}
```

### 2. Derived query methods
Spring parses the method name:
```java
List<Product> findByCategory(String category);
List<Product> findByCategoryAndPriceLessThan(String category, BigDecimal max);
List<Product> findByNameContainingIgnoreCase(String fragment);           // LIKE %x%
List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
List<Product> findByCategoryIn(Collection<String> categories);
List<Product> findByCreatedAtAfter(Instant since);
List<Product> findByDescriptionIsNull();
List<Product> findByActiveTrueOrderByPriceDesc();
Optional<Product> findFirstByCategoryOrderByPriceAsc(String category);
List<Product> findTop5ByOrderBySoldCountDesc();
boolean existsBySku(String sku);
long countByCategory(String category);
@Transactional long deleteByCategory(String category);                   // loads then deletes each!
List<Order> findByCustomer_Email(String email);                           // nested property traversal
Stream<Product> streamAllByActiveTrue();                                  // must be used inside a transaction + close
Page<Product> findByCategory(String category, Pageable pageable);
Slice<Product> findByActiveTrue(Pageable pageable);                       // no count query
List<Product> findByCategory(String category, Sort sort);
```
Keywords: `And, Or, Is, Equals, Not, Between, LessThan(Equal), GreaterThan(Equal), After, Before, IsNull, IsNotNull, Like, NotLike, StartingWith, EndingWith, Containing, In, NotIn, True, False, IgnoreCase, OrderBy, Top/First, Distinct`.
Long names = smell → switch to `@Query` or Specifications.

### 3. `@Query`
```java
@Query("select p from Product p where p.category = :cat and p.price <= :max")
List<Product> cheapIn(@Param("cat") String category, @Param("max") BigDecimal max);

@Query(value = "select * from product where tsv @@ plainto_tsquery(:q)", nativeQuery = true)
List<Product> fullText(@Param("q") String q);

@Query(value = "select p from Product p where p.category = :cat",
       countQuery = "select count(p) from Product p where p.category = :cat")
Page<Product> pageByCategory(@Param("cat") String cat, Pageable pageable);

@Modifying(clearAutomatically = true, flushAutomatically = true)    // keeps persistence context consistent
@Transactional
@Query("update Product p set p.price = p.price * :factor where p.category = :cat")
int reprice(@Param("cat") String cat, @Param("factor") BigDecimal factor);

@Query("select p from Product p where p.name like %:term%")         // SpEL-ish LIKE shorthand
```
SpEL: `#{#entityName}` for generic repositories.

### 4. Projections
```java
// Interface-based (closed) — Spring selects only needed columns
public interface ProductSummary { Long getId(); String getName(); BigDecimal getPrice(); }
List<ProductSummary> findByCategory(String category);

// Class/record-based (DTO)
public record ProductView(Long id, String name) {}
List<ProductView> findByActiveTrue();

// Dynamic projections
<T> List<T> findByCategory(String category, Class<T> type);
repo.findByCategory("books", ProductView.class);

// Open projection with SpEL (loads full entity — less efficient)
@Value("#{target.name + ' (' + target.category + ')'}") String getLabel();
```

### 5. Specifications — dynamic search
```java
public final class ProductSpecs {
    public static Specification<Product> hasCategory(String c) {
        return (root, query, cb) -> c == null ? null : cb.equal(root.get("category"), c);
    }
    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, q, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.le(root.get("price"), max);
            if (max == null) return cb.ge(root.get("price"), min);
            return cb.between(root.get("price"), min, max);
        };
    }
    public static Specification<Product> nameLike(String term) {
        return (root, q, cb) -> term == null ? null : cb.like(cb.lower(root.get("name")), "%" + term.toLowerCase() + "%");
    }
}

Page<Product> page = repo.findAll(
    Specification.where(hasCategory(f.category())).and(priceBetween(f.min(), f.max())).and(nameLike(f.q())),
    pageable);
```
`null` predicates are ignored → clean optional filters. Alternatives: **Querydsl**, Query by Example (`Example.of(probe, ExampleMatcher.matching().withIgnoreCase())`).

### 6. Pagination & sorting
```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("price").descending().and(Sort.by("name")));
Page<Product> page = repo.findAll(pageable);
page.getContent(); page.getTotalElements(); page.getTotalPages(); page.hasNext(); page.getNumber();
```
`Page` runs an extra `COUNT` query; `Slice` doesn't (just checks if there's a next element); Spring Data 3.1 `Window`/`ScrollPosition` for keyset scrolling:
```java
Window<Product> findFirst20ByActiveTrueOrderByIdAsc(ScrollPosition position);
```
Web: `?page=0&size=20&sort=price,desc`; limit max size with `spring.data.web.pageable.max-page-size=100`.

### 7. Custom repository implementations
```java
public interface ProductRepositoryCustom { List<Product> complexSearch(SearchCriteria c); }

class ProductRepositoryCustomImpl implements ProductRepositoryCustom {   // name must end with Impl
    @PersistenceContext private EntityManager em;
    public List<Product> complexSearch(SearchCriteria c) { /* Criteria API / native SQL */ }
}

public interface ProductRepository extends JpaRepository<Product, Long>, ProductRepositoryCustom {}
```

### 8. `save()` semantics & gotchas
- `save(entity)`: if `isNew` (id null for wrapper types, or `@Version` null, or implements `Persistable`) → `persist`, else `merge` (SELECT then UPDATE). With app-assigned ids (UUID set manually) every save triggers a SELECT first → implement `Persistable<ID>` with `isNew()`.
- Inside a transaction, modifying a managed entity needs no `save()`.
- `deleteAll()` / `deleteBy...` load entities and delete one by one (triggers cascades, callbacks); `deleteAllInBatch()` issues one DELETE (no cascades).
- `findById` hits DB (or 1st level cache); `getReferenceById` returns a lazy proxy without hitting DB.
- `saveAll` with IDENTITY ids doesn't batch.

### 9. Other Spring Data modules (same programming model)
| Module | Store |
|--------|-------|
| Spring Data JDBC | Relational, aggregate-oriented, no lazy loading |
| Spring Data R2DBC | Reactive relational |
| Spring Data MongoDB | `MongoRepository`, `MongoTemplate` |
| Spring Data Redis | `RedisTemplate`, `@RedisHash` repositories |
| Spring Data Elasticsearch | Search |
| Spring Data Cassandra, Neo4j, Couchbase | |
| Spring Data REST | Auto-expose repositories as REST/HAL (prototypes) |

## Part B — Transaction Management

### 10. ACID refresher
- **Atomicity**: all or nothing.
- **Consistency**: constraints hold before and after.
- **Isolation**: concurrent transactions don't see each other's intermediate state (to a configurable degree).
- **Durability**: committed data survives crashes.

### 11. Spring's transaction abstraction
```
@Transactional  ─►  TransactionInterceptor (AOP proxy)
                         │
                         ▼
                PlatformTransactionManager
                 ├─ JpaTransactionManager          (JPA; Boot default with data-jpa)
                 ├─ DataSourceTransactionManager   (JDBC)
                 ├─ JtaTransactionManager          (distributed XA — rare)
                 ├─ KafkaTransactionManager, RabbitTransactionManager
                 └─ ReactiveTransactionManager     (R2DBC, reactive Mongo)
```
The transaction (connection/EntityManager) is bound to the current **thread** (`TransactionSynchronizationManager` → ThreadLocal). Work done on another thread (`@Async`, `CompletableFuture`, parallel streams) is **not** part of the transaction.

### 12. Declarative transactions
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)                  // class default for queries
public class TransferService {

    private final AccountRepository accounts;
    private final TransferRepository transfers;

    @Transactional                                // overrides: read-write
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accounts.findForUpdate(fromId).orElseThrow();
        Account to = accounts.findForUpdate(toId).orElseThrow();
        from.withdraw(amount);                    // throws InsufficientBalanceException (runtime) → rollback
        to.deposit(amount);
        transfers.save(new Transfer(fromId, toId, amount));
    }                                             // commit → flush dirty entities → COMMIT

    public BigDecimal balance(Long id) { return accounts.findById(id).orElseThrow().getBalance(); }
}
```
Put `@Transactional` on the **service layer** (business use case boundary), not controllers or repositories (Spring Data repos already have their own).

Programmatic alternative:
```java
transactionTemplate.execute(status -> {
    ...;
    if (problem) status.setRollbackOnly();
    return result;
});
```

### 13. `@Transactional` attributes
| Attribute | Default | Meaning |
|-----------|---------|---------|
| `propagation` | `REQUIRED` | How to behave with an existing tx |
| `isolation` | `DEFAULT` (DB's default) | Isolation level |
| `readOnly` | false | Hint: no flush/dirty checking in Hibernate; driver/replica routing |
| `timeout` | none | Seconds before rollback |
| `rollbackFor` / `rollbackForClassName` | — | Extra exceptions causing rollback |
| `noRollbackFor` | — | Exceptions that should still commit |
| `transactionManager` / `value` | primary | Which manager (multiple DBs) |
| `label` | — | Metadata |

### 14. Rollback rules ⚠️
- **Default: roll back on unchecked exceptions (`RuntimeException`, `Error`); COMMIT on checked exceptions.**
```java
@Transactional
public void importFile() throws IOException {
    repo.save(a);
    throw new IOException("bad file");      // checked → transaction COMMITS a!
}
@Transactional(rollbackFor = Exception.class)     // fix
```
- Catching an exception inside the method and not rethrowing → commit (unless you call `setRollbackOnly`).
- `UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only` → an inner `REQUIRED` method threw (marking the shared tx rollback-only), the outer caught it and tried to commit.

### 15. Propagation
| Propagation | Existing tx | No tx |
|-------------|-------------|-------|
| **REQUIRED** (default) | Join it | Create new |
| **REQUIRES_NEW** | **Suspend** it, create new independent tx | Create new |
| **NESTED** | Savepoint inside existing tx; inner rollback returns to the savepoint (JDBC savepoints — best with DataSourceTransactionManager; with JPA the EntityManager state isn't rolled back, so use carefully) | Create new |
| **SUPPORTS** | Join | Run non-transactionally |
| **NOT_SUPPORTED** | Suspend, run non-tx | Run non-tx |
| **MANDATORY** | Join | **Throw** |
| **NEVER** | **Throw** | Run non-tx |

Classic use of `REQUIRES_NEW`: audit/failure logging that must persist even if the main transaction rolls back:
```java
@Service class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String event) { auditRepo.save(new AuditEntry(event)); }
}
```
Notes: REQUIRES_NEW takes a **second DB connection** while the first is held → pool deadlock risk under load. And it must be in a **different bean** (proxy!).

### 16. Isolation levels & anomalies
| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|:-:|:-:|:-:|
| READ_UNCOMMITTED | possible | possible | possible |
| **READ_COMMITTED** (Postgres, Oracle, SQL Server default) | ✘ | possible | possible |
| **REPEATABLE_READ** (MySQL InnoDB default) | ✘ | ✘ | possible (InnoDB mostly prevents) |
| SERIALIZABLE | ✘ | ✘ | ✘ |

- **Dirty read**: reading uncommitted changes of another tx.
- **Non-repeatable read**: same row read twice returns different values (another tx committed an update).
- **Phantom read**: same query returns different set of rows (another tx inserted/deleted).
- **Lost update**: two txs read-modify-write the same row; one overwrites the other → solve with optimistic locking (`@Version`), `SELECT FOR UPDATE`, or atomic `UPDATE ... SET x = x + 1`.
- Postgres/Oracle use MVCC: readers don't block writers. Higher isolation → more aborts/locks → retry logic needed (serialization failures).

### 17. Common `@Transactional` pitfalls (very frequently asked)
1. **Self-invocation** — calling a `@Transactional` method from the same class bypasses the proxy.
2. **Non-public methods** — historically ignored (Spring 6 supports protected/package-private with class-based proxies, but keep them public).
3. **Checked exceptions don't roll back** by default.
4. **Swallowing exceptions** → commit.
5. **Wrong layer** — on controller/private helper.
6. **Long transactions** — calling remote HTTP APIs/sending emails inside a transaction holds DB connections & locks. Do I/O outside, or use events (`@TransactionalEventListener(AFTER_COMMIT)`) / outbox.
7. **Async/threads** — new thread = no transaction.
8. **Multiple transaction managers** — specify which one.
9. **`readOnly = true` isn't a security guarantee** — it's an optimization hint (Postgres JDBC does set the connection read-only).
10. **Testing** — `@Transactional` on tests rolls back at the end; but code using `REQUIRES_NEW` or another thread will commit/not see data.
11. **`@Transactional` on interface vs class** — put it on the concrete class.
12. **Expecting a DB constraint error at `save()`** — errors appear at flush/commit, outside your try/catch. Use `saveAndFlush` if you need to catch it inside.

### 18. Distributed transactions
- XA / 2PC (JTA with Atomikos/Narayana) — strong consistency across resources but slow, blocking, poorly supported in cloud/microservices.
- Microservices use **Saga** (sequence of local transactions + compensations) and **Transactional Outbox** (write event to an outbox table in the same local transaction; relay publishes it to Kafka) → eventual consistency. See microservices notes.

### 19. Transaction-bound events
```java
@Transactional
public Order place(OrderRequest r) {
    Order o = repo.save(...);
    events.publishEvent(new OrderPlaced(o.getId()));
    return o;
}
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void notify(OrderPlaced e) { emailService.sendConfirmation(e.orderId()); }   // never emails for rolled-back orders
```

### 20. Multiple data sources (outline)
Define two `DataSource`, `LocalContainerEntityManagerFactoryBean`, and `PlatformTransactionManager` beans (one `@Primary`), use `@EnableJpaRepositories(basePackages=..., entityManagerFactoryRef=..., transactionManagerRef=...)` per package, and `@Transactional("ordersTransactionManager")`. Read replicas: `AbstractRoutingDataSource` + `LazyConnectionDataSourceProxy` routing `readOnly` transactions to replicas.

## 21. Interview Q&A
**Q. How does Spring Data generate repository implementations?** A proxy backed by `SimpleJpaRepository`; derived queries parsed from method names; `@Query` executed via EntityManager.

**Q. `CrudRepository` vs `JpaRepository`?** JpaRepository adds JPA specifics: flush, batch deletes, `getReferenceById`, List return types, Example queries.

**Q. `findById` vs `getReferenceById`?** findById queries immediately and returns Optional; getReferenceById returns a lazy proxy (exception on access if the row doesn't exist).

**Q. How do you implement dynamic filtering?** Specifications (Criteria API), Querydsl, Query by Example, or a custom repository.

**Q. Page vs Slice?** Page includes total count (extra query); Slice only knows whether there's a next slice.

**Q. How does `@Transactional` work?** AOP proxy intercepts the call, TransactionManager begins tx (binds connection to thread), invokes method, commits or rolls back based on exception rules.

**Q. Default rollback behaviour?** Rollback for RuntimeException and Error, commit for checked exceptions.

**Q. REQUIRED vs REQUIRES_NEW?** REQUIRED joins the current tx; REQUIRES_NEW suspends it and runs an independent tx that commits/rolls back separately.

**Q. Why doesn't `@Transactional` work on a private method / internal call?** Proxies only intercept external calls on public (overridable) methods.

**Q. Explain isolation levels and anomalies.** (Table above.)

**Q. What does `readOnly = true` do?** Hibernate sets FlushMode.MANUAL/skips dirty checking & snapshots; drivers may optimize; can be used to route to replicas.

**Q. How to prevent lost updates?** Optimistic locking with `@Version`, pessimistic locks, or atomic update statements.
