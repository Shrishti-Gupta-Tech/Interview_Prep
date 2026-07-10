# 6. SQL & MongoDB — Deep Dive

> **Goal:** Master relational database fundamentals, write efficient SQL, understand indexing, and confidently work with NoSQL (MongoDB).

---

## 6.0 Why Databases?

Apps need to **persist** data — survive restarts, be shared, be queryable.

Two families:
- **Relational (SQL)** — Tables, rows, columns; ACID transactions; strict schema. Examples: MySQL, PostgreSQL, Oracle, SQL Server.
- **NoSQL** — Flexible schemas, horizontal scaling, various models (document, key-value, wide-column, graph). Examples: MongoDB (document), Redis (KV), Cassandra (wide-column), Neo4j (graph).

**Rule of thumb:** Start with SQL. Reach for NoSQL only when you have a specific reason (scale, schema flexibility, specific data model).

---

## 6.1 SQL Basics — SELECT / WHERE / ORDER BY / LIMIT

### Analogy
A table is like an Excel spreadsheet. A row is a record. A column is an attribute.

### Sample data — `books` table

| id | title | author | price | published_at |
|----|-------|--------|-------|--------------|
| 1 | Java | Bloch | 500 | 2018-01-01 |
| 2 | Python | Lutz | 400 | 2019-06-15 |
| 3 | Go | Donovan | 350 | 2020-03-10 |

### Simple queries

```sql
-- Select all columns
SELECT * FROM books;

-- Select specific columns
SELECT id, title, price FROM books;

-- Filter with WHERE
SELECT * FROM books WHERE price > 400;

-- Multiple conditions
SELECT * FROM books
WHERE price > 300 AND author = 'Bloch';

-- OR
SELECT * FROM books WHERE price < 400 OR author = 'Bloch';

-- Pattern matching
SELECT * FROM books WHERE title LIKE 'J%';        -- starts with J
SELECT * FROM books WHERE title LIKE '%va%';      -- contains va

-- List membership
SELECT * FROM books WHERE author IN ('Bloch', 'Lutz');

-- Range
SELECT * FROM books WHERE price BETWEEN 300 AND 500;

-- NULL check
SELECT * FROM books WHERE description IS NULL;

-- Sorting
SELECT * FROM books ORDER BY price DESC;
SELECT * FROM books ORDER BY author ASC, price DESC;

-- Pagination
SELECT * FROM books ORDER BY id LIMIT 10 OFFSET 20;   -- page 3, 10 per page
```

### Aggregation — GROUP BY / HAVING

**Aggregate functions:** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

```sql
-- Book count per author
SELECT author, COUNT(*) AS book_count
FROM books
GROUP BY author;

-- Authors with more than 2 books, sorted
SELECT author, COUNT(*) AS book_count, AVG(price) AS avg_price
FROM books
GROUP BY author
HAVING COUNT(*) > 2
ORDER BY book_count DESC;
```

**WHERE vs HAVING:**
- **WHERE** filters rows *before* grouping (raw rows)
- **HAVING** filters *after* grouping (aggregate results)

```sql
SELECT author, AVG(price) AS avg_p
FROM books
WHERE price > 100                    -- filter individual books first
GROUP BY author
HAVING AVG(price) > 300;             -- then filter groups
```

---

## 6.2 JOINs — Combining Tables

### Setup: `authors` and `books` tables

```
authors                  books
+----+-------+           +----+---------+-----------+
| id | name  |           | id | title   | author_id |
+----+-------+           +----+---------+-----------+
| 1  | Bloch |           | 1  | Java    | 1         |
| 2  | Lutz  |           | 2  | Python  | 2         |
| 3  | Kant  |           | 3  | Effect. | 1         |
+----+-------+           +----+---------+-----------+
                                (author_id → authors.id)
```

### INNER JOIN — only matching rows

```sql
SELECT b.title, a.name
FROM books b
INNER JOIN authors a ON b.author_id = a.id;
```

Result: only books that have a matching author.

### LEFT JOIN — all from left, matches from right

```sql
SELECT a.name, b.title
FROM authors a
LEFT JOIN books b ON b.author_id = a.id;
```

