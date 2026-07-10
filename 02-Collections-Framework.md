# 2. Java Collections Framework — Deep Dive

> **Goal:** After reading, you'll pick the right collection every time, know how `HashMap` works internally, and answer any collections question fluently.

---

## 2.0 What is a Collection?

A **collection** is a container that holds a group of objects. Just like real-world containers (shopping bag, filing cabinet, mailbox), each Java collection has different characteristics:

| Real-world thing | Java collection |
|-------------------|-----------------|
| A shopping list (ordered items, allow duplicates) | `List` (`ArrayList`) |
| A set of unique friends | `Set` (`HashSet`) |
| A dictionary (word → meaning) | `Map` (`HashMap`) |
| A queue at a ticket counter (first-come-first-served) | `Queue` (`ArrayDeque`) |
| A stack of plates (last-plate-first-out) | `Deque` used as Stack |

---

## 2.1 The Big Picture — Hierarchy

```
                Iterable<E>
                     │
                Collection<E>
        ┌────────────┼────────────┐
        │            │            │
      List<E>      Set<E>      Queue<E>
        │            │            │
        ├─ ArrayList ├─ HashSet   ├─ PriorityQueue
        ├─ LinkedList├─ LinkedHashSet└─ Deque<E>
        ├─ Vector    └─ TreeSet         ├─ ArrayDeque
        └─ Stack                        └─ LinkedList

Map<K,V>   (separate — NOT a Collection!)
├─ HashMap
├─ LinkedHashMap
├─ TreeMap
├─ Hashtable (legacy)
└─ ConcurrentHashMap
```

**Key insight:** `Map` is **not** a `Collection` because it stores key-value pairs, not single elements.

### How to choose?

Ask yourself:
1. **Order matters?** → `LinkedHashMap`/`LinkedHashSet` (insertion order) or `TreeMap`/`TreeSet` (sorted)
2. **Duplicates allowed?** → `List` yes, `Set` no
3. **Fast lookup?** → `HashMap`/`HashSet` (O(1) avg)
4. **Sorted?** → `TreeMap`/`TreeSet`
5. **Thread-safe?** → `ConcurrentHashMap`, `CopyOnWriteArrayList`

---

## 2.2 List — Ordered Sequence with Duplicates

### 2.2.1 ArrayList — Backed by a Growable Array

**Mental model:** A parking lot where cars are lined up in numbered slots. Getting car in slot #47 is instant, but inserting a new car in the middle requires shifting others.

**Internals:**
- Backed by `Object[]` array
- Default initial capacity: **10**
- When full, grows by **50%**: `newCap = oldCap + (oldCap >> 1)`
- Access by index: **O(1)**
- Insert/delete at middle: **O(n)** (shift elements)

```java
List<String> books = new ArrayList<>();   // capacity 10
books.add("Java");            // add at end - O(1) amortized
books.add(0, "Python");       // insert at index 0 - O(n) (shifts everything)
books.get(0);                 // O(1)
books.set(0, "Go");           // replace - O(1)
books.remove(0);              // O(n) (shifts remaining)
books.contains("Java");       // O(n) linear scan
books.size();                 // O(1)
```

**Try it:** watch capacity growth:
```java
ArrayList<Integer> list = new ArrayList<>(2);   // capacity 2
list.add(1);
list.add(2);         // full
list.add(3);         // triggers resize → capacity becomes 3 (2 + 2/2 = 3)
```

**Best for:** Read-heavy scenarios, sequential access.

### 2.2.2 LinkedList — Doubly Linked List

**Mental model:** A treasure hunt where each clue points to the next AND previous location.

```
[null | 1 | ●]─────►[● | 2 | ●]─────►[● | 3 | null]
                        ◄────────────────
```

**Internals:**
- Each node stores value + `prev` + `next` references
- Access by index: **O(n)** (walk from head/tail)
- Insert/delete at ends: **O(1)**
- Implements both `List` and `Deque` (double-ended queue)

```java
LinkedList<Integer> ll = new LinkedList<>();
ll.addFirst(1);              // O(1)
ll.addLast(2);               // O(1)
ll.removeFirst();            // O(1)
ll.get(1000);                // O(n) — walks from nearest end
```

**Best for:** Frequent insert/delete at ends, queue/deque usage.

