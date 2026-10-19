# 08 — Concurrency & Multithreading

> Backend servers are concurrent by nature: Tomcat serves each request on a thread from a pool (default 200). Every Spring singleton bean is shared across all of those threads. Understanding concurrency is how you avoid data corruption, deadlocks and thread-pool exhaustion.

## 1. Concepts
| Term | Meaning |
|------|---------|
| **Process** | Running program with its own memory space |
| **Thread** | Unit of execution inside a process; threads share heap, have own stack |
| **Concurrency** | Dealing with many tasks at once (interleaving) |
| **Parallelism** | Executing many tasks literally at the same time (multiple cores) |
| **Context switch** | CPU saves one thread's state and loads another's — costly |
| **Race condition** | Outcome depends on timing of threads |
| **Critical section** | Code accessing shared mutable state |
| **Thread-safe** | Behaves correctly when used by multiple threads without extra sync |

## 2. Creating threads
```java
// 1. Extend Thread (avoid — wastes inheritance)
class Worker extends Thread { public void run() { work(); } }
new Worker().start();

// 2. Runnable (no result)
Thread t = new Thread(() -> work(), "worker-1");
t.start();         // start() creates a new thread; run() would execute in the CURRENT thread!

// 3. Callable (returns result, can throw checked exceptions) — via ExecutorService
Callable<Integer> task = () -> compute();

// 4. Java 21 builders
Thread.ofPlatform().name("p-", 0).start(task);
Thread.ofVirtual().start(task);
```

### Thread lifecycle
```
NEW ──start()──► RUNNABLE ◄──────────────┐
                    │                     │
      ┌─────────────┼───────────────┐     │
      ▼             ▼               ▼     │
   BLOCKED      WAITING      TIMED_WAITING│
 (waiting for  (wait(), join(), (sleep(ms),│
  monitor lock) park())         wait(ms))  │
      └─────────────┴───────────────┴─────┘
                    │
              run() ends
                    ▼
               TERMINATED
```

### Useful Thread methods
| Method | Meaning |
|--------|---------|
| `sleep(ms)` | Pause; **keeps** locks held |
| `join()` | Wait for thread to finish |
| `interrupt()` | Request cancellation (sets flag / wakes blocking calls with `InterruptedException`) |
| `isInterrupted()` / `Thread.interrupted()` | Check flag (static version clears it) |
| `setDaemon(true)` | JVM won't wait for it on exit |
| `yield()` | Hint to scheduler; rarely useful |
| `setPriority` | Hint; OS-dependent |
| `stop()`, `suspend()` | Deprecated/removed — unsafe |

**Cooperative cancellation**
```java
while (!Thread.currentThread().isInterrupted()) { doChunk(); }
```

## 3. The Java Memory Model (JMM)

Two problems in multithreaded code:
1. **Visibility** — one thread's write may sit in a CPU cache/register and never be seen by another.
2. **Ordering** — compiler/CPU reorder instructions.
3. (plus **Atomicity** — `count++` is read-modify-write, three steps.)

```java
// Broken visibility
private boolean running = true;
void stop() { running = false; }          // thread A
void loop() { while (running) { } }       // thread B may loop forever
```

