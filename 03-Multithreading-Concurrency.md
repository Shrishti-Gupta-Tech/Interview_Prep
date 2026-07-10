# 3. Multithreading & Concurrency — Deep Dive

> **Goal:** Understand *why* concurrency is hard, master the tools Java gives you, and be able to design thread-safe code confidently.

---

## 3.0 Why Multithreading?

### Real-world analogy

A restaurant with 1 chef vs 4 chefs. With 1 chef:
- Only one order cooked at a time
- Idle time when waiting for oven

With 4 chefs (threads):
- Parallel work
- Better utilization of resources (CPU cores, memory, network)
- BUT: they need to share the same kitchen (memory), so coordination is needed!

### Benefits
- **Performance:** utilize multi-core CPUs
- **Responsiveness:** don't block UI while server call happens
- **Throughput:** handle many requests concurrently

### Costs
- **Complexity:** race conditions, deadlocks
- **Debugging pain:** bugs are non-deterministic
- **Overhead:** context switching, synchronization

---

## 3.1 Process vs Thread

| Process | Thread |
|---------|--------|
| An independent program with its own memory | A lightweight unit of execution inside a process |
| Heavy weight (own PID, memory map) | Light weight (shares memory with siblings) |
| Communication via IPC (pipes, sockets) | Communication via shared memory (variables) |
| Crashing doesn't affect other processes | Crashing (uncaught exception) can crash the whole JVM |

**Example:** Chrome opens each tab as a separate process (crash-isolation). Inside a tab, multiple threads render, run JS, network.

---

## 3.2 Thread Lifecycle

```
     new Thread(...)
          │
          ▼
       [NEW] ──start()──► [RUNNABLE] ─────► JVM scheduler picks it
                              │                  │
                              │                  ▼
                              │              (RUNNING)
                              ▼
                  ┌── wait() / join() no timeout ─► [WAITING]
                  ├── sleep(ms), wait(ms), join(ms) ─► [TIMED_WAITING]
                  └── waiting for monitor lock ─────► [BLOCKED]
                              │
                              ▼
                        [TERMINATED]
```

```java
Thread t = new Thread(() -> System.out.println("Hi"));
System.out.println(t.getState());   // NEW
t.start();
System.out.println(t.getState());   // RUNNABLE (or already TERMINATED for short tasks)
t.join();                            // main waits until t finishes
System.out.println(t.getState());   // TERMINATED
```

---

## 3.3 Creating Threads — 4 Ways

### 1) Extending Thread class

```java
class Worker extends Thread {
    public void run() {
        System.out.println("Working: " + Thread.currentThread().getName());
    }
}
new Worker().start();       // don't call run() directly — that runs on caller thread
```

**Downside:** Java allows single inheritance — you lose the ability to extend anything else.

### 2) Implementing Runnable (preferred)

```java
Runnable r = () -> System.out.println("Task");
new Thread(r, "worker-1").start();
```

### 3) Callable + Future (for tasks that return a value)

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

ExecutorService es = Executors.newSingleThreadExecutor();
Future<Integer> future = es.submit(task);
Integer result = future.get();     // blocks until done
es.shutdown();
```

| Runnable | Callable |
|----------|----------|
| `void run()` | `V call() throws Exception` |
| No return | Returns a value |
| Cannot throw checked exception | Can throw checked exception |

### 4) CompletableFuture (Java 8+ async pipeline)

Covered in section 3.7.

---

## 3.4 The Big Picture — Concurrency Problems

Three classic issues when threads share state:

### 1) Race Condition

Two threads read → modify → write, and their operations interleave badly.

```java
class Counter {
    int count = 0;
    void increment() { count++; }   // NOT atomic! (read, add, write)
}