### 2.2.3 ArrayList vs LinkedList — Choose Wisely

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| `get(i)` | **O(1)** | O(n) |
| `add(e)` at end | O(1) amortized | O(1) |
| `add(i, e)` middle | O(n) | O(n) walk + O(1) insert |
| `remove(i)` | O(n) | O(n) walk + O(1) delete |
| Memory | Compact | 2 extra refs per node |
| Cache friendly | ✅ (contiguous) | ❌ |

**Rule of thumb:** Default to `ArrayList`. Use `LinkedList` only when you're doing many insertions/removals at both ends (queue/deque). Even then, `ArrayDeque` is usually faster.

### 2.2.4 Vector & Stack — Legacy

- **`Vector`** — like `ArrayList` but every method is `synchronized`. Slow, rarely needed. Replace with `Collections.synchronizedList()` or `CopyOnWriteArrayList`.
- **`Stack`** extends `Vector` — LIFO. **Don't use.** Prefer `Deque`:

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);
stack.pop();   // 2
stack.peek();  // 1
```

---

## 2.3 Queue & Deque

### 2.3.1 Queue (FIFO — First In, First Out)

Like a ticket line — first person in is first served.

Methods (two flavors — the second throws exception):
| Purpose | Throws | Returns special |
|---------|--------|------------------|
| Insert | `add(e)` | `offer(e)` (returns false if full) |
| Remove head | `remove()` | `poll()` (returns null if empty) |
| Examine head | `element()` | `peek()` (returns null if empty) |

### 2.3.2 PriorityQueue — Heap-based

Elements are ordered by priority, not insertion order.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();      // min-heap by default
pq.offer(5); pq.offer(1); pq.offer(3);
pq.poll();    // 1 (smallest)
pq.poll();    // 3
pq.poll();    // 5

// Max-heap
PriorityQueue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder());

// Custom priority — tasks by deadline
PriorityQueue<Task> tasks = new PriorityQueue<>(
    Comparator.comparing(Task::deadline)
);
```

Operations: `offer` O(log n), `poll` O(log n), `peek` O(1).

**Real use case:** Top-K problems, Dijkstra's algorithm, task scheduling.

### 2.3.3 Deque (Double-ended queue)

Insert/remove from **both ends**.

```java
Deque<Integer> dq = new ArrayDeque<>();
dq.addFirst(1);
dq.addLast(2);
dq.pollFirst();   // 1
dq.pollLast();    // 2
```

**`ArrayDeque` is the modern replacement for both `Stack` and `LinkedList` (as queue).**

---

## 2.4 Set — Uniqueness Enforced

### 2.4.1 HashSet — Uses HashMap Internally

**Mental model:** A guest list where each guest can appear only once. Bouncer checks the ID at the door in O(1).

```java
Set<String> emails = new HashSet<>();
emails.add("a@x.com");
emails.add("a@x.com");        // duplicate — ignored, add returns false
emails.contains("a@x.com");   // O(1) average
emails.size();                 // 1
```

Internally: uses `HashMap` where every key maps to a dummy `Object PRESENT`.

**Best for:** Existence check, deduplication.

### 2.4.2 LinkedHashSet — Insertion Order Preserved

```java
Set<String> s = new LinkedHashSet<>();
s.add("banana"); s.add("apple"); s.add("cherry");
System.out.println(s);   // [banana, apple, cherry] — insertion order
```

Slightly slower than `HashSet` (extra linked list overhead), but preserves order.

### 2.4.3 TreeSet — Sorted (Red-Black Tree)

```java
TreeSet<Integer> t = new TreeSet<>();
t.add(3); t.add(1); t.add(2);
System.out.println(t);       // [1, 2, 3] — sorted

t.first();      // 1
t.last();       // 3
t.floor(2);     // 2 (≤ 2)
t.ceiling(2);   // 2 (≥ 2)
t.higher(2);    // 3 (strictly > 2)
t.headSet(2);   // [1]
t.tailSet(2);   // [2, 3]
```

Operations: all O(log n). Requires elements to be `Comparable` or provide `Comparator`.

### Comparison

| | HashSet | LinkedHashSet | TreeSet |
|-|---------|---------------|---------|
| Order | None | Insertion | Sorted |
| Nulls | 1 allowed | 1 allowed | ❌ (needs comparison) |
| Add/contains | O(1) | O(1) | O(log n) |
| Backed by | HashMap | LinkedHashMap | TreeMap |