Result: all authors, even those without books (title will be NULL for them).

### RIGHT JOIN — reverse of LEFT

```sql
SELECT a.name, b.title
FROM authors a
RIGHT JOIN books b ON b.author_id = a.id;
```

### FULL OUTER JOIN — all rows from both

```sql
SELECT a.name, b.title
FROM authors a
FULL OUTER JOIN books b ON b.author_id = a.id;
```

(Not supported in MySQL, but in PostgreSQL and SQL Server.)

### SELF JOIN — table joined with itself

Employees + their manager:
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### CROSS JOIN — Cartesian product

```sql
SELECT s.name, c.title
FROM students s CROSS JOIN courses c;   -- every student × every course
```

### Visual summary

```
INNER JOIN:      A ∩ B     (both must match)
LEFT JOIN:       A ⊃ (A∩B) (all A, matching B)
RIGHT JOIN:      B ⊃ (A∩B)
FULL OUTER JOIN: A ∪ B     (everything)
CROSS JOIN:      A × B     (every combination)
```

---

## 6.3 Subqueries & CTEs

### Subquery — query inside a query

```sql
-- Books more expensive than average
SELECT * FROM books
WHERE price > (SELECT AVG(price) FROM books);

-- Authors with at least one book (using EXISTS)
SELECT * FROM authors a
WHERE EXISTS (SELECT 1 FROM books b WHERE b.author_id = a.id);

-- Using IN
SELECT * FROM books
WHERE author_id IN (SELECT id FROM authors WHERE country = 'US');
```

### Correlated subquery — references outer query

```sql
-- Each book vs average price of its author's books
SELECT b.title, b.price,
       (SELECT AVG(price) FROM books WHERE author_id = b.author_id) AS author_avg
FROM books b;
```

### CTE (Common Table Expression) — named subquery

```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 100000
)
SELECT department, COUNT(*)
FROM high_earners
GROUP BY department;
```

**Recursive CTE** — for hierarchies:
```sql
WITH RECURSIVE org_tree AS (
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL             -- CEO
    UNION ALL
    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM employees e
    JOIN org_tree o ON e.manager_id = o.id
)
SELECT * FROM org_tree ORDER BY level;
```

---

## 6.4 Window Functions — Powerful Analytics

Operate on a "window" of rows without collapsing them (unlike GROUP BY).

### Example: rank employees by salary within their department

```sql
SELECT name, department, salary,
       RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS overall_row,
       SUM(salary)  OVER (PARTITION BY department) AS dept_total,
       AVG(salary)  OVER (PARTITION BY department) AS dept_avg,
       LAG(salary)  OVER (ORDER BY hire_date) AS prev_salary,
       LEAD(salary) OVER (ORDER BY hire_date) AS next_salary
FROM employees;
```

### RANK vs DENSE_RANK vs ROW_NUMBER

Salaries: 100, 100, 90:
- ROW_NUMBER: 1, 2, 3
- RANK: 1, 1, 3 (gap)
- DENSE_RANK: 1, 1, 2 (no gap)

### Common window functions
- `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE(n)`
- `LAG(col)`, `LEAD(col)` — previous/next row
- `SUM/AVG/MIN/MAX OVER(...)` — running/accumulated
- `FIRST_VALUE(col)`, `LAST_VALUE(col)`

**Real example — running total:**
```sql
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM transactions;
```

---

## 6.5 Views, Stored Procedures, Triggers

### Views — saved queries

```sql
CREATE VIEW top_books AS
SELECT title, price FROM books WHERE price > 500;

SELECT * FROM top_books;                -- use like a table
```

### Stored procedures

```sql
DELIMITER //
CREATE PROCEDURE increase_price(IN pct INT)
BEGIN
    UPDATE books SET price = price * (1 + pct / 100.0);
END //
DELIMITER ;

CALL increase_price(10);              -- raise all by 10%
```

### Triggers

```sql
CREATE TRIGGER audit_price_change
AFTER UPDATE ON books
FOR EACH ROW
INSERT INTO price_history(book_id, old_price, new_price, changed_at)
VALUES (OLD.id, OLD.price, NEW.price, NOW());
```

