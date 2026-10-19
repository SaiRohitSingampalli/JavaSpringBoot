# 03 — Object-Oriented Programming in Depth

## 1. Classes and objects

```java
public class BankAccount {
    // ---- fields (state) ----
    private final String accountNumber;     // immutable after construction
    private BigDecimal balance = BigDecimal.ZERO;
    private static int totalAccounts = 0;   // shared by all instances

    // ---- constructor ----
    public BankAccount(String accountNumber) {
        this.accountNumber = Objects.requireNonNull(accountNumber);
        totalAccounts++;
    }

    // ---- behaviour ----
    public void deposit(BigDecimal amount) {
        if (amount.signum() <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance = balance.add(amount);
    }

    public BigDecimal getBalance() { return balance; }
    public static int getTotalAccounts() { return totalAccounts; }
}
```

### Object creation: what happens on `new BankAccount("A1")`
1. Class loaded & initialized if needed (static fields/blocks).
2. Memory allocated on heap; fields set to defaults (0/null/false).
3. Instance initializers & field initializers run (in textual order).
4. Constructor body runs (after implicit/explicit `super(...)` call chain up to `Object`).
5. Reference returned.

### Initialization order (with inheritance)
```
Parent static blocks → Child static blocks          (once, at class init)
Parent instance init + Parent constructor
Child instance init + Child constructor             (each new object)
```

---

## 2. Constructors in detail
- Default constructor generated **only if no constructor is declared**.
- First statement is always `this(...)` or `super(...)`; if omitted, compiler inserts `super()`. If the parent has no no-arg constructor → compile error.
- Constructors are not inherited and not overridden.
- Calling overridable methods from a constructor is dangerous: the subclass override runs before the subclass fields are initialized.

```java
class Parent { Parent() { init(); } void init() {} }
class Child extends Parent {
    private String name = "child";
    @Override void init() { System.out.println(name.length()); }  // NPE! name not set yet
}
```

### Builder pattern for many parameters
```java
public class Pizza {
    private final String size; private final boolean cheese; private final boolean olives;
    private Pizza(Builder b) { size = b.size; cheese = b.cheese; olives = b.olives; }
    public static Builder builder(String size) { return new Builder(size); }

    public static class Builder {
        private final String size; private boolean cheese; private boolean olives;
        private Builder(String size) { this.size = size; }
        public Builder cheese() { cheese = true; return this; }
        public Builder olives() { olives = true; return this; }
        public Pizza build() { return new Pizza(this); }
    }
}
Pizza p = Pizza.builder("L").cheese().olives().build();
```
(Lombok `@Builder` generates this.)

---

## 3. Encapsulation
**Definition**: bundle data + methods that operate on it, and restrict direct access to internal state.

Good encapsulation is **not** "private field + public getter + public setter for everything" — that's just public fields with extra steps. Expose **behaviour** (`account.deposit(x)`) rather than raw setters (`account.setBalance(...)`), and enforce invariants inside.

Benefits: invariants protected, implementation can change, easier debugging (one place mutates state), thread-safety easier.

---

## 4. Inheritance

```java
public class Employee {
    protected final String name;
    protected BigDecimal baseSalary;
    public Employee(String name, BigDecimal baseSalary) { this.name = name; this.baseSalary = baseSalary; }
    public BigDecimal monthlyPay() { return baseSalary.divide(BigDecimal.valueOf(12), 2, RoundingMode.HALF_UP); }
}

public class Manager extends Employee {
    private final BigDecimal bonus;
    public Manager(String name, BigDecimal salary, BigDecimal bonus) {
        super(name, salary);
        this.bonus = bonus;
    }
    @Override
    public BigDecimal monthlyPay() { return super.monthlyPay().add(bonus); }
}
```

### Types of inheritance in Java
- Single (A → B) ✔, multilevel (A → B → C) ✔, hierarchical (A → B, A → C) ✔
- Multiple inheritance of **classes** ✘ (diamond problem) — but multiple inheritance of **type/behaviour** via interfaces ✔.

### Diamond problem with default methods
```java
interface A { default void hi() { System.out.println("A"); } }
interface B { default void hi() { System.out.println("B"); } }
class C implements A, B {
    public void hi() { A.super.hi(); }   // must override and choose explicitly
}
```
Rules: class method wins over interface default; more specific interface wins; otherwise must override.

### `super` keyword
- `super.method()` calls parent version; `super(...)` calls parent constructor; `super.field` accesses hidden parent field.

### Fields are not polymorphic (hiding)
```java
class P { String name = "P"; }
class C extends P { String name = "C"; }
P obj = new C();
obj.name;   // "P" — fields resolved by reference type
```

