# 02 — Java Language Basics (in depth)

## 1. Program structure

```java
package com.example.app;                 // namespace; matches folder com/example/app

import java.util.List;                   // import a class
import static java.lang.Math.max;        // static import

public class Calculator {                // file must be Calculator.java

    private static final double PI = 3.14159;   // constant

    public static void main(String[] args) {    // entry point
        System.out.println(max(3, 7));
    }
}
```

- One **public top-level class** per file; file name = class name.
- `main` must be `public static void main(String[] args)` (Java 25 also allows instance `void main()` in compact source files).
- Run a single file directly (Java 11+): `java Calculator.java`.

### Naming conventions
| Element | Convention | Example |
|---------|-----------|---------|
| Class/Interface/Record/Enum | PascalCase | `OrderService` |
| Method/variable | camelCase | `calculateTotal` |
| Constant | UPPER_SNAKE | `MAX_RETRIES` |
| Package | lowercase reverse domain | `com.company.orders` |

---

## 2. Data types

### 2.1 Primitives (8)
| Type | Bits | Range | Literal |
|------|------|-------|---------|
| byte | 8 | -128..127 | `(byte) 10` |
| short | 16 | -32,768..32,767 | |
| int | 32 | -2³¹..2³¹-1 (~±2.1 billion) | `10`, `0x1F`, `0b1010`, `1_000_000` |
| long | 64 | -2⁶³..2⁶³-1 | `10L` |
| float | 32 | ~7 decimal digits | `3.14f` |
| double | 64 | ~15-16 digits | `3.14`, `1e-3` |
| char | 16 | 0..65,535 (UTF-16 code unit) | `'A'`, `'A'` |
| boolean | — | true/false | |

### 2.2 Reference types
Everything else: classes, interfaces, arrays, enums, records. Variable holds a reference (or `null`).

### 2.3 Integer overflow — silent!
```java
int max = Integer.MAX_VALUE;
System.out.println(max + 1);              // -2147483648 (wraps around)
Math.addExact(max, 1);                    // throws ArithmeticException — use for money/counters
```

### 2.4 Floating point
- IEEE 754 binary → can't represent 0.1 exactly. `0.1 + 0.2 == 0.30000000000000004`.
- Never use `float/double` for money → **`BigDecimal`**:
```java
BigDecimal price = new BigDecimal("19.99");          // String constructor! new BigDecimal(0.1) is inexact
BigDecimal total = price.multiply(BigDecimal.valueOf(3))
                        .setScale(2, RoundingMode.HALF_UP);
price.compareTo(other) == 0;   // use compareTo, NOT equals ("2.0".equals("2.00") is false due to scale)
```
- Special values: `Double.NaN`, `Infinity`. `NaN != NaN` → use `Double.isNaN()`.
- Integer division by zero → `ArithmeticException`; double division by zero → `Infinity`.

### 2.5 Type conversion
- **Widening** (automatic): `byte → short → int → long → float → double`, `char → int`.
- **Narrowing** (explicit cast, may lose data): `(int) 3.99` → 3, `(byte) 200` → -56.
- **Promotion in expressions**: `byte + byte` → `int`. So `byte b = b1 + b2;` doesn't compile; `b += 1` does (compound ops include implicit cast).

### 2.6 `var` (Java 10)
```java
var list = new ArrayList<String>();     // inferred ArrayList<String>
var stream = list.stream();
```
- Only for **local variables** with initializer (also in for-loops, lambdas params from Java 11).
- Not for fields, method params, return types. Still statically typed.

---

## 3. Operators & precedence

Precedence (high → low, abbreviated): postfix `x++` → unary `++x ! ~` → `* / %` → `+ -` → shifts → relational `< > instanceof` → `== !=` → `&` → `^` → `|` → `&&` → `||` → ternary `?:` → assignment.

```java
int i = 5;
int a = i++ + ++i;   // 5 + 7 = 12, i = 7  (don't write code like this!)

boolean safe = obj != null && obj.isValid();   // && short-circuits — prevents NPE
int flags = READ | WRITE;                       // bitwise OR for bitmasks
boolean canRead = (flags & READ) != 0;
int half = n >> 1;    // arithmetic shift (keeps sign);  >>> logical shift (fills 0)
```

**`instanceof` with pattern matching (Java 16)**
```java
if (obj instanceof String s && !s.isBlank()) {
    System.out.println(s.length());   // s already cast
}
```

---

## 4. Control flow

### 4.1 if / else, ternary
```java
String category = age < 13 ? "child" : age < 18 ? "teen" : "adult";
```

### 4.2 switch — three generations
```java
// 1) Classic statement (falls through without break!)
switch (status) {
    case "NEW":
    case "PENDING":
        process(); break;
    default:
        log();
}

// 2) Arrow form (Java 14) — no fall-through
switch (status) {
    case "NEW", "PENDING" -> process();
    default -> log();
}

// 3) Switch expression with yield
int score = switch (grade) {
    case 'A' -> 90;
    case 'B' -> 80;
    default -> {
        log("unknown");
        yield 0;            // yield returns a value from a block
    }
};

// 4) Pattern matching switch (Java 21) — types, null, guards
String describe(Object o) {
    return switch (o) {
        case null -> "null!";
        case Integer i when i > 100 -> "big int";
        case Integer i -> "int " + i;
        case String s -> "string of length " + s.length();
        default -> "something else";
    };
}
```
Switch works on: `int` and smaller, `char`, their wrappers, `String`, enums, and (Java 21) any type via patterns.