Use sparingly — makes app logic scattered and harder to debug.

---

## 6.6 Keys & Constraints

### Primary Key
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

### Foreign Key
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

`ON DELETE`: `CASCADE`, `SET NULL`, `RESTRICT`, `NO ACTION`.

### Common constraints
- `NOT NULL`
- `UNIQUE`
- `CHECK (age >= 18)`
- `DEFAULT 'active'`

### Composite key
```sql
PRIMARY KEY (student_id, course_id)
```

---

## 6.7 Indexes — The Speed-Up

An index is like a book's index — instead of scanning every page, you jump directly.

### Create index
```sql
CREATE INDEX idx_book_title    ON books(title);
CREATE UNIQUE INDEX idx_isbn   ON books(isbn);
CREATE INDEX idx_author_price  ON books(author_id, price);  -- composite
```

### Types
- **B-Tree** (default) — good for range queries, ORDER BY, equality
- **Hash** — only equality (used by memory engines)
- **Full-text** — text search (`MATCH() AGAINST()`)
- **Spatial** — GIS queries
- **Bitmap** — low-cardinality data (data warehouses)

### When to add an index?
- Columns in **WHERE**, **JOIN**, **ORDER BY**
- High-cardinality columns (many distinct values)
- **NOT** on columns rarely queried
- **NOT** on frequently-updated columns (indexes cost writes!)

### Composite index — column order matters

Index on `(author_id, price)` supports:
- `WHERE author_id = ?`
- `WHERE author_id = ? AND price > ?`
- `WHERE author_id = ? ORDER BY price`

But NOT:
- `WHERE price > ?` alone (leftmost prefix rule!)

### Downsides
- Slows down INSERT / UPDATE / DELETE
- Takes disk space
- Poorly chosen indexes = wasted resources

### EXPLAIN — see the query plan

```sql
EXPLAIN SELECT * FROM books WHERE title = 'Java';
EXPLAIN ANALYZE ...           -- with actual timings (PG)
```

Look for:
- `type: ALL` → full table scan (bad!)
- `type: ref/range/const` → using index (good)
- `Extra: Using filesort` → sorting outside index (may indicate missing sort index)

---

## 6.8 Normalization & Denormalization

### Normalization — reduce redundancy

**1NF** — atomic values, no repeating groups
Bad:
```
id | name | phones
1  | Alice | 111,222,333   -- multi-value!
```
Good: split to separate rows / another table.

**2NF** — 1NF + no partial dependency on composite PK
If your PK is `(order_id, product_id)`, don't store `product_name` here (depends only on `product_id`).

**3NF** — 2NF + no transitive dependency (non-key → non-key)
Bad: `students(id, name, department_id, department_name)` — `department_name` depends on `department_id`, not on `id`.
Good: split `departments` table.

**BCNF** — stricter version of 3NF.

### Denormalization — controlled redundancy for performance

For read-heavy analytics or reports, sometimes duplicate data:
- Store `user_name` in `orders` to avoid a JOIN
- Materialized views
- Precomputed counters

Trade-off: more storage & update complexity for faster reads.

---

## 6.9 Transactions & ACID (recap)

```sql
BEGIN;                                                  -- START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- or ROLLBACK;
```

**ACID:**
- **A**tomicity — all or nothing
- **C**onsistency — constraints enforced
- **I**solation — concurrent tx don't step on each other
- **D**urability — committed data survives crashes

**Isolation levels** (from weakest to strongest):
1. READ UNCOMMITTED — dirty reads possible
2. READ COMMITTED — most DBs default
3. REPEATABLE READ — MySQL InnoDB default
4. SERIALIZABLE — strongest, slowest

---

## 6.10 SQL Performance Tips

1. **Index frequently queried columns** (WHERE / JOIN / ORDER BY)
2. **Avoid `SELECT *`** — fetch only needed columns
3. **Use JOIN instead of correlated subqueries** where possible
4. **Batch INSERTs**: `INSERT INTO t VALUES (…),(…),(…)` — much faster than one-by-one
5. **Pagination**: prefer keyset over OFFSET for large tables
   ```sql
   -- OFFSET is slow for big pages
   SELECT * FROM books ORDER BY id LIMIT 20 OFFSET 100000;
   -- keyset (much faster)
   SELECT * FROM books WHERE id > 100000 ORDER BY id LIMIT 20;
   ```
