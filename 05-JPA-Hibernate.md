# 5. Spring Data JPA & Hibernate — Deep Dive

> **Goal:** Understand the abstraction stack (JDBC → JPA → Hibernate → Spring Data), master relationships, avoid the N+1 trap, and be confident with transactions.

---

## 5.0 The Stack — Why So Many Layers?

Imagine you want to talk to a database. You could:

1. **JDBC** — raw. Open a connection, write SQL strings, iterate a ResultSet, close everything. Very manual.
2. **JPA** (Java Persistence API) — a **specification**. Says: "Here's how mapping and queries should work" but no implementation.
3. **Hibernate** — the most popular **implementation** of JPA. Actually does the work.
4. **Spring Data JPA** — thin layer on top. Removes boilerplate: you just define a repository interface, Spring auto-generates the implementation.

```
Your Code
   ↓
Spring Data JPA      (Repository interfaces)
   ↓
JPA API              (EntityManager, JPQL)
   ↓
Hibernate            (implementation, SQL generation)
   ↓
JDBC                 (low-level connection)
   ↓
Database             (MySQL, PostgreSQL, etc.)
```

### Why not just JDBC?
- Boilerplate (connection, statement, ResultSet, exceptions, close)
- No object-relational mapping — you manually convert rows to objects
- Not DB-independent — SQL varies

### What ORM gives you
- Map classes ↔ tables, fields ↔ columns automatically
- Object-oriented queries (JPQL)
- Caching (1st & 2nd level)
- Lazy loading
- Dirty checking (auto UPDATE on changes)
- Transaction management integration

---

## 5.1 Entity — Your Java-to-Table Bridge

An `@Entity` class represents a row in a table.

```java
@Entity
@Table(name = "books")
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "title", nullable = false, length = 200)
    private String title;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Enumerated(EnumType.STRING)      // stores enum name as text
    private Status status;

    @Transient                         // NOT persisted
    private String tempCache;

    // JPA needs a no-arg constructor
    protected Book() {}

    // Business constructor
    public Book(String title) { this.title = title; }

    // getters/setters...
}
```

### Common annotations

| Annotation | Purpose |
|------------|---------|
| `@Entity` | Marks class as a JPA entity |
| `@Table(name=)` | Custom table name |
| `@Id` | Primary key |
| `@GeneratedValue` | Auto-generate ID |
| `@Column` | Customize column (name, nullable, length, unique) |
| `@Enumerated(EnumType.STRING)` | How to store enums |
| `@Transient` | Ignore field |
| `@Lob` | Large object (BLOB/CLOB) |
| `@Temporal` (JPA 1.x) | Date type — obsolete with `java.time` |
| `@Version` | Optimistic locking version column |
| `@CreationTimestamp` / `@UpdateTimestamp` (Hibernate) | Auto-populate |

### ID generation strategies

```java
@GeneratedValue(strategy = GenerationType.IDENTITY)   // DB auto-increment
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq")
@GeneratedValue(strategy = GenerationType.AUTO)        // Hibernate picks
@GeneratedValue(strategy = GenerationType.UUID)        // JPA 3.1+
```

**Rule of thumb:**
- MySQL/H2 → `IDENTITY`
- PostgreSQL/Oracle → `SEQUENCE` (better for batch inserts)

---

## 5.2 Persistence Context & Entity Lifecycle

Think of the **persistence context** as an in-memory cache of your session. It tracks all managed entities and flushes changes to DB at the right moment.

### Entity states

```
                                   ┌───────────────────────────────┐
                                   │                               │
[NEW]  ──persist()──►  [MANAGED]  ────detach()──►  [DETACHED]  ────┘
                        │  ▲          ▲                merge()
                        │  │          │
                        │  └─ managed by persistence context
                        │
                        └──remove()──►  [REMOVED]  ──flush/commit──►  DB delete
                                                                     ▲
                                                                     │
                        (any change) ─────dirty check────► auto UPDATE
```