Counter c = new Counter();
Runnable r = () -> { for (int i = 0; i < 100_000; i++) c.increment(); };
Thread t1 = new Thread(r), t2 = new Thread(r);
t1.start(); t2.start(); t1.join(); t2.join();
System.out.println(c.count);   // Expected 200_000; actual likely less!
```

**Fix:** `synchronized`, `AtomicInteger`, `Lock`.

### 2) Visibility

Changes made by one thread may not be visible to another due to CPU caches.

```java
class Runner {
    boolean stop = false;    // NOT volatile
    void run() {
        while (!stop) { /* work */ }        // may never see stop=true!
    }
}
```

**Fix:** `volatile`, `synchronized`, `Atomic*`.

### 3) Deadlock

Two threads each hold a lock and wait for the other's lock forever.

```java
Object A = new Object(), B = new Object();

Thread t1 = new Thread(() -> {
    synchronized (A) { sleep(100); synchronized (B) { /* work */ } }
});
Thread t2 = new Thread(() -> {
    synchronized (B) { sleep(100); synchronized (A) { /* work */ } }
});
```

**Fix:** consistent lock ordering, `tryLock(timeout)`, avoid nested locks.

---

## 3.5 synchronized — Java's Built-in Lock

Every object has an implicit monitor (lock).

```java
class Counter {
    private int count;
    public synchronized void increment() { count++; }             // instance lock (this)
    public synchronized int get() { return count; }
    public static synchronized void staticMethod() { }            // class lock (Counter.class)

    public void other() {
        synchronized (this) {                                     // explicit block
            // critical section
        }
        synchronized (Counter.class) {                             // class-level block
            // ...
        }
    }
}
```

### Rules
1. Only one thread can hold a given monitor at a time
2. Reentrant — same thread can re-acquire the same lock (avoids self-deadlock)
3. Released automatically when method/block ends (even on exception)

### Analogy
Bathroom with one key hanging outside. Only one person at a time. Others wait outside. When you come out, you hang the key back.

---

## 3.6 volatile — Guarantee Visibility (Not Atomicity!)

`volatile` tells JVM: always read this variable from main memory, don't cache in registers/L1.

```java
class Runner {
    private volatile boolean running = true;

    public void stop() { running = false; }         // visible to all threads

    public void run() {
        while (running) { /* work */ }              // will exit when stop() called
    }
}
```

**BUT: `volatile` doesn't make compound operations atomic!**
```java
volatile int counter = 0;
counter++;             // still NOT thread-safe (read + add + write)
```

For that, use `AtomicInteger` or `synchronized`.

### When to use `volatile`
- Flags (stop, ready)
- Publication of an immutable object reference (safe publication)

---

## 3.7 Atomic Classes — Lock-Free Concurrency

Use CPU-level **CAS (Compare-And-Swap)** — atomic without locks.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
counter.getAndAdd(5);
counter.compareAndSet(10, 20);              // only sets if current is 10

AtomicReference<String> ref = new AtomicReference<>("A");
ref.compareAndSet("A", "B");                // atomic ref swap

AtomicLong al = new AtomicLong();
al.updateAndGet(v -> v * 2);                // Java 8 functional

// For high contention → LongAdder is faster than AtomicLong
LongAdder adder = new LongAdder();
adder.increment();
long total = adder.sum();
```

### How CAS works
```
Try to change x from expected old value E to new value N:
1. Read current value X of x
2. If X == E, atomically set x = N. Return success.
3. Else, retry (loop).
```

CAS is implemented via CPU instructions (`CMPXCHG` on x86). Truly lock-free.

**Rule of thumb:** For a single counter, `AtomicInteger`. For many concurrent increments (high contention), `LongAdder`.

---

## 3.8 Locks (java.util.concurrent.locks)

More flexible than `synchronized`.

### ReentrantLock

```java
Lock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock();      // ALWAYS in finally!
}
```

### Features that `synchronized` lacks

**1) tryLock — non-blocking or with timeout**
```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { /* work */ }
    finally { lock.unlock(); }
} else {
    // couldn't get lock in 1 sec → back off
}
```