6. **`EXPLAIN`** every non-trivial query
7. **Avoid functions on indexed columns** in WHERE:
   ```sql
   WHERE UPPER(email) = 'X'          -- ignores index
   WHERE email = 'x'                 -- uses index
   ```
8. **Use appropriate data types** (`VARCHAR(20)` not `TEXT` for short strings)
9. **Archive old data** — smaller tables = faster queries
10. **Analyze / rebuild indexes** periodically (`ANALYZE TABLE`, `OPTIMIZE TABLE`)

---

## 6.11 Common Interview SQL Queries

### 1) Nth highest salary

```sql
-- Method 1: LIMIT/OFFSET
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 4;                        -- 5th highest

-- Method 2: DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t WHERE rnk = 5;
```

### 2) Employees earning more than their manager

```sql
SELECT e.name
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

### 3) Duplicate rows

```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

### 4) Delete duplicates keeping one

```sql
DELETE u1 FROM users u1
JOIN users u2 ON u1.email = u2.email AND u1.id > u2.id;
```

### 5) Top N per group (top 3 highest paid per dept)

```sql
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
) t WHERE rnk <= 3;
```

### 6) Consecutive dates (gaps and islands)

```sql
-- Find users who logged in every day in a week
SELECT user_id
FROM logins
WHERE login_date BETWEEN '2025-01-01' AND '2025-01-07'
GROUP BY user_id
HAVING COUNT(DISTINCT login_date) = 7;
```

### 7) Running total

```sql
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) AS running_total
FROM transactions;
```

### 8) Percentage of total

```sql
SELECT department, SUM(salary) AS dept_total,
       SUM(salary) * 100.0 / SUM(SUM(salary)) OVER () AS pct_of_all
FROM employees
GROUP BY department;
```

---

## 6.12 DELETE vs TRUNCATE vs DROP

| | DELETE | TRUNCATE | DROP |
|-|--------|----------|------|
| Removes | rows | all rows | entire table |
| WHERE | yes | no | no |
| Rollbackable | yes (DML) | usually no (DDL) | no (DDL) |
| Triggers fire | yes | no | no |
| Auto-increment reset | no | yes | – |
| Speed | slower | fast | fastest |

---

## 6.13 MongoDB — NoSQL Document DB

### Concepts mapping

| SQL | MongoDB |
|-----|---------|
| Database | Database |
| Table | Collection |
| Row | Document (JSON/BSON) |
| Column | Field |
| Primary Key | `_id` (auto-created ObjectId) |
| JOIN | `$lookup` (aggregation) or embed |

### Documents look like JSON

```json
{
  "_id": ObjectId("..."),
  "title": "Java",
  "author": { "name": "Bloch", "country": "US" },
  "tags": ["programming", "backend"],
  "price": 500,
  "reviews": [
    { "user": "alice", "rating": 5, "comment": "Great!" }
  ]
}
```

Compare with SQL where you'd need 4 tables to represent this. **Embedded documents** vs **references** — a design choice.

### Basic operations (mongo shell)

```javascript
// Insert
db.books.insertOne({ title: "Java", price: 500 });
db.books.insertMany([{...}, {...}]);

// Find
db.books.find();                                    // all
db.books.find({ author: "Bloch" });                 // filter
db.books.find({ price: { $gt: 300 } });             // greater than
db.books.findOne({ _id: ObjectId("...") });

// Complex query
db.books.find({
    $and: [
        { price: { $gte: 300, $lte: 600 } },
        { $or: [
            { "author.country": "US" },
            { tags: "programming" }
        ]}
    ]
})
.sort({ price: -1 })
.skip(10)
.limit(5);

// Update
db.books.updateOne({ _id: id }, { $set: { price: 400 } });
db.books.updateMany({ author: "Bloch" }, { $inc: { price: 50 } });
db.books.updateOne({ _id: id }, { $push: { tags: "advanced" } });

// Delete
db.books.deleteOne({ _id: id });
db.books.deleteMany({ price: { $lt: 100 } });

// Upsert (insert if not exists)
db.books.updateOne(
    { title: "New" },
    { $set: { price: 500 } },
    { upsert: true }
);
```

