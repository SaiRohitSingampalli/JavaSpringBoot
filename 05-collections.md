# 05 — Collections Framework Deep Dive

## 1. The big picture

```
                       Iterable<E>
                           │
                      Collection<E>
         ┌─────────────────┼─────────────────────┐
       List<E>           Set<E>               Queue<E>
         │                 │                     │
  ArrayList          HashSet                PriorityQueue
  LinkedList         LinkedHashSet          Deque<E>
  Vector/Stack(legacy) SortedSet→NavigableSet  ├─ ArrayDeque
  CopyOnWriteArrayList   └─ TreeSet            └─ LinkedList
                     EnumSet                 BlockingQueue (concurrent)

          Map<K,V>  (not a Collection)
          ├─ HashMap ─ LinkedHashMap
          ├─ SortedMap → NavigableMap → TreeMap
          ├─ Hashtable (legacy), EnumMap, WeakHashMap, IdentityHashMap
          └─ ConcurrentMap → ConcurrentHashMap, ConcurrentSkipListMap

Java 21: SequencedCollection / SequencedSet / SequencedMap
         (getFirst, getLast, addFirst, addLast, reversed)
```

## 2. Choosing the right collection (cheat sheet)

| Need | Use |
|------|-----|
| Ordered list, index access, mostly appends | `ArrayList` |
| Stack or queue | `ArrayDeque` |
| Unique elements, fast lookup | `HashSet` |
| Unique + insertion order | `LinkedHashSet` |
| Unique + sorted / range queries | `TreeSet` |
| Key-value, fast lookup | `HashMap` |
| Key-value + insertion order / LRU | `LinkedHashMap` |
| Sorted keys, floor/ceiling queries | `TreeMap` |
| Priority / top-K | `PriorityQueue` |
| Enum keys | `EnumMap` / `EnumSet` |
| Thread-safe map | `ConcurrentHashMap` |
| Thread-safe list, read-heavy | `CopyOnWriteArrayList` |
| Producer–consumer | `ArrayBlockingQueue`, `LinkedBlockingQueue` |
| Immutable | `List.of`, `Set.of`, `Map.of`, `List.copyOf` |

## 3. Big-O summary

| Operation | ArrayList | LinkedList | HashSet/HashMap | TreeSet/TreeMap | PriorityQueue | ArrayDeque |
|-----------|-----------|-----------|-----------------|-----------------|---------------|-----------|
| add (end) | O(1) amortized | O(1) | O(1) avg | O(log n) | O(log n) | O(1) |
| get(index) | O(1) | O(n) | — | — | — | — |
| contains | O(n) | O(n) | O(1) avg | O(log n) | O(n) | O(n) |
| remove | O(n) | O(1) at iterator / O(n) find | O(1) avg | O(log n) | O(log n) (poll) | O(1) ends |
| iteration order | insertion | insertion | none | sorted | heap order | insertion |

---

## 4. List

### 4.1 ArrayList internals
- Backed by `Object[] elementData`. Default capacity 10 (allocated lazily on first add).
- When full, grows by **50%** (`newCapacity = old + old >> 1`) and copies (`Arrays.copyOf`).
- `add(index, e)` / `remove(index)` shift elements → O(n).
- Pre-size when size known: `new ArrayList<>(10_000)`.
- Implements `RandomAccess`.

### 4.2 LinkedList internals
- Doubly linked nodes (`prev`, `item`, `next`) — also implements `Deque`.
- Poor cache locality, extra memory per node (~24+ bytes). In practice **ArrayList/ArrayDeque almost always win**.

### 4.3 Useful List API
```java
List<String> list = new ArrayList<>(List.of("b", "a", "c"));
list.add(0, "z"); list.set(1, "y"); list.remove("c"); list.remove(0); // remove(int index) vs remove(Object)!
list.indexOf("a"); list.subList(0, 2);        // view, not copy
list.sort(null);                              // natural order
list.sort(Comparator.comparing(String::length).thenComparing(Comparator.reverseOrder()));
list.removeIf(s -> s.startsWith("x"));
list.replaceAll(String::toUpperCase);
Collections.reverse(list); Collections.shuffle(list); Collections.swap(list, 0, 1);
Collections.frequency(list, "a"); Collections.max(list); Collections.nCopies(3, "x");
Collections.unmodifiableList(list);           // read-only VIEW (changes in original show through)
List.copyOf(list);                            // true immutable copy
list.getFirst(); list.getLast(); list.reversed();   // Java 21
```

