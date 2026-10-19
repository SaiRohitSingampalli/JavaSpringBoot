# 15 — Validation & Global Exception Handling

## Part A — Bean Validation (Jakarta Validation 3 / Hibernate Validator)

Add `spring-boot-starter-validation` (not included in `-web` since Boot 2.3).

### 1. Built-in constraints (`jakarta.validation.constraints`)
| Constraint | Applies to | Notes |
|-----------|-----------|-------|
| `@NotNull` | any | not null |
| `@NotEmpty` | String, Collection, Map, array | not null and size > 0 |
| `@NotBlank` | String | not null and contains non-whitespace |
| `@Size(min, max)` | String, Collection, Map, array | |
| `@Min`, `@Max` | numbers | long values |
| `@DecimalMin("0.01")`, `@DecimalMax` | BigDecimal, strings | inclusive flag |
| `@Positive`, `@PositiveOrZero`, `@Negative`, `@NegativeOrZero` | numbers | |
| `@Digits(integer=10, fraction=2)` | numbers | money |
| `@Email` | String | basic format |
| `@Pattern(regexp = "...")` | String | |
| `@Past`, `@PastOrPresent`, `@Future`, `@FutureOrPresent` | dates/times | |
| `@AssertTrue`, `@AssertFalse` | boolean | e.g., `isPasswordMatching()` |
| `@Null` | any | must be null (e.g., id on create) |
Hibernate extras: `@Length`, `@Range`, `@URL`, `@UUID`, `@CreditCardNumber`.

Note: `null` is considered **valid** by most constraints (`@Size`, `@Email`, `@Pattern`...) — combine with `@NotNull`/`@NotBlank` when required.

### 2. Validating request bodies
```java
public record RegisterRequest(
        @NotBlank @Size(min = 3, max = 50) String username,
        @NotBlank @Email String email,
        @NotBlank @Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d).{8,}$",
                           message = "{password.weak}") String password,
        @NotNull @Past LocalDate birthDate,
        @Valid @NotNull AddressDto address,                 // @Valid cascades into nested object
        @NotEmpty List<@NotBlank String> interests) {}      // container element constraints

@PostMapping("/register")
public ResponseEntity<UserResponse> register(@Valid @RequestBody RegisterRequest req) { ... }
```
Failure → `MethodArgumentNotValidException` (400).

### 3. Validating path variables & request params
```java
@RestController
@Validated                                        // class-level → method validation
class OrderController {
    @GetMapping("/orders/{id}")
    OrderResponse get(@PathVariable @Positive Long id,
                      @RequestParam @Min(1) @Max(100) int limit) { ... }
}
```
Failure → Spring 6.1+: `HandlerMethodValidationException` (built-in MVC method validation); with `@Validated` AOP: `ConstraintViolationException`.

### 4. Validating in the service layer
```java
@Service
@Validated
public class TransferService {
    public void transfer(@NotNull Long from, @NotNull Long to, @Positive BigDecimal amount) { ... }
    public @Valid Receipt pay(@Valid PaymentCommand cmd) { ... }
}
```
Or programmatically:
```java
Set<ConstraintViolation<Order>> violations = validator.validate(order);
```