### Common operators

**Query:** `$eq`, `$ne`, `$gt/$gte/$lt/$lte`, `$in`, `$nin`, `$and`, `$or`, `$not`, `$exists`, `$type`, `$regex`.

**Update:** `$set`, `$unset`, `$inc`, `$mul`, `$push`, `$pull`, `$addToSet`, `$rename`, `$currentDate`.

### Aggregation Pipeline

Chain stages. Each transforms the output of the previous.

```javascript
db.orders.aggregate([
    { $match: { status: "completed" } },                            // filter
    { $group: {                                                     // group
        _id: "$customerId",
        totalSpent: { $sum: "$amount" },
        orderCount: { $sum: 1 }
    }},
    { $sort: { totalSpent: -1 } },                                  // sort
    { $limit: 10 },                                                 // top 10
    { $lookup: {                                                    // join
        from: "customers",
        localField: "_id",
        foreignField: "_id",
        as: "customer"
    }},
    { $unwind: "$customer" },                                       // flatten array
    { $project: {                                                   // reshape
        customerName: "$customer.name",
        totalSpent: 1,
        orderCount: 1
    }}
]);
```

Common stages: `$match`, `$group`, `$sort`, `$limit`, `$skip`, `$project`, `$lookup`, `$unwind`, `$facet`, `$addFields`.

### Indexes

```javascript
db.books.createIndex({ title: 1 });                      // ascending
db.books.createIndex({ title: 1, price: -1 });           // compound
db.books.createIndex({ email: 1 }, { unique: true });    // unique
db.books.createIndex({ description: "text" });           // text search
db.books.createIndex({ location: "2dsphere" });          // geospatial

db.books.find({ title: "Java" }).explain("executionStats");
```

### Replication (High Availability)

**Replica set:** 1 primary + N secondaries + optional arbiter.
- All writes go to primary
- Reads can go to secondaries (eventual consistency)
- Automatic failover if primary crashes

### Sharding (Horizontal Scaling)

Split data across shards using a **shard key**.

Components:
- **Shard** — data node (each is a replica set)
- **Mongos** — router
- **Config server** — metadata

Choose shard key wisely:
- High cardinality
- Even distribution (avoid hot shards)
- Support your query patterns

### Transactions (Multi-Document, MongoDB 4.0+)

```javascript
const session = client.startSession();
session.startTransaction();
try {
    db.accounts.updateOne({ _id: 1 }, { $inc: { balance: -100 } }, { session });
    db.accounts.updateOne({ _id: 2 }, { $inc: { balance:  100 } }, { session });
    session.commitTransaction();
} catch (e) {
    session.abortTransaction();
} finally {
    session.endSession();
}
```

---

## 6.14 MongoDB with Spring Data

Add: `spring-boot-starter-data-mongodb`

```java
@Document(collection = "books")
public class Book {
    @Id
    private String id;                              // ObjectId as String
    private String title;
    private Author author;                          // embedded
    private List<String> tags;
    private double price;
    @Indexed(unique = true)
    private String isbn;
}

public interface BookRepository extends MongoRepository<Book, String> {
    List<Book> findByAuthor_Name(String name);
    List<Book> findByPriceGreaterThan(double p);

    @Query("{ 'tags': ?0, 'price': { $gt: ?1 } }")
    List<Book> findByTagAndMinPrice(String tag, double min);
}
```

MongoTemplate for advanced queries:
```java
@Autowired MongoTemplate mongoTemplate;

Query q = new Query();
q.addCriteria(Criteria.where("author.name").is("Bloch"));
q.addCriteria(Criteria.where("price").gt(300));
List<Book> books = mongoTemplate.find(q, Book.class);
```

---

## 6.15 SQL vs NoSQL — When to Use What?