### 4.3 Loops
```java
for (int i = 0, j = 10; i < j; i++, j--) { }
for (var item : items) { }            // enhanced — Iterable or array
while (iterator.hasNext()) { }
do { input = read(); } while (!input.equals("q"));

outer:
for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
        if (j == 3) continue outer;     // jumps to next i
        if (i == 4) break outer;        // exits both loops
    }
}
```

---

## 5. Methods

### 5.1 Anatomy
```java
public static <T> List<T> repeat(T value, int times) throws IllegalArgumentException { ... }
// modifiers  generic  return     name   parameters          checked exceptions
```
**Signature** = name + parameter types (not return type, not exceptions).

### 5.2 Overloading resolution order
1. Exact match → 2. Widening primitive → 3. Autoboxing → 4. Varargs.
```java
void f(long x) {}  void f(Integer x) {}  void f(int... x) {}
f(5);   // calls f(long) — widening beats boxing
```

### 5.3 Pass-by-value (always)
```java
void change(Person p) {
    p.setName("B");          // visible to caller — same object
    p = new Person("C");     // NOT visible — only local copy of reference changed
}
void inc(int x) { x++; }     // caller's int unchanged
```

### 5.4 Varargs
```java
static int sum(int... nums) { int s = 0; for (int n : nums) s += n; return s; }
sum(); sum(1); sum(1, 2, 3); sum(new int[]{1, 2});
```
Must be the last parameter; only one per method.

### 5.5 Recursion
```java
long factorial(int n) { return n <= 1 ? 1 : n * factorial(n - 1); }
```
Each call adds a stack frame; no tail-call optimization in Java → deep recursion → `StackOverflowError`. Convert to iteration for big inputs.

---

## 6. Strings in depth