**2) Fair lock (FIFO order)**
```java
Lock fairLock = new ReentrantLock(true);   // longest-waiting thread wins
```

**3) Interruptible**
```java
lock.lockInterruptibly();
```

**4) Multiple conditions per lock**
```java
Lock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

// producer:
lock.lock();
try {
    while (queue.isFull()) notFull.await();
    queue.add(item);
    notEmpty.signal();
} finally { lock.unlock(); }
```

### ReadWriteLock — many readers or one writer

```java
ReadWriteLock rw = new ReentrantReadWriteLock();

// Reader:
rw.readLock().lock();
try { /* multiple readers can hold this simultaneously */ }
finally { rw.readLock().unlock(); }

// Writer:
rw.writeLock().lock();
try { /* exclusive */ }
finally { rw.writeLock().unlock(); }
```

Great for caches — many reads, occasional writes.

### synchronized vs ReentrantLock

| synchronized | ReentrantLock |
|--------------|---------------|
| Implicit | Explicit lock/unlock |
| No timeout | `tryLock(timeout)` |
| Not interruptible | `lockInterruptibly()` |
| No fairness | Optional fair mode |
| One wait-set | Multiple `Condition`s |
| Auto-release on exception | Manual in `finally` |
| Simpler | More powerful |

---

## 3.9 wait / notify / notifyAll — Classic Coordination

Object monitor methods. Must be called inside `synchronized`.

- `wait()` — release lock, sleep until `notify()`
- `notify()` — wake ONE waiting thread
- `notifyAll()` — wake ALL

**Classic Producer-Consumer:**
```java
class Buffer {
    private final Queue<Integer> q = new LinkedList<>();
    private final int CAPACITY = 5;

    public synchronized void produce(int val) throws InterruptedException {
        while (q.size() == CAPACITY) wait();
        q.add(val);
        notifyAll();
    }

    public synchronized int consume() throws InterruptedException {
        while (q.isEmpty()) wait();
        int v = q.poll();
        notifyAll();
        return v;
    }
}
```

**Key rules:**
1. Always call `wait()` in a `while` loop (guard against spurious wakeups).
2. Prefer `notifyAll()` unless you're 100% sure only one waiter should wake.

Modern equivalent: `BlockingQueue`.

---

## 3.10 ExecutorService & Thread Pools

Creating a new `Thread` for every task is expensive. Use a pool.

### Analogy
Instead of hiring a new chef for every order, keep 4 chefs on payroll and hand them orders as they come.

```java
ExecutorService es = Executors.newFixedThreadPool(4);
for (int i = 0; i < 10; i++) {
    int taskId = i;
    es.submit(() -> System.out.println("Task " + taskId + " on " + Thread.currentThread().getName()));
}
es.shutdown();                              // stop accepting; finish running
es.awaitTermination(1, TimeUnit.MINUTES);
```

### Factory methods (Executors class)

| Method | What you get |
|--------|--------------|
| `newFixedThreadPool(n)` | n permanent threads |
| `newSingleThreadExecutor()` | 1 thread — sequential |
| `newCachedThreadPool()` | grows/shrinks (60s idle timeout) |
| `newScheduledThreadPool(n)` | supports delayed/periodic tasks |
| `newWorkStealingPool()` | Java 8, ForkJoinPool with work-stealing |

### Building your own with ThreadPoolExecutor

```java
ExecutorService es = new ThreadPoolExecutor(
    4,                                          // corePoolSize
    10,                                         // maximumPoolSize
    60, TimeUnit.SECONDS,                       // keepAliveTime
    new LinkedBlockingQueue<>(100),             // task queue
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()   // rejection policy
);
```

### Rejection policies (queue full & max threads reached)

- `AbortPolicy` — throws `RejectedExecutionException` (default)
- `CallerRunsPolicy` — the calling thread runs the task (natural backpressure!)
- `DiscardPolicy` — silently drops
- `DiscardOldestPolicy` — drops the oldest queued task