| State | Description | Tracked by JPA? | In DB? |
|-------|-------------|-----------------|--------|
| **NEW/Transient** | Just created via `new` | ❌ | ❌ |
| **MANAGED/Persistent** | Attached to persistence context | ✅ | Will be at flush |
| **DETACHED** | Was managed; context closed or `detach()` | ❌ | ✅ |
| **REMOVED** | Marked for delete | ✅ | Will be deleted |

### Example

```java
EntityManager em = ...;                        // usually injected

Book b = new Book("Java");                      // NEW
em.persist(b);                                  // → MANAGED (b.id is set!)

b.setTitle("Java Advanced");                    // MANAGED — dirty check
// no explicit save needed — auto UPDATE on flush

em.flush();                                     // sends SQL, still MANAGED

em.detach(b);                                   // → DETACHED
b.setTitle("...");                              // change NOT tracked

Book merged = em.merge(b);                      // → MANAGED again (returns tracked copy)

em.remove(merged);                              // → REMOVED
// DELETE on flush/commit
```

### Why `save()` returns an entity (Spring Data)

`repo.save(book)` may return a **different instance** than what you passed. Always use the returned one:

```java
Book saved = repo.save(book);        // use `saved`, not `book`, going forward
```

---

## 5.3 Spring Data JPA — Repositories

The magic — define an interface, get an implementation for free.

### Hierarchy

```
Repository<T, ID>                    (marker)
   ↑
CrudRepository<T, ID>                (save, findById, delete, count)
   ↑
PagingAndSortingRepository<T, ID>    (findAll(Pageable), findAll(Sort))
   ↑
JpaRepository<T, ID>                 (flush, saveAll, findAllInBatch, ...)
```

**Almost always extend `JpaRepository`.**

### Basic repository

```java
public interface BookRepository extends JpaRepository<Book, Long> {
}
```

That's it — you get:
- `findAll()`, `findAll(Pageable)`, `findAll(Sort)`
- `findById(Long)`
- `save(Book)`, `saveAll(Iterable<Book>)`
- `delete(Book)`, `deleteById(Long)`, `deleteAll()`
- `count()`, `existsById(Long)`

### Derived query methods — Spring parses the name

Spring reads your method name and generates SQL:

```java
List<Book> findByTitle(String title);
List<Book> findByTitleContainingIgnoreCase(String kw);
Optional<Book> findByIsbn(String isbn);
List<Book> findByPriceGreaterThan(double p);
List<Book> findByAuthorAndPriceLessThan(String author, double p);
List<Book> findTop5ByOrderByPriceDesc();
long countByAuthor(String author);
boolean existsByIsbn(String isbn);
```

Keywords: `And`, `Or`, `Between`, `LessThan/GreaterThan`, `Like`, `Containing`, `StartingWith`/`EndingWith`, `IsNull`, `In`, `OrderBy…Asc/Desc`, `Top5By…`, `Distinct`.

### Custom queries with `@Query`

**JPQL** (uses entity/field names):
```java
@Query("SELECT b FROM Book b WHERE b.author = :author AND b.price > :min")
List<Book> findExpensiveByAuthor(@Param("author") String a, @Param("min") double m);
```

**Native SQL** (uses table/column names):
```java
@Query(value = "SELECT * FROM books WHERE price > ?1", nativeQuery = true)
List<Book> findExpensiveNative(double min);
```

**Update/Delete queries** (need `@Modifying`):
```java
@Modifying
@Query("UPDATE Book b SET b.price = :price WHERE b.id = :id")
int updatePrice(Long id, double price);
```

Note: `@Modifying` queries **don't trigger dirty checking or cascading**.

### DTO projection — avoid loading full entity

```java
public interface BookSummary {
    Long getId();
    String getTitle();
    double getPrice();
}

@Query("SELECT b.id AS id, b.title AS title, b.price AS price FROM Book b")
List<BookSummary> findSummaries();
```

Or a record projection:
```java
@Query("SELECT new com.example.dto.BookDto(b.id, b.title) FROM Book b")
List<BookDto> findAllDtos();
```

---

## 5.4 Entity Relationships — The Heart of JPA

### The four cardinalities

