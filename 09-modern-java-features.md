# 09 — Modern Java: Features from Java 8 to 25

> LTS releases: **8, 11, 17, 21, 25**. Spring Boot 3 requires **17+**; most new projects use **21** (virtual threads) or **25**.
> Release cadence: a new version every 6 months (March & September); LTS every 2 years.

## Java 8 (2014) — the big one
| Feature | Example |
|---------|---------|
| Lambdas & functional interfaces | `list.forEach(x -> ...)` |
| Streams API | `list.stream().filter(...).toList()` |
| Method references | `String::length` |
| Default & static interface methods | `default void log() {}` |
| `Optional` | `Optional.ofNullable(x)` |
| `java.time` | `LocalDate.now()` |
| `CompletableFuture` | async pipelines |
| Nashorn JS engine (removed in 15) | |
| Metaspace replaces PermGen | |
| `String.join`, `Base64`, `StampedLock`, `LongAdder` | |

## Java 9 (2017)
- **Module system (JPMS / Project Jigsaw)** — `module-info.java`:
  ```java
  module com.example.orders {
      requires java.sql;
      exports com.example.orders.api;      // only this package is visible to others
      opens com.example.orders.model;       // allow reflection (e.g., Jackson/Hibernate)
  }
  ```
  Strong encapsulation of JDK internals; `jlink` creates trimmed runtimes. Most Spring apps still use the classpath, not modules.
- **JShell** REPL.
- Collection factories: `List.of`, `Set.of`, `Map.of`, `Map.ofEntries`.
- Private interface methods.
- Stream: `takeWhile`, `dropWhile`, `iterate` with predicate, `ofNullable`.
- Optional: `ifPresentOrElse`, `or`, `stream`.
- `Process` API improvements, reactive `Flow` API, `try-with-resources` on effectively final variables.
- G1 becomes default GC. Compact Strings.

## Java 10 (2018)
- **`var`** local variable type inference.
- `List.copyOf`, `Collectors.toUnmodifiableList`, `Optional.orElseThrow()`.
- Container awareness (respects Docker CPU/memory limits).

## Java 11 (2018, LTS)
- `String`: `isBlank`, `strip`, `stripLeading/Trailing`, `lines`, `repeat`.
- `Files.readString`, `Files.writeString`.
- **HTTP Client** (`java.net.http`) — standard, supports HTTP/2 and async:
  ```java
  HttpClient client = HttpClient.newHttpClient();
  HttpRequest req = HttpRequest.newBuilder(URI.create("https://api.example.com/users"))
      .header("Accept", "application/json").GET().build();
  HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
  client.sendAsync(req, BodyHandlers.ofString()).thenApply(HttpResponse::body);
  ```
- Run single-file programs: `java Hello.java`.
- `var` in lambda parameters (for annotations): `(@NonNull var x) -> ...`.
- `Predicate.not`, `Collection.toArray(IntFunction)`.
- Removed Java EE modules (JAXB, JAX-WS) & CORBA; JavaFX unbundled.
- Epsilon & ZGC (experimental), Flight Recorder open-sourced.

## Java 12–13
- Switch expressions (preview), `Collectors.teeing`, `String.indent/transform`, `CompactNumberFormat`.
- Text blocks (preview in 13).

## Java 14 (2020)
- **Switch expressions** standard (`->`, `yield`).
- **Helpful NullPointerExceptions** (tells which variable was null).
- Records & pattern matching `instanceof` (preview). CMS GC removed.

## Java 15
- **Text blocks** standard.
- Sealed classes (preview), hidden classes, ZGC & Shenandoah production-ready. Nashorn removed.

## Java 16 (2021)
- **Records** standard.
- **Pattern matching for `instanceof`** standard: `if (o instanceof String s)`.
- `Stream.toList()`, `Stream.mapMulti`.
- Strong encapsulation of JDK internals by default.
- Unix domain sockets, `jpackage`.

## Java 17 (2021, LTS) — Spring Boot 3 baseline
- **Sealed classes/interfaces** standard.
- Pattern matching for switch (preview).
- `RandomGenerator` API, enhanced pseudo-random generators.
- Strongly encapsulated JDK internals (no `--illegal-access`).
- Deprecated Security Manager & Applet API; removed RMI activation, AOT/Graal JIT experimental.
- macOS AArch64 port.

## Java 18–20
- UTF-8 by default (18) — `Charset.defaultCharset()` is UTF-8 everywhere.
- Simple web server: `jwebserver` (18).
- `@snippet` in Javadoc.
- Record patterns, virtual threads, structured concurrency, scoped values in preview/incubator.

## Java 21 (2023, LTS) — big release
- **Virtual threads** (JEP 444) standard.
- **Record patterns** (JEP 440):
  ```java
  record Point(int x, int y) {}
  if (obj instanceof Point(int x, int y)) System.out.println(x + y);
  ```