---

## 2.5 Map — Key-Value Store

### 2.5.1 HashMap — The Workhorse

**Everyday operations:**
```java
Map<String, Integer> stock = new HashMap<>();
stock.put("apple", 5);
stock.put("banana", 3);
stock.get("apple");                          // 5
stock.getOrDefault("cherry", 0);             // 0
stock.putIfAbsent("apple", 99);              // no-op (apple exists)
stock.computeIfAbsent("mango", k -> 10);     // adds mango=10
stock.merge("apple", 1, Integer::sum);       // apple = 5 + 1 = 6
stock.remove("banana");
stock.containsKey("apple");
stock.size();

// Iteration
for (Map.Entry<String, Integer> e : stock.entrySet()) {
    System.out.println(e.getKey() + "=" + e.getValue());
}
stock.forEach((k, v) -> System.out.println(k + "=" + v));
```

**Deep dive follows in section 2.10.**

### 2.5.2 LinkedHashMap — Order Preserved

```java
Map<String, Integer> ordered = new LinkedHashMap<>();
ordered.put("a", 1); ordered.put("b", 2); ordered.put("c", 3);
System.out.println(ordered);   // {a=1, b=2, c=3}
```

**Access order mode → build an LRU cache in 5 lines:**
```java
Map<String, String> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 3;      // max 3 entries
    }
};
lru.put("a","A"); lru.put("b","B"); lru.put("c","C");
lru.get("a");                    // touch 'a' — moves to end
lru.put("d","D");                // 'b' evicted (least recently used)
System.out.println(lru);         // {c=C, a=A, d=D}
```

### 2.5.3 TreeMap — Sorted by Key

```java
TreeMap<String, Integer> tm = new TreeMap<>();
tm.put("banana", 2); tm.put("apple", 1); tm.put("cherry", 3);
System.out.println(tm);              // {apple=1, banana=2, cherry=3}
tm.firstKey();                        // apple
tm.floorKey("berry");                 // banana
tm.subMap("apple", "cherry");         // {apple=1, banana=2}
```

Backed by a Red-Black tree. All ops O(log n).

### 2.5.4 Hashtable — Legacy

- Thread-safe (every method synchronized) — slow!
- No null key or value
- Replaced by `ConcurrentHashMap`

### 2.5.5 ConcurrentHashMap — Modern Concurrent Map

Thread-safe with high concurrency. Deep dive in section 2.11.

```java
ConcurrentMap<String, Integer> cm = new ConcurrentHashMap<>();
cm.put("a", 1);
cm.putIfAbsent("a", 99);       // atomic
cm.compute("a", (k, v) -> v == null ? 1 : v + 1);   // atomic increment
```

**Never allows `null` keys or values.**

---

## 2.6 Comparable vs Comparator — Sorting Objects

### Comparable — natural order (built into the class)

```java
class Employee implements Comparable<Employee> {
    int id;
    String name;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);   // ascending by id
    }
}

List<Employee> list = ...;
Collections.sort(list);       // uses compareTo
```

### Comparator — external, multiple orderings

```java
Comparator<Employee> byName = Comparator.comparing(Employee::getName);
Comparator<Employee> bySalary = Comparator.comparingDouble(Employee::getSalary);
Comparator<Employee> combined = bySalary.reversed().thenComparing(byName);

list.sort(combined);
```

### Contract & pitfalls

- `compareTo(x)` returns negative / 0 / positive
- Should be **consistent with equals** (i.e., `compareTo == 0` iff `equals == true`)
- Never subtract! `a.id - b.id` can overflow. Use `Integer.compare(a.id, b.id)`.

| Comparable | Comparator |
|------------|------------|
| Interface implemented by class itself | Separate class / lambda |
| `compareTo(T)` | `compare(T, T)` |
| One "natural" order | Many orderings |
| `Collections.sort(list)` | `list.sort(comparator)` |

---

## 2.7 Iteration — Iterator, ListIterator, forEach