- **@OneToOne** — Person ↔ Address
- **@OneToMany / @ManyToOne** — Author → Books (both sides)
- **@ManyToMany** — Student ↔ Courses

### @ManyToOne + @OneToMany — the classic

```java
@Entity
public class Author {
    @Id @GeneratedValue Long id;
    private String name;

    @OneToMany(mappedBy = "author",           // 'author' is the field in Book
               cascade = CascadeType.ALL,
               orphanRemoval = true,
               fetch = FetchType.LAZY)
    private List<Book> books = new ArrayList<>();

    // helper — ALWAYS use to maintain both sides
    public void addBook(Book b) {
        books.add(b);
        b.setAuthor(this);
    }
    public void removeBook(Book b) {
        books.remove(b);
        b.setAuthor(null);
    }
}

@Entity
public class Book {
    @Id @GeneratedValue Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)         // ← the OWNING side (has @JoinColumn)
    @JoinColumn(name = "author_id")            // FK column in books table
    private Author author;
}
```

### Owning vs inverse side

- **Owning side** has `@JoinColumn`. The **FK is here**.
- **Inverse side** has `mappedBy`. Only for JPA object graph — doesn't affect the DB.

**Both sides must be updated in memory for consistency.** JPA only writes what the *owning* side says.

### @ManyToMany

```java
@Entity
public class Student {
    @Id @GeneratedValue Long id;
    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id @GeneratedValue Long id;
    private String title;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

**Pro tip:** if the relationship carries extra data (e.g., enrollment date, grade), **don't use `@ManyToMany`**. Create a **join entity**:

```java
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne @MapsId("studentId") Student student;
    @ManyToOne @MapsId("courseId")  Course course;

    private LocalDate enrolledAt;
    private String grade;
}
```

### @OneToOne

```java
@Entity
public class Person {
    @Id @GeneratedValue Long id;
    private String name;

    @OneToOne(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "address_id")
    private Address address;
}
```

---

## 5.5 Fetching — Lazy vs Eager (Critical!)

### Defaults

| Relationship | Default |
|--------------|---------|
| `@OneToOne` | **EAGER** |
| `@ManyToOne` | **EAGER** |
| `@OneToMany` | **LAZY** |
| `@ManyToMany` | **LAZY** |

**Best practice:** always use `LAZY` explicitly. Override to eager only where absolutely needed.

```java
@ManyToOne(fetch = FetchType.LAZY)
private Author author;
```

### What is a proxy?

Lazy fields are backed by Hibernate proxies. The proxy stores only the ID until you actually call a getter.

```java
Book b = repo.findById(1L).get();
System.out.println(b.getAuthor().getClass());   // Author$HibernateProxy$xxx
System.out.println(b.getAuthor().getName());    // triggers SQL → real data loaded
```

### LazyInitializationException — the #1 JPA bug

```java
// In a service (transactional)
Book b = repo.findById(1L).get();
return b;                              // returns to controller

