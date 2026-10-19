# 07 — Functional Programming, Lambdas, Streams & Optional

## 1. Imperative vs functional
```java
// Imperative: HOW
List<String> result = new ArrayList<>();
for (Employee e : employees) {
    if (e.salary() > 50_000) result.add(e.name().toUpperCase());
}
Collections.sort(result);

// Functional/declarative: WHAT
List<String> result = employees.stream()
    .filter(e -> e.salary() > 50_000)
    .map(e -> e.name().toUpperCase())
    .sorted()
    .toList();
```
Functional style principles: functions as values, **no side effects**, immutability, composition.

## 2. Lambda expressions

### 2.1 Syntax
```java
() -> 42
x -> x * 2
(x, y) -> x + y
(String s) -> s.length()
(a, b) -> {
    int sum = a + b;
    return sum * 2;
}
```
A lambda is an implementation of a **functional interface** (one abstract method). The compiler infers types from the target type.

### 2.2 Variable capture
Lambdas can use local variables only if they are **effectively final**:
```java
int base = 10;
Function<Integer, Integer> add = x -> x + base;   // ok
base = 20;                                         // now compile error above
```
Why? Locals live on the stack and may be gone when the lambda runs on another thread; Java captures the value, so it must not change. Instance fields can be modified (captured via `this`) — but that's a side effect, avoid it.

### 2.3 `this` in a lambda
Refers to the enclosing instance (unlike anonymous classes, where `this` is the anonymous object).

### 2.4 How lambdas are compiled
Not anonymous classes — they use `invokedynamic` + `LambdaMetafactory`; the body becomes a private synthetic method. Cheaper than inner classes.

## 3. Built-in functional interfaces (`java.util.function`)

| Interface | Abstract method | Typical use | Composition |
|-----------|----------------|-------------|-------------|
| `Supplier<T>` | `T get()` | lazy values, factories | — |
| `Consumer<T>` | `void accept(T)` | side effects (print, save) | `andThen` |
| `BiConsumer<T,U>` | `void accept(T,U)` | `map.forEach` | `andThen` |
| `Function<T,R>` | `R apply(T)` | transform | `andThen`, `compose`, `identity` |
| `BiFunction<T,U,R>` | `R apply(T,U)` | combine two values | `andThen` |
| `UnaryOperator<T>` | `T apply(T)` | same-type transform | |
| `BinaryOperator<T>` | `T apply(T,T)` | reduce | `minBy`, `maxBy` |
| `Predicate<T>` | `boolean test(T)` | filter | `and`, `or`, `negate`, `not`, `isEqual` |
| `BiPredicate<T,U>` | `boolean test(T,U)` | | |

Primitive specializations avoid boxing: `IntPredicate`, `IntFunction<R>`, `ToIntFunction<T>`, `IntUnaryOperator`, `IntBinaryOperator`, `IntSupplier`, `ObjIntConsumer`...

```java
Predicate<String> notBlank = Predicate.not(String::isBlank);
Predicate<User> activeAdmin = User::isActive;
activeAdmin = activeAdmin.and(u -> u.role() == Role.ADMIN);

Function<String, String> trim = String::trim;
Function<String, Integer> trimThenLength = trim.andThen(String::length);

Supplier<List<String>> listFactory = ArrayList::new;
```

### Custom functional interface
```java
@FunctionalInterface
public interface TriFunction<A, B, C, R> { R apply(A a, B b, C c); }
```

## 4. Method references
| Kind | Syntax | Lambda equivalent |
|------|--------|------------------|
| Static | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Bound instance | `System.out::println` | `x -> System.out.println(x)` |
| Unbound instance | `String::toUpperCase` | `s -> s.toUpperCase()` |
| Constructor | `ArrayList::new` | `() -> new ArrayList<>()` |
| Array constructor | `int[]::new` | `n -> new int[n]` |

## 5. Streams API

### 5.1 Anatomy
```
Source ──► intermediate op ──► intermediate op ──► terminal op
(collection,   (lazy, return a Stream)            (triggers execution,
 array, gen.)                                       produces result/side effect)
```
- **Lazy**: nothing runs until a terminal op.
- **Single-use**: a stream can't be reused after terminal op (`IllegalStateException`).
- **Doesn't modify the source.**
- Processes elements **vertically** (one element passes through the whole pipeline before the next), enabling short-circuiting.