```java
// Standard for-each (uses Iterator internally)
for (String s : list) System.out.println(s);

// Explicit Iterator (allows safe removal)
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.startsWith("x")) it.remove();      // safe
}

// ❌ Bad — throws ConcurrentModificationException
for (String s : list) {
    if (s.startsWith("x")) list.remove(s);
}

// ListIterator — bidirectional, only for List
ListIterator<String> lit = list.listIterator();
while (lit.hasNext()) {
    String s = lit.next();
    if (s.length() > 5) lit.set(s.toUpperCase());
}

// forEach + lambda
list.forEach(System.out::println);
map.forEach((k, v) -> System.out.println(k + "=" + v));
```

---

## 2.8 Fail-Fast vs Fail-Safe Iterators

### Fail-Fast (default for most collections)

Tracks internal `modCount`. If the collection is structurally modified during iteration → throws `ConcurrentModificationException` immediately.

Examples: `ArrayList`, `HashMap`, `HashSet`.

```java
List<Integer> list = new ArrayList<>(List.of(1,2,3));
for (Integer i : list) {
    list.add(4);          // 💥 ConcurrentModificationException on next iteration
}
```

### Fail-Safe

Works on a snapshot / uses different mechanism — does NOT throw.

Examples: `CopyOnWriteArrayList`, `ConcurrentHashMap`.

```java
List<Integer> list = new CopyOnWriteArrayList<>(List.of(1,2,3));
for (Integer i : list) {
    list.add(4);          // ✅ no exception, but change not seen during this iteration
}
```

`CopyOnWriteArrayList` — every write creates a new copy of the array. Great for read-heavy, write-rare data (e.g., listener lists).

---

## 2.9 Time & Space Complexity Cheat Sheet

| Op | ArrayList | LinkedList | HashMap | TreeMap | HashSet | TreeSet |
|----|-----------|------------|---------|---------|---------|---------|
| add | O(1)* | O(1) at ends | O(1) avg | O(log n) | O(1) | O(log n) |
| remove | O(n) | O(1) at ends | O(1) | O(log n) | O(1) | O(log n) |
| get(idx) | O(1) | O(n) | – | – | – | – |
| get(key)/contains | O(n) | O(n) | O(1) | O(log n) | O(1) | O(log n) |

*O(1) amortized (occasional resize is O(n))

---

## 2.10 HashMap Internals — The Most Asked Question

### Data structure (Java 8+)

- Array of "buckets" (`Node<K,V>[] table`), default size **16**
- Each bucket is a **linked list**; if it grows too long, converted to a **Red-Black Tree** (Java 8+)
- Load factor: 0.75 (resize threshold)

```
table[0]  →  null
table[1]  →  [k1,v1] → [k5,v5] → null           (linked list)
table[2]  →  Red-Black tree (if len ≥ 8)
table[3]  →  [k3,v3] → null
...
table[15] →  null
```

Each entry:
```java
static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

### The hash function

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

Then bucket index: `index = (n - 1) & hash` where `n` is table length (always a power of 2).

**Why XOR the high 16 bits?** Because the low bits of hashCode often have poor distribution. XOR-ing with high bits mixes them → fewer collisions.

### put(K, V) — step by step

```
1. Compute h = hash(key)
2. Compute idx = (n-1) & h
3. If table[idx] is null → put node there
4. Else, walk the list:
     - If key equals existing → replace value, return old
     - Else if last → append new node
     - If chain length ≥ 8 AND table.length ≥ 64 → treeify
5. size++
6. If size > threshold (capacity * loadFactor) → resize
```

### Resize (the expensive part)

- Doubles the capacity: `newCap = 2 * oldCap`
- Rehashes every entry — O(n)
- Chose 0.75 as load factor: balance between space wasted (empty buckets) and collisions.

### Treeification thresholds

- List → Tree when bucket has **≥ 8** nodes AND table size **≥ 64**
- Tree → List when bucket has **≤ 6** nodes

Why 8? Poisson distribution: probability of a bucket having ≥ 8 entries under a well-distributed hash is ~1 in 10 million. So treeification is rare but crucial protection against DoS attacks (attacker crafts many keys with same hash).

### equals() + hashCode() contract — CRITICAL

```
1. If a.equals(b) → a.hashCode() == b.hashCode()   (MUST)
2. If a.hashCode() == b.hashCode() → a.equals(b) may or may not be true
3. hashCode() must be consistent (same object → same value across calls)
```

**Violation demo:**
```java
class BadKey {
    int id;
    BadKey(int id) { this.id = id; }
    @Override public boolean equals(Object o) {
        return o instanceof BadKey && ((BadKey) o).id == id;
    }
    // hashCode() NOT overridden → uses Object's identity hash
}