// In the controller (no tx!)
String name = b.getAuthor().getName(); // 💥 LazyInitializationException
// Persistence context is closed; can't load lazy fields
```

**Solutions:**

1. **Fetch what you need inside the transaction:**
   ```java
   @Transactional(readOnly = true)
   public BookDto getBookWithAuthor(Long id) {
       Book b = repo.findById(id).get();
       b.getAuthor().getName();          // trigger load
       return toDto(b);
   }
   ```

2. **JOIN FETCH** (single-query loading):
   ```java
   @Query("SELECT b FROM Book b JOIN FETCH b.author WHERE b.id = :id")
   Optional<Book> findByIdWithAuthor(Long id);
   ```

3. **@EntityGraph** — declarative eager loading:
   ```java
   @EntityGraph(attributePaths = {"author", "reviews"})
   Optional<Book> findById(Long id);
   ```

4. **DTO projection** (see 5.3).

---

## 5.6 N+1 — The Silent Performance Killer

### Symptom

Load 100 authors, then access each one's books:

```java
List<Author> authors = repo.findAll();               // 1 query
for (Author a : authors) {
    System.out.println(a.getBooks().size());          // 1 query PER AUTHOR = 100 more!
}
// Total: 1 + 100 = 101 queries
```

If Hibernate SQL logging is on, you'd see:
```
select * from authors;
select * from books where author_id = 1;
select * from books where author_id = 2;
...
```

### Fix 1 — JOIN FETCH

```java
@Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
List<Author> findAllWithBooks();
```

One query with a JOIN. `DISTINCT` de-dupes authors caused by the join.

### Fix 2 — @EntityGraph

```java
@EntityGraph(attributePaths = {"books"})
List<Author> findAll();
```

### Fix 3 — @BatchSize

```java
@OneToMany(mappedBy = "author")
@BatchSize(size = 20)
private List<Book> books;
```

Now Hibernate fetches children in batches of 20 (1 + 5 queries for 100 parents).

### Fix 4 — DTO projection

If you only need author name + book count, don't load full entities:
```java
@Query("SELECT new com.example.dto.AuthorSummary(a.name, size(a.books)) FROM Author a")
List<AuthorSummary> findSummaries();
```

**Interview tip:** always mention N+1 when they ask "how would you optimize JPA?"

---

## 5.7 Cascade & Orphan Removal

Cascade propagates operations from parent to children.

| Type | Effect |
|------|--------|
| `CascadeType.PERSIST` | Save children when parent saved |
| `CascadeType.MERGE` | Merge children when parent merged |
| `CascadeType.REMOVE` | Delete children when parent deleted ⚠️ |
| `CascadeType.REFRESH` | Refresh children |
| `CascadeType.DETACH` | Detach children |
| `CascadeType.ALL` | All of the above |

**`orphanRemoval = true`** — when a child is removed from the collection, delete it from DB.

```java
@OneToMany(mappedBy = "author",
           cascade = CascadeType.ALL,
           orphanRemoval = true)
private List<Book> books;

// Usage
author.getBooks().remove(book);    // triggers DELETE FROM books WHERE id = book.id
```

**⚠️ Danger:** `CascadeType.REMOVE` on a large collection means deleting the parent will issue N DELETE statements. Avoid on huge relations.

---

## 5.8 Queries — All the Options

### 1) Derived queries (5.3)

### 2) JPQL — Java Persistence Query Language

Object-oriented; uses entity and field names, not table/column names.

```java
@Query("SELECT b FROM Book b WHERE b.author.name = :name AND b.price > :min")
List<Book> byAuthorMinPrice(String name, double min);

// aggregation
@Query("SELECT AVG(b.price) FROM Book b WHERE b.author = :author")
Double avgPriceForAuthor(Author author);
```

### 3) Native SQL

```java
@Query(value = "SELECT * FROM books WHERE MATCH(title, description) AGAINST(?1)",
       nativeQuery = true)
List<Book> fullTextSearch(String term);
```

Native gives you full DB power (window functions, hints) at the cost of portability.

### 4) Named queries — declared on entity

```java
@Entity
@NamedQuery(name = "Book.findCheap", query = "SELECT b FROM Book b WHERE b.price < 100")
public class Book { }

// Spring Data picks it up automatically
public interface BookRepository extends JpaRepository<Book, Long> {
    List<Book> findCheap();
}
```

### 5) Criteria API — type-safe dynamic queries

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Book> q = cb.createQuery(Book.class);
Root<Book> b = q.from(Book.class);

Predicate authorMatch = cb.equal(b.get("author").get("name"), "Rowling");
Predicate priceMatch = cb.gt(b.get("price"), 100.0);

q.select(b).where(cb.and(authorMatch, priceMatch));

List<Book> res = em.createQuery(q).getResultList();
```

Verbose but 100% type-safe. Alternative: **jOOQ**, **Blaze-Persistence**, or **QueryDSL**.

### 6) Specifications — Spring Data's clean dynamic queries