```java
Stream.of("a", "bb", "ccc")
    .filter(s -> { System.out.println("filter " + s); return s.length() > 1; })
    .map(s -> { System.out.println("map " + s); return s.toUpperCase(); })
    .findFirst();
// filter a, filter bb, map bb  → stops. "ccc" never processed.
```

### 5.2 Creating streams
```java
list.stream(); set.parallelStream();
Arrays.stream(arr); Stream.of(1, 2, 3);
IntStream.range(0, 10); IntStream.rangeClosed(1, 10);
Stream.iterate(1, n -> n * 2).limit(10);
Stream.iterate(1, n -> n < 1000, n -> n * 2);       // Java 9 with predicate
Stream.generate(UUID::randomUUID).limit(5);
Stream.empty(); Stream.ofNullable(x);
"text".chars(); Files.lines(path); Pattern.compile(",").splitAsStream(csv);
map.entrySet().stream();
```

### 5.3 Intermediate operations
| Op | Notes |
|----|-------|
| `filter(Predicate)` | Keep matching |
| `map(Function)` | Transform 1→1 |
| `mapToInt/Long/Double` | To primitive stream |
| `flatMap(Function<T, Stream<R>>)` | Transform 1→many and flatten |
| `mapMulti` (Java 16) | Imperative alternative to flatMap |
| `distinct()` | Uses equals/hashCode (stateful) |
| `sorted()` / `sorted(Comparator)` | Stateful — needs all elements |
| `peek(Consumer)` | Debugging only |
| `limit(n)`, `skip(n)` | Short-circuiting / pagination |
| `takeWhile`, `dropWhile` (Java 9) | Ordered prefix ops |
| `boxed()` | IntStream → Stream<Integer> |

### 5.4 Terminal operations
| Op | Returns |
|----|---------|
| `forEach`, `forEachOrdered` | void |
| `collect(Collector)` | anything |
| `toList()` (Java 16) | unmodifiable List |
| `toArray()` | array |
| `reduce(identity, accumulator)` | T |
| `count()`, `min()`, `max()` | long / Optional |
| `sum()`, `average()`, `summaryStatistics()` | primitive streams |
| `anyMatch`, `allMatch`, `noneMatch` | boolean (short-circuit) |
| `findFirst`, `findAny` | Optional (short-circuit) |

### 5.5 map vs flatMap
```java
List<List<String>> nested = List.of(List.of("a", "b"), List.of("c"));
nested.stream().map(List::size).toList();             // [2, 1]
nested.stream().flatMap(List::stream).toList();        // [a, b, c]

// Real example: all distinct skills across employees
employees.stream().flatMap(e -> e.skills().stream()).distinct().sorted().toList();

// Optional chaining is flatMap too
user.flatMap(User::getAddress).map(Address::city);
```

### 5.6 reduce
```java
int sum = nums.stream().reduce(0, Integer::sum);
Optional<Integer> max = nums.stream().reduce(Integer::max);
BigDecimal total = orders.stream().map(Order::amount).reduce(BigDecimal.ZERO, BigDecimal::add);
```
Identity must be a true identity for the accumulator; accumulator must be associative (needed for parallel).

## 6. Collectors (the most interview-heavy part)