- **Pattern matching for switch** (JEP 441) standard — type patterns, `case null`, guards with `when`, exhaustiveness with sealed types.
  ```java
  String describe(Shape s) {
      return switch (s) {
          case Circle c when c.radius() > 10 -> "big circle";
          case Circle c -> "circle";
          case Rectangle(var w, var h) when w == h -> "square";
          case Rectangle r -> "rectangle";
      };
  }
  ```
- **Sequenced collections** (JEP 431): `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `reversed()` on `List`, `Deque`, `LinkedHashSet`, `SortedSet`; `SequencedMap` (`firstEntry`, `pollLastEntry`, `putFirst`).
- **Generational ZGC**.
- Key Encapsulation Mechanism API.
- Previews: string templates (later withdrawn), unnamed patterns/variables `_`, unnamed classes & instance main, structured concurrency, scoped values.

## Java 22 (2024)
- **Unnamed variables & patterns** `_` standard:
  ```java
  try { ... } catch (NumberFormatException _) { return 0; }
  map.forEach((_, v) -> sum += v);
  case Point(var x, _) -> x;
  ```
- **Foreign Function & Memory API** standard (Project Panama) — call native code without JNI.
- Launch multi-file source programs: `java Main.java` compiles referenced files.
- Statements before `super(...)` (preview).
- Stream gatherers (preview).

## Java 23 (2024)
- Markdown in Javadoc comments (`///`).
- Generational mode default for ZGC.
- Previews: primitive types in patterns, module import declarations, flexible constructor bodies, structured concurrency, scoped values.

## Java 24 (2025)
- **Stream Gatherers** standard (JEP 485) — custom intermediate operations:
  ```java
  Stream.of(1,2,3,4,5,6).gather(Gatherers.windowFixed(2)).toList();     // [[1,2],[3,4],[5,6]]
  Stream.of(1,2,3,4).gather(Gatherers.windowSliding(2)).toList();        // [[1,2],[2,3],[3,4]]
  ```
- **Virtual threads no longer pin on `synchronized`** (JEP 491).
- Class-File API standard.
- Ahead-of-Time class loading & linking (JEP 483, Project Leyden) — faster startup with an AOT cache.
- Quantum-resistant crypto (ML-KEM, ML-DSA). Security Manager permanently disabled.

## Java 25 (September 2025, LTS)
- **Compact source files & instance main methods** (JEP 512) — beginner-friendly:
  ```java
  void main() {
      IO.println("Hello, World!");      // java.lang.IO helper
  }
  ```
- **Module import declarations** (JEP 511): `import module java.base;`
- **Flexible constructor bodies** (JEP 513) — statements (validation) before `super(...)`:
  ```java
  public Employee(String name, int age) {
      if (age < 18) throw new IllegalArgumentException();   // allowed before super now
      super(name);
  }
  ```
- **Scoped values** (JEP 506) final — immutable, inheritable alternative to ThreadLocal, ideal for virtual threads.
- **Compact object headers** (JEP 519) — smaller objects (product feature, opt-in).
- AOT command-line ergonomics & method profiling (Leyden), JFR improvements, Generational Shenandoah.
- Structured concurrency still in preview; primitive patterns preview.

---

## Migration notes for backend devs
**Java 8 → 11/17**
- `javax.xml.bind` (JAXB) removed → add dependency `jakarta.xml.bind-api`.
- Illegal reflective access blocked → upgrade libraries (Lombok, Mockito, ByteBuddy, old Hibernate).
- `--add-opens` as temporary workaround.

**Spring Boot 2 → 3**
- Java 17 baseline; `javax.*` → `jakarta.*` namespace (Servlet, JPA, Validation, Annotation).
- Use OpenRewrite recipes / Spring Boot Migrator to automate.

**Java 17 → 21**
- Mostly smooth; enable virtual threads gradually; watch for pinning (synchronized + blocking) on 21.

## Interview Q&A
**Q. Which Java version do you use and why?** e.g., "Java 21 LTS — records, pattern matching, sealed types, and virtual threads which we enabled in Spring Boot 3.2 for our I/O-heavy services."

**Q. What were the major changes in Java 8?** Lambdas, streams, Optional, default methods, java.time, CompletableFuture, Metaspace.

**Q. What is the module system and do Spring Boot apps use it?** JPMS gives strong encapsulation and reliable configuration via module-info; most Spring Boot apps still run on the classpath.

**Q. Explain records, sealed classes and pattern matching together.** Records = data carriers, sealed = closed hierarchies, pattern-matching switch = exhaustive type-safe deconstruction → algebraic-data-type style modelling (e.g., `PaymentResult` = Success | Failure | Pending).

**Q. What are sequenced collections?** Java 21 interfaces giving a uniform API for first/last elements and reversed views across ordered collections.

**Q. What's new in Java 25 for everyday code?** Instance main/compact source files, flexible constructor bodies, module imports, scoped values final, compact object headers.