**Pitfall**: `List<Integer> l; l.remove(1)` removes index 1, not value 1. Use `l.remove(Integer.valueOf(1))`.

### 4.4 Arrays.asList vs List.of
| | `Arrays.asList` | `List.of` |
|--|--|--|
| Size | Fixed (add/remove throw) | Immutable |
| set() | Allowed (writes through to array) | Throws |
| nulls | Allowed | NPE |

---

## 5. Set

### 5.1 HashSet
Backed by a `HashMap` (elements are keys, value is a dummy `PRESENT` object). Relies entirely on `hashCode/equals`.

### 5.2 LinkedHashSet
HashSet + doubly linked list across entries → predictable insertion order.

### 5.3 TreeSet
Backed by `TreeMap` (red-black tree). Elements must be `Comparable` or you provide a `Comparator`. Uniqueness determined by `compareTo == 0`, **not equals** — keep them consistent.
```java
TreeSet<Integer> ts = new TreeSet<>(List.of(10, 20, 30, 40));
ts.first(); ts.last();
ts.floor(25);    // 20  (≤)
ts.ceiling(25);  // 30  (≥)
ts.lower(20);    // 10  (<)
ts.higher(20);   // 30  (>)
ts.headSet(30);  // [10, 20]
ts.tailSet(30);  // [30, 40]
ts.descendingSet();
```

### 5.4 Mutable keys trap
If you change a field used in `hashCode` after inserting into a HashSet, the element is "lost" (in the wrong bucket). Keys should be immutable.

---

## 6. Queue & Deque

```java
Queue<String> q = new ArrayDeque<>();
q.offer("a");  q.peek();  q.poll();       // return null/false on failure
q.add("a");    q.element(); q.remove();   // throw on failure

Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.pop(); stack.peek();  // LIFO — replaces legacy Stack class

PriorityQueue<Task> pq = new PriorityQueue<>(Comparator.comparing(Task::priority));
// min-heap; iteration order is NOT sorted — only poll() gives sorted order

// Top-K largest numbers: keep a min-heap of size K
PriorityQueue<Integer> heap = new PriorityQueue<>();
for (int n : nums) { heap.offer(n); if (heap.size() > k) heap.poll(); }
```
`ArrayDeque` doesn't allow nulls; it's faster than `Stack` (synchronized) and `LinkedList`.

---

## 7. Map

### 7.1 Core API (know these by heart)
```java
Map<String, Integer> m = new HashMap<>();
m.put("a", 1);                       // returns previous value or null
m.get("x");                          // null if absent
m.getOrDefault("x", 0);
m.containsKey("a"); m.containsValue(1);
m.putIfAbsent("a", 5);
m.remove("a"); m.remove("a", 1);     // conditional remove
m.replace("a", 2);
m.keySet(); m.values(); m.entrySet();

// Java 8 compute family — great for counting/grouping
m.merge(word, 1, Integer::sum);                              // word count
m.computeIfAbsent(dept, k -> new ArrayList<>()).add(emp);    // multimap
m.computeIfPresent("a", (k, v) -> v + 1);
m.compute("a", (k, v) -> v == null ? 1 : v + 1);
m.forEach((k, v) -> System.out.println(k + "=" + v));
m.replaceAll((k, v) -> v * 2);

// Iterate efficiently
for (Map.Entry<String, Integer> e : m.entrySet()) { e.getKey(); e.getValue(); }

Map.of("a", 1, "b", 2);              // immutable, up to 10 pairs
Map.ofEntries(Map.entry("a", 1));
Map.copyOf(m);
```

### 7.2 HashMap internals (most asked interview topic)

**Structure**: `Node<K,V>[] table` — each slot is a "bucket". `Node` has `hash, key, value, next`.

**put(key, value)**
1. `hash = h ^ (h >>> 16)` where `h = key.hashCode()` — spreads high bits into low bits.
2. Bucket index = `hash & (n - 1)` (n is a power of two → fast modulo).
3. Bucket empty → place node.
4. Bucket occupied → walk the chain: if a node has same hash and `equals` key → replace value. Otherwise append to the end.
5. If a chain length reaches **8** (TREEIFY_THRESHOLD) **and** table size ≥ 64 → convert bucket to a **red-black tree** (O(log n) worst case instead of O(n)). Below 64, it resizes instead. Shrinks back to list at 6.
6. If `size > capacity × loadFactor` (default 16 × 0.75 = 12) → **resize**: double the table and redistribute (each node either stays at index `i` or moves to `i + oldCap`).