### Scheduled tasks

```java
ScheduledExecutorService ses = Executors.newScheduledThreadPool(2);
ses.schedule(() -> System.out.println("after 3s"), 3, TimeUnit.SECONDS);
ses.scheduleAtFixedRate(() -> heartbeat(), 0, 5, TimeUnit.SECONDS);
ses.scheduleWithFixedDelay(() -> pollDb(), 0, 5, TimeUnit.SECONDS);
```

`FixedRate` fires every N sec regardless of task duration; `FixedDelay` waits N sec **after** each task ends.

---

## 3.11 Future & CompletableFuture

### Future — blocking

```java
Future<Integer> f = es.submit(() -> heavyCalc());
Integer result = f.get();                          // blocks
Integer result2 = f.get(1, TimeUnit.SECONDS);      // throws TimeoutException
f.cancel(true);
f.isDone();
```

### CompletableFuture — async pipeline (Java 8)

Chain, combine, handle errors.

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchUser())              // starts task
    .thenApply(user -> user.getEmail())          // transform
    .thenApply(String::toUpperCase);

cf.thenAccept(System.out::println);              // consume result
```

**Combine two futures:**
```java
CompletableFuture<Integer> a = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> b = CompletableFuture.supplyAsync(() -> 20);
CompletableFuture<Integer> sum = a.thenCombine(b, Integer::sum);
sum.thenAccept(System.out::println);             // 30
```

**Wait for all / any:**
```java
CompletableFuture.allOf(a, b).join();            // wait for both
CompletableFuture.anyOf(a, b).thenAccept(...);   // whichever finishes first
```

**Error handling:**
```java
cf.exceptionally(ex -> "default value on error")
  .handle((result, ex) -> ex != null ? "err" : result);
```

**Real-world parallel API calls:**
```java
CompletableFuture<Profile> p = CompletableFuture.supplyAsync(() -> fetchProfile(userId));
CompletableFuture<List<Order>> o = CompletableFuture.supplyAsync(() -> fetchOrders(userId));
CompletableFuture<Recs> r = CompletableFuture.supplyAsync(() -> fetchRecs(userId));

CompletableFuture.allOf(p, o, r).join();
Dashboard dash = new Dashboard(p.join(), o.join(), r.join());
```

Result: 3 sec (max) instead of 3+3+3 = 9 sec.

---

## 3.12 Fork/Join Framework & Parallel Streams

Divide-and-conquer parallelism with **work-stealing**.

```java
class SumTask extends RecursiveTask<Long> {
    private final long[] arr;
    private final int lo, hi;
    private static final int THRESHOLD = 10_000;

    SumTask(long[] arr, int lo, int hi) {
        this.arr = arr; this.lo = lo; this.hi = hi;
    }

    protected Long compute() {
        if (hi - lo <= THRESHOLD) {
            long s = 0;
            for (int i = lo; i < hi; i++) s += arr[i];
            return s;
        }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();                        // async
        long rightResult = right.compute(); // sync
        long leftResult = left.join();
        return leftResult + rightResult;
    }
}

long[] arr = new long[10_000_000];
Arrays.setAll(arr, i -> i);
long total = ForkJoinPool.commonPool().invoke(new SumTask(arr, 0, arr.length));
```

### Parallel streams — the easy version

```java
long sum = LongStream.range(0, 10_000_000).parallel().sum();
```

Under the hood uses the common ForkJoinPool.

**Watch out:**
- Not always faster (overhead of splitting)
- Shared mutable state = disaster
- Order may be different (use `forEachOrdered` if it matters)
- Blocks in stream ops harm the common pool → be careful in web servers

---

## 3.13 Concurrent Collections

### Overview

| Collection | Purpose |
|------------|---------|
| `ConcurrentHashMap` | Thread-safe map |
| `CopyOnWriteArrayList` | Thread-safe list, best for read-heavy |
| `ConcurrentLinkedQueue` | Lock-free non-blocking queue |
| `LinkedBlockingQueue` | Blocking (with capacity) queue |
| `ArrayBlockingQueue` | Bounded blocking queue |
| `PriorityBlockingQueue` | Priority + blocking |
| `DelayQueue` | Blocking, elements available only after delay |
| `SynchronousQueue` | Zero-capacity — every put waits for a take |

### BlockingQueue — the producer-consumer champion

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

// producer
queue.put(task);          // blocks if full
queue.offer(task, 1, TimeUnit.SECONDS);

// consumer
Task t = queue.take();    // blocks if empty
Task t2 = queue.poll(1, TimeUnit.SECONDS);
```