```java
import static java.util.stream.Collectors.*;

// To collections
stream.collect(toList()); stream.collect(toSet());
stream.collect(toCollection(TreeSet::new));
stream.collect(toUnmodifiableList());

// To map — beware duplicates (IllegalStateException) and null values (NPE)
Map<Long, Employee> byId = emps.stream().collect(toMap(Employee::id, Function.identity()));
Map<String, BigDecimal> salaryByName = emps.stream()
    .collect(toMap(Employee::name, Employee::salary, BigDecimal::add, LinkedHashMap::new));
//                         key          value      merge on duplicate   map supplier

// Joining
names.stream().collect(joining(", ", "[", "]"));      // [a, b, c]

// Aggregations
emps.stream().collect(counting());
emps.stream().collect(averagingDouble(e -> e.salary().doubleValue()));
emps.stream().collect(summingInt(Employee::age));
emps.stream().collect(maxBy(Comparator.comparing(Employee::salary)));
IntSummaryStatistics stats = emps.stream().collect(summarizingInt(Employee::age));
stats.getMin(); stats.getMax(); stats.getAverage(); stats.getCount();

// groupingBy — SQL GROUP BY
Map<String, List<Employee>> byDept = emps.stream().collect(groupingBy(Employee::dept));
Map<String, Long> countByDept = emps.stream().collect(groupingBy(Employee::dept, counting()));
Map<String, Double> avgSalaryByDept = emps.stream()
    .collect(groupingBy(Employee::dept, averagingDouble(e -> e.salary().doubleValue())));
Map<String, Optional<Employee>> topEarnerByDept = emps.stream()
    .collect(groupingBy(Employee::dept, maxBy(Comparator.comparing(Employee::salary))));
Map<String, Set<String>> namesByDept = emps.stream()
    .collect(groupingBy(Employee::dept, TreeMap::new, mapping(Employee::name, toCollection(TreeSet::new))));
Map<String, Map<String, List<Employee>>> byDeptThenCity = emps.stream()
    .collect(groupingBy(Employee::dept, groupingBy(Employee::city)));

// partitioningBy — always two keys: true / false
Map<Boolean, List<Employee>> seniors = emps.stream().collect(partitioningBy(e -> e.age() > 40));

// filtering / flatMapping (Java 9) as downstream collectors
Map<String, List<Employee>> highEarnersByDept = emps.stream()
    .collect(groupingBy(Employee::dept, filtering(e -> e.salary().compareTo(LIMIT) > 0, toList())));

// collectingAndThen — post-process
List<String> unmodifiable = names.stream().collect(collectingAndThen(toList(), Collections::unmodifiableList));
Map<String, Employee> topByDept = emps.stream().collect(groupingBy(Employee::dept,
    collectingAndThen(maxBy(Comparator.comparing(Employee::salary)), Optional::get)));

// teeing (Java 12) — two collectors, then merge
var minMax = nums.stream().collect(teeing(minBy(naturalOrder()), maxBy(naturalOrder()),
        (min, max) -> List.of(min.get(), max.get())));
```

## 7. Classic stream interview problems
```java
// 1. Second highest salary
emps.stream().map(Employee::salary).distinct()
    .sorted(Comparator.reverseOrder()).skip(1).findFirst();

// 2. Find duplicates
Set<Integer> seen = new HashSet<>();
nums.stream().filter(n -> !seen.add(n)).collect(toSet());       // side-effect version
nums.stream().collect(groupingBy(identity(), counting()))
    .entrySet().stream().filter(e -> e.getValue() > 1).map(Map.Entry::getKey).toList();

// 3. Character frequency in a string
"programming".chars().mapToObj(c -> (char) c)
    .collect(groupingBy(identity(), LinkedHashMap::new, counting()));

// 4. First non-repeating character
"swiss".chars().mapToObj(c -> (char) c)
    .collect(groupingBy(identity(), LinkedHashMap::new, counting()))
    .entrySet().stream().filter(e -> e.getValue() == 1).map(Map.Entry::getKey).findFirst();

// 5. Reverse each word
Arrays.stream(sentence.split(" "))
    .map(w -> new StringBuilder(w).reverse().toString()).collect(joining(" "));

// 6. Sum of squares of even numbers
nums.stream().filter(n -> n % 2 == 0).mapToInt(n -> n * n).sum();

// 7. Highest paid employee per department (name only)
emps.stream().collect(groupingBy(Employee::dept,
    collectingAndThen(maxBy(comparing(Employee::salary)), o -> o.map(Employee::name).orElse(""))));

// 8. Merge two lists without duplicates, sorted
Stream.concat(a.stream(), b.stream()).distinct().sorted().toList();

// 9. Longest string
words.stream().max(comparingInt(String::length));

// 10. Convert List<Employee> to Map<dept, totalSalary>
emps.stream().collect(toMap(Employee::dept, Employee::salary, BigDecimal::add));

// 11. Check if string is palindrome
IntStream.range(0, s.length() / 2).allMatch(i -> s.charAt(i) == s.charAt(s.length() - 1 - i));

// 12. Fibonacci first 10
Stream.iterate(new long[]{0, 1}, f -> new long[]{f[1], f[0] + f[1]}).limit(10).map(f -> f[0]).toList();
```

