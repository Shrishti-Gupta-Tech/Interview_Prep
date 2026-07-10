# 11. Design Patterns — Deep Dive

> **Goal:** Understand *why* each pattern exists (the problem it solves), see real-world Java examples, and know when NOT to use them.

---

## 11.0 What Are Design Patterns?

Design patterns are **reusable solutions to common problems** in software design. They're not code you copy-paste; they're **templates** you adapt.

### Why learn them?
- **Shared vocabulary** — say "Singleton" and every dev knows what you mean
- **Proven solutions** — vetted by decades of use
- **Better designs** — force you to think about extensibility
- **Interviews** love them

### The three families (GoF — "Gang of Four" 1994)

| Family | Purpose | Examples |
|--------|---------|----------|
| **Creational** | How objects are created | Singleton, Factory, Builder, Prototype, Abstract Factory |
| **Structural** | How objects/classes are composed | Adapter, Decorator, Facade, Proxy, Composite, Bridge |
| **Behavioral** | How objects communicate | Strategy, Observer, Command, Template Method, Iterator, State, Chain of Responsibility |

### Warning

**Don't over-use patterns!** Simple code is better than patterned code when the pattern isn't needed. Only apply a pattern when it clearly solves your problem.

---

## 11.1 Singleton — One Instance Only

### Problem
You need exactly one instance of a class (e.g., logger, connection pool, config), globally accessible.

### Real-world analogy
The President of a country. There's only one. Everyone refers to "the President".

### Naive (broken) implementation
```java
public class Singleton {
    private static Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) instance = new Singleton();      // ❌ race condition
        return instance;
    }
}
```

Two threads can both see `instance == null` → create two instances!

### Double-Checked Locking (DCL) — thread-safe, lazy