**Happens-before** guarantees (if A happens-before B, B sees A's effects):
- Program order within a thread.
- Unlock of a monitor → later lock of the same monitor.
- Write to a `volatile` field → later read of it.
- `Thread.start()` → everything in the new thread.
- Everything in a thread → another thread's successful `join()` on it.
- Constructor end → finalizer; `final` field freeze after constructor (safe publication of immutable objects).
- Actions before submitting a task to an Executor → the task's execution; `Future.get()` sees the task's effects.

## 4. synchronized

```java
public class Counter {
    private int count;
    public synchronized void increment() { count++; }   // locks on `this`
    public synchronized int get() { return count; }      // reads must sync too (visibility)
}

public static synchronized void m() {}   // locks on Counter.class

private final Object lock = new Object();
void update() {
    synchronized (lock) { /* critical section */ }       // prefer a private lock object
}
```
- Every object has an intrinsic lock (**monitor**).
- **Reentrant**: a thread holding a lock can re-acquire it.
- Provides **mutual exclusion + visibility**.
- Waiting threads are BLOCKED and can't be interrupted or time out → use `ReentrantLock` for that.
- Don't synchronize on `String` literals, boxed integers, or `this` of publicly reachable objects (others can lock them too).

### wait / notify (low-level)
```java
synchronized (queue) {
    while (queue.isEmpty()) queue.wait();     // ALWAYS in a while loop (spurious wakeups)
    item = queue.poll();
}
synchronized (queue) { queue.add(x); queue.notifyAll(); }
```
Must be called holding the object's monitor (else `IllegalMonitorStateException`). `wait()` releases the lock; `sleep()` doesn't. Prefer `BlockingQueue` / `Condition`.

## 5. volatile
```java
private volatile boolean running = true;
```
- Guarantees **visibility** and prevents reordering around it.
- Does **NOT** make compound operations atomic: `volatile int c; c++` is still a race.
- Use for flags, status fields, and in double-checked locking.

```java
// Double-checked locking singleton — volatile is required
public class Config {
    private static volatile Config instance;
    public static Config get() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) instance = new Config();
            }
        }
        return instance;
    }
}
// Simpler: holder idiom or enum singleton
class Holder { static final Config INSTANCE = new Config(); }   // lazy & thread-safe via class init
```

## 6. Atomic variables (lock-free, CAS)
```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
counter.compareAndSet(expected, newValue);       // CAS: Compare-And-Swap CPU instruction
counter.updateAndGet(x -> x * 2);
counter.accumulateAndGet(5, Integer::sum);

AtomicLong, AtomicBoolean, AtomicReference<T>, AtomicIntegerArray
LongAdder hits = new LongAdder();  hits.increment();  hits.sum();   // best for high-contention counters (metrics)
```
CAS loop: read value → compute new → CAS; if another thread changed it, retry. **ABA problem**: value changed A→B→A; solved with `AtomicStampedReference`.

## 7. Explicit locks (`java.util.concurrent.locks`)
```java
private final ReentrantLock lock = new ReentrantLock(true);   // fair = FIFO (slower)

public void transfer() {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {     // timeout — avoids deadlock hang
        try { /* critical */ }
        finally { lock.unlock(); }               // ALWAYS unlock in finally
    } else { throw new TimeoutException(); }
}
lock.lockInterruptibly();
```
| Feature | synchronized | ReentrantLock |
|---------|-------------|---------------|
| Try with timeout | ✘ | ✔ `tryLock` |
| Interruptible wait | ✘ | ✔ |
| Fairness | ✘ | ✔ optional |
| Multiple conditions | one (wait/notify) | many `newCondition()` |
| Auto-release | ✔ | Must `unlock()` in finally |

**ReadWriteLock** — many readers or one writer:
```java
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock(); try { read(); } finally { rw.readLock().unlock(); }
```
**StampedLock** (Java 8) — optimistic reads, even faster for read-heavy data (not reentrant).

**Condition** — like wait/notify but multiple per lock:
```java
Condition notEmpty = lock.newCondition();
notEmpty.await(); notEmpty.signal();
```

## 8. Deadlock, livelock, starvation
**Deadlock** — four conditions (Coffman): mutual exclusion, hold and wait, no preemption, circular wait.
```java
// Thread 1: lock(A) then lock(B)     Thread 2: lock(B) then lock(A)   → deadlock
```
Prevention:
- **Lock ordering**: always acquire locks in a global order (e.g., by account id).
- `tryLock` with timeout.
- Minimize lock scope; avoid calling external code (alien methods) while holding locks.
- Detect: `jstack` prints "Found one Java-level deadlock".

**Livelock** — threads keep responding to each other but make no progress. **Starvation** — a thread never gets CPU/lock (unfair locks, high priority hogs).

## 9. Executor framework (thread pools)

Creating threads is expensive (~1MB stack, OS call). Pools reuse them.

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
Future<Report> f = pool.submit(() -> buildReport());
Report r = f.get(5, TimeUnit.SECONDS);       // blocks; TimeoutException / ExecutionException
f.cancel(true);

List<Future<Integer>> all = pool.invokeAll(tasks);     // waits for all
Integer fastest = pool.invokeAny(tasks);               // first successful

pool.shutdown();                                       // no new tasks; finish queued
if (!pool.awaitTermination(30, TimeUnit.SECONDS)) pool.shutdownNow();   // interrupt running
```
`execute(Runnable)` → no Future; exceptions go to the thread's uncaught handler. `submit()` → exceptions are captured inside the Future (silently lost if you never call `get()`!).

### Executors factory methods (and their traps)
| Factory | Queue | Trap |
|---------|-------|------|
| `newFixedThreadPool(n)` | **Unbounded** LinkedBlockingQueue | Queue can grow until OOM |
| `newCachedThreadPool()` | SynchronousQueue, unlimited threads | Can create thousands of threads |
| `newSingleThreadExecutor()` | Unbounded | Sequential |
| `newScheduledThreadPool(n)` | Delay queue | — |
| `newWorkStealingPool()` | ForkJoinPool | — |
| `newVirtualThreadPerTaskExecutor()` | none (Java 21) | — |

### ThreadPoolExecutor — configure it yourself in production
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                                   // corePoolSize
    50,                                   // maximumPoolSize
    60, TimeUnit.SECONDS,                 // keepAlive for threads above core
    new ArrayBlockingQueue<>(500),        // BOUNDED queue
    new ThreadFactoryBuilder("orders-%d"),// named threads (Guava / custom)
    new ThreadPoolExecutor.CallerRunsPolicy()); // rejection policy → backpressure
```
**Task submission algorithm**:
1. Threads < core → create new thread.
2. Else → put in queue.
3. Queue full and threads < max → create new thread.
4. Else → **rejection policy**: `AbortPolicy` (throw, default), `CallerRunsPolicy` (submitter runs it — natural backpressure), `DiscardPolicy`, `DiscardOldestPolicy`.

**Sizing**: CPU-bound ≈ number of cores (+1). I/O-bound ≈ cores × (1 + wait time / compute time). Measure!

### ScheduledExecutorService
```java
ScheduledExecutorService s = Executors.newScheduledThreadPool(2);
s.schedule(task, 10, TimeUnit.SECONDS);
s.scheduleAtFixedRate(task, 0, 1, TimeUnit.MINUTES);      // fixed rate (from start)
s.scheduleWithFixedDelay(task, 0, 1, TimeUnit.MINUTES);   // fixed delay (after finish)
```
An uncaught exception cancels future executions silently — wrap the body in try/catch.

## 10. CompletableFuture (async composition)

```java
ExecutorService io = Executors.newFixedThreadPool(20);

CompletableFuture<User> userF   = CompletableFuture.supplyAsync(() -> userClient.get(id), io);
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderClient.list(id), io);

CompletableFuture<Dashboard> dash = userF
    .thenCombine(ordersF, Dashboard::new)                 // both done → combine
    .orTimeout(2, TimeUnit.SECONDS)                       // Java 9
    .exceptionally(ex -> Dashboard.empty());              // fallback

Dashboard d = dash.join();                                // unchecked exception version of get()
```
| Method | Purpose |
|--------|---------|
| `supplyAsync / runAsync` | Start (default: common ForkJoinPool — pass your own executor for I/O!) |
| `thenApply` | Transform result (like map) |
| `thenCompose` | Chain another async call (like flatMap) |
| `thenCombine` | Combine two independent futures |
| `thenAccept / thenRun` | Consume / run after |
| `allOf(...)` / `anyOf(...)` | Wait for all / first |
| `exceptionally` | Recover from error |
| `handle((res, ex) -> ...)` | Handle both success and failure |
| `whenComplete` | Side effect, doesn't change result |
| `completeOnTimeout(value, t, unit)` | Default value on timeout |
| `...Async` variants | Run the stage on another thread |

```java
// Fan-out / fan-in
List<CompletableFuture<Price>> futures = vendors.stream()
    .map(v -> CompletableFuture.supplyAsync(() -> v.quote(item), io)).toList();
List<Price> prices = CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).toList())
    .join();
```

## 11. Fork/Join framework
Divide-and-conquer with **work stealing** (idle threads steal tasks from others' deques). Backbone of parallel streams.
```java
class SumTask extends RecursiveTask<Long> {
    private final int[] arr; private final int lo, hi;
    protected Long compute() {
        if (hi - lo <= 10_000) { long s = 0; for (int i = lo; i < hi; i++) s += arr[i]; return s; }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(arr, lo, mid); left.fork();
        return new SumTask(arr, mid, hi).compute() + left.join();
    }
}
ForkJoinPool.commonPool().invoke(new SumTask(arr, 0, arr.length));
```

## 12. Synchronizers
| Class | What | Example |
|-------|------|---------|
| `CountDownLatch` | Wait until N events happen; one-shot | Main waits for 5 services to warm up |
| `CyclicBarrier` | N threads wait for each other; reusable | Parallel simulation steps |
| `Semaphore` | N permits; limit concurrent access | Max 10 concurrent calls to a partner API |
| `Phaser` | Flexible multi-phase barrier | |
| `Exchanger` | Two threads swap objects | |

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) pool.submit(() -> { load(); latch.countDown(); });
latch.await(10, TimeUnit.SECONDS);

Semaphore permits = new Semaphore(10);
permits.acquire();
try { callPartner(); } finally { permits.release(); }
```

## 13. Producer–consumer with BlockingQueue
```java
BlockingQueue<Order> queue = new ArrayBlockingQueue<>(100);
// producer
queue.put(order);                       // blocks when full → backpressure
// consumer
while (true) { Order o = queue.take(); process(o); }   // blocks when empty
```

## 14. ThreadLocal
Per-thread variable — each thread sees its own copy.
```java
private static final ThreadLocal<DateTimeFormatter> FMT = ThreadLocal.withInitial(() -> ...);
private static final ThreadLocal<String> TENANT = new ThreadLocal<>();

try {
    TENANT.set(request.getHeader("X-Tenant"));
    chain.doFilter(request, response);
} finally {
    TENANT.remove();          // CRUCIAL in thread pools, otherwise next request sees old value + leaks
}
```
Used by Spring for `SecurityContextHolder`, `TransactionSynchronizationManager`, `RequestContextHolder`, and SLF4J's **MDC** (log correlation IDs). That's why these don't propagate automatically into `@Async` / `CompletableFuture` threads (use `TaskDecorator`, context-propagation library). With virtual threads, prefer **ScopedValue** (finalized in Java 25).

## 15. Virtual threads (Java 21) — Project Loom
- Platform thread = 1:1 OS thread (expensive, ~thousands max).
- **Virtual thread** = lightweight, JVM-managed, mounted on a small pool of carrier (platform) threads. When it blocks on I/O, it **unmounts** and the carrier runs another one. Millions possible.
- Write simple blocking code ("thread-per-request") and get async-level scalability.

```java
try (ExecutorService ex = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> ex.submit(() -> {
        Thread.sleep(Duration.ofSeconds(1));      // 10k concurrent sleeps, finishes in ~1s
        return i;
    }));
}
```
Spring Boot 3.2+: `spring.threads.virtual.enabled=true` → Tomcat, `@Async`, scheduling use virtual threads.

**Guidelines**
- Great for **I/O-bound** work (DB, HTTP calls). No benefit for CPU-bound work.
- **Don't pool** virtual threads — create one per task.
- **Pinning**: in Java 21–23, blocking inside `synchronized` pins the carrier; prefer `ReentrantLock` there. Java 24 (JEP 491) fixed most synchronized pinning.
- Still limit access to scarce resources (DB connection pool size, Semaphore for rate limits) — virtual threads make it easy to overwhelm downstream systems.
- Avoid heavy ThreadLocal caches (one copy per virtual thread → memory).

**Structured concurrency** (preview in 21–25): `StructuredTaskScope` treats a group of subtasks as one unit — cancellation and error propagation handled together.

## 16. Thread safety strategies (design level)
1. **Don't share** — local variables, stateless beans (Spring services should be stateless!).
2. **Immutability** — records, `final`, `List.copyOf`.
3. **Thread confinement** — ThreadLocal, single-threaded executors, actors.
4. **Use thread-safe libraries** — `ConcurrentHashMap`, atomics, `BlockingQueue`.
5. **Synchronize** — as last resort, with minimal scope.

**Spring singleton pitfall**
```java
@Service
public class ReportService {
    private List<String> rows = new ArrayList<>();   // shared by ALL requests → data mixes up!
    public Report build() { rows.clear(); ... }
}
```
Keep request data in local variables / method parameters.

**Check-then-act** race (even with thread-safe collections):
```java
if (!map.containsKey(k)) map.put(k, v);      // race
map.putIfAbsent(k, v);                        // atomic ✔
map.computeIfAbsent(k, key -> load(key));     // atomic ✔
```

## 17. Interview Q&A
**Q. Difference between process and thread?** Processes have separate memory; threads share the process heap, have own stack; threads are cheaper to create and switch.

**Q. `start()` vs `run()`?** start creates a new thread that calls run; calling run directly executes in the current thread.

**Q. `sleep()` vs `wait()`?** sleep is a Thread static method, keeps locks, no notify needed. wait is an Object method, must hold the monitor, releases it, woken by notify/notifyAll/timeout.

**Q. `synchronized` vs `volatile`?** volatile gives visibility and ordering for single reads/writes; synchronized also gives mutual exclusion (atomicity of compound actions).

**Q. What is a race condition? Example?** Two threads doing `count++` concurrently lose updates because read-increment-write isn't atomic.

**Q. How do you prevent deadlocks?** Consistent lock ordering, tryLock timeouts, smaller critical sections, avoid nested locks.

**Q. Callable vs Runnable?** Callable returns a value and can throw checked exceptions.

**Q. `Future` vs `CompletableFuture`?** Future only supports blocking get/cancel. CompletableFuture supports non-blocking composition, combining, exception handling, manual completion.

**Q. Why not use `Executors.newFixedThreadPool` in production?** Unbounded queue may grow without limit → OOM, hides overload. Use `ThreadPoolExecutor` with bounded queue and a rejection policy.

**Q. What are virtual threads and when would you use them?** JVM-scheduled lightweight threads for high-concurrency blocking I/O; enable in Spring Boot with one property; not for CPU-bound tasks.

**Q. How does ConcurrentHashMap differ from Collections.synchronizedMap?** Fine-grained locking + lock-free reads vs one lock for everything; weakly consistent iterators vs fail-fast.

**Q. What is CAS?** Atomic CPU instruction: update a memory location only if it still holds the expected value; basis of atomics and lock-free algorithms.

**Q. Why must you call `ThreadLocal.remove()`?** Pool threads are reused; stale values leak into other requests and keep objects alive.