### Static methods are hidden, not overridden
```java
P.staticMethod() vs C.staticMethod() — call resolved by reference type at compile time.
```

### Composition over inheritance
Inheritance creates tight coupling: changes in the parent break children ("fragile base class"), and you inherit everything including things you don't want (e.g., `Stack extends Vector` exposes `insertElementAt`).

```java
// Instead of class LoggingList extends ArrayList — wrap it:
class LoggingList<E> {
    private final List<E> delegate = new ArrayList<>();
    public boolean add(E e) { log(e); return delegate.add(e); }
}
```
Use inheritance only for true "is-a" relationships where the subclass is substitutable (Liskov).

---

## 5. Polymorphism

### 5.1 Compile-time (static) — overloading
Same method name, different parameter lists; resolved by the compiler from the **reference/argument types**.

### 5.2 Runtime (dynamic) — overriding
Method chosen at runtime based on the **actual object** (virtual method dispatch via vtable).

```java
List<Shape> shapes = List.of(new Circle(2), new Square(3));
for (Shape s : shapes) System.out.println(s.area());   // each calls its own area()
```
This is what lets Spring inject `PaymentGateway` and you don't care whether it's `StripeGateway` or `RazorpayGateway`.

### 5.3 Overriding rules (memorize)
| Rule | Detail |
|------|--------|
| Signature | Same name & parameter types |
| Return type | Same or **covariant** (subtype) |
| Access | Same or **wider** (protected → public OK; public → protected ✘) |
| Checked exceptions | Same, narrower, or none — not broader/new |
| Can't override | `final`, `static` (hidden instead), `private` (not inherited) methods, constructors |
| `@Override` | Optional but always use — compiler catches typos |

### 5.4 Upcasting & downcasting
```java
Animal a = new Dog();          // upcast — implicit, always safe
Dog d = (Dog) a;               // downcast — explicit; ClassCastException if wrong
if (a instanceof Dog dog) dog.bark();   // safe pattern
```

---

## 6. Abstraction

### 6.1 Abstract classes
```java
public abstract class Notification {
    private final String recipient;
    protected Notification(String recipient) { this.recipient = recipient; }

    // Template method: fixed algorithm, customizable steps
    public final void send() {
        validate();
        String body = buildBody();
        deliver(recipient, body);
        audit();
    }
    protected void validate() { if (recipient.isBlank()) throw new IllegalStateException(); }
    protected abstract String buildBody();
    protected abstract void deliver(String to, String body);
    private void audit() { System.out.println("sent to " + recipient); }
}

public class EmailNotification extends Notification {
    public EmailNotification(String to) { super(to); }
    protected String buildBody() { return "<html>...</html>"; }
    protected void deliver(String to, String body) { /* SMTP */ }
}
```
- Cannot be instantiated; may have constructors, fields, concrete methods.
- A class with any abstract method must be abstract. Abstract class may have zero abstract methods (prevents instantiation).

### 6.2 Interfaces (evolution)
| Java version | Interface can contain |
|-------------|----------------------|
| ≤7 | abstract methods, constants |
| 8 | + `default` methods, `static` methods |
| 9 | + `private` methods (helpers for defaults) |
| 17 | + `sealed` / `permits` |

```java
public interface PaymentGateway {
    String PROVIDER_HEADER = "X-Provider";              // public static final
    PaymentResult charge(Money amount, Card card);      // public abstract
    default boolean supportsRefunds() { return true; }
    static PaymentGateway noop() { return (amt, card) -> PaymentResult.ok(); }
}
```

### 6.3 Functional interfaces
Exactly one abstract method → can be implemented by a lambda. Mark with `@FunctionalInterface`.

### 6.4 Marker interfaces
No methods; tag a capability: `Serializable`, `Cloneable`, `RandomAccess`. Modern code often uses annotations instead.

### 6.5 Abstract class vs interface — decision guide
| Question | Choose |
|----------|--------|
| Need shared state / constructor logic? | Abstract class |
| Unrelated classes share a capability (Comparable, Runnable)? | Interface |
| Need multiple inheritance of type? | Interface |
| Defining an API boundary for DI (Spring)? | Interface |
| Template method with protected hooks? | Abstract class |

---

## 7. `Object` class methods

| Method | Notes |
|--------|-------|
| `equals(Object)` | Default = reference equality |
| `hashCode()` | Default = identity-based |
| `toString()` | Default = `ClassName@hexHash` — override for logs |
| `getClass()` | Runtime class |
| `clone()` | Shallow copy; requires `Cloneable`; broken design — prefer copy constructors |
| `finalize()` | Deprecated for removal — use try-with-resources / `Cleaner` |
| `wait()`, `notify()`, `notifyAll()` | Thread coordination on the object's monitor |

