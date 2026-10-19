# 06 — Generics

## 1. Why generics?
Before Java 5:
```java
List list = new ArrayList();
list.add("hello");
list.add(42);                         // compiles
String s = (String) list.get(1);      // ClassCastException at runtime
```
With generics:
```java
List<String> list = new ArrayList<>();   // diamond operator (Java 7)
list.add(42);                            // compile error ✔
String s = list.get(0);                  // no cast
```
Benefits: **compile-time type safety, no casts, reusable algorithms**.

## 2. Generic classes and interfaces
```java
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

public interface Repository<T, ID> {
    Optional<T> findById(ID id);
    T save(T entity);
    List<T> findAll();
}

public class Pair<K, V> {
    private final K key; private final V value;
    public Pair(K key, V value) { this.key = key; this.value = value; }
}
// or simply: public record Pair<K, V>(K key, V value) {}
```
Naming convention: `T` type, `E` element, `K` key, `V` value, `N` number, `R` result, `ID` identifier.

## 3. Generic methods
```java
public static <T> T firstOrDefault(List<T> list, T defaultValue) {
    return list.isEmpty() ? defaultValue : list.get(0);
}
String s = firstOrDefault(names, "none");             // T inferred as String
Integer i = Util.<Integer>firstOrDefault(nums, 0);    // explicit type witness
```
The `<T>` before the return type declares the method's own type parameter.

## 4. Bounded type parameters
```java
// Upper bound
public static <T extends Number> double sum(List<T> nums) {
    return nums.stream().mapToDouble(Number::doubleValue).sum();
}

// Multiple bounds: class first, then interfaces
public static <T extends Number & Comparable<T>> T max(List<T> list) { ... }

// Recursive bound — "T comparable to itself"
public static <T extends Comparable<? super T>> T max(Collection<T> c) { ... }
```

## 5. Wildcards
```java
List<?>               // unknown type: read as Object, can't add (except null)
List<? extends Number> // Number or subtype: READ as Number, can't add
List<? super Integer>  // Integer or supertype: can ADD Integer, read as Object
```

### Why `List<Integer>` is NOT a `List<Number>`
Generics are **invariant**. If it were allowed:
```java
List<Integer> ints = new ArrayList<>();
List<Number> nums = ints;    // suppose allowed
nums.add(3.14);              // now a Double is in a List<Integer>!
```
Arrays are covariant (and fail at runtime with ArrayStoreException); generics prevent this at compile time.

### PECS — Producer Extends, Consumer Super
```java
// src PRODUCES T values (we read) → extends
// dst CONSUMES T values (we write) → super
public static <T> void copy(List<? super T> dst, List<? extends T> src) {
    for (T t : src) dst.add(t);
}
```
Example in JDK: `Collections.copy`, `Collections.max(Collection<? extends T>, Comparator<? super T>)`.

## 6. Type erasure
The compiler checks types, then **erases** them:
- `T` → `Object` (or its first bound, e.g., `Number`).
- Inserts casts where needed.
- Generates **bridge methods** to preserve polymorphism.

Consequences / limitations:
```java
new T();                         // ✘ can't instantiate type param
T[] arr = new T[10];             // ✘ generic array creation
if (obj instanceof List<String>) // ✘ (List<?> is OK)
static T field;                  // ✘ static fields can't use class type param
List<int>                        // ✘ primitives not allowed — use Integer
void m(List<String> l) {} void m(List<Integer> l) {}   // ✘ same erasure → clash
```
Workaround for creating instances — pass a `Class<T>` or `Supplier<T>`:
```java
public static <T> T create(Supplier<T> factory) { return factory.get(); }
create(ArrayList::new);
```

### Getting generic types at runtime: "super type token"
Erasure removes type args from objects, but they're kept in **class signatures**. Jackson's `TypeReference` and Spring's `ParameterizedTypeReference` exploit this:
```java
List<User> users = objectMapper.readValue(json, new TypeReference<List<User>>() {});
ResponseEntity<List<User>> r = restTemplate.exchange(url, GET, null,
        new ParameterizedTypeReference<List<User>>() {});
```
The anonymous subclass captures `List<User>` in its superclass signature.

## 7. Raw types and heap pollution
```java
List raw = new ArrayList<String>();   // raw type: compiler warning, no checks
```
Avoid raw types — they exist only for backward compatibility.
**Heap pollution**: a variable of a parameterized type refers to an object of a different parameterization (often via varargs of generic types) → `@SafeVarargs` on methods that are truly safe.

## 8. Generics in real backend code
```java
// Generic API response wrapper
public record ApiResponse<T>(boolean success, T data, String error) {
    public static <T> ApiResponse<T> ok(T data) { return new ApiResponse<>(true, data, null); }
    public static <T> ApiResponse<T> fail(String err) { return new ApiResponse<>(false, null, err); }
}

// Generic paged result
public record PageResult<T>(List<T> content, int page, int size, long total) {
    public <R> PageResult<R> map(Function<? super T, ? extends R> fn) {
        return new PageResult<>(content.stream().<R>map(fn).toList(), page, size, total);
    }
}

// Spring Data: JpaRepository<T, ID> is generic
public interface OrderRepository extends JpaRepository<Order, Long> {}

// Generic service base
public abstract class CrudService<T, ID> {
    protected final JpaRepository<T, ID> repo;
    protected CrudService(JpaRepository<T, ID> repo) { this.repo = repo; }
    public T get(ID id) { return repo.findById(id).orElseThrow(); }
}
```
Spring can even inject by generic type: `@Autowired List<Validator<Order>> validators` picks beans whose generic parameter matches.

## 9. Interview Q&A
**Q. What is type erasure?** Generic type info is removed at compile time; runtime only sees raw types plus casts. Ensures backward compatibility with pre-Java-5 bytecode.

**Q. Difference between `<? extends T>` and `<? super T>`?** Extends: read-only producer of T. Super: write-capable consumer of T. (PECS)

**Q. Why can't we create generic arrays?** Arrays need their component type at runtime for store checks; erased generic types don't have it, which would break type safety.

**Q. `List<?>` vs `List<Object>`?** `List<Object>` accepts only `List<Object>` but you can add anything. `List<?>` accepts any list but you can't add (except null).

**Q. Can you overload methods that differ only in generic parameter type?** No — same erasure → compile error.

**Q. How does Jackson deserialize `List<User>` if types are erased?** With a `TypeReference` subclass that captures the type in its generic superclass signature.
