# 1. Java Fundamentals (Core Java) — Deep Dive

> **Goal of this chapter:** After reading, you'll understand *why* Java works the way it does — not just *what* it does. Every concept comes with a real-world analogy, code you can run, and interview-ready explanation.

---

## 1.0 How Java Works (Big Picture)

### Real-world analogy

Imagine you write a novel in English but want people in Japan, France, and Brazil to read it. Two options:

1. **Translate the entire novel to each language** (like C/C++ which compiles separately for each OS).
2. **Write once in a universal script + give every reader a translator device** — that's Java!

### The journey of your code

```
    Hello.java  (source code)
         │
         │  javac (compiler)
         ▼
   Hello.class  (bytecode)  ← universal, works on ANY JVM
         │
         │  java command → JVM
         ▼
   Machine code (0s & 1s)   ← what the CPU actually executes
```

### Try it hands-on

Create `Hello.java`:
```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

Run:
```bash
javac Hello.java         # compile → produces Hello.class
javap -c Hello           # peek at bytecode instructions
java Hello               # JVM executes bytecode
```

**Interview one-liner:** *"Java is platform-independent because the JVM abstracts the OS — the same bytecode runs anywhere a JVM exists."*

---

## 1.1 JDK vs JRE vs JVM

### Car analogy

- **JVM** = the engine (runs the car)
- **JRE** = engine + all parts to actually drive (JVM + core libraries)
- **JDK** = the mechanic's full garage (JRE + tools to *build* the car — compilers, debuggers, docs)

| Component | Purpose | Contains |
|-----------|---------|----------|
| **JVM** | Executes bytecode | ClassLoader, Bytecode verifier, Interpreter, JIT compiler, GC |
| **JRE** | Runs Java apps | JVM + core libraries (`java.lang`, `java.util`, `java.io`…) |
| **JDK** | Develops Java apps | JRE + `javac`, `javadoc`, `jar`, `jdb`, `jshell` |

**Relationship:** `JDK ⊃ JRE ⊃ JVM`

### JVM internals — what happens when you run `java Hello`

```
┌───────────────────────────────────────────────────────────┐
│                          JVM                              │
│                                                           │
│  1. ClassLoader Subsystem                                 │
│     ├── Bootstrap ClassLoader (loads java.* core classes) │
│     ├── Extension ClassLoader (loads ext/*.jar)           │
│     └── App ClassLoader       (loads your app's classpath)│
│                                                           │
│  2. Bytecode Verifier — validates the .class is safe      │
│                                                           │
│  3. Runtime Data Areas                                    │
│     ├── Heap          (objects — shared)                  │
│     ├── Method Area   (class metadata, static)            │
│     ├── Stack         (one per thread — method frames)    │
│     ├── PC Register   (per thread — current instruction)  │
│     └── Native Stack  (per thread — for JNI calls)        │
│                                                           │
│  4. Execution Engine                                      │
│     ├── Interpreter                                       │
│     ├── JIT Compiler (hot code → native machine code)     │
│     └── Garbage Collector                                 │
│                                                           │
│  5. Native Method Interface (JNI) — talk to C/C++         │
└───────────────────────────────────────────────────────────┘
```

**Interpreter vs JIT:** The interpreter runs bytecode line-by-line (slow but simple). When JVM notices a "hot" method (called many times), the JIT compiles it to native code and caches it — subsequent calls are as fast as C++.

### Interview questions

- *Can you run Java without JDK?* → Yes, JRE is enough to run. JDK is needed only to compile.
- *Is JVM platform-dependent or independent?* → **JVM is platform-dependent** (different for Windows/Linux/Mac). **Bytecode is platform-independent**.

---

## 1.2 Java Memory Model — Where Does Every Variable Live?

Understanding memory is the difference between an average and a strong Java developer.

### The two main areas

Think of a **hotel**:
- **Heap** = the shared parking lot where all guests park cars (objects) — everyone can access it.
- **Stack** = each guest's private safe in their room — only they can open it (local variables & method frames).

```java
public class MemoryDemo {
    static int companyCount = 100;       // Method Area (Metaspace)

    String name;                          // Heap (lives inside the object)

    public void greet(String message) {   // 'message' is a parameter → Stack
        int localVar = 10;                // Stack
        String greeting = new String("Hi");
        //  ^^^^^^^^ reference on Stack, the actual String object on Heap
    }
}
```

### Detailed regions

```
┌─────────────── HEAP ────────────────┐
│                                     │
│  ┌────── Young Generation ──────┐   │
│  │  ┌─────┐ ┌──────┐ ┌──────┐  │   │
│  │  │Eden │ │S0    │ │S1    │  │   │
│  │  └─────┘ └──────┘ └──────┘  │   │
│  │  New objects live here first │   │
│  └──────────────────────────────┘   │
│                                     │
│  ┌────── Old (Tenured) Gen ─────┐   │
│  │  Objects that survived many   │   │
│  │  minor GCs are promoted here  │   │
│  └───────────────────────────────┘   │
└─────────────────────────────────────┘

┌────── Metaspace (Java 8+) ──────────┐
│  Class metadata, static variables,  │
│  method bytecode, constants         │
│  (Replaces old PermGen; grows       │
│   in native memory, not heap)       │
└─────────────────────────────────────┘

┌────── Stack (per thread) ───────────┐
│  Method frame 3 (top)               │
│  Method frame 2                     │
│  Method frame 1 (bottom = main)     │
│  Each frame: local vars, params,    │
│  operand stack, return address      │
└─────────────────────────────────────┘
```

### Errors related to memory

| Error | Cause | Example |
|-------|-------|---------|
| `StackOverflowError` | Stack exhausted (usually infinite recursion) | `void f() { f(); }` |
| `OutOfMemoryError: Java heap space` | Heap full | Creating too many objects, memory leak |
| `OutOfMemoryError: Metaspace` | Loading too many classes | Loading classes dynamically without bound |

### Practical: See it yourself

```java
public class HeapDemo {
    public static void main(String[] args) {
        Runtime rt = Runtime.getRuntime();
        System.out.println("Total memory:  " + rt.totalMemory() / 1024 / 1024 + " MB");
        System.out.println("Free memory:   " + rt.freeMemory()  / 1024 / 1024 + " MB");
        System.out.println("Max memory:    " + rt.maxMemory()   / 1024 / 1024 + " MB");
    }
}
```

Tune with JVM flags:
```bash
java -Xms256m -Xmx1g HeapDemo    # min 256MB, max 1GB heap
```

### Interview one-liner

*"Local variables and primitives live on the stack per thread. Objects live on the heap and are shared across threads. Class metadata lives in the Metaspace since Java 8 (replacing PermGen)."*

---

## 1.3 Garbage Collection (GC) — Java's Auto-Cleaner

### Analogy

Imagine your desk gets messy with papers you no longer need. In C++, you must throw them away manually (`free()`). In Java, a **cleaning robot** silently comes at intervals and throws away anything you're not touching anymore.

### How Java decides "not needed"

An object is eligible for GC when **no live reference points to it**.

```java
Book b = new Book();     // Book object created on heap, 'b' points to it
b = null;                // now no reference → eligible for GC

// Or reassigning
Book b1 = new Book();
Book b2 = new Book();
b1 = b2;                 // the first Book object is now unreachable
```

### Generational hypothesis

Most objects die young. GC leverages this by splitting the heap:

```
NEW OBJECT
    │
    ▼
  Eden  ─── minor GC ───► Survivor S0 ─── minor GC ───► Survivor S1
    │                          │                            │
    │            (age counter increments each survival)     │
    │                                                       ▼
    │                                                Old (Tenured) — after ~15 survivals
    │                                                       │
    │                                                       ▼
    │                                              Full GC (major GC)
```

### GC algorithms

| Algorithm | Best for | Notes |
|-----------|----------|-------|
| **Serial GC** | Small apps (single core) | Stops the world, single thread |
| **Parallel GC** | Batch processing (throughput) | Multi-thread, still stops the world |
| **CMS** (Concurrent Mark Sweep) | Deprecated | Low pause but fragmentation |
| **G1** (Garbage First) — default since Java 9 | Balanced (predictable pauses) | Heap divided into regions |
| **ZGC / Shenandoah** | Very low latency (< 10ms pauses) | Java 11+, huge heaps |

Set via: `java -XX:+UseG1GC MyApp`

### Types of GC events

- **Minor GC** — cleans only Young Gen (fast)
- **Major GC** — cleans Old Gen (slower)
- **Full GC** — cleans entire heap + Metaspace (slowest, "stop-the-world")

### Common GC tuning flags

```bash
-Xms512m                # initial heap size
-Xmx4g                  # max heap size
-XX:+UseG1GC            # use G1 collector
-XX:MaxGCPauseMillis=200 # target max pause
-XX:+PrintGCDetails     # log GC events
-Xlog:gc*               # Java 9+ unified logging
```

### Practical: force a GC (not guaranteed!)

```java
System.gc();      // "please run GC"; JVM may ignore
```

**Never rely on `System.gc()` in production code** — it's a hint, not a command.

### finalize() — deprecated, don't use

Use `try-with-resources` or `java.lang.ref.Cleaner` instead.

```java
try (BufferedReader br = new BufferedReader(new FileReader("x.txt"))) {
    // auto-closed even on exception
}
```

### Common interview questions

1. *Can we force garbage collection?* → No, only request it.
2. *Difference between Minor GC and Major GC?*
3. *What is memory leak in Java?* → Objects still referenced but no longer needed (e.g., static Collections holding refs, listeners not deregistered).
4. *Why did they replace PermGen with Metaspace?* → PermGen had fixed size → `OutOfMemoryError: PermGen space`; Metaspace grows in native memory.

---

## 1.4 Access Modifiers — Who Can See Your Code?

### Bank vault analogy

- `private` → only the account holder
- default (no keyword) → only people in the same branch (package)
- `protected` → same branch + family members (subclasses) in any city
- `public` → everyone on the planet

| Modifier | Same Class | Same Package | Subclass (diff pkg) | World |
|----------|:---------:|:------------:|:-------------------:|:-----:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| default | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

### Practical example

```java
package bank;

public class Account {
    private double balance;              // only Account can touch
    String accountNumber;                 // package-private (default)
    protected String branch;              // subclasses across packages
    public String bankName = "SBI";       // anyone
}
```

**Rule of thumb:** Start with `private` and open up only as needed (Principle of Least Privilege).

---

## 1.5 Data Types & Variables

### Primitives — stored directly on stack (fast, small)

| Type | Size | Range | Default |
|------|------|-------|---------|
| `byte` | 1 byte | -128 to 127 | 0 |
| `short` | 2 bytes | -32,768 to 32,767 | 0 |
| `int` | 4 bytes | ~ -2.1B to 2.1B | 0 |
| `long` | 8 bytes | huge | 0L |
| `float` | 4 bytes | ~7 digit precision | 0.0f |
| `double` | 8 bytes | ~15 digit precision | 0.0d |
| `char` | 2 bytes | Unicode 0..65535 | '\u0000' |
| `boolean` | 1 bit* | true/false | false |

### Wrapper classes — objects that "box" primitives

```java
int    prim  = 5;
Integer obj  = 5;              // autoboxing: prim → wrapper
int    back  = obj;             // unboxing: wrapper → prim

// Wrappers can be null (primitives can't)
Integer nullable = null;
int broken = nullable;          // 💥 NullPointerException!
```

**Why wrappers exist:**
- Collections need objects: `List<Integer>` (can't use primitives)
- Utility methods: `Integer.parseInt("42")`, `Integer.MAX_VALUE`

### Integer cache trap

```java
Integer a = 100, b = 100;
Integer c = 200, d = 200;
System.out.println(a == b);      // true  (Integer cache -128..127)
System.out.println(c == d);      // false (new Integer objects)
System.out.println(c.equals(d)); // true  (always use equals!)
```

---

## 1.6 Pass By Value — The Truth

Java is **always pass-by-value**, but the confusion comes from *what value* is passed:
- For primitives → the *actual value*
- For objects → the *reference value* (address), NOT the object itself

### Example 1: Primitives

```java
public static void change(int x) {
    x = 100;
}
int a = 5;
change(a);
System.out.println(a);   // 5 — unchanged
```

### Example 2: Objects (reference value)

```java
public static void modify(List<Integer> list) {
    list.add(99);                   // uses the SAME object → visible outside
    list = new ArrayList<>();       // rebinds the local copy → NOT visible outside
    list.add(-1);
}

List<Integer> nums = new ArrayList<>(List.of(1, 2, 3));
modify(nums);
System.out.println(nums);           // [1, 2, 3, 99]  ← 99 was added; new list ignored
```

### Mental model

```
Before call:                  After call:
nums ──┐                      nums ──┐
       ▼                             ▼
   [1,2,3]                       [1,2,3,99]
                                       ▲
                              param "list" is a COPY of the reference
                              pointing to the SAME object
```

---

## 1.7 OOP — The Four Pillars (Deep Dive)

OOP is about modelling code the way we think in the real world.

### 1) Encapsulation — hide the wiring

Wrap data and methods together; expose only what's needed.

**Bad (no encapsulation):**
```java
class Account {
    public double balance;
}

Account a = new Account();
a.balance = -1_000_000;   // any code can corrupt it
```

**Good (encapsulated):**
```java
public class Account {
    private double balance;

    public double getBalance() { return balance; }

    public void deposit(double amt) {
        if (amt <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amt;
    }

    public void withdraw(double amt) {
        if (amt > balance) throw new IllegalStateException("Insufficient funds");
        balance -= amt;
    }
}
```

**Benefits:** validation, invariants, easier to change internals later.

### 2) Inheritance — reuse via IS-A

`Dog` IS-A `Animal` → `Dog` inherits `Animal`'s state & behavior.

```java
class Animal {
    protected String name;
    public void eat() { System.out.println(name + " is eating"); }
}

class Dog extends Animal {
    public void bark() { System.out.println(name + " says Woof!"); }
}

Dog d = new Dog();
d.name = "Rex";
d.eat();     // inherited
d.bark();    // own
```

**Java supports single class inheritance** (one parent) but **multiple interface inheritance** (many interfaces).

### 3) Polymorphism — same method, different behavior

Two flavors:

**a) Compile-time (Overloading)** — same method name, different parameters
```java
class Calc {
    int add(int a, int b)         { return a + b; }
    double add(double a, double b){ return a + b; }
    int add(int a, int b, int c)  { return a + b + c; }
}
```

**b) Runtime (Overriding)** — subclass provides its own version
```java
class Animal { void sound() { System.out.println("Some sound"); } }
class Dog extends Animal { void sound() { System.out.println("Woof"); } }
class Cat extends Animal { void sound() { System.out.println("Meow"); } }

Animal a1 = new Dog();
Animal a2 = new Cat();
a1.sound();    // Woof   ← chosen at RUNTIME based on actual object
a2.sound();    // Meow
```

**Interview:** JVM uses a **virtual method table (vtable)** to pick the right method at runtime — this is *dynamic dispatch*.

### 4) Abstraction — expose *what*, hide *how*

```java
// You care that a car drives; not how the engine works.
abstract class Vehicle {
    abstract void start();                 // no body — subclass must define
    void stop() { System.out.println("Stop"); }  // concrete method allowed
}

class Car extends Vehicle {
    void start() { System.out.println("Turn key"); }
}

class ElectricCar extends Vehicle {
    void start() { System.out.println("Press button"); }
}
```

Two abstraction tools:
| Feature | `abstract class` | `interface` |
|---------|------------------|-------------|
| State (fields) | Yes | Constants only (`public static final`) |
| Constructors | Yes | No |
| Multiple inheritance | No | Yes |
| Concrete methods | Yes | `default` and `static` (Java 8+) |
| Use when | Some shared code + template | Pure contract / capability |

### Composition vs Inheritance — the "prefer composition" rule

```java
// Inheritance (tight coupling)
class Car extends Engine { }        // Car IS-A Engine? No — awkward

// Composition (looser, flexible)
class Car {
    private Engine engine;           // Car HAS-A Engine
    Car(Engine e) { this.engine = e; }
}
```

**Rule:** Use inheritance only when there's a genuine IS-A relationship. Prefer composition for HAS-A.

---

## 1.8 `this`, `super`, `static`, `final`

### `this`

Refers to the current object.

```java
class Person {
    String name;
    Person(String name) {
        this.name = name;             // disambiguates field vs parameter
    }
    Person() {
        this("Unknown");              // call other constructor
    }
}
```

### `super`

Refers to the parent.

```java
class Animal {
    Animal() { System.out.println("Animal ctor"); }
    void greet() { System.out.println("Hi from Animal"); }
}
class Dog extends Animal {
    Dog() {
        super();                       // implicit if omitted
        System.out.println("Dog ctor");
    }
    void greet() {
        super.greet();                 // call parent version
        System.out.println("Woof!");
    }
}
```

### `static` — belongs to the class, not the instance

```java
class Counter {
    static int count = 0;              // shared by all instances
    int id;
    Counter() { id = ++count; }
    static void reset() { count = 0; } // no 'this'
}

Counter.count;    // access via class name
Counter.reset();
```

**Static blocks** run once when class is loaded:
```java
class Config {
    static Map<String,String> props;
    static {
        props = new HashMap<>();
        props.put("env", "dev");
    }
}
```

### `final` — the "no change" keyword

- `final variable` → constant (must be initialized)
- `final method` → cannot be overridden
- `final class` → cannot be extended (e.g., `String`, `Integer`)
- `final parameter` → cannot reassign inside method

```java
final int MAX = 100;
MAX = 200;                             // ❌ compile error

final class Utils { }
class MyUtils extends Utils { }        // ❌ compile error
```

### `final` vs `finally` vs `finalize`

| | What it is |
|-|--|
| **final** | keyword — for constants / no-inherit / no-override |
| **finally** | block in try-catch that always runs |
| **finalize()** | method called by GC before destroying an object (deprecated) |

---

## 1.9 Immutable Classes — Write Once, Trust Forever

A class whose objects **cannot be changed** after creation. String is the most famous example.

### Why immutable?
- Thread-safe by default (no locks needed)
- Safe as HashMap keys (hashCode never changes)
- Easy to reason about (no surprises)
- Great for caching / interning

### 5 rules

1. Mark class `final` (no subclassing)
2. All fields `private final`
3. No setters
4. Deep-copy mutable inputs in constructor
5. Return defensive copies from getters

### Full example

```java
public final class Employee {
    private final String name;
    private final Date joinDate;         // mutable type!
    private final List<String> skills;    // mutable type!

    public Employee(String name, Date joinDate, List<String> skills) {
        this.name = name;
        this.joinDate = new Date(joinDate.getTime());        // deep copy in
        this.skills = new ArrayList<>(skills);
    }

    public String getName() { return name; }

    public Date getJoinDate() { return new Date(joinDate.getTime()); } // copy out

    public List<String> getSkills() {
        return Collections.unmodifiableList(skills);          // wrap so caller can't mutate
    }
}
```

Try:
```java
List<String> skills = new ArrayList<>(List.of("Java"));
Employee e = new Employee("A", new Date(), skills);
skills.add("Python");                    // ❌ doesn't affect e (defensive copy)
e.getSkills().add("Go");                 // ❌ throws UnsupportedOperationException
```

---

## 1.10 Strings Deep Dive

### Why String is special

- Very commonly used → performance matters
- Cached in **String Pool** (part of heap) to save memory
- **Immutable** → safe for hashing, caching, security (e.g., DB URLs, passwords)

### String Pool visualized

```java
String a = "java";                       // pool
String b = "java";                       // reuses same pooled instance
String c = new String("java");           // NEW heap object (not pooled)
String d = c.intern();                   // pushes to pool → same as 'a'

a == b       // true  (same pool ref)
a == c       // false (different heap objects)
a.equals(c)  // true  (content same)
a == d       // true  (intern gives pooled ref)
```

Diagram:
```
Heap:
  ┌──────────┐              ┌────────────────┐
  │ "java"   │◄──── a       │  new String    │◄── c
  │ (pool)   │◄──── b       │  content:"java"│
  └──────────┘              └────────────────┘
       ▲
       └────── d (after c.intern())
```

### String vs StringBuilder vs StringBuffer

| | Mutable? | Thread-safe? | Speed |
|-|----------|--------------|-------|
| **String** | ❌ | ✅ (immutable) | Slow for concat (creates new each time) |
| **StringBuilder** | ✅ | ❌ | Fastest |
| **StringBuffer** | ✅ | ✅ (synchronized) | Slower than SB |

**Performance demo:**
```java
long t1 = System.currentTimeMillis();
String s = "";
for (int i = 0; i < 100_000; i++) s += "a";            // creates 100k Strings!
System.out.println("String: " + (System.currentTimeMillis() - t1) + " ms");

long t2 = System.currentTimeMillis();
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100_000; i++) sb.append("a");
System.out.println("StringBuilder: " + (System.currentTimeMillis() - t2) + " ms");
```
You'll see StringBuilder is 100× faster.

### `==` vs `equals()` cheat sheet

```java
String a = "x";
String b = "x";
String c = new String("x");

a == b           // true  (both from pool)
a == c           // false (different heap objects)
a.equals(c)      // true  (compare content)
```

**Rule:** Always use `equals()` for content comparison.

### `hashCode()` contract

If `a.equals(b)` is true → `a.hashCode() == b.hashCode()` **must** be true.  
Reverse is not required.

Violating this **breaks HashMap / HashSet**.

```java
class Key {
    int id;
    String name;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Key)) return false;
        Key other = (Key) o;
        return id == other.id && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}
```

---

## 1.11 Exception Handling Mastery

### Hierarchy

```
Throwable
├── Error                       (never catch! JVM problems)
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── RuntimeException        (UNCHECKED — programmer errors)
    │   ├── NullPointerException
    │   ├── ArithmeticException
    │   ├── ArrayIndexOutOfBoundsException
    │   └── ClassCastException
    └── (all other Exception)   (CHECKED — must handle or declare)
        ├── IOException
        ├── SQLException
        └── ClassNotFoundException
```

### Checked vs Unchecked

- **Checked** — compiler forces you to handle: `IOException`, `SQLException`. Represents *expected* problems (file missing, DB down).
- **Unchecked** — bugs in your code: `NullPointerException`, `IllegalArgumentException`. Fix the code, not the caller.

### try-catch-finally

```java
try {
    riskyOperation();
} catch (SpecificException e) {
    // handle specific case
} catch (Exception e) {
    // catch-all fallback
    log.error("Unexpected", e);
} finally {
    // always runs — cleanup (close files, release locks)
}
```

**Fun fact:** `finally` runs **even if you `return` inside try**!

```java
public int test() {
    try { return 1; }
    finally { System.out.println("bye"); }  // prints "bye", then returns 1
}
```

### try-with-resources (Java 7+) — auto-close

Any class implementing `AutoCloseable` works:

```java
try (BufferedReader br = new BufferedReader(new FileReader("x.txt"))) {
    return br.readLine();
}
// br.close() called automatically, even if an exception was thrown
```

Multiple resources — closed in reverse order:
```java
try (Connection c = DriverManager.getConnection(url);
     PreparedStatement ps = c.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    // work
}
```

### Custom exceptions

```java
public class InsufficientBalanceException extends RuntimeException {
    private final double balance;
    private final double requested;

    public InsufficientBalanceException(double balance, double requested) {
        super("Balance " + balance + " < requested " + requested);
        this.balance = balance;
        this.requested = requested;
    }

    public double getBalance()   { return balance; }
    public double getRequested() { return requested; }
}
```

Use custom exceptions to carry business-relevant info the caller might want.

### Exception propagation

If not caught, exceptions bubble up the call stack:

```
main() → level1() → level2() → level3() throws
                                    │
   ┌────────────────────────────────┘
   ▼
level2() (no catch → propagates)
   ▼
level1() (no catch → propagates)
   ▼
main() (no catch → JVM prints stack trace & exits)
```

### Best practices

1. **Catch specific first, generic last.**
2. **Never swallow** exceptions silently — at least log.
3. **Don't catch `Throwable`** or `Error`.
4. **Use `finally` or try-with-resources** for cleanup.
5. **Wrap and rethrow** with context when crossing layers:
   ```java
   catch (SQLException e) {
       throw new BookRepositoryException("Failed to load book " + id, e);
   }
   ```

---

## 1.12 Java 8+ Features — With Real Examples

### Lambda Expressions

Anonymous functions — replace verbose anonymous inner classes.

**Before Java 8:**
```java
Runnable r = new Runnable() {
    @Override public void run() {
        System.out.println("Hello");
    }
};
```

**With lambda:**
```java
Runnable r = () -> System.out.println("Hello");
```

Sorting:
```java
List<String> names = new ArrayList<>(List.of("Bob", "Alice", "Charlie"));
names.sort((a, b) -> a.length() - b.length());
// [Bob, Alice, Charlie]
```

### Functional Interfaces (in `java.util.function`)

An interface with exactly **one abstract method**. Lambdas target these.

| Interface | Abstract Method | Purpose | Example |
|-----------|-----------------|---------|---------|
| `Function<T,R>` | `R apply(T)` | transform | `s -> s.length()` |
| `Predicate<T>` | `boolean test(T)` | test/filter | `n -> n > 0` |
| `Consumer<T>` | `void accept(T)` | side effect | `System.out::println` |
| `Supplier<T>` | `T get()` | create | `() -> new Book()` |
| `BiFunction<T,U,R>` | `R apply(T,U)` | 2 → 1 | `(a,b) -> a+b` |
| `UnaryOperator<T>` | `T apply(T)` | same-type transform | `x -> x*2` |
| `BinaryOperator<T>` | `T apply(T,T)` | reduce | `Integer::sum` |

Try:
```java
Function<String, Integer> length = String::length;
Predicate<Integer> positive = n -> n > 0;
Consumer<String> print = System.out::println;
Supplier<List<Integer>> list = ArrayList::new;

System.out.println(length.apply("hello"));    // 5
System.out.println(positive.test(-2));        // false
print.accept("hi");                            // hi
System.out.println(list.get());               // []
```

Custom:
```java
@FunctionalInterface
interface StringTransformer {
    String transform(String input);
}

StringTransformer upper = s -> s.toUpperCase();
StringTransformer reverse = s -> new StringBuilder(s).reverse().toString();
System.out.println(upper.transform("hi"));      // HI
System.out.println(reverse.transform("java"));  // avaj
```

### Streams API — The Game Changer

A **stream** is a pipeline of operations on data. Not a data structure; it doesn't store anything.

**Anatomy:**
```
Source (collection/array/generator)
   │
   ▼
Intermediate ops (lazy — filter, map, sorted, distinct, limit)
   │
   ▼
Terminal op (eager — collect, forEach, reduce, count)
```

**Example — top 3 highest-paid engineers, uppercased name:**

```java
record Employee(String name, String dept, double salary) {}

List<Employee> emps = List.of(
    new Employee("Alice", "Eng", 90_000),
    new Employee("Bob",   "Eng", 120_000),
    new Employee("Carol", "HR",  70_000),
    new Employee("Dave",  "Eng", 110_000),
    new Employee("Eve",   "Eng", 100_000)
);

List<String> top3 = emps.stream()
    .filter(e -> "Eng".equals(e.dept()))
    .sorted(Comparator.comparingDouble(Employee::salary).reversed())
    .limit(3)
    .map(e -> e.name().toUpperCase())
    .collect(Collectors.toList());

System.out.println(top3);   // [BOB, DAVE, EVE]
```

**Grouping / counting:**
```java
Map<String, Long> byDept = emps.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
// {Eng=4, HR=1}
```

**Partitioning (Predicate → 2 groups):**
```java
Map<Boolean, List<Employee>> highLow = emps.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() > 100_000));
```

**Reducing:**
```java
double totalSalary = emps.stream()
    .mapToDouble(Employee::salary)
    .sum();

Optional<Employee> highest = emps.stream()
    .max(Comparator.comparingDouble(Employee::salary));
```

**`map` vs `flatMap`:**
- `map` — one-to-one transform
- `flatMap` — one-to-many (flatten nested streams)

```java
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4));
List<Integer> flat = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());   // [1,2,3,4]
```

**Parallel streams — use with care!**
```java
long count = list.parallelStream().filter(x -> x > 0).count();
```
Uses `ForkJoinPool.commonPool()`. Watch out for:
- Shared mutable state
- Ordering (use `.forEachOrdered`)
- Non-CPU-heavy tasks may be slower (overhead)

### Method References — cleaner lambdas

| Kind | Syntax | Example |
|------|--------|---------|
| Static | `Class::staticMethod` | `Integer::parseInt` |
| Bound instance | `object::method` | `System.out::println` |
| Unbound instance | `Class::method` | `String::toUpperCase` |
| Constructor | `Class::new` | `ArrayList::new` |

### Optional — Say goodbye to NullPointerException

Wraps a value that *might* be absent.

```java
Optional<Book> book = repository.findById(1L);

// ❌ bad — defeats the purpose
if (book.isPresent()) System.out.println(book.get().getTitle());

// ✅ functional style
book.ifPresent(b -> System.out.println(b.getTitle()));

// default
Book b = book.orElse(new Book("Default"));

// throw if absent
Book b2 = book.orElseThrow(() -> new BookNotFoundException(1L));

// map / chain
String title = book.map(Book::getTitle).map(String::toUpperCase).orElse("N/A");
```

**When NOT to use Optional:**
- As a field (Optional isn't serializable, and there's overhead)
- As a method parameter
- With collections — return empty list, not `Optional<List>`

### Default & Static Methods in Interfaces

Before Java 8, adding a method to an interface broke every implementation. Java 8 solves this:

```java
interface Vehicle {
    void start();                                    // abstract

    default void warmUp() {                          // has default body
        System.out.println("Warming up...");
    }

    static Vehicle defaultVehicle() {                // static utility
        return () -> System.out.println("Started");
    }
}
```

### java.time — Modern Date/Time API

The old `Date`, `Calendar` are mutable, thread-unsafe, and confusing. Java 8's `java.time` fixes all that.

```java
LocalDate today = LocalDate.now();
LocalTime now = LocalTime.now();
LocalDateTime dt = LocalDateTime.now();
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));

LocalDate birthday = LocalDate.of(1995, Month.JUNE, 15);
long age = ChronoUnit.YEARS.between(birthday, today);

LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastMonth = today.minusMonths(1);
boolean isBefore = today.isBefore(nextWeek);

Duration d = Duration.between(LocalTime.of(9,0), LocalTime.of(17,30));
System.out.println(d.toHours());        // 8

DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");
System.out.println(today.format(fmt));  // "07 Oct 2025"
LocalDate parsed = LocalDate.parse("2025-01-15");
```

All immutable, thread-safe. Always use `java.time` in new code.

---

## 1.13 Hands-on Mini Project — Immutable BankAccount

Combines: encapsulation, immutability, exceptions, Optional, Streams.

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

public final class BankAccount {
    private final String accountNumber;
    private final String holder;
    private final List<Transaction> transactions;

    public BankAccount(String accountNumber, String holder) {
        if (accountNumber == null || accountNumber.isBlank())
            throw new IllegalArgumentException("Account number required");
        this.accountNumber = accountNumber;
        this.holder = holder;
        this.transactions = new ArrayList<>();
    }

    public double getBalance() {
        return transactions.stream()
            .mapToDouble(Transaction::amount)
            .sum();
    }

    public void deposit(double amt) {
        if (amt <= 0) throw new IllegalArgumentException("Deposit must be positive");
        transactions.add(new Transaction(amt, "DEPOSIT", LocalDateTime.now()));
    }

    public void withdraw(double amt) {
        if (amt <= 0) throw new IllegalArgumentException("Withdraw must be positive");
        if (amt > getBalance()) throw new InsufficientFundsException(getBalance(), amt);
        transactions.add(new Transaction(-amt, "WITHDRAW", LocalDateTime.now()));
    }

    public List<Transaction> getTransactions() {
        return Collections.unmodifiableList(transactions);
    }

    public Optional<Transaction> lastTransaction() {
        return transactions.isEmpty()
            ? Optional.empty()
            : Optional.of(transactions.get(transactions.size() - 1));
    }

    public double totalDeposits() {
        return transactions.stream()
            .filter(t -> "DEPOSIT".equals(t.type()))
            .mapToDouble(Transaction::amount)
            .sum();
    }

    public record Transaction(double amount, String type, LocalDateTime at) {}

    public static class InsufficientFundsException extends RuntimeException {
        public InsufficientFundsException(double balance, double requested) {
            super("Balance " + balance + " < requested " + requested);
        }
    }

    public static void main(String[] args) {
        BankAccount acc = new BankAccount("SBI-001", "Alice");
        acc.deposit(1000);
        acc.deposit(500);
        acc.withdraw(300);
        System.out.println("Balance: " + acc.getBalance());         // 1200
        System.out.println("Deposits total: " + acc.totalDeposits());// 1500
        acc.lastTransaction().ifPresent(System.out::println);
    }
}
```

---

## 1.14 Interview Question Bank (with short answers)

1. **Why is String immutable?**  
   → Thread safety, caching in String pool, safe as HashMap key (hashcode stable), security (passwords in String can't be modified).

2. **Difference between abstract class and interface?**  
   → Abstract class: partial implementation + state + single inheritance. Interface: contract + multiple inheritance + default/static methods (Java 8+).

3. **`==` vs `equals()`?**  
   → `==` compares references; `equals()` compares content (when overridden).

4. **Why override `hashCode()` when overriding `equals()`?**  
   → The contract: equal objects must have equal hash codes. Violating breaks HashMap/HashSet.

5. **Checked vs unchecked exceptions?**  
   → Checked: caller must handle (compile-time enforced). Unchecked: programmer errors, not enforced.

6. **What are functional interfaces?**  
   → Interface with exactly one abstract method. Used as target for lambdas.

7. **`map` vs `flatMap`?**  
   → `map`: 1-to-1 transform. `flatMap`: 1-to-many, then flattens.

8. **What's new in Java 8?**  
   → Lambdas, streams, Optional, default methods, java.time, method references.

9. **Explain garbage collection.**  
   → Automatic reclamation of unreachable objects. Generational (young → old). Algorithms: Serial, Parallel, G1, ZGC.

10. **`ArrayList` vs `LinkedList`?**  
    → ArrayList: dynamic array, fast random access, slower insert/delete. LinkedList: doubly linked, fast insert/delete at ends, slower random access.

11. **How does JVM achieve platform independence?**  
    → JVM abstracts the OS; bytecode is compiled once and runs on any JVM.

12. **What is autoboxing?**  
    → Automatic conversion between primitives and their wrapper classes (`int` ↔ `Integer`).

13. **Can you override a static method?**  
    → No, you can only *hide* it (redeclare in subclass). Dispatch is based on the reference type, not object type.

14. **What is the difference between `throw` and `throws`?**  
    → `throw`: actually throws an exception. `throws`: declares that a method might throw one.

15. **Explain the try-with-resources.**  
    → Auto-closes resources implementing `AutoCloseable`. No explicit `finally` block needed. Suppressed exceptions are attached to the primary.

---

## 1.15 Quick Cheat Sheet

```
JDK ⊃ JRE ⊃ JVM
Heap = objects; Stack = local vars/frames per thread
Java is pass-by-value (reference value for objects)
final class      → cannot extend
final method     → cannot override
final variable   → cannot reassign
== reference; equals() content
Streams are lazy; collect() triggers execution
Optional to avoid nulls; never store in fields
java.time for dates; forget Date/Calendar
```

---

## Practical Assignment (do this before your interview!)

1. Convert a legacy `Date`-using class to `java.time`.
2. Rewrite an imperative loop that filters + maps + sums into a Stream one-liner.
3. Design an immutable `Address` class following the 5 rules.
4. Write a custom checked exception for a domain (e.g., `OrderNotFoundException`) and a global handler.
5. Use `Optional` correctly in a `UserRepository.findByEmail` method.

Master these and you're 60% done with any Java interview! 🚀