```java
public interface BookRepository
    extends JpaRepository<Book, Long>, JpaSpecificationExecutor<Book> { }

public class BookSpecs {
    public static Specification<Book> hasAuthor(String name) {
        return (root, q, cb) -> cb.equal(root.get("author").get("name"), name);
    }
    public static Specification<Book> priceGreaterThan(double p) {
        return (root, q, cb) -> cb.gt(root.get("price"), p);
    }
}

// Usage
repo.findAll(BookSpecs.hasAuthor("X").and(BookSpecs.priceGreaterThan(100)));
```

---

## 5.9 Transactions — `@Transactional` Deep Dive

Wraps a method in a DB transaction.

```java
@Service
public class BookService {

    @Transactional                                        // start tx, commit on return, rollback on RuntimeException
    public Book create(Book b) { return repo.save(b); }

    @Transactional(readOnly = true)                       // hint: no writes → optimizations
    public List<Book> findAll() { return repo.findAll(); }

    @Transactional(rollbackFor = Exception.class)         // rollback on ANY exception (default: RuntimeException only)
    public void risky() throws IOException { }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void audit(String msg) { auditRepo.save(new AuditEntry(msg)); }

    @Transactional(isolation = Isolation.REPEATABLE_READ, timeout = 30)
    public void complex() { }
}
```

### Propagation levels

| Type | Behavior |
|------|----------|
| **REQUIRED** (default) | Join existing tx; else start new |
| **REQUIRES_NEW** | Suspend existing, always start new (great for audits) |
| **SUPPORTS** | Join if exists; else no tx |
| **NOT_SUPPORTED** | Suspend if exists; run without tx |
| **MANDATORY** | Must have existing tx (else throw) |
| **NEVER** | Must NOT have existing tx |
| **NESTED** | Savepoint within existing tx |

### Isolation levels

| Level | Prevents |
|-------|----------|
| **READ_UNCOMMITTED** | Nothing (dirty read possible) |
| **READ_COMMITTED** | Dirty read (default in most DBs) |
| **REPEATABLE_READ** | Dirty + non-repeatable read (MySQL default) |
| **SERIALIZABLE** | All (dirty + non-repeatable + phantom) |

- **Dirty read** — read uncommitted data from another tx
- **Non-repeatable read** — same query returns different values within same tx
- **Phantom read** — same query returns different set of rows within same tx

### ACID

- **A**tomicity — all-or-nothing
- **C**onsistency — valid state → valid state (constraints enforced)
- **I**solation — concurrent tx behave as if serial
- **D**urability — committed changes survive crashes

### Dirty Checking Magic

You don't call `save()` to update — just modify a managed entity:

```java
@Transactional
public void updatePrice(Long id, double newPrice) {
    Book b = repo.findById(id).orElseThrow();
    b.setPrice(newPrice);              // no save() needed!
    // At tx commit, Hibernate compares snapshot vs current → issues UPDATE
}
```

### @Transactional Gotchas

**1) Only on public methods.** Spring uses proxies. Private/protected calls bypass the proxy.

**2) Self-invocation doesn't work:**
```java
@Service
public class MyService {
    public void outer() {
        inner();                  // ❌ NOT transactional (bypasses proxy)
    }
    @Transactional
    public void inner() { }
}
```
Fix: split into two beans, or inject self, or use `TransactionTemplate`.

**3) Default rollback is RuntimeException only:**
```java
@Transactional                                  // ❌ won't rollback on IOException
public void x() throws IOException { }

@Transactional(rollbackFor = Exception.class)   // ✅ rolls back on any exception
public void y() throws IOException { }
```

---

## 5.10 Optimistic vs Pessimistic Locking

Concurrent updates? Two strategies:

### Optimistic Locking — check version on write

```java
@Entity
public class Book {
    @Id Long id;
    private String title;

    @Version                       // Hibernate manages this
    private int version;
}
```

Every UPDATE:
```sql
UPDATE books SET title = ?, version = version + 1
WHERE id = ? AND version = ?
```

If another tx updated first, `version` doesn't match → 0 rows updated → `OptimisticLockException`. Retry or notify user.

### Pessimistic Locking — hold DB lock

```java
@Query("SELECT b FROM Book b WHERE b.id = :id")
@Lock(LockModeType.PESSIMISTIC_WRITE)
Book findByIdForUpdate(Long id);
```