```java
public class Singleton {
    private static volatile Singleton instance;     // volatile is critical!

    private Singleton() { }                          // private ctor prevents new

    public static Singleton getInstance() {
        if (instance == null) {                      // fast path (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {              // double check
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

Why `volatile`? Prevents JVM instruction reordering that could publish a partially-initialized object.

### Bill Pugh — cleaner, lazy, thread-safe, no sync

```java
public class Singleton {
    private Singleton() { }

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

Uses class-loading semantics — Holder is loaded (and INSTANCE created) only when getInstance() is called.

### Enum — Joshua Bloch's recommendation (safest)

```java
public enum Singleton {
    INSTANCE;

    public void doSomething() { }
}

// Usage
Singleton.INSTANCE.doSomething();
```

Advantages:
- Thread-safe by JVM guarantee
- Serialization-safe (no risk of extra instances)
- Reflection-safe (`newInstance()` throws)

### Real-world Java examples
- `Runtime.getRuntime()`
- `System` class
- Spring beans (singleton scope by default)
- Logger factories

### When NOT to use
- Testing — hard to mock singletons
- Global state is often a smell
- In Spring, use `@Component` with default scope instead

---

## 11.2 Factory Method — Delegate Creation

### Problem
Client code shouldn't know which concrete class to instantiate. Let a factory decide.

### Real-world analogy
You order a "pizza". The pizza restaurant decides internally whether to make a Margherita, Pepperoni, or Veggie based on your choice — you don't hand-craft it yourself.

### Simple factory

```java
public interface Shape {
    void draw();
}

public class Circle implements Shape { public void draw() { } }
public class Square implements Shape { public void draw() { } }
public class Triangle implements Shape { public void draw() { } }

public class ShapeFactory {
    public static Shape create(String type) {
        return switch (type.toLowerCase()) {
            case "circle"   -> new Circle();
            case "square"   -> new Square();
            case "triangle" -> new Triangle();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}

// Usage
Shape s = ShapeFactory.create("circle");
s.draw();
```

### Real Java examples
- `Integer.valueOf(int)` — may return cached values
- `Calendar.getInstance()` — locale-specific Calendar subclass
- `LocalDate.of(2025, 1, 15)`
- Spring `BeanFactory.getBean(name)`
- `Executors.newFixedThreadPool(4)` — returns an ExecutorService

---

## 11.3 Abstract Factory — Factory of Factories

### Problem
Create **families** of related objects that must be used together.

### Example — cross-platform UI

```java
interface Button { void click(); }
interface Checkbox { void check(); }

class WindowsButton implements Button { public void click() { } }
class WindowsCheckbox implements Checkbox { public void check() { } }

class MacButton implements Button { public void click() { } }
class MacCheckbox implements Checkbox { public void check() { } }

// Abstract factory
interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

class WindowsFactory implements GUIFactory {
    public Button createButton()     { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

class MacFactory implements GUIFactory {
    public Button createButton()     { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client
class App {
    private final GUIFactory factory;
    public App(GUIFactory factory) { this.factory = factory; }

    public void render() {
        Button b = factory.createButton();
        Checkbox c = factory.createCheckbox();
        b.click(); c.check();
    }
}

new App(new WindowsFactory()).render();
```

Use when you have multiple product families and want to switch between them.

---

## 11.4 Builder — Construct Complex Objects Step-by-Step

### Problem
An object has many optional parameters. Telescoping constructors get ugly:

```java
new Pizza(30, true, false, true, false, null, false);
// What does each argument mean?!
```

### Solution — fluent builder

```java
public class Pizza {
    private final int size;
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushrooms;
    private final boolean bacon;
    private final String sauce;

    private Pizza(Builder b) {
        this.size = b.size;
        this.cheese = b.cheese;
        this.pepperoni = b.pepperoni;
        this.mushrooms = b.mushrooms;
        this.bacon = b.bacon;
        this.sauce = b.sauce;
    }

    public static class Builder {
        // required
        private final int size;
        // optional (with defaults)
        private boolean cheese = false;
        private boolean pepperoni = false;
        private boolean mushrooms = false;
        private boolean bacon = false;
        private String sauce = "tomato";

        public Builder(int size) { this.size = size; }
        public Builder cheese()     { this.cheese = true; return this; }
        public Builder pepperoni()  { this.pepperoni = true; return this; }
        public Builder mushrooms()  { this.mushrooms = true; return this; }
        public Builder bacon()      { this.bacon = true; return this; }
        public Builder sauce(String s) { this.sauce = s; return this; }
        public Pizza build()        { return new Pizza(this); }
    }
}

// Usage — readable!
Pizza p = new Pizza.Builder(30)
    .cheese()
    .pepperoni()
    .sauce("bbq")
    .build();
```

### Lombok shortcut

```java
@Builder
public class Pizza {
    private int size;
    private boolean cheese;
    private boolean pepperoni;
    // ...
}

Pizza p = Pizza.builder().size(30).cheese(true).build();
```

### Real Java examples
- `StringBuilder` / `StringBuffer` (mutable builder)
- `Stream.Builder`
- `HttpRequest.newBuilder()...build()`
- Records + Lombok `@Builder`
- `ImmutableList.<String>builder().add("a").build()` (Guava)

---

## 11.5 Prototype — Clone Instead of Constructing

### Problem
Creating an object is expensive (heavy DB query, complex init). Cloning an existing one is cheaper.

```java
public class Document implements Cloneable {
    private String content;
    private List<String> pages;

    public Document(String content) { /* heavy init */ }

    @Override
    public Document clone() {
        try {
            Document copy = (Document) super.clone();
            copy.pages = new ArrayList<>(this.pages);      // deep copy mutable fields!
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}

Document original = new Document("heavy setup");
Document copy = original.clone();
```

### Warning — Cloneable is broken

Java's Cloneable is famously bad. Alternatives:
- **Copy constructor:** `new Document(other)`
- **Static factory:** `Document.from(other)`
- Libraries like MapStruct, Jackson serialization

---

## 11.6 Adapter — Bridge Incompatible Interfaces

### Problem
You have code expecting interface A, but the class you want to use has interface B.

### Real-world analogy
Your laptop has a US plug; the wall in Europe has a Type-C socket. An **adapter** bridges them.

### Example

```java
// Target — what our code expects
public interface MediaPlayer {
    void play(String file);
}

// Adaptee — third-party class we can't change
public class VLCPlayer {
    public void playVlcFile(String filename) { System.out.println("Playing " + filename); }
}

// Adapter
public class VLCAdapter implements MediaPlayer {
    private final VLCPlayer vlc = new VLCPlayer();

    @Override
    public void play(String file) {
        vlc.playVlcFile(file);
    }
}

// Client
MediaPlayer player = new VLCAdapter();
player.play("song.vlc");
```

### Real Java examples
- `Arrays.asList(array)` — adapts array to List
- `InputStreamReader` — adapts InputStream to Reader
- JDBC drivers — adapt DB-specific APIs to the JDBC standard

---

## 11.7 Decorator — Add Behavior Dynamically

### Problem
You want to add responsibilities to objects without changing their class.

### Real-world analogy
Buying a car and adding options: leather seats, sunroof, spoiler. Each is a "decorator" wrapping the base car.

### Example — coffee shop

```java
public interface Coffee {
    double cost();
    String description();
}

// Base
public class SimpleCoffee implements Coffee {
    public double cost() { return 50; }
    public String description() { return "Coffee"; }
}

// Base decorator
public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee coffee;
    public CoffeeDecorator(Coffee c) { this.coffee = c; }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee c) { super(c); }
    public double cost() { return coffee.cost() + 20; }
    public String description() { return coffee.description() + " + Milk"; }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee c) { super(c); }
    public double cost() { return coffee.cost() + 5; }
    public String description() { return coffee.description() + " + Sugar"; }
}

public class CaramelDecorator extends CoffeeDecorator {
    public CaramelDecorator(Coffee c) { super(c); }
    public double cost() { return coffee.cost() + 15; }
    public String description() { return coffee.description() + " + Caramel"; }
}

// Usage
Coffee c = new CaramelDecorator(
              new SugarDecorator(
                  new MilkDecorator(
                      new SimpleCoffee())));
System.out.println(c.description() + " = " + c.cost());
// Coffee + Milk + Sugar + Caramel = 90
```

### Real Java examples
- **`java.io`** streams! `new BufferedReader(new FileReader("x"))` — BufferedReader decorates FileReader.
- `Collections.unmodifiableList(list)` — decorates with read-only behavior
- Servlet filters, Spring interceptors

---

## 11.8 Facade — Simplify a Complex Subsystem

### Problem
A complex subsystem with many classes is hard to use. Provide a single "front desk" interface.

### Real-world analogy
Hotel concierge. Instead of calling housekeeping, restaurant, laundry, valet separately, you tell the concierge and they coordinate.

### Example

```java
// Complex subsystem
class CPU     { void start() { }  }
class Memory  { void load()  { }  }
class HardDrive { void read() { } }

// Facade
public class ComputerFacade {
    private final CPU cpu = new CPU();
    private final Memory memory = new Memory();
    private final HardDrive hd = new HardDrive();

    public void start() {
        hd.read();
        memory.load();
        cpu.start();
        System.out.println("Computer started");
    }
}

// Client
new ComputerFacade().start();
```

### Real Java examples
- **Spring's `JdbcTemplate`** — wraps JDBC's boilerplate
- `RestTemplate` — HTTP abstraction
- SLF4J's `LoggerFactory` — hides logging framework choice

---

## 11.9 Proxy — Placeholder Controlling Access

### Problem
You want to control access to an object (lazy load, security, remote, cache).

### Types
- **Virtual** — lazy loading (create expensive object on demand)
- **Protection** — access control
- **Remote** — represents an object on another server (RPC/RMI)
- **Caching** — cache results

### Virtual proxy example

```java
public interface Image {
    void display();
}

public class RealImage implements Image {
    private final String filename;
    public RealImage(String f) { this.filename = f; loadFromDisk(); }
    private void loadFromDisk() { System.out.println("Loading " + filename); }
    public void display() { System.out.println("Displaying " + filename); }
}

public class ImageProxy implements Image {
    private final String filename;
    private RealImage real;

    public ImageProxy(String f) { this.filename = f; }

    public void display() {
        if (real == null) real = new RealImage(filename);   // load only when needed
        real.display();
    }
}

// Client
Image img = new ImageProxy("huge.png");
// nothing loaded yet
img.display();      // NOW it loads and displays
img.display();      // subsequent calls use loaded image
```

### Real Java examples
- **Hibernate lazy loading** — `Book.getAuthor()` returns a proxy until you actually use it
- **Spring AOP** — `@Transactional`, `@Cacheable` create proxies
- **Java Dynamic Proxy** (`java.lang.reflect.Proxy`)
- **RMI stubs**

---

## 11.10 Composite — Tree Structures

### Problem
Represent tree hierarchies where both **individual objects** and **groups** should be treated the same way.

### Real-world analogy
Filesystem: a folder can contain files AND other folders. Operations like "size" or "delete" work on both.

### Example

```java
public interface FileSystem {
    long size();
    void print(String indent);
}

public class File implements FileSystem {
    private final String name;
    private final long size;
    public File(String name, long size) { this.name = name; this.size = size; }
    public long size() { return size; }
    public void print(String indent) { System.out.println(indent + name + " (" + size + ")"); }
}

public class Folder implements FileSystem {
    private final String name;
    private final List<FileSystem> children = new ArrayList<>();
    public Folder(String name) { this.name = name; }
    public void add(FileSystem fs) { children.add(fs); }
    public long size() {
        return children.stream().mapToLong(FileSystem::size).sum();
    }
    public void print(String indent) {
        System.out.println(indent + name + "/");
        children.forEach(c -> c.print(indent + "  "));
    }
}

Folder root = new Folder("root");
root.add(new File("a.txt", 100));
Folder sub = new Folder("sub");
sub.add(new File("b.txt", 200));
root.add(sub);
root.print("");
System.out.println("Total size: " + root.size());
```

Real Java example: Swing `Component`/`Container` hierarchy.

---

## 11.11 Strategy — Interchangeable Algorithms

### Problem
You have multiple ways to do something (sort algorithms, payment methods, compression). Client picks at runtime.

### Real-world analogy
Choosing a route in Google Maps: fastest, shortest, no highways. Same input (start, end), different algorithm.

### Example — payment

```java
public interface PaymentStrategy {
    void pay(double amount);
}

public class CreditCardPayment implements PaymentStrategy {
    public void pay(double a) { System.out.println("Paid $" + a + " via credit card"); }
}

public class UPIPayment implements PaymentStrategy {
    public void pay(double a) { System.out.println("Paid $" + a + " via UPI"); }
}

public class CryptoPayment implements PaymentStrategy {
    public void pay(double a) { System.out.println("Paid $" + a + " in crypto"); }
}

// Context
public class Checkout {
    private PaymentStrategy strategy;
    public void setStrategy(PaymentStrategy s) { this.strategy = s; }
    public void pay(double amount) { strategy.pay(amount); }
}

Checkout c = new Checkout();
c.setStrategy(new UPIPayment());
c.pay(500);
c.setStrategy(new CryptoPayment());
c.pay(1000);
```

### With Java 8 lambdas — strategies are just functions

```java
Map<String, PaymentStrategy> strategies = Map.of(
    "cc",     a -> System.out.println("CC:  " + a),
    "upi",    a -> System.out.println("UPI: " + a),
    "crypto", a -> System.out.println("BTC: " + a)
);
strategies.get("upi").pay(500);
```

### Real Java examples
- `Comparator` (sorting strategy)
- `Runnable`, `Callable`
- `java.util.function.*` (Function, Predicate, etc.)
- Spring's `AuthenticationProvider`

---

## 11.12 Observer — Publish/Subscribe

### Problem
When one object changes state, many other objects need to react — without tight coupling.

### Real-world analogy
YouTube channel. Subscribers get notified whenever the channel publishes a new video.

### Example

```java
public interface Observer {
    void update(String event);
}

public class Subject {
    private final List<Observer> observers = new ArrayList<>();

    public void attach(Observer o) { observers.add(o); }
    public void detach(Observer o) { observers.remove(o); }

    public void notifyAll(String event) {
        for (Observer o : observers) o.update(event);
    }
}

public class EmailNotifier implements Observer {
    public void update(String e) { System.out.println("Email: " + e); }
}

public class SmsNotifier implements Observer {
    public void update(String e) { System.out.println("SMS: " + e); }
}

Subject youtube = new Subject();
youtube.attach(new EmailNotifier());
youtube.attach(new SmsNotifier());
youtube.notifyAll("New video: 'Design Patterns'");
```

### Real examples
- **Spring `ApplicationEvent` + `@EventListener`** — pub/sub in Spring
- **JavaFX property bindings**
- **RxJS Observables**
- **DOM event listeners** (`addEventListener`)
- **Kafka consumer groups**

---

## 11.13 Command — Encapsulate Requests as Objects

### Problem
You want to queue, log, undo, or parameterize operations.

### Real-world analogy
Restaurant order slip. Waiter writes "Chicken Burger, Large Coke". The slip is queued in the kitchen. Kitchen can execute later, log it, cancel it.

### Example

```java
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class Light {
    void on()  { System.out.println("Light ON"); }
    void off() { System.out.println("Light OFF"); }
}

// Concrete commands
public class LightOnCommand implements Command {
    private final Light light;
    public LightOnCommand(Light l) { this.light = l; }
    public void execute() { light.on(); }
    public void undo()    { light.off(); }
}

// Invoker
public class RemoteControl {
    private final Deque<Command> history = new ArrayDeque<>();
    public void press(Command c) { c.execute(); history.push(c); }
    public void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```

### Real Java examples
- **Runnable** — command for Threads/Executors
- Undo/redo in editors
- Task queues (`ExecutorService.submit(Runnable)`)
- CQRS pattern's Commands

---

## 11.14 Template Method — Skeleton with Customizable Steps

### Problem
Multiple algorithms share a structure but differ in a few steps.

### Real-world analogy
Cooking recipes: "Make hot beverage" = boil water → add ingredient → pour into cup → add condiments. Tea and coffee follow the same skeleton but differ in "add ingredient" and "condiments".

### Example

```java
public abstract class DataProcessor {

    public final void process() {          // template — final so subclasses can't change skeleton
        read();
        transform();
        write();
        cleanup();
    }

    protected abstract void read();
    protected abstract void transform();
    protected abstract void write();
    protected void cleanup() {              // hook — subclasses can override optionally
        System.out.println("Default cleanup");
    }
}

public class CsvProcessor extends DataProcessor {
    protected void read()      { System.out.println("Reading CSV"); }
    protected void transform() { System.out.println("Parsing rows"); }
    protected void write()     { System.out.println("Saving to DB"); }
}
```

### Real Java examples
- Spring's `JdbcTemplate.query(...)` — sets up connection, runs your callback, cleans up
- Servlet's `doGet`, `doPost`
- `HttpServlet.service(req, res)`
- `AbstractList` — skeleton implementation

---

## 11.15 Iterator — Sequential Access Without Exposing Internals

Built into Java as `Iterator`/`Iterable`.

```java
public class BookCollection implements Iterable<Book> {
    private final List<Book> books = new ArrayList<>();

    public Iterator<Book> iterator() {
        return books.iterator();       // delegates
    }
}

// Client can use for-each
for (Book b : collection) { ... }
```

Every Java Collection implements Iterable → Iterator.

---

## 11.16 State — Behavior Changes with State

### Problem
An object's behavior depends on its state, and it should change class-like behavior at runtime.

### Example — traffic light

```java
public interface TrafficLightState {
    void handle(TrafficLight ctx);
}

public class GreenState implements TrafficLightState {
    public void handle(TrafficLight ctx) {
        System.out.println("Green — go");
        ctx.setState(new YellowState());
    }
}

public class YellowState implements TrafficLightState {
    public void handle(TrafficLight ctx) {
        System.out.println("Yellow — slow down");
        ctx.setState(new RedState());
    }
}

public class RedState implements TrafficLightState {
    public void handle(TrafficLight ctx) {
        System.out.println("Red — stop");
        ctx.setState(new GreenState());
    }
}

public class TrafficLight {
    private TrafficLightState state = new GreenState();
    public void setState(TrafficLightState s) { this.state = s; }
    public void next() { state.handle(this); }
}

TrafficLight t = new TrafficLight();
for (int i = 0; i < 6; i++) t.next();
```

### Strategy vs State — subtle difference

| Strategy | State |
|----------|-------|
| Client chooses which algorithm | State transitions internally |
| Interchangeable at any time | Follows a state machine |
| e.g., sorting algorithm | e.g., order status flow |

---

## 11.17 Chain of Responsibility

### Problem
Multiple handlers might handle a request; you want to pass it along a chain until someone handles it.

### Real-world analogy
Escalating a support ticket: L1 → L2 → L3 → Manager. Each level either handles or passes up.

### Example

```java
public abstract class Handler {
    protected Handler next;
    public void setNext(Handler n) { this.next = n; }
    public abstract void handle(Request r);
}

public class AuthHandler extends Handler {
    public void handle(Request r) {
        if (!r.isAuthenticated()) throw new RuntimeException("401");
        if (next != null) next.handle(r);
    }
}
public class RateLimitHandler extends Handler {
    public void handle(Request r) {
        if (r.rateExceeded()) throw new RuntimeException("429");
        if (next != null) next.handle(r);
    }
}
public class BusinessHandler extends Handler {
    public void handle(Request r) { System.out.println("Handled: " + r); }
}

// Build the chain
Handler auth = new AuthHandler();
Handler rate = new RateLimitHandler();
Handler biz  = new BusinessHandler();
auth.setNext(rate);
rate.setNext(biz);
auth.handle(request);
```

### Real Java examples
- **Servlet Filter chains**
- **Spring Security filter chain**
- **Logging levels** (DEBUG → INFO → WARN → ERROR)
- **AOP interceptors**

---

## 11.18 Dependency Injection Pattern

Not GoF, but universal in modern Java.

### Problem
Hard-coded dependencies make code rigid, hard to test.

### Example

```java
// Bad — tight coupling
public class OrderService {
    private OrderRepository repo = new OrderRepositoryImpl();  // ❌
}

// Good — DI
public class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) {                // ✅
        this.repo = repo;
    }
}
```

Frameworks: **Spring**, **Guice**, **Dagger**, **CDI**.

Benefits: testable, swappable, loosely coupled.

---

## 11.19 SOLID Meets Patterns

Many patterns exist to satisfy SOLID:
- **Strategy** → Open/Closed (add new strategies without changing client)
- **Factory** → Dependency Inversion (depend on abstractions)
- **Decorator** → Open/Closed (add features without changing class)
- **Observer** → Dependency Inversion (subject depends on Observer abstraction)

See `12-System-Design.md` for SOLID details.

---

## 11.20 Anti-Patterns to Avoid

Just as important as good patterns:

- **God Class** — one class doing everything
- **Spaghetti Code** — no structure
- **Copy-Paste Programming** — should be extracted
- **Magic Numbers/Strings** — use constants
- **Golden Hammer** — using one pattern for everything
- **Premature Optimization** — Knuth's "root of all evil"
- **Yo-Yo Problem** — over-inheritance (jumping up and down the class hierarchy)

---

## 11.21 Real Framework Usage Recap

| Pattern | Real usage |
|---------|-----------|
| Singleton | Spring beans (default scope), Runtime, Logger |
| Factory | `Calendar.getInstance()`, `LocalDate.of()`, Spring BeanFactory |
| Builder | `StringBuilder`, Lombok `@Builder`, `HttpRequest.newBuilder()` |
| Adapter | `Arrays.asList()`, `InputStreamReader`, JDBC drivers |
| Decorator | `java.io` streams, `Collections.unmodifiableList()`, Servlet filters |
| Facade | `JdbcTemplate`, `RestTemplate`, SLF4J |
| Proxy | Hibernate lazy loading, Spring AOP (`@Transactional`), Java Dynamic Proxy |
| Composite | Swing components, filesystem, expression trees |
| Strategy | `Comparator`, `Runnable`, `Function<T,R>` |
| Observer | Spring `ApplicationEvent`, `@EventListener`, RxJS |
| Command | `Runnable` in Executors, undo/redo |
| Template Method | Spring `JdbcTemplate`, HttpServlet, `AbstractList` |
| Iterator | Every Java `Collection` |
| State | Order status machines, TCP state |
| Chain of Responsibility | Servlet filter chain, Spring Security |
| DI | Spring, Guice |

---

## 11.22 Interview Question Bank

1. **What are design patterns? Why use them?**  
   Reusable solutions to common problems; provide shared vocabulary and proven templates.

2. **Difference between Factory and Abstract Factory?**  
   Factory creates one type. Abstract Factory creates families of related types.

3. **Best way to implement Singleton?**  
   Enum (Bloch's recommendation). Otherwise Bill Pugh or DCL with volatile.

4. **What is Builder? When useful?**  
   Constructs complex objects step-by-step. Useful when many optional params (avoids telescoping constructors).

5. **Decorator vs Proxy?**  
   Decorator adds behavior. Proxy controls access (lazy, security, remote). Structure is similar; intent differs.

6. **Strategy vs State?**  
   Strategy: client chooses algorithm. State: object transitions between states internally.

7. **When to use Template Method?**  
   When multiple classes share a workflow but differ in specific steps. E.g., Spring's JdbcTemplate.

8. **Real-world Observer examples?**  
   Spring ApplicationEvent, RxJS, DOM listeners, Kafka consumer groups.

9. **What is SOLID? How do patterns relate?**  
   S: SRP, O: OCP (Strategy/Decorator), L: LSP, I: ISP, D: DIP (DI/Factory).

10. **Give example of Facade in Spring.**  
    `JdbcTemplate` hides JDBC boilerplate. `RestTemplate` hides HTTP details.

11. **Difference between Composition and Inheritance in patterns?**  
    Most GoF patterns favor composition (has-a) over inheritance (is-a) for flexibility.

12. **What is CQRS?**  
    Command Query Responsibility Segregation — separate models for reads and writes.

13. **Chain of Responsibility example?**  
    Servlet filters, Spring Security filter chain.

14. **Prototype vs Copy Constructor?**  
    Prototype uses `clone()` (broken in Java). Copy constructor is safer.

15. **Adapter vs Bridge?**  
    Adapter: make existing incompatible things work together. Bridge: separate abstraction from implementation so they can vary independently.

16. **What is Dependency Injection?**  
    Providing dependencies from outside instead of the class creating them.

17. **Command pattern use cases?**  
    Undo/redo, task queues, macros, `Runnable` on Executors.

18. **When NOT to use patterns?**  
    When simpler code works. Don't force patterns for their own sake — YAGNI.

19. **What is the DAO pattern?**  
    Data Access Object — abstracts persistence. Now largely replaced by Spring Data repositories.

20. **What is Repository pattern?**  
    Mediates domain and data mapping — collection-like abstraction over persistence.

---

## 11.23 Cheat Sheet

```
Creational: Singleton, Factory, Abstract Factory, Builder, Prototype
Structural: Adapter, Decorator, Facade, Proxy, Composite, Bridge
Behavioral: Strategy, Observer, Command, Template Method, Iterator, State,
            Chain of Responsibility

Singleton     → Enum > Bill Pugh > DCL with volatile
Factory       → Return interface; hide concrete type
Builder       → Fluent chaining for many optional params
Adapter       → Interface translator (like plug adapter)
Decorator     → Wrap to add behavior (java.io streams)
Facade        → Simple front door to complex subsystem
Proxy         → Placeholder controlling access (lazy, security)
Strategy      → Choose algorithm at runtime (Comparator)
Observer      → 1-to-many notification (Spring events)
Command       → Encapsulate action (Runnable)
Template      → Skeleton with customizable steps (JdbcTemplate)
```

---

## Practical Assignments

1. Implement a `Logger` singleton using all four techniques and compare.
2. Design a `NotificationSenderFactory` that returns Email/SMS/Push impls based on preference.
3. Build a `PizzaBuilder` (or use Lombok's `@Builder`) with all optional toppings.
4. Wrap an existing library class with an `Adapter` to fit your app's interface.
5. Implement Coffee ordering with `Decorator` (milk, sugar, caramel).
6. Refactor an `if/else` payment method chain into `Strategy`.
7. Model a TrafficLight with `State` pattern.
8. Build a simple servlet-like filter chain (`Chain of Responsibility`).

Master these and design pattern interviews are yours. 🚀