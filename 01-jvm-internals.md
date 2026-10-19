# 01 — JVM Internals, Memory & Garbage Collection

> Why this matters for a backend dev: every production incident involving `OutOfMemoryError`, long GC pauses, high CPU, or slow startup comes back to how the JVM works. Interviewers love this topic for 3+ years experience.

---

## 1. From source code to running program

```
HelloWorld.java ──javac──► HelloWorld.class (bytecode) ──► JVM
                                                           │
                       ┌───────────────────────────────────┤
                       ▼                                   ▼
               Class Loader Subsystem              Execution Engine
               (load → link → init)        (Interpreter + JIT + GC)
                       │                                   │
                       └────────► Runtime Data Areas ◄─────┘
                                 (Heap, Stacks, Metaspace...)
```

### 1.1 JDK vs JRE vs JVM

| Component | Contains | Who needs it |
|-----------|----------|--------------|
| **JVM** | Class loader, bytecode verifier, interpreter, JIT compiler, garbage collector | Abstract spec + concrete implementation (HotSpot, OpenJ9, GraalVM) |
| **JRE** | JVM + standard class libraries (`java.lang`, `java.util`...) | Anyone *running* Java (since Java 11 Oracle doesn't ship separate JRE; use `jlink` to make custom runtimes) |
| **JDK** | JRE + dev tools: `javac`, `java`, `jar`, `jshell`, `javadoc`, `jdb`, `jcmd`, `jstack`, `jmap`, `jconsole`, `jfr` | Developers |

**Platform independence**: bytecode is the same everywhere; only the JVM is OS-specific. "Write once, run anywhere."

**Distributions**: Oracle JDK, OpenJDK builds (Eclipse Temurin, Amazon Corretto, Azul Zulu, Microsoft Build of OpenJDK, Red Hat). Functionally the same codebase — pick an LTS (17, 21, 25).

---

## 2. Class Loading

### 2.1 Three phases
1. **Loading** – find the `.class` bytes and create a `Class` object.
2. **Linking**
   - *Verification* – bytecode is safe (no stack overflow tricks, correct types).
   - *Preparation* – allocate static fields with default values (0, null, false).
   - *Resolution* – symbolic references → direct references (can be lazy).
3. **Initialization** – run static initializers and assign static field values (`static int x = 5;`, `static { }` blocks). Happens on **first active use** (new instance, static method call, static field access, reflection, subclass init).

### 2.2 Class loader hierarchy (Java 9+)
| Loader | Loads |
|--------|-------|
| **Bootstrap** (native) | Core modules: `java.base` (`java.lang.String`, etc.) |
| **Platform** (was Extension) | Other Java SE modules (`java.sql`, ...) |
| **Application / System** | Your classpath / module path |
| Custom | Tomcat, Spring Boot's `LaunchedClassLoader` (reads nested jars in fat jar) |

**Parent delegation model**: a loader first asks its parent. Only if the parent can't find the class does it try itself. Why? Security (nobody can replace `java.lang.String`) and no duplicate classes.

A class's identity = **fully qualified name + class loader**. Same class loaded by two loaders → two different types → `ClassCastException: com.x.A cannot be cast to com.x.A` (classic app-server / DevTools restart issue).

```java
System.out.println(String.class.getClassLoader());          // null → bootstrap
System.out.println(MyApp.class.getClassLoader());            // app loader
```

**Errors**:
- `ClassNotFoundException` (checked) – explicit load by name failed (`Class.forName("x")`).
- `NoClassDefFoundError` (error) – class was present at compile time but missing at runtime, or its static initializer failed earlier.

---

## 3. Runtime Data Areas (JVM memory)

```
┌──────────────────────── Shared by all threads ───────────────────────┐
│  HEAP                                                                 │
│  ┌──────────── Young Generation ────────────┐  ┌── Old Generation ──┐ │
│  │  Eden   │ Survivor S0 │ Survivor S1       │  │  (Tenured)         │ │
│  └──────────────────────────────────────────┘  └────────────────────┘ │
│  METASPACE (native memory): class metadata, method bytecode            │
│  Code Cache: JIT-compiled native code                                  │
└───────────────────────────────────────────────────────────────────────┘
┌──── Per thread ────┐
│ JVM Stack (frames) │  each frame: local variables array, operand stack, ref to constant pool
│ PC Register        │  address of current instruction
│ Native Method Stack│
└────────────────────┘
```

### 3.1 Heap
- All objects and arrays live here (escape analysis may place some on the stack/scalar-replace them).
- Size: `-Xms512m` (initial), `-Xmx2g` (max). In containers, prefer `-XX:MaxRAMPercentage=75.0` (JVM is container-aware since Java 10).
- **String pool** lives in the heap (moved from PermGen in Java 7).

### 3.2 Stack
- One per thread, created with the thread. Default size ~512KB–1MB (`-Xss`).
- Each method call pushes a **frame**; return pops it.
- Holds **primitive locals and references**; the objects themselves are in the heap.
- Too-deep recursion → `StackOverflowError`.

```java
void m() {
    int x = 10;                 // x on stack
    Person p = new Person();    // reference p on stack, Person object on heap
}
```

### 3.3 Metaspace (Java 8+)
- Replaced **PermGen**. Stores class metadata in **native memory**, grows automatically.
- Limit with `-XX:MaxMetaspaceSize`. Leaks happen with repeated class loading (hot redeploys, dynamic proxies generated without caching) → `OutOfMemoryError: Metaspace`.

### 3.4 Common OutOfMemoryErrors
| Message | Cause | First step |
|---------|-------|-----------|
| `Java heap space` | Too many live objects / leak / heap too small | Heap dump, look at dominator tree |
| `GC overhead limit exceeded` | >98% time in GC, <2% heap recovered | Same as above |
| `Metaspace` | Class loader leak | Count loaded classes, check proxies |
| `unable to create native thread` | Too many threads / OS limit | Thread dump, use pools/virtual threads |
| `Direct buffer memory` | NIO direct buffers (Netty) | `-XX:MaxDirectMemorySize` |

---

## 4. Execution Engine

### 4.1 Interpreter + JIT
- Bytecode is first **interpreted** (fast startup, slow execution).
- HotSpot profiles code; "hot" methods (called often, loops) are compiled to native code by the **JIT**.
- **Tiered compilation**: C1 (client compiler, quick, light optimizations) → C2 (server compiler, aggressive optimizations using profile data).
- Optimizations: **inlining** (most important), escape analysis, loop unrolling, dead code elimination, lock elision/coarsening, intrinsics (e.g., `Math`, `System.arraycopy`).
- **Deoptimization**: if an assumption breaks (e.g., a new subclass loaded invalidates a monomorphic call site), compiled code is discarded.
- **Warm-up**: why the first requests to a fresh Spring Boot service are slower. Mitigations: CDS/AppCDS, CRaC (checkpoint/restore), GraalVM native image, Project Leyden AOT cache (Java 24+).

### 4.2 GraalVM Native Image (relevant to Spring Boot 3)
- **AOT** compiles the app into a standalone native executable.
- Pros: startup in milliseconds, low memory — great for serverless/scale-to-zero.
- Cons: long build times, closed-world assumption (reflection/proxies need hints; Spring Boot 3 AOT engine generates them), lower peak throughput than JIT in some cases.
- `mvn -Pnative native:compile` or `mvn -Pnative spring-boot:build-image`.

---

## 5. Garbage Collection

### 5.1 What is garbage?
An object is eligible when it is **not reachable** from any **GC root**:
- Local variables in active stack frames
- Static fields of loaded classes
- Active threads
- JNI references

Reference cycles are **not** a problem (unlike ref counting) — if the cycle isn't reachable from roots, it's collected.

### 5.2 Generational hypothesis
"Most objects die young." So the heap is split:
1. New objects allocated in **Eden** (fast bump-pointer allocation in per-thread TLABs).
2. Eden full → **Minor GC**: live objects copied to a Survivor space; age +1.
3. Survivors bounce between S0 and S1; after reaching the tenuring threshold (default up to 15) → promoted to **Old gen**.
4. Old gen full → **Major/Full GC** (expensive).

### 5.3 Core algorithms
- **Mark** – trace from roots, mark live objects.
- **Sweep** – reclaim unmarked (leaves fragmentation).
- **Compact** – move live objects together (no fragmentation, costs time).
- **Copy** – copy live objects to another space (used in young gen; cost proportional to live objects only).

**Stop-the-world (STW)** pause: all application threads halted. Modern collectors do most work **concurrently**.

### 5.4 Collectors
| Collector | Flag | Notes |
|-----------|------|-------|
| Serial | `-XX:+UseSerialGC` | Single thread. Tiny heaps / 1 CPU containers |
| Parallel | `-XX:+UseParallelGC` | Multi-threaded, maximizes throughput; longer pauses. Batch jobs |
| CMS | removed in Java 14 | Historic low-pause collector |
| **G1** | `-XX:+UseG1GC` (default since 9) | Heap split into equal **regions**; collects regions with most garbage first ("garbage first"). Pause target `-XX:MaxGCPauseMillis=200` |
| **ZGC** | `-XX:+UseZGC` | Sub-millisecond pauses, heaps up to TBs. Generational since Java 21 (default generational from 23) |
| Shenandoah | `-XX:+UseShenandoahGC` | Low pause, concurrent compaction (Red Hat) |
| Epsilon | `-XX:+UseEpsilonGC` | No-op GC, for testing |

**Choosing**: default G1 is fine for most services. Latency-sensitive APIs with big heaps → ZGC. Small container (≤1 CPU, <1.7GB) → JVM auto-selects Serial.

### 5.5 Reference types (`java.lang.ref`)
| Type | Collected when | Use |
|------|---------------|-----|
| Strong | Never while reachable | Normal references |
| Soft | Memory is low | Memory-sensitive caches |
| Weak | Next GC if only weakly reachable | `WeakHashMap`, canonicalizing maps, listeners |
| Phantom | After finalization; used with `ReferenceQueue` | Cleanup actions (`Cleaner`) — replacement for deprecated `finalize()` |

### 5.6 Memory leaks in Java (yes, they exist)
- Static collections that only grow (`static Map cache` without eviction).
- Unclosed resources (streams, connections) → use try-with-resources.
- `ThreadLocal` values not removed in thread pools (Tomcat threads live forever) → always `remove()` in `finally`.
- Listeners/callbacks registered but never unregistered.
- Inner (non-static) classes holding implicit reference to outer instance.
- Wrong `equals/hashCode` → duplicates pile up in HashMap.

---

## 6. JVM tuning & troubleshooting toolkit

### 6.1 Common flags
```bash
java -Xms1g -Xmx1g \                     # equal sizes avoid resize pauses
     -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
     -Xlog:gc*:file=gc.log:time,uptime \  # unified GC logging (Java 9+)
     -XX:MaxRAMPercentage=75 \            # in containers instead of -Xmx
     -jar app.jar
```

### 6.2 Diagnostic tools
| Tool | Purpose |
|------|---------|
| `jps` | List Java processes |
| `jcmd <pid> help` | Swiss army knife: `GC.heap_info`, `Thread.print`, `GC.heap_dump`, `VM.flags`, `JFR.start` |
| `jstack <pid>` | Thread dump — find deadlocks, blocked threads, hot loops |
| `jmap -dump:live,format=b,file=heap.hprof <pid>` | Heap dump |
| `jstat -gcutil <pid> 1000` | GC stats every second |
| **JFR** (Java Flight Recorder) + **JMC** | Low-overhead production profiling |
| **Eclipse MAT** / VisualVM | Analyze heap dumps (dominator tree, leak suspects) |
| async-profiler | CPU/alloc flame graphs |
| Spring Boot Actuator | `/actuator/heapdump`, `/actuator/threaddump`, `/actuator/metrics/jvm.memory.used` |

### 6.3 Troubleshooting playbook
- **High CPU**: `top -H -p <pid>` → find hot thread id → convert to hex → find in `jstack` output. Or JFR/async-profiler.
- **App frozen**: take 3 thread dumps 5s apart; look for `BLOCKED` threads waiting on the same lock, or "Found one Java-level deadlock".
- **Memory growing**: GC logs show old gen not dropping after full GC → leak → heap dump → MAT "Leak Suspects".
- **Long pauses**: check GC logs, allocation rate, consider bigger young gen or ZGC.

---

## 7. Java Memory Model (JMM) — short version (details in concurrency notes)
- Defines what values a thread may see when reading shared variables.
- Without synchronization, the CPU/compiler may **reorder** instructions and threads may see **stale cached values**.
- **Happens-before** rules guarantee visibility: unlock → subsequent lock of same monitor; write to `volatile` → subsequent read; `Thread.start()` → actions in started thread; actions in thread → `join()` returns.

---

## 8. Interview Q&A

**Q1. Explain JVM architecture.**
Class loader subsystem (loading, linking, initialization) → runtime data areas (heap & metaspace shared; stack, PC register, native stack per thread) → execution engine (interpreter, JIT, GC) + JNI for native libs.

**Q2. Where are objects, local variables and static variables stored?**
Objects → heap. Local primitives & references → thread stack. Static variables → since Java 8 the static fields are stored with the `Class` object in the heap; class metadata in Metaspace.

**Q3. PermGen vs Metaspace?**
PermGen was a fixed-size part of the heap (`OOM: PermGen space` was common). Metaspace (Java 8+) uses native memory, auto-grows, configurable with `MaxMetaspaceSize`.

**Q4. How does GC decide an object is garbage?**
Reachability analysis from GC roots; unreachable objects are garbage, even if they reference each other.

**Q5. Minor vs Major vs Full GC?**
Minor = young gen only, frequent, cheap. Major = old gen. Full = entire heap + metaspace, usually STW and expensive.

**Q6. Can you force garbage collection?**
No. `System.gc()` is a hint (and can be disabled by `-XX:+DisableExplicitGC`).

**Q7. What is JIT and why is the first request slow?**
JIT compiles hot bytecode to native code at runtime based on profiling. Before warm-up, code is interpreted, plus lazy class loading & Spring bean init — so early requests are slower.

**Q8. ClassNotFoundException vs NoClassDefFoundError?**
CNFE: checked exception from dynamic loading by name. NCDFE: error — class existed at compile time but isn't loadable now (missing jar, or failed static init).

**Q9. How would you debug a memory leak in production?**
Confirm with GC logs/metrics (old gen grows after full GCs) → capture heap dump (`jcmd GC.heap_dump` or auto on OOM) → analyze in MAT (dominator tree, paths to GC roots) → fix code (unbounded cache, ThreadLocal, listener).

**Q10. Which GC would you choose for a low-latency API?**
ZGC (or Shenandoah) for sub-ms pauses with large heaps; G1 with tuned pause target is a reasonable default.