Issues `SELECT ... FOR UPDATE`. Blocks other writers. Use sparingly — can cause bottlenecks/deadlocks.

**Rule:** default to optimistic; use pessimistic for hot rows (like inventory counters).

---

## 5.11 Caching

### First-level (per session, always on)

Within one `EntityManager`/tx, fetching the same entity twice hits DB only once.

```java
@Transactional
public void demo() {
    Book b1 = em.find(Book.class, 1L);   // SELECT
    Book b2 = em.find(Book.class, 1L);   // no SELECT — from L1 cache
    System.out.println(b1 == b2);         // true — same instance
}
```

### Second-level (across sessions, opt-in)

Store entities in a shared cache (EhCache, Hazelcast, Redis). Great for reference data.

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... }
```

### Query cache — cache query results

```properties
spring.jpa.properties.hibernate.cache.use_query_cache=true
```
Then annotate query hint: `@QueryHint(name = "org.hibernate.cacheable", value = "true")`.

Use only for read-mostly reference data (list of countries, categories).

---

## 5.12 Common Performance Optimizations

1. **Solve N+1** — JOIN FETCH, EntityGraph, BatchSize.
2. **Use projections/DTOs** for read-only endpoints (skip entity overhead).
3. **`@Transactional(readOnly = true)`** for read endpoints — enables optimizations, skips dirty checking.
4. **Batch inserts/updates:**
   ```properties
   spring.jpa.properties.hibernate.jdbc.batch_size=50
   spring.jpa.properties.hibernate.order_inserts=true
   spring.jpa.properties.hibernate.order_updates=true
   ```
5. **Index DB columns** used in WHERE/JOIN/ORDER BY.
6. **Tune HikariCP** connection pool.
7. **Avoid CascadeType.REMOVE** on huge collections; use bulk DELETE.
8. **Enable SQL logging in dev** to inspect what's being sent:
   ```properties
   spring.jpa.show-sql=true
   spring.jpa.properties.hibernate.format_sql=true
   logging.level.org.hibernate.SQL=DEBUG
   logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
   ```
9. **Use pagination** for large lists.
10. **Avoid `@OneToMany` with `List` and duplicates** — Hibernate can't optimize; use `Set` or `Map`.

---

## 5.13 Auditing — `@CreatedDate`, `@LastModifiedDate`

```java
@EnableJpaAuditing                          // in @Configuration