Full producer-consumer in 20 lines:
```java
BlockingQueue<Integer> q = new ArrayBlockingQueue<>(10);

Thread producer = new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        try { q.put(i); System.out.println("Produced " + i); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
});

Thread consumer = new Thread(() -> {
    while (true) {
        try { int v = q.take(); System.out.println("Consumed " + v); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
    }
});

producer.start(); consumer.start();
```

---

## 3.14 Concurrency Utilities — Synchronizers

### CountDownLatch — wait for N events

```java
CountDownLatch latch = new CountDownLatch(3);

// 3 worker threads
Runnable worker = () -> {
    try { doWork(); } finally { latch.countDown(); }
};

new Thread(worker).start();
new Thread(worker).start();
new Thread(worker).start();

latch.await();          // main blocks until countdown reaches 0
System.out.println("All workers done");
```

Use case: waiting for all micro-services to boot before serving traffic.

### CyclicBarrier — N threads wait at a rendezvous point

```java
CyclicBarrier barrier = new CyclicBarrier(3,
    () -> System.out.println("All arrived — starting phase 2"));

Runnable task = () -> {
    System.out.println(Thread.currentThread().getName() + " arrived");
    try { barrier.await(); } catch (Exception e) {}
    System.out.println(Thread.currentThread().getName() + " going");
};

new Thread(task).start(); new Thread(task).start(); new Thread(task).start();
```

Difference from `CountDownLatch`: barrier is **reusable** (`reset()`).

### Semaphore — limit concurrent access

Analogy: a coffee shop with 3 tables — only 3 customers at once.

```java
Semaphore sem = new Semaphore(3);

Runnable customer = () -> {
    try {
        sem.acquire();      // wait if 3 already inside
        System.out.println("Inside");
        Thread.sleep(1000);
    } catch (InterruptedException e) {}
    finally { sem.release(); }
};
```

Use case: rate-limit external API calls, DB connections.

### Phaser — flexible barrier

Advanced multi-phase coordination. Rarely asked.

### Exchanger — one-shot two-thread value swap

```java
Exchanger<String> ex = new Exchanger<>();

new Thread(() -> {
    String reply = ex.exchange("Hello");
    System.out.println("A got " + reply);
}).start();
new Thread(() -> {
    String reply = ex.exchange("World");
    System.out.println("B got " + reply);
}).start();
```

---

## 3.15 ThreadLocal — Per-Thread Data

Each thread has its own copy — no synchronization needed.

**Classic use: date formatters (SimpleDateFormat is NOT thread-safe).**

```java
private static final ThreadLocal<SimpleDateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

String today = FMT.get().format(new Date());
```

**Spring uses ThreadLocal heavily** — user context, transaction context, security context.

**⚠️ Memory leak trap:** In thread pools, the same thread lives forever. If you `set()` a value and forget to `remove()`, it stays. Best practice:
```java
try {
    FMT.set(new SimpleDateFormat(...));
    // work
} finally {
    FMT.remove();
}
```

Java 21+ introduces **`ScopedValue`** — safer, immutable alternative.

---

## 3.16 Deadlock — Detection & Prevention

### The dining philosophers problem