**get(key)**: compute hash → bucket → compare hash then `equals` along chain/tree.

**null key**: allowed once, stored in bucket 0. `Hashtable` and `ConcurrentHashMap` don't allow null keys/values.

**Why must keys be immutable?** If hashCode changes after insertion, `get` looks in the wrong bucket.

**Why capacity power of two?** Bitwise AND instead of modulo; resize splits cleanly.

**Not thread-safe**: concurrent puts can lose updates (in Java 7 resize could even create an infinite loop).

### 7.3 LinkedHashMap — and an LRU cache
```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LruCache(int capacity) {
        super(capacity, 0.75f, true);     // accessOrder = true
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```
(In production use Caffeine.)

### 7.4 TreeMap
Red-black tree; sorted keys; `firstKey, lastKey, floorKey, ceilingKey, headMap, tailMap, subMap, descendingMap`. Useful for time-series, ranges, leaderboards.

### 7.5 Other maps
- `EnumMap` – array indexed by ordinal; fastest for enum keys.
- `WeakHashMap` – keys weakly referenced; entry removed when key unreachable.
- `IdentityHashMap` – uses `==` instead of equals (serialization graphs, proxies).
- `ConcurrentSkipListMap` – concurrent sorted map.

### 7.6 HashMap vs Hashtable vs ConcurrentHashMap vs synchronizedMap
| | HashMap | Hashtable | Collections.synchronizedMap | ConcurrentHashMap |
|--|--|--|--|--|
| Thread-safe | No | Yes (every method synchronized) | Yes (one lock) | Yes (fine-grained) |
| Null key/values | 1 key, many values | No | Depends on backing map | No |
| Performance under contention | — | Poor | Poor | Excellent |
| Iterator | Fail-fast | Fail-fast (enumerations not) | Fail-fast (must sync manually) | Weakly consistent |

**ConcurrentHashMap (Java 8+)**: CAS for empty bins, `synchronized` on the first node of a bin for updates; reads are lock-free. Atomic compound ops: `putIfAbsent`, `computeIfAbsent`, `merge`. `size()` is an estimate under concurrency; use `mappingCount()`.

---

## 8. Iteration, fail-fast, fail-safe

```java
for (String s : list) {
    if (s.isEmpty()) list.remove(s);   // ConcurrentModificationException!
}
```
Each collection has a `modCount`; the iterator checks it on `next()` → **fail-fast** (best-effort, not guaranteed).

Correct ways:
```java
list.removeIf(String::isEmpty);
Iterator<String> it = list.iterator();
while (it.hasNext()) if (it.next().isEmpty()) it.remove();
```
**Fail-safe / weakly consistent** iterators (`CopyOnWriteArrayList`, `ConcurrentHashMap`) work on a snapshot or tolerate concurrent changes, never throw CME.

`ListIterator` adds bidirectional traversal, `set`, `add`, `previousIndex`.

---

## 9. Comparable vs Comparator

| | Comparable | Comparator |
|--|--|--|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T a, T b)` |
| Where | Inside the class (natural order) | Separate — many orderings |
| Example | `String`, `Integer`, `LocalDate` | Sort employees by salary, then name |

```java
public record Employee(String name, String dept, BigDecimal salary, int age) implements Comparable<Employee> {
    public int compareTo(Employee o) { return name.compareTo(o.name); }
}

employees.sort(Comparator
    .comparing(Employee::dept)
    .thenComparing(Employee::salary, Comparator.reverseOrder())
    .thenComparingInt(Employee::age));