### 5. Validation groups (different rules for create vs update)
```java
public interface OnCreate {}
public interface OnUpdate {}

public record ProductDto(
    @Null(groups = OnCreate.class) @NotNull(groups = OnUpdate.class) Long id,
    @NotBlank(groups = {OnCreate.class, OnUpdate.class}) String name) {}

@PostMapping public void create(@Validated(OnCreate.class) @RequestBody ProductDto dto) {}
@PutMapping  public void update(@Validated(OnUpdate.class) @RequestBody ProductDto dto) {}
```
(`@Valid` doesn't support groups; `@Validated` does.) Many teams simply use separate request records instead of groups.

### 6. Custom constraint
```java
@Documented
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueEmail {
    String message() default "Email already registered";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component                                            // Spring injects dependencies into validators
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository repo;
    public UniqueEmailValidator(UserRepository repo) { this.repo = repo; }
    @Override public boolean isValid(String email, ConstraintValidatorContext ctx) {
        return email == null || !repo.existsByEmailIgnoreCase(email);
    }
}
```
Note: a uniqueness check here is still racy — also keep a DB unique constraint and handle `DataIntegrityViolationException`.

### 7. Class-level (cross-field) constraint
```java
@PasswordsMatch
public record ChangePasswordRequest(String newPassword, String confirmPassword) {}

public class PasswordsMatchValidator implements ConstraintValidator<PasswordsMatch, ChangePasswordRequest> {
    public boolean isValid(ChangePasswordRequest r, ConstraintValidatorContext ctx) {
        boolean ok = Objects.equals(r.newPassword(), r.confirmPassword());
        if (!ok) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("Passwords do not match")
               .addPropertyNode("confirmPassword").addConstraintViolation();
        }
        return ok;
    }
}
```
Quick alternative: `@AssertTrue boolean isPasswordConfirmed() { ... }` inside a class.

### 8. Messages & i18n
`src/main/resources/ValidationMessages.properties` (or Boot's `messages.properties` via `LocalValidatorFactoryBean`):
```properties
password.weak=Password must have 8+ chars, one uppercase letter and one digit
```
Use `message = "{password.weak}"`. Interpolation: `"must be at least {min} characters"`, `${validatedValue}`.

### 9. Where to validate — layers
| Layer | What |
|-------|------|
| API (DTO) | Format/shape: required, lengths, patterns |
| Service | Business rules: stock available, user allowed, state transitions → throw business exceptions |
| Database | Final guarantee: NOT NULL, UNIQUE, FK, CHECK constraints |

## Part B — Exception Handling in Spring MVC

### 10. Mechanisms (from local to global)
1. `try/catch` in controller — avoid; clutters code.
2. `@ResponseStatus` on custom exception class:
   ```java
   @ResponseStatus(HttpStatus.NOT_FOUND)
   public class UserNotFoundException extends RuntimeException { ... }
   ```
3. `ResponseStatusException` — throw ad hoc: `throw new ResponseStatusException(HttpStatus.NOT_FOUND, "User not found");`
4. `@ExceptionHandler` method inside a controller — local to that controller.
5. **`@RestControllerAdvice` + `@ExceptionHandler`** — global, **recommended**.
6. `ErrorResponse` / `ProblemDetail` / `ErrorResponseException` (Spring 6).
7. Fallback: Boot's `BasicErrorController` at `/error` (whitelabel / JSON `{timestamp,status,error,path}`).

### 11. RFC 9457 Problem Details
Standard JSON error format (`application/problem+json`):
```json
{
  "type": "https://api.shop.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Product with id 42 not found",
  "instance": "/api/v1/products/42",
  "errorCode": "PRODUCT_NOT_FOUND",
  "timestamp": "2026-10-19T10:15:30Z"
}
```
Enable for Spring's built-in exceptions: `spring.mvc.problemdetails.enabled=true`.

### 12. Production-grade global handler
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    // extending gives ProblemDetail handling for all standard Spring MVC exceptions
    // (missing params, type mismatch, unsupported media type, method not allowed, not readable JSON...)

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex, HttpServletRequest req) {
        return problem(HttpStatus.NOT_FOUND, "Resource Not Found", ex.getMessage(), ex.getErrorCode(), req);
    }

    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusiness(BusinessException ex, HttpServletRequest req) {
        return problem(HttpStatus.UNPROCESSABLE_ENTITY, "Business Rule Violation", ex.getMessage(), ex.getErrorCode(), req);
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ProblemDetail handleConflict(DataIntegrityViolationException ex, HttpServletRequest req) {
        log.warn("Constraint violation", ex);
        return problem(HttpStatus.CONFLICT, "Conflict", "Resource already exists or violates a constraint", "CONFLICT", req);
    }

    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ProblemDetail handleOptimisticLock(Exception ex, HttpServletRequest req) {
        return problem(HttpStatus.CONFLICT, "Concurrent Modification", "Resource was modified by someone else; reload and retry", "STALE_DATA", req);
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ProblemDetail handleForbidden(AccessDeniedException ex, HttpServletRequest req) {
        return problem(HttpStatus.FORBIDDEN, "Forbidden", "You don't have permission", "FORBIDDEN", req);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ProblemDetail handleConstraintViolation(ConstraintViolationException ex, HttpServletRequest req) {
        ProblemDetail pd = problem(HttpStatus.BAD_REQUEST, "Validation Failed", "Invalid request parameters", "VALIDATION_ERROR", req);
        pd.setProperty("errors", ex.getConstraintViolations().stream()
            .map(v -> Map.of("field", v.getPropertyPath().toString(), "message", v.getMessage())).toList());
        return pd;
    }

    // Override the parent's handling of @Valid @RequestBody failures to add field errors
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        ProblemDetail pd = ex.getBody();
        pd.setTitle("Validation Failed");
        pd.setDetail("One or more fields are invalid");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of("field", fe.getField(), "message", Objects.requireNonNullElse(fe.getDefaultMessage(), "invalid"),
                              "rejectedValue", String.valueOf(fe.getRejectedValue())))
            .toList());
        pd.setProperty("errorCode", "VALIDATION_ERROR");
        return handleExceptionInternal(ex, pd, headers, status, request);
    }

    // Catch-all: never leak stack traces or internal messages to clients
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest req) {
        log.error("Unhandled error on {} {}", req.getMethod(), req.getRequestURI(), ex);
        return problem(HttpStatus.INTERNAL_SERVER_ERROR, "Internal Server Error",
                "Something went wrong. Reference: " + MDC.get("traceId"), "INTERNAL_ERROR", req);
    }

    private ProblemDetail problem(HttpStatus status, String title, String detail, String code, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(status, detail);
        pd.setTitle(title);
        pd.setType(URI.create("https://api.shop.com/errors/" + code.toLowerCase().replace('_', '-')));
        pd.setInstance(URI.create(req.getRequestURI()));
        pd.setProperty("errorCode", code);
        pd.setProperty("timestamp", Instant.now());
        return pd;
    }
}
```

### 13. Exception → status mapping guide
| Exception | Status |
|-----------|--------|
| Validation (`MethodArgumentNotValidException`, `ConstraintViolationException`, `HandlerMethodValidationException`) | 400 |
| `HttpMessageNotReadableException` (malformed JSON) | 400 |
| `MethodArgumentTypeMismatchException` (`/users/abc`) | 400 |
| `MissingServletRequestParameterException` | 400 |
| Authentication failure | 401 (handled by Spring Security entry point) |
| `AccessDeniedException` | 403 |
| Not found | 404 |
| `HttpRequestMethodNotSupportedException` | 405 |
| `HttpMediaTypeNotSupportedException` | 415 |
| Duplicate / optimistic lock / state conflict | 409 |
| Business rule violation | 422 (or 400/409 — be consistent) |
| Downstream timeout | 504 / 503 |
| Everything else | 500 |

### 14. Important details
- `@RestControllerAdvice(basePackages = "com.shop.api")` / `assignableTypes` to scope advice; `@Order` for multiple advices.
- Most specific `@ExceptionHandler` wins (closest in the exception hierarchy).
- Exceptions thrown in **filters** (including Spring Security) happen before DispatcherServlet → not handled by `@ControllerAdvice`. Security uses `AuthenticationEntryPoint` (401) and `AccessDeniedHandler` (403); for other filters, delegate to `HandlerExceptionResolver` manually.
- Don't expose internal messages, SQL, class names or stack traces (`server.error.include-stacktrace=never`, Boot default).
- Log server errors (5xx) at ERROR with stack trace; client errors (4xx) at WARN/DEBUG without stack trace noise.
- Include a trace/correlation id in error responses for support.
- WebFlux equivalents: `@ControllerAdvice` works; also `ErrorWebExceptionHandler`.

### 15. Testing error handling
```java
mockMvc.perform(post("/api/v1/products").contentType(APPLICATION_JSON).content("{\"name\":\"\"}"))
    .andExpect(status().isBadRequest())
    .andExpect(jsonPath("$.title").value("Validation Failed"))
    .andExpect(jsonPath("$.errors[0].field").value("name"));
```

## 16. Interview Q&A
**Q. `@Valid` vs `@Validated`?** `@Valid` (Jakarta) triggers validation & cascades into nested objects; `@Validated` (Spring) supports validation groups and enables method-level validation on beans.

**Q. `@NotNull` vs `@NotEmpty` vs `@NotBlank`?** NotNull: not null. NotEmpty: not null and not empty (strings/collections). NotBlank: string not null and has non-whitespace.

**Q. How do you implement global exception handling?** `@RestControllerAdvice` with `@ExceptionHandler` methods returning `ProblemDetail`/`ResponseEntity`, optionally extending `ResponseEntityExceptionHandler`.

**Q. How to create a custom validator?** Annotation with `@Constraint(validatedBy=...)` + `ConstraintValidator` implementation; can inject Spring beans.

**Q. Why aren't exceptions thrown in filters handled by @ControllerAdvice?** Filters run before DispatcherServlet; advice only covers handler execution.

**Q. What is ProblemDetail?** Spring 6's implementation of RFC 9457 standard error response body.