5 philosophers around a table, 5 forks. Each needs 2 forks to eat. If all grab the left fork simultaneously → deadlock.

### Prevention techniques

**1) Consistent lock order** (always lock A before B):
```java
public void transfer(Account from, Account to, double amt) {
    Account first  = from.id < to.id ? from : to;
    Account second = from.id < to.id ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.debit(amt); to.credit(amt);
        }
    }
}
```

**2) tryLock with timeout**
```java
if (lockA.tryLock(1, SECONDS)) {
    try {
        if (lockB.tryLock(1, SECONDS)) {
            try { /* work */ }
            finally { lockB.unlock(); }
        }
    } finally { lockA.unlock(); }
}
```

**3) Avoid nested locks** if possible.

### Detect deadlocks
- `jstack <pid>` — prints stack traces; shows "Found deadlock"
- `ThreadMXBean.findDeadlockedThreads()`
- IntelliJ / VisualVM

---

## 3.17 Thread Safety Strategies

Ranked from most to least preferred:

1. **Don't share state (immutability)** — if there's nothing to change, no bug possible.
2. **ThreadLocal** — everyone gets a copy.
3. **Atomic classes / concurrent collections** — designed for you.
4. **synchronized / Lock** — last resort.

---

## 3.18 Real-World Mini Project — Web Scraper with Bounded Parallelism

```java
import java.net.*;
import java.util.*;
import java.util.concurrent.*;

public class Scraper {
    private static final Semaphore RATE_LIMIT = new Semaphore(5); // max 5 concurrent

    public static String fetch(String url) throws Exception {
        RATE_LIMIT.acquire();
        try (var in = new URL(url).openStream()) {
            return new String(in.readAllBytes()).substring(0, 100);
        } finally {
            RATE_LIMIT.release();
        }
    }

    public static void main(String[] args) throws Exception {
        List<String> urls = List.of(
            "https://example.com", "https://example.org", "https://example.net"
            /* ... hundreds ... */
        );

        ExecutorService es = Executors.newFixedThreadPool(20);
        List<CompletableFuture<String>> futures = urls.stream()
            .map(u -> CompletableFuture.supplyAsync(() -> {
                try { return fetch(u); }
                catch (Exception e) { return "ERR: " + e.getMessage(); }
            }, es))
            .toList();

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        futures.forEach(f -> System.out.println(f.join()));
        es.shutdown();
    }
}
```

Techniques used: `Semaphore`, `CompletableFuture`, `ExecutorService`, `allOf`.

---

## 3.19 Thread-Safe Singleton (Design Pattern Interview Classic)

**Wrong:**
```java
public class Singleton {
    private static Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) instance = new Singleton();  // race condition!
        return instance;
    }
}
```

**DCL (Double-Checked Locking) — thread-safe & lazy:**
```java
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```
`volatile` prevents instruction reordering (crucial!).