| Use SQL when | Use MongoDB when |
|--------------|------------------|
| ACID transactions critical (finance, orders) | High write volume, flexible schema |
| Structured relationships (multi-table joins) | Hierarchical / nested data |
| Complex reporting / analytics | Rapidly evolving schema |
| Data integrity constraints matter | Horizontal scaling needed |
| Team knows SQL well | Document-oriented app (CMS, catalogs) |

**Real projects often use both** (polyglot persistence).

---

## 6.16 Interview Question Bank

1. **Difference between INNER JOIN and LEFT JOIN?**  
   INNER: only matching. LEFT: all from left + matches or nulls.

2. **What is index? Types?**  
   Data structure for fast lookup. B-Tree (default), hash, full-text, spatial, bitmap.

3. **Normalization vs Denormalization?**  
   Normalize to reduce redundancy (write efficiency, integrity). Denormalize for read speed.

4. **What is ACID?**  
   Atomicity, Consistency, Isolation, Durability.

5. **DELETE vs TRUNCATE vs DROP?**  
   DELETE — rows with WHERE, rollbackable. TRUNCATE — all rows, fast, not rollbackable. DROP — entire table.

6. **What is a stored procedure?**  
   Pre-compiled SQL routine stored in DB. `CALL proc_name(args)`.

7. **WHERE vs HAVING?**  
   WHERE filters rows before grouping. HAVING filters groups after aggregation.

8. **What is a window function?**  
   Operates on a window of rows without collapsing them (RANK, LAG, running SUM).

9. **How do you optimize a slow query?**  
   Add indexes on filtered columns, run EXPLAIN, avoid SELECT *, use JOINs over correlated subqueries, batch inserts.

10. **Difference between primary key and unique key?**  
    PK: unique + not null + one per table. Unique: unique but nullable, many per table.

11. **What is a view?**  
    Named saved query. Simplifies complex queries or restricts access.

12. **When to use MongoDB over SQL?**  
    Flexible schema, high write throughput, hierarchical data, horizontal scaling.

13. **What is the aggregation pipeline?**  
    MongoDB's data processing framework — chain stages ($match, $group, $sort…).

14. **Sharding vs Replication?**  
    Sharding — split data across nodes (scale writes). Replication — copies for HA / read scaling.

15. **What's `_id` in MongoDB?**  
    Auto-generated primary key. Type ObjectId (12 bytes: timestamp + machine + PID + counter).

16. **Composite index — column order matters why?**  
    Leftmost-prefix rule. `(a,b)` supports queries filtering `a` or `a,b` but not `b` alone.

17. **What causes a full table scan?**  
    No usable index, `WHERE` on non-indexed column, using functions on indexed column, `NOT` on indexed column.

18. **What is CAP theorem?**  
    In distributed systems, choose 2 of: Consistency, Availability, Partition tolerance.

19. **How does isolation level affect performance?**  
    Higher = more locking = safer but slower.

20. **What is a self-JOIN?**  
    Table joined with itself; used for hierarchical / same-table relationships (e.g., employee-manager).

---

## 6.17 Cheat Sheet

```
JOIN types: INNER (∩), LEFT (all A), RIGHT (all B), FULL OUTER (A∪B), CROSS (A×B), SELF
WHERE filters rows; HAVING filters groups
Index the columns in WHERE / JOIN / ORDER BY
EXPLAIN before optimizing
Batch INSERTs; use keyset pagination for large offsets
Window functions rank/partition without collapsing rows
ACID: Atomicity, Consistency, Isolation, Durability
Normalize by default; denormalize when reads dominate
MongoDB: use aggregation pipeline for complex reports
```

---

## Practical Assignments

1. Given `employees(id, name, dept, salary, manager_id)`, write queries for: 2nd highest salary, dept-wise avg, employees earning more than manager, top 3 per dept.
2. Design a schema for an e-commerce store (Users, Products, Orders, OrderItems) with proper keys and indexes.
3. Explain a slow query using EXPLAIN. Add an appropriate index and see the plan change.
4. Model a "blog with posts, comments, tags" in MongoDB (embedded vs referenced trade-offs).
5. Write an aggregation pipeline to compute monthly revenue per category.

Nail these and DB interviews are done. 🚀