## 8. Parallel streams
```java
long count = bigList.parallelStream().filter(this::isPrime).count();
```
- Uses the **common ForkJoinPool** (size = CPU cores − 1) — shared with the whole JVM (including other requests in a web server!).
- Good when: large data (100k+), CPU-bound, stateless, cheap-to-split sources (ArrayList, arrays, IntStream.range).
- Bad when: I/O-bound (blocking DB/HTTP calls), small data, LinkedList/iterate sources, order-dependent ops (`findFirst`, `limit`, `sorted`), shared mutable state.
- Never do this in a parallel stream: `list.parallelStream().forEach(results::add)` with a non-thread-safe list. Use `collect`.
- In web apps prefer explicit executors (or virtual threads) over parallel streams.

## 9. Optional

### 9.1 Purpose
A return type that explicitly says "might be empty", forcing callers to handle absence. Not a general null replacement.

```java
Optional<User> findByEmail(String email);

// Creating
Optional.of(value);          // NPE if value null
Optional.ofNullable(value);
Optional.empty();

// Consuming — prefer these
opt.map(User::getName).orElse("guest");
opt.orElseGet(() -> expensiveDefault());       // lazy
opt.orElseThrow(() -> new UserNotFoundException(email));
opt.orElseThrow();                              // NoSuchElementException (Java 10)
opt.ifPresent(u -> send(u));
opt.ifPresentOrElse(this::send, () -> log.warn("not found"));   // Java 9
opt.filter(User::isActive);
opt.flatMap(User::getManager);                 // when mapper returns Optional
opt.or(() -> findInCache(email));               // Java 9 — alternative Optional
opt.stream();                                   // Java 9 — 0 or 1 element stream
opt.isPresent(); opt.isEmpty();
```

### 9.2 orElse vs orElseGet
```java
opt.orElse(createDefault());        // createDefault() ALWAYS runs
opt.orElseGet(this::createDefault); // runs only if empty
```

### 9.3 Anti-patterns
| Don't | Why / Instead |
|-------|---------------|
| `opt.get()` without check | Same as NPE risk → `orElseThrow` |
| `if (opt.isPresent()) { x = opt.get(); }` | Use `map/orElse/ifPresent` |
| Optional fields in entities/DTOs | Not serializable; use nullable field |
| Optional method parameters | Overload or nullable |
| `Optional<List<T>>` | Return empty list |
| Returning `null` from a method that returns Optional | Defeats the purpose |

## 10. Functional design in backend code
```java
// Strategy pattern with lambdas
Map<PaymentType, Function<Order, Receipt>> processors = Map.of(
    PaymentType.CARD, cardService::pay,
    PaymentType.UPI,  upiService::pay);
processors.get(order.type()).apply(order);

// Retry helper with Supplier
static <T> T retry(int times, Supplier<T> action) {
    RuntimeException last = null;
    for (int i = 0; i < times; i++) {
        try { return action.get(); } catch (RuntimeException e) { last = e; }
    }
    throw last;
}
retry(3, () -> client.fetchRates());

// Validation chain with Predicates
List<Predicate<Order>> rules = List.of(o -> o.total().signum() > 0, o -> !o.items().isEmpty());
boolean valid = rules.stream().allMatch(r -> r.test(order));
```

## 11. Interview Q&A
**Q. What is a functional interface? Is `@FunctionalInterface` required?** One abstract method (default/static/Object methods don't count). Annotation is optional but makes the compiler enforce it.

**Q. Intermediate vs terminal operations?** Intermediate are lazy and return streams; terminal trigger processing and return a non-stream result.

**Q. map vs flatMap?** map: one input → one output. flatMap: one input → stream of outputs, flattened.

**Q. Can a stream be reused?** No — `IllegalStateException: stream has already been operated upon or closed`.

**Q. `findFirst` vs `findAny`?** findFirst respects encounter order; findAny may return any (faster in parallel).

**Q. Collection vs Stream?** Collection stores data (eager, reusable); Stream computes data on demand (lazy, single-use, no storage).

**Q. `Stream.toList()` vs `collect(Collectors.toList())`?** `toList()` returns an unmodifiable list and allows nulls; `Collectors.toList()` currently returns a mutable ArrayList (not guaranteed).

**Q. Why must lambda-captured variables be effectively final?** Captured by value; mutability would create race conditions and confusing semantics.

**Q. When not to use parallel streams?** Small data, I/O work, shared mutable state, order-sensitive operations, inside request-handling code sharing the common pool.