### 7.1 equals contract
Reflexive (`x.equals(x)`), symmetric, transitive, consistent, `x.equals(null) == false`.

### 7.2 hashCode contract
Equal objects **must** have equal hash codes. Unequal objects *may* collide.

### 7.3 What breaks if you override only `equals`?
```java
Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));
set.contains(new Point(1, 2));  // false! Different hash buckets
```

### 7.4 Correct implementation
```java
public final class Point {
    private final int x, y;
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
    @Override public String toString() { return "Point[x=%d, y=%d]".formatted(x, y); }
}
```
**JPA entity tip**: don't base equals/hashCode on a generated `@Id` that's null before persist; use a business key or the Hibernate-recommended pattern (`id != null && id.equals(other.id)` with constant hashCode `getClass().hashCode()`).

### 7.5 Shallow vs deep copy
- Shallow: copies field values; referenced objects shared.
- Deep: recursively copies referenced objects. Use copy constructors or serialization libs.

---

## 8. `static`, `final`, `this`

### static
- Static fields: class-level, one copy. Static methods: no `this`, can't be overridden.
- Static nested classes don't need an outer instance.
- Static init block runs once on class init.
- Overuse = global mutable state = hard to test (Spring beans replace most static usage).

### final
| On | Effect |
|----|--------|
| variable | Assign once (reference can't change; object can still mutate!) |
| method | Can't override |
| class | Can't extend (`String`, `Integer`, records) |
| parameter | Can't reassign inside method |

**Effectively final**: a local variable never reassigned — can be captured by lambdas/inner classes.

---

## 9. Nested classes

| Type | Declared | Access to outer instance | Use |
|------|----------|--------------------------|-----|
| Static nested | `static class X` inside class | No | Builders, helper types (`Map.Entry`) |
| Inner (member) | `class X` inside class | Yes (`Outer.this`) | Iterators tightly bound to outer |
| Local | Inside a method | Yes + effectively final locals | Rare |
| Anonymous | `new Interface() { ... }` | Yes | Pre-Java-8 callbacks; now mostly lambdas |

Prefer **static nested** unless you need the outer instance — inner classes hold a hidden reference that can cause memory leaks.

---

## 10. Enums in depth

```java
public enum OrderStatus {
    NEW("N") {
        @Override public OrderStatus next() { return PAID; }
    },
    PAID("P") {
        @Override public OrderStatus next() { return SHIPPED; }
    },
    SHIPPED("S") {
        @Override public OrderStatus next() { return DELIVERED; }
    },
    DELIVERED("D") {
        @Override public OrderStatus next() { throw new IllegalStateException("final state"); }
    };

    private final String code;
    OrderStatus(String code) { this.code = code; }
    public String getCode() { return code; }
    public abstract OrderStatus next();          // constant-specific behaviour

    private static final Map<String, OrderStatus> BY_CODE =
        Arrays.stream(values()).collect(Collectors.toMap(OrderStatus::getCode, s -> s));
    public static OrderStatus fromCode(String c) { return BY_CODE.get(c); }
}
```
- Enums extend `java.lang.Enum` implicitly; can implement interfaces.
- `values()`, `valueOf(String)`, `ordinal()`, `name()`, `compareTo()`.
- `EnumMap` / `EnumSet` are extremely efficient (array/bit-vector based).
- **JPA**: use `@Enumerated(EnumType.STRING)` — `ORDINAL` breaks when you reorder constants.
- Enum singleton: `enum Registry { INSTANCE; }` — thread-safe, serialization-safe, reflection-safe.

---

## 11. Records (Java 16)

```java
public record Money(BigDecimal amount, String currency) {
    public Money {                                    // compact canonical constructor
        Objects.requireNonNull(amount);
        if (amount.signum() < 0) throw new IllegalArgumentException("negative");
        currency = currency.toUpperCase();            // can reassign params before fields set
    }
    public Money add(Money o) {                       // extra methods allowed
        if (!currency.equals(o.currency)) throw new IllegalArgumentException();
        return new Money(amount.add(o.amount), currency);
    }
    public static Money inr(String v) { return new Money(new BigDecimal(v), "INR"); }
}
```
- Implicitly `final`, fields `private final`, accessors `amount()` (not `getAmount()`), auto `equals/hashCode/toString`.
- Can implement interfaces; can't extend classes; can't declare instance fields beyond components.
- **Shallow immutability**: a `List` component can still be mutated → use `List.copyOf()` in the constructor.
- Great for DTOs, API requests/responses, value objects, map keys, `@ConfigurationProperties`. **Not** for JPA entities (entities need no-arg constructor, mutability, proxies).

**Record patterns (Java 21)**
```java
if (obj instanceof Money(BigDecimal amt, String cur) && cur.equals("INR")) { ... }
```

---

## 12. Sealed types (Java 17)

```java
public sealed interface PaymentResult permits Success, Failure, Pending {}
public record Success(String txnId) implements PaymentResult {}
public record Failure(String reason) implements PaymentResult {}
public final class Pending implements PaymentResult {}

String message(PaymentResult r) {
    return switch (r) {               // compiler checks exhaustiveness; no default needed
        case Success s -> "Paid: " + s.txnId();
        case Failure f -> "Failed: " + f.reason();
        case Pending p -> "Waiting";
    };
}
```
Permitted subclasses must be `final`, `sealed`, or `non-sealed`. Models closed domain hierarchies (algebraic data types) — adding a new subtype forces you to handle it everywhere.

---

## 13. Immutability

Recipe:
1. Class `final` (or private constructor + factory).
2. All fields `private final`.
3. No setters.
4. Defensive copy mutable inputs and outputs.
```java
public final class Team {
    private final List<String> members;
    public Team(List<String> members) { this.members = List.copyOf(members); }
    public List<String> members() { return members; }    // already unmodifiable
}
```
Benefits: thread-safe without locks, safe as map keys, easy reasoning, cacheable.

---

## 14. SOLID with examples

**S — Single Responsibility**: `OrderService` shouldn't also send emails and generate PDFs. Split into `OrderService`, `EmailService`, `InvoiceGenerator`.

**O — Open/Closed**: add new discount types without editing existing code:
```java
interface DiscountPolicy { BigDecimal apply(Order o); }
class FestiveDiscount implements DiscountPolicy { ... }
class LoyaltyDiscount implements DiscountPolicy { ... }
// Spring injects List<DiscountPolicy>; adding a class adds behaviour.
```

**L — Liskov Substitution**: subtypes must honour the parent contract. Classic violation: `Square extends Rectangle` where `setWidth` also changes height → code expecting a Rectangle breaks.

**I — Interface Segregation**: don't force `ReadOnlyRepository` implementers to implement `delete()`. Split `Readable`/`Writable` interfaces.

**D — Dependency Inversion**: high-level modules depend on abstractions; details injected:
```java
class OrderService {
    private final PaymentGateway gateway;           // interface
    OrderService(PaymentGateway gateway) { this.gateway = gateway; }
}
```
This principle is exactly what Spring's IoC container automates.

### Other principles
- **DRY** (Don't Repeat Yourself), **KISS**, **YAGNI**.
- **Law of Demeter**: talk to friends, not strangers (`order.getCustomer().getAddress().getCity()` is a smell).
- **Tell, don't ask**.
- **High cohesion, low coupling**.

---

## 15. Coupling and cohesion
- **Coupling**: how much one class depends on another's internals. Aim low (interfaces, DI, events).
- **Cohesion**: how focused a class is on a single purpose. Aim high.

---

## 16. Interview Q&A

**Q. What are the four pillars of OOP?** Encapsulation, Abstraction, Inheritance, Polymorphism — explain each with a domain example (bank account, notifications, employees, shapes/payment gateways).

**Q. Can we override static or private methods?** No. Static methods are hidden (resolved at compile time); private methods aren't visible to subclasses.

**Q. Can an abstract class have a constructor?** Yes — called via `super()` from subclasses to initialize shared state.

**Q. Can an interface have a constructor or state?** No constructor, no instance state; only constants.

**Q. Why is multiple inheritance not supported for classes?** Diamond problem — ambiguity in which state/implementation to inherit. Interfaces allow multiple type inheritance; default-method conflicts must be resolved explicitly.

**Q. What happens if hashCode is not overridden but equals is?** Hash-based collections fail to find logically equal objects; duplicates appear.

**Q. Records vs Lombok @Data?** Records are language-level immutable carriers with no setters; @Data generates mutable getters/setters. Use records for DTOs, classes for JPA entities.

**Q. What is a covariant return type?** Overriding method returning a subtype of the parent's return type (e.g., `clone()` returning `Employee` instead of `Object`).

**Q. Composition vs aggregation vs association?** Association: general relationship. Aggregation: has-a with independent lifecycles (Department–Professor). Composition: strong ownership, part dies with whole (House–Room).

**Q. Explain Liskov with a real example.** A `ReadOnlyList` subclass of `List` throwing on `add()` breaks callers expecting a fully functional List — violates LSP.