Comparator.nullsFirst(Comparator.naturalOrder());
```
Contract: `sgn(compare(a,b)) == -sgn(compare(b,a))`, transitive. **Don't** use `a - b` for ints (overflow) → use `Integer.compare(a, b)`.

---

## 10. Collections utility & immutability
```java
Collections.emptyList(); Collections.singletonList(x);
Collections.synchronizedList(list);      // wrap with a lock
Collections.unmodifiableMap(map);        // read-only view
Collections.addAll(list, "a", "b");
Collections.binarySearch(sortedList, key);
```
**Unmodifiable view vs immutable copy**: views reflect changes to the underlying collection; `List.copyOf` doesn't.

`List.of / Set.of / Map.of`: no nulls; `Set.of` & `Map.of` iteration order is randomized per JVM run; duplicates in `Set.of("a","a")` throw `IllegalArgumentException`.

---

## 11. Concurrent collections (overview)
| Class | Notes |
|-------|-------|
| `ConcurrentHashMap` | Default thread-safe map |
| `CopyOnWriteArrayList/Set` | Copies array on every write; perfect for read-mostly listener lists |
| `ConcurrentLinkedQueue/Deque` | Lock-free non-blocking queues |
| `ArrayBlockingQueue` | Bounded, one lock |
| `LinkedBlockingQueue` | Optionally bounded, two locks (put/take) — used by `newFixedThreadPool` (unbounded!) |
| `PriorityBlockingQueue` | Unbounded priority |
| `DelayQueue` | Elements available after delay |
| `SynchronousQueue` | No capacity; handoff — used by `newCachedThreadPool` |
| `ConcurrentSkipListMap/Set` | Sorted, concurrent |

BlockingQueue methods: `put/take` (block), `offer/poll` (return immediately or with timeout).

---

## 12. Common interview coding patterns with collections
```java
// 1. Word frequency
Map<String, Long> freq = Arrays.stream(text.split("\\s+"))
    .collect(Collectors.groupingBy(String::toLowerCase, Collectors.counting()));

// 2. First non-repeated character
Map<Character, Integer> counts = new LinkedHashMap<>();
for (char c : s.toCharArray()) counts.merge(c, 1, Integer::sum);
counts.entrySet().stream().filter(e -> e.getValue() == 1).map(Map.Entry::getKey).findFirst();

// 3. Two-sum with HashMap
Map<Integer, Integer> seen = new HashMap<>();
for (int i = 0; i < nums.length; i++) {
    Integer j = seen.get(target - nums[i]);
    if (j != null) return new int[]{j, i};
    seen.put(nums[i], i);
}

// 4. Group anagrams
Map<String, List<String>> groups = new HashMap<>();
for (String w : words) {
    char[] c = w.toCharArray(); Arrays.sort(c);
    groups.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(w);
}

// 5. Remove duplicates keeping order
List<Integer> unique = new ArrayList<>(new LinkedHashSet<>(list));

// 6. Sort a map by value
map.entrySet().stream().sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
   .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue, (a, b) -> a, LinkedHashMap::new));
```

---

## 13. Interview Q&A

**Q. How does HashMap work internally?** (See §7.2 — hash spreading, bucket index, chaining, treeify at 8 when capacity ≥ 64, resize at load factor 0.75, equals for matching.)

**Q. What happens if two keys have the same hashCode?** Collision — they go into the same bucket; `equals` distinguishes them. Too many collisions degrade performance (tree bins limit worst case to O(log n)).

**Q. Why override hashCode and equals for HashMap keys?** Lookups use hashCode to find the bucket and equals to find the key; default identity versions make logically equal keys unfindable.

**Q. ArrayList vs LinkedList?** ArrayList: contiguous array, O(1) random access, cache-friendly. LinkedList: O(1) insert/remove at known node but O(n) access, more memory. Default to ArrayList.

**Q. HashSet vs TreeSet?** O(1) unordered vs O(log n) sorted with range ops.

**Q. How does ConcurrentHashMap achieve thread safety?** Java 8+: CAS + per-bin synchronization, lock-free reads, no global lock; Java 7 used segments (16 locks).

**Q. Fail-fast vs fail-safe?** Fail-fast throws CME on structural modification during iteration (modCount check). Fail-safe iterates over a snapshot/weakly consistent view.

**Q. Why is `Map` not part of `Collection`?** It stores pairs, not single elements; `add(E)` doesn't make sense.

**Q. Can we add null to TreeSet?** No (NPE) with natural ordering, since compareTo is invoked; possible with a null-friendly comparator.

**Q. What is the initial capacity and load factor of HashMap? Why 0.75?** 16 and 0.75 — balance between space waste and collision probability.

**Q. How to make a collection read-only?** `List.copyOf`/`List.of` (immutable) or `Collections.unmodifiableList` (view).