### 6.1 Immutability and the String pool
```java
String a = "hello";                 // pool
String b = "hello";                 // same pooled object
String c = new String("hello");     // new heap object
a == b;          // true
a == c;          // false
a.equals(c);     // true  ← always use equals
c.intern() == a; // true
```
Why immutable: security (class names, URLs, DB credentials can't be changed after validation), safe sharing across threads, pool reuse, cached `hashCode` (great as HashMap keys).

Since Java 9 strings use **compact strings** (byte[] with LATIN-1 when possible → half memory).

### 6.2 Concatenation
```java
String s = "a" + "b";      // compile-time constant → "ab"
// In loops this creates many objects:
String r = ""; for (...) r += x;            // O(n²) — bad
StringBuilder sb = new StringBuilder();      // O(n) — good
for (...) sb.append(x).append(',');
String result = sb.toString();
```
Java 9+ compiles `+` with `invokedynamic` (`StringConcatFactory`) — efficient for single expressions, but loops still need StringBuilder.

| | String | StringBuilder | StringBuffer |
|--|--|--|--|
| Mutable | No | Yes | Yes |
| Thread-safe | Yes (immutable) | No | Yes (synchronized) |
| Speed | — | Fastest | Slower |

### 6.3 Must-know API
```java
s.length(); s.charAt(i); s.indexOf("x"); s.lastIndexOf('x');
s.substring(2); s.substring(2, 5);          // end exclusive
s.equalsIgnoreCase(t); s.compareTo(t);       // lexicographic
s.startsWith("ab"); s.endsWith(".java"); s.contains("x");
s.trim();      // removes chars <= ' '
s.strip();     // Unicode-aware whitespace (Java 11)
s.isEmpty(); s.isBlank();
s.toLowerCase(); s.toUpperCase();
s.replace('a','b'); s.replaceAll("\\d+", "#");  // regex
s.split(",");  s.split("\\s+");
String.join("-", "a", "b"); String.valueOf(42);
s.chars();  s.lines();  s.repeat(3);
s.toCharArray(); new String(chars);
"Name: %s, Age: %d, Score: %.2f".formatted(name, age, score);
```

### 6.4 Text blocks (Java 15)
```java
String sql = """
    SELECT id, name
    FROM users
    WHERE status = 'ACTIVE'
    """;
```
Indentation stripped based on the closing `"""`. `\` at end of line joins lines; `\s` keeps a trailing space.

### 6.5 Regex quick reference
```java
Pattern p = Pattern.compile("^[\\w.+-]+@[\\w-]+\\.[a-z]{2,}$");   // compile once, reuse
boolean valid = p.matcher(email).matches();
Matcher m = Pattern.compile("(\\d{3})-(\\d{4})").matcher(text);
while (m.find()) { m.group(1); }
```

---

## 7. Wrapper classes & autoboxing
| Primitive | Wrapper |
|-----------|---------|
| int | Integer |
| char | Character |
| others | Byte, Short, Long, Float, Double, Boolean |

```java
Integer a = 100;              // Integer.valueOf(100) — autoboxing
int b = a;                    // a.intValue() — unboxing
Integer.parseInt("42");       // → int
Integer.valueOf("42");        // → Integer (cached)
Integer.MAX_VALUE; Integer.toBinaryString(10); Character.isDigit('5');
```
**Pitfalls**
1. Cache: `Integer.valueOf` caches **-128..127**. `Integer x=1000, y=1000; x==y` → false.
2. `Integer count = map.get(key); int c = count;` → NPE if key missing.
3. Boxing in hot loops is slow: `Long sum = 0L; for(...) sum += i;` creates millions of objects.

---

## 8. Arrays

```java
int[] nums = new int[5];                  // [0,0,0,0,0]
String[] names = {"a", "b"};
int[][] grid = new int[3][4];
int[][] jagged = {{1}, {2, 3}, {4, 5, 6}};
nums.length;                              // field, not method

Arrays.sort(nums);                         // dual-pivot quicksort for primitives
Arrays.sort(names, Comparator.reverseOrder());
Arrays.binarySearch(nums, 3);              // array must be sorted
Arrays.fill(nums, -1);
int[] copy = Arrays.copyOf(nums, 10);      // pads with 0
int[] part = Arrays.copyOfRange(nums, 1, 3);
Arrays.equals(a, b); Arrays.deepEquals(m1, m2); Arrays.toString(nums); Arrays.deepToString(grid);
List<String> list = Arrays.asList(names);  // fixed-size view backed by array! add() throws
Arrays.stream(nums).sum();
```
- Arrays are **covariant**: `Object[] o = new String[1]; o[0] = 1;` → `ArrayStoreException` at runtime (generics avoid this).
- Index out of range → `ArrayIndexOutOfBoundsException`.

---

## 9. `java.time` (dates done right)
| Class | Represents | Example |
|-------|-----------|---------|
| `LocalDate` | Date, no time zone | birthday |
| `LocalTime` | Time | store opening hour |
| `LocalDateTime` | Date + time, no zone | — avoid for events |
| `ZonedDateTime` | Date-time with region zone | meeting in Asia/Kolkata |
| `OffsetDateTime` | Date-time with fixed offset | DB/JSON timestamps |
| `Instant` | Point on UTC timeline | store timestamps in DB |
| `Duration` | Time-based amount (hours, seconds) | timeout |
| `Period` | Date-based amount (years, months, days) | age |

```java
LocalDate d = LocalDate.of(2026, Month.OCTOBER, 19);
d.plusMonths(1).withDayOfMonth(1);
d.getDayOfWeek(); d.isLeapYear();
Period.between(birth, LocalDate.now()).getYears();
Instant now = Instant.now();
ZonedDateTime ist = now.atZone(ZoneId.of("Asia/Kolkata"));
DateTimeFormatter f = DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm");
LocalDateTime.parse("2026-10-19T10:15:30");
ChronoUnit.DAYS.between(d1, d2);
```
Best practice for backends: store **UTC `Instant`** (or `timestamptz`), convert to user zone only at the edge. All java.time classes are immutable & thread-safe (old `SimpleDateFormat` is NOT thread-safe).

---

## 10. Access modifiers & packages

| Modifier | Same class | Same package | Subclass (other pkg) | Everywhere |
|----------|:-:|:-:|:-:|:-:|
| private | ✔ | | | |
| (package-private) | ✔ | ✔ | | |
| protected | ✔ | ✔ | ✔ (via inheritance) | |
| public | ✔ | ✔ | ✔ | ✔ |

Top-level classes can only be `public` or package-private. Prefer the most restrictive access that works.

---

## 11. `Math`, `Random`, `Scanner` quick hits
```java
Math.max(a,b); Math.abs(-3); Math.pow(2,10); Math.sqrt(16); Math.round(2.5); Math.floorMod(-7, 3);
ThreadLocalRandom.current().nextInt(1, 101);    // prefer in multithreaded code
new SecureRandom();                             // for tokens/passwords
Scanner sc = new Scanner(System.in); int n = sc.nextInt();
```

---

## 12. Interview Q&A

**Q. Why is `String` immutable and final?** Security, thread safety, string pool, hashCode caching. `final` stops subclasses from breaking immutability.

**Q. How many objects does `new String("abc")` create?** Up to two: "abc" literal in the pool (if not already there) + a new heap object.

**Q. `==` vs `equals`?** `==` compares references (or primitive values); `equals` compares logical content (if overridden).

**Q. Why use BigDecimal for money?** Binary floating point can't represent most decimal fractions exactly; BigDecimal stores exact decimal values with explicit rounding.

**Q. What does `Integer a = 127, b = 127; a == b` print? And with 128?** true, false — Integer cache range -128..127.

**Q. Is Java pass-by-reference?** No, always pass-by-value; for objects, the value is the reference.

**Q. Difference between `trim()` and `strip()`?** `trim` removes ASCII ≤ space; `strip` uses Unicode whitespace definition.

**Q. Why is `SimpleDateFormat` dangerous in a Spring singleton?** It holds mutable state and is not thread-safe; use `DateTimeFormatter`.