@Entity
@EntityListeners(AuditingEntityListener.class)
public class Book {
    @Id @GeneratedValue Long id;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

`AuditorAware` bean tells Spring who the current user is (usually from SecurityContext).

---

## 5.14 Mini Project — Book & Author with Relationships

**Author.java**
```java
@Entity
public class Author {
    @Id @GeneratedValue Long id;
    private String name;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<Book> books = new ArrayList<>();

    public void addBook(Book b) { books.add(b); b.setAuthor(this); }
    // getters/setters
}
```

**Book.java**
```java
@Entity
public class Book {
    @Id @GeneratedValue Long id;
    private String title;
    private double price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;

    @Version
    private int version;
    // getters/setters
}
```

**Repositories:**
```java
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @EntityGraph(attributePaths = "books")
    Optional<Author> findWithBooksById(Long id);
}

public interface BookRepository extends JpaRepository<Book, Long>,
                                        JpaSpecificationExecutor<Book> {
    List<Book> findByAuthorId(Long authorId);
}
```

**Service:**
```java
@Service
@Transactional
public class LibraryService {
    private final AuthorRepository authorRepo;
    private final BookRepository bookRepo;

    public LibraryService(AuthorRepository a, BookRepository b) {
        this.authorRepo = a; this.bookRepo = b;
    }

    public Book addBookToAuthor(Long authorId, Book newBook) {
        Author a = authorRepo.findById(authorId).orElseThrow();
        a.addBook(newBook);                            // maintains both sides
        return newBook;                                // saved via cascade
    }

    @Transactional(readOnly = true)
    public Author getWithBooks(Long id) {
        return authorRepo.findWithBooksById(id).orElseThrow();
    }
}
```

---

## 5.15 Interview Question Bank

1. **JPA vs Hibernate?**  
   JPA is a spec. Hibernate is the most popular implementation.

2. **What is the Persistence Context?**  
   In-memory first-level cache; tracks all managed entities in a session/tx.

3. **Explain entity lifecycle states.**  
   NEW → MANAGED (via persist) → DETACHED (via detach or close) → REMOVED (via remove). Merge brings detached back to managed.

4. **`save()` vs `persist()` vs `merge()`?**  
   `persist` — insert new; throws if entity already exists.  
   `merge` — for detached entities; copies state into managed one, returns managed instance.  
   `save` (Spring/Hibernate) — chooses between them.

5. **Lazy vs Eager — which is better?**  
   LAZY by default. Eager only for small, mandatory relations.

6. **N+1 problem — how to fix?**  
   JOIN FETCH, EntityGraph, BatchSize, DTO projection.

7. **`get()` vs `load()` (Hibernate)?**  
   `get` — returns null / hits DB immediately.  
   `load` — returns proxy, hits DB on first access, throws if not found.

8. **@Transactional propagation levels?**  
   REQUIRED (default), REQUIRES_NEW, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER, NESTED.

9. **What is dirty checking?**  
   Hibernate compares entity's current state with snapshot; auto-issues UPDATE at flush.

10. **What is orphan removal?**  
    Deletes child from DB when removed from parent's collection.

11. **JPQL vs Native SQL — trade-offs?**  
    JPQL — portable, object-oriented. Native — full DB power, less portable.

12. **Second-level cache — when to use?**  
    Read-mostly reference data (countries, categories, small lookups).

13. **Optimistic vs pessimistic locking?**  
    Optimistic uses version; retry on conflict. Pessimistic holds DB lock; blocks.

14. **Unidirectional vs bidirectional mapping?**  
    Uni — one side knows about the other. Bi — both sides. Bi requires helper methods to maintain consistency.

15. **JPA performance optimizations in your project?**  
    JOIN FETCH, batch fetching, DTO projections, read-only tx, HikariCP tuning, DB indexes, `hibernate.jdbc.batch_size`.

16. **What is `@EntityGraph`?**  
    Declarative way to fetch related entities eagerly for a specific query.

17. **Why is `@ManyToMany` often problematic?**  
    Loses ability to add relationship attributes; sometimes generates surprising SQL. Prefer a join entity when relationship has data.

18. **What is a JPA proxy?**  
    Lazy-loading placeholder. Real data loaded on first access.

19. **How does Spring Data know to create the query?**  
    Parses your method name (derived queries) or reads `@Query`.

20. **Can `@Transactional` be applied to private methods?**  
    No. Spring uses proxies; only intercepts public method calls from outside.

---

## 5.16 Cheat Sheet

```
Entity lifecycle: new → persist → managed → detach → merge → managed → remove
Default fetch: @*ToOne EAGER (change to LAZY!), @*ToMany LAZY
Solve N+1 with: JOIN FETCH / @EntityGraph / @BatchSize
Owning side has @JoinColumn; inverse side has mappedBy
Always maintain both sides in helpers
@Transactional default rollback: RuntimeException only
@Transactional(readOnly = true) for reads
Prefer optimistic locking (@Version) over pessimistic
Use DTO projections for read-heavy endpoints
Enable spring.jpa.show-sql in dev to spot bad queries
```

---

## Practical Assignments

1. Design an `Order → OrderItem → Product` schema with proper relationships & cascades. Save an order with items in one call.
2. Reproduce an N+1 (parent + lazy children in a loop), then fix it with `@EntityGraph` and see the SQL log.
3. Add optimistic locking with `@Version`; write a test simulating two threads updating the same entity.
4. Write a repository with a `Specification`-based dynamic search (title, priceMin, priceMax filters).
5. Create an audited entity with `@CreatedDate`/`@LastModifiedDate` using `AuditorAware`.

Master these and JPA/Hibernate interviews are yours! 🚀