**Bill Pugh — cleanest, lazy, thread-safe, no sync:**
```java
public class Singleton {
    private Singleton() {}
    private static class Holder { static final Singleton INSTANCE = new Singleton(); }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

**Enum — Joshua Bloch's recommendation:**
```java
public enum Singleton {
    INSTANCE;
    public void doThing() {}
}
```

---

## 3.20 Java Memory Model & happens-before

The JMM defines what memory operations are guaranteed to be visible across threads.

**"happens-before" relationships:**
- Anything before `Thread.start()` is visible to the new thread
- Anything a thread does before terminating is visible after `Thread.join()`
- `unlock` happens-before subsequent `lock` on same monitor
- Write to `volatile` happens-before subsequent read of same volatile
- Constructor of an object happens-before its finalizer

**Practical implication:** the compiler & CPU can reorder instructions unless there's a happens-before ordering. That's why `volatile` and `synchronized` are essential — they establish ordering.

---

## 3.21 Interview Question Bank

1. **Runnable vs Callable?**  
   Runnable: `void run()`. Callable: `V call() throws Exception` returns value, can throw checked.

2. **What is thread pool? Why use it?**  
   Pool of pre-created threads. Avoids overhead of thread creation, limits max concurrency, provides queueing.

3. **synchronized vs volatile vs Atomic?**  
   synchronized: mutual exclusion + visibility. volatile: visibility only (no atomicity). Atomic: lock-free atomic ops via CAS.

4. **How to solve deadlock?**  
   Consistent lock order, tryLock with timeout, avoid nested locks, use higher-level utilities (BlockingQueue).

5. **wait() vs sleep()?**  
   wait(): Object method, must be inside synchronized, releases lock, wakes on notify. sleep(): Thread static, does NOT release lock.

6. **wait() vs join()?**  
   wait: waits until notified. join: waits until the target thread dies.

7. **What is CompletableFuture? Why better than Future?**  
   Non-blocking async pipeline. Chain callbacks, combine, handle errors. Future only offers blocking `get()`.

8. **Explain ConcurrentHashMap internals.**  
   Java 8+: CAS for empty buckets, synchronized on head node for non-empty ones. No global lock.

9. **What is ThreadLocal? Use cases?**  
   Per-thread state. Used for SimpleDateFormat, transaction context (Spring), security context, request tracing.

10. **submit() vs execute()?**  
    submit returns Future; execute is void (Runnable only, exceptions get lost).

11. **What is Fork/Join? Work-stealing?**  
    Divide-and-conquer framework. Idle threads steal work from busy threads' deques. Used by parallel streams.

12. **volatile guarantees ordering?**  
    Yes — reads/writes to volatile establish happens-before ordering. Prevents reordering.

13. **Best way to implement Singleton in multi-threaded env?**  
    Enum > Bill Pugh > Double-Checked Locking with volatile.

14. **Why is HashMap not thread-safe?**  
    Concurrent puts can corrupt internal structure (in Java 7 could cause infinite loop). Use `ConcurrentHashMap`.

15. **Race condition vs deadlock?**  
    Race: wrong result due to interleaving. Deadlock: threads stuck waiting for each other's locks.

16. **How does CAS work?**  
    Compare-And-Swap: atomically check-then-set. Hardware primitive (`CMPXCHG`). Basis for atomic classes.

17. **What are daemon threads?**  
    Background threads that don't prevent JVM exit. `t.setDaemon(true)` before `start()`. Example: GC.

18. **How would you handle a slow producer, fast consumer?**  
    Use a bounded `BlockingQueue`; consumer's `take()` blocks when empty.

19. **What is virtual thread (Java 21)?**  
    Lightweight threads managed by JVM, not OS. Millions can run concurrently. Great for IO-bound work.

20. **Explain the happens-before relationship.**  
    Guarantees when one action's effect is visible to another. Established by locks, volatile, thread start/join.

---

## 3.22 Cheat Sheet

```
Prefer immutability > ThreadLocal > Atomic/concurrent collections > synchronized/Lock
volatile = visibility, NOT atomicity
synchronized = visibility + atomicity + reentrant
Locks: always lock in try, unlock in finally
BlockingQueue solves 90% of producer-consumer scenarios
Never call thread.run() directly — call thread.start()
Always shutdown() ExecutorService in finally / try-with-resources
CompletableFuture.allOf for parallel calls; join() to get results
Use G1/ZGC GC and profile before micro-optimizing threads
```

---

## Practical Assignment

1. Rewrite a synchronized counter using `AtomicInteger` and measure the performance difference.
2. Implement `parallel API aggregation`: fetch user profile, orders, and preferences in parallel using `CompletableFuture`.
3. Build a rate limiter using `Semaphore`.
4. Write a bounded blocking queue with `ReentrantLock` + `Condition` from scratch.
5. Design a producer-consumer with two thread pools and a `LinkedBlockingQueue`.

Get comfortable with these and you'll ace concurrency interviews. 🚀