Map<BadKey, String> m = new HashMap<>();
m.put(new BadKey(1), "hi");
m.get(new BadKey(1));        // null! different hashCodes → different buckets
```

**Golden rule:** If you override `equals()`, you MUST override `hashCode()`.

Best template with `Objects.hash()`:
```java
class Book {
    private String isbn;
    private String title;

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Book)) return false;
        Book b = (Book) o;
        return Objects.equals(isbn, b.isbn);
    }
    @Override public int hashCode() { return Objects.hash(isbn); }
}
```

### Why HashMap is NOT thread-safe

In Java 7, a concurrent resize could form a cycle in a bucket's linked list → infinite loop on `get()`. In Java 8+ this specific bug is gone, but you can still get lost updates and inconsistent iterators.

**Solutions:**
- `Collections.synchronizedMap(new HashMap<>())` — lock on every op (slow)
- `ConcurrentHashMap` — much better

---

## 2.11 ConcurrentHashMap Deep Dive (Java 8+)

Thread-safe, high throughput.

**How it works:**
- Same bucket array as HashMap
- **Empty bucket** → uses **CAS** (Compare-And-Swap, lock-free) to insert
- **Non-empty bucket** → synchronizes only on that bucket's head node
- Multiple threads can update different buckets in parallel

```
Thread A puts to bucket[3]  ─┐
Thread B puts to bucket[7]  ─┤─► No contention
Thread C puts to bucket[10] ─┘

Thread D puts to bucket[3]  ─► synchronized(bucket[3].head)
```

**No null keys or values:** ambiguity — is `null` "not present" or "value is null"? — worse in concurrent context.

**Atomic compound operations:**
```java
map.putIfAbsent(k, v);
map.replace(k, oldV, newV);          // CAS
map.compute(k, (key, val) -> ...);   // atomic
map.merge(k, delta, Integer::sum);   // atomic increment
```

**Real use — thread-safe counter map:**
```java
ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();
// concurrent event handlers:
counters.computeIfAbsent(event, k -> new LongAdder()).increment();
```

---

## 2.12 Practical Recipes — Common Interview Coding

### 1) Word frequency counter

```java
String text = "to be or not to be";
Map<String, Long> freq = Arrays.stream(text.split(" "))
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {to=2, be=2, or=1, not=1}
```

### 2) Group employees by department

```java
Map<String, List<Employee>> byDept = emps.stream()
    .collect(Collectors.groupingBy(Employee::getDept));
```

### 3) LRU Cache (see LinkedHashMap section)

### 4) Deduplication preserving order

```java
List<Integer> input = List.of(3, 1, 3, 2, 1, 4);
List<Integer> unique = new ArrayList<>(new LinkedHashSet<>(input));
// [3, 1, 2, 4]
```

### 5) Sort map by value (descending)

```java
Map<String, Integer> sortedByValue = map.entrySet().stream()
    .sorted(Map.Entry.<String,Integer>comparingByValue().reversed())
    .collect(Collectors.toMap(
        Map.Entry::getKey, Map.Entry::getValue,
        (a,b) -> a, LinkedHashMap::new));
```

### 6) Convert List → Map

```java
Map<Long, Book> byId = books.stream()
    .collect(Collectors.toMap(Book::getId, Function.identity()));
```

Handle duplicates:
```java
Map<Long, Book> byId = books.stream()
    .collect(Collectors.toMap(Book::getId, b -> b, (a, b) -> a));
```

### 7) Immutable collections (Java 9+)

```java
List<Integer> l = List.of(1, 2, 3);            // immutable
Set<String>   s = Set.of("a", "b");            // immutable
Map<String,Integer> m = Map.of("a", 1, "b", 2);// immutable

// Larger maps:
Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)
);
```

---

## 2.13 Mini Project — In-Memory Cache with Statistics

Combines: Map, LinkedHashMap (LRU), AtomicLong, Comparable.

```java
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

public class InMemoryCache<K, V> {
    private final int capacity;
    private final LinkedHashMap<K, V> store;
    private final AtomicLong hits = new AtomicLong();
    private final AtomicLong misses = new AtomicLong();

    public InMemoryCache(int capacity) {
        this.capacity = capacity;
        this.store = new LinkedHashMap<>(16, 0.75f, true) {   // access order
            @Override protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
                return size() > InMemoryCache.this.capacity;
            }
        };
    }

    public synchronized V get(K key) {
        V v = store.get(key);
        if (v == null) misses.incrementAndGet(); else hits.incrementAndGet();
        return v;
    }

    public synchronized void put(K key, V value) { store.put(key, value); }

    public String stats() {
        long h = hits.get(), m = misses.get();
        double rate = (h + m == 0) ? 0 : (100.0 * h / (h + m));
        return String.format("size=%d hits=%d misses=%d hitRate=%.1f%%",
                             store.size(), h, m, rate);
    }

    public static void main(String[] args) {
        InMemoryCache<String, String> cache = new InMemoryCache<>(3);
        cache.put("a", "A"); cache.put("b", "B"); cache.put("c", "C");
        cache.get("a"); cache.get("b"); cache.get("x");
        cache.put("d", "D");                     // evicts 'c' (least recently used)
        System.out.println(cache.stats());
    }
}
```

---

## 2.14 Interview Question Bank

1. **HashMap vs Hashtable vs ConcurrentHashMap?**  
   Hashtable = legacy, sync every method (slow). ConcurrentHashMap = modern, per-bucket lock + CAS. HashMap = not thread-safe.

2. **Internal working of HashMap in Java 8?**  
   Array of nodes → linked list per bucket → red-black tree if bucket size ≥ 8. Uses `hashCode() ^ (hashCode() >>> 16)` for better distribution. Load factor 0.75 triggers resize (doubling capacity).

3. **Why load factor 0.75?**  
   Balance between space and time. Lower = more space, fewer collisions; higher = more collisions.

4. **Why treeify at 8?**  
   Poisson probability of ≥8 collisions ~1e-8 with good hash → indicates hostile input or bad hash. Tree gives O(log n) worst case vs O(n) for list.

5. **Fail-fast vs fail-safe iterators?**  
   Fail-fast throws `ConcurrentModificationException` on structural change (via `modCount`). Fail-safe iterates over snapshot (no exception, may miss updates).

6. **ArrayList vs LinkedList — when to use which?**  
   ArrayList for random access; LinkedList only for heavy insert/delete at ends (and usually `ArrayDeque` is better).

7. **Can HashMap have null keys/values?**  
   HashMap: 1 null key + many null values. ConcurrentHashMap and Hashtable: NO nulls.

8. **How does HashSet ensure uniqueness?**  
   Internally uses HashMap with a dummy value `PRESENT`. Adds return false if key already exists (checked via equals/hashCode).

9. **Can you sort a HashMap?**  
   No, but you can sort its entries by copying to `TreeMap` (by key) or `LinkedHashMap` after sorting the entry stream (by value).

10. **What is a WeakHashMap?**  
    Keys wrapped in `WeakReference`. When GC removes the referent, the map entry disappears. Great for cache-like structures.

11. **What is an IdentityHashMap?**  
    Uses `==` (reference equality) instead of `equals()` for keys.

12. **Difference between `HashSet` and `TreeSet`?**  
    HashSet = unordered, O(1). TreeSet = sorted, O(log n).

13. **How does ConcurrentHashMap achieve concurrency in Java 8?**  
    CAS for empty buckets, synchronized on head node for occupied buckets. No global lock.

14. **What is Comparable vs Comparator?**  
    Comparable = natural ordering, `compareTo`, in same class. Comparator = external, `compare(T,T)`, multiple orderings.

15. **How to make an ArrayList thread-safe?**  
    `Collections.synchronizedList(list)` (wrapper) or `CopyOnWriteArrayList` (better for read-heavy).

---

## 2.15 Quick Cheat Sheet

```
Use ArrayList by default.
Use HashMap for O(1) key lookup.
Use LinkedHashMap for LRU (accessOrder=true) or insertion-ordered map.
Use TreeMap when you need sorted keys.
Use ConcurrentHashMap in multithreaded code (never Hashtable).
NEVER use Vector or Stack — use ArrayDeque.
Always override hashCode() when overriding equals().
Iterating & mutating → use Iterator.remove() or CopyOnWriteArrayList.
```

Practice one implementation puzzle before your interview:  
**"Build an LRU cache in 10 minutes."** (Answered above with LinkedHashMap.)