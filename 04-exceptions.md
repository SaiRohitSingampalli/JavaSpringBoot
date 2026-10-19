# 04 — Exception Handling in Depth

## 1. Hierarchy

```
java.lang.Throwable
├── Error                               ← JVM/system problems; don't catch
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── NoClassDefFoundError, ExceptionInInitializerError
└── Exception                           ← checked (unless RuntimeException)
    ├── IOException
    │   ├── FileNotFoundException
    │   └── UncheckedIOException (wrapper, runtime)
    ├── SQLException
    ├── InterruptedException
    ├── ReflectiveOperationException → ClassNotFoundException
    └── RuntimeException                ← unchecked
        ├── NullPointerException
        ├── IllegalArgumentException → NumberFormatException
        ├── IllegalStateException
        ├── IndexOutOfBoundsException → ArrayIndexOutOfBounds / StringIndexOutOfBounds
        ├── ArithmeticException
        ├── ClassCastException
        ├── UnsupportedOperationException
        ├── ConcurrentModificationException
        └── DateTimeException
```

## 2. Checked vs unchecked

| | Checked | Unchecked |
|--|--|--|
| Extends | `Exception` (not Runtime) | `RuntimeException` / `Error` |
| Compiler | Must catch or declare `throws` | Not enforced |
| Meaning | Recoverable, external condition (file missing, network) | Programming bug or business rule violation |
| Examples | `IOException`, `SQLException` | NPE, `IllegalArgumentException` |

**Modern trend** (Spring, Hibernate, most frameworks): use **unchecked** exceptions. Spring translates checked `SQLException` into unchecked `DataAccessException` hierarchy. Checked exceptions also don't play well with lambdas/streams.

## 3. Syntax

```java
try {
    riskyOperation();
} catch (FileNotFoundException e) {        // more specific first
    log.warn("missing file", e);
} catch (IOException | ParseException e) { // multi-catch; e is effectively final
    throw new DataLoadException("failed to load", e);   // wrap & preserve cause
} finally {
    cleanup();                             // always runs
}
```
- Catch order: subclasses before superclasses, otherwise compile error (unreachable).
- `finally` runs even if `try` returns. It doesn't run if the JVM exits (`System.exit`), the thread is killed, or power is lost.
- **Never `return` from `finally`** — it swallows exceptions and overrides the try's return.

```java
int test() {
    try { return 1; }
    finally { return 2; }      // returns 2, and would swallow any exception — don't!
}
```

## 4. try-with-resources (Java 7)

```java
try (Connection con = dataSource.getConnection();
     PreparedStatement ps = con.prepareStatement(SQL);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { ... }
}   // closed automatically in REVERSE order: rs, ps, con
```
- Works with anything implementing `AutoCloseable`.
- If both the body and `close()` throw, the body exception is thrown and the close exception is attached as a **suppressed exception** (`e.getSuppressed()`).
- Java 9: can use effectively-final variables declared outside: `try (existingStream) { }`.

## 5. throw vs throws
```java
public User find(long id) throws UserNotFoundException {   // declares
    return repo.findById(id).orElseThrow(() -> new UserNotFoundException(id));  // throws
}
```

## 6. Custom exceptions — designing a hierarchy for a backend

```java
// Base for all business errors
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;
    protected BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    protected BusinessException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }
    public String getErrorCode() { return errorCode; }
}

public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String resource, Object id) {
        super("NOT_FOUND", "%s with id %s not found".formatted(resource, id));
    }
}

public class InsufficientBalanceException extends BusinessException {
    public InsufficientBalanceException(BigDecimal needed, BigDecimal available) {
        super("INSUFFICIENT_BALANCE", "Needed %s but only %s available".formatted(needed, available));
    }
}
```
Then map them to HTTP responses in a `@RestControllerAdvice` (see Spring REST notes).

**Rules for good custom exceptions**
- Name ends with `Exception`, describes *what went wrong*.
- Always provide a constructor that takes a `cause`.
- Include context (ids, values) in the message — but **no secrets/PII**.
- Don't create one exception class per method; group by meaning.

## 7. Exception chaining
```java
try {
    parse(file);
} catch (IOException e) {
    throw new ConfigException("Cannot read config " + file, e);   // keep the root cause!
}
```
Stack trace shows `Caused by: ...`. Losing the cause (`throw new X(e.getMessage())`) is one of the most common mistakes.

## 8. Best practices (and anti-patterns)

| Do | Don't |
|----|-------|
| Catch specific exceptions | `catch (Exception e)` everywhere / `catch (Throwable t)` |
| Log OR rethrow (once) | Log and rethrow at every layer → duplicated stack traces |
| Wrap low-level exceptions in domain ones | Leak `SQLException` to controllers |
| Fail fast — validate inputs early | Return `null` / magic values like `-1` for errors |
| Use `Optional` for "not found" in lookups where absence is normal | Use exceptions for normal control flow (slow, unclear) |
| Clean up with try-with-resources | Empty catch blocks `catch (Exception e) {}` |
| Restore interrupt flag | Swallow `InterruptedException` |

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();     // restore flag so callers know
    throw new IllegalStateException("interrupted", e);
}
```

**Performance**: creating an exception captures the stack trace (expensive). For high-frequency, expected failures you can override `fillInStackTrace` or use constructor `super(msg, null, false, false)` — but usually, just don't use exceptions for control flow.

## 9. Exceptions in lambdas / streams
Lambdas for `Function`, `Consumer`, etc. can't throw checked exceptions.
```java
// Option 1: wrap inside
files.stream().map(f -> {
    try { return Files.readString(f); }
    catch (IOException e) { throw new UncheckedIOException(e); }
}).toList();

// Option 2: helper
@FunctionalInterface interface ThrowingFunction<T, R> { R apply(T t) throws Exception; }
static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> f) {
    return t -> { try { return f.apply(t); } catch (Exception e) { throw new RuntimeException(e); } };
}
```

## 10. Exceptions and inheritance
- Overriding method may throw **fewer or narrower checked** exceptions, never broader/new checked ones. Unchecked: anything.
- Constructor of a subclass must declare the checked exceptions thrown by the super constructor.

## 11. NullPointerException — prevention toolkit
- Helpful NPE messages (Java 14+): `Cannot invoke "String.length()" because "user.name" is null`.
- `Objects.requireNonNull(x, "x must not be null")` at boundaries.
- Return empty collections, not null.
- `Optional` for maybe-absent return values.
- `"literal".equals(variable)` instead of `variable.equals("literal")`.
- Annotations `@NonNull/@Nullable` (JSpecify — adopted across Spring Framework 7) + static analysis.
- Bean Validation `@NotNull` on API inputs.

## 12. Exceptions in Spring (preview)
- `@Transactional` rolls back on **unchecked** exceptions by default, not on checked ones (`rollbackFor` to change).
- `@ExceptionHandler` + `@RestControllerAdvice` for centralized HTTP error mapping.
- `ResponseStatusException` for quick status responses.
- Spring's `DataAccessException` (unchecked): `DuplicateKeyException`, `DataIntegrityViolationException`, `OptimisticLockingFailureException`, `EmptyResultDataAccessException`.

## 13. Interview Q&A

**Q. Difference between Error and Exception?** Errors are serious JVM problems apps shouldn't try to handle (OOM, StackOverflow). Exceptions are conditions apps may handle.

**Q. Can `finally` be skipped?** Only by `System.exit()`, JVM crash, or the thread dying (e.g., killed / infinite loop in try).

**Q. What happens if an exception is thrown in a finally block?** It replaces any exception from the try block (original is lost) — another reason to use try-with-resources, which preserves suppressed exceptions.

**Q. Can we have try without catch?** Yes: `try-finally` or try-with-resources alone.

**Q. Why prefer unchecked exceptions in Spring apps?** Less boilerplate, works with lambdas, transaction rollback default, and most failures can't be recovered at the call site anyway — handle centrally.

**Q. What is exception chaining?** Wrapping a lower-level exception as the cause of a higher-level one to preserve root-cause information while abstracting layers.

**Q. Should you catch `NullPointerException`?** No — fix the bug. It indicates a programming error.

**Q. What's a suppressed exception?** An exception thrown during resource closing in try-with-resources, attached to the primary exception rather than replacing it.
