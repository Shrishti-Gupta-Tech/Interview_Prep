# 12. System Design — Deep Dive

> **Goal:** Master SOLID principles (LLD basis), tackle low-level design questions with confidence, and approach high-level design in a structured way.

---

## 12.0 Two Levels of "System Design"

- **LLD (Low-Level Design)** — class-level design. "Design a Parking Lot / URL Shortener." Focuses on classes, interfaces, relationships, patterns.
- **HLD (High-Level Design)** — architecture-level. "Design WhatsApp / Instagram / Netflix." Focuses on components, DBs, caching, scaling, trade-offs.

Both levels want to see:
1. Do you ask the right clarifying questions?
2. Do you think about trade-offs?
3. Do you consider scale, failure, security?
4. Can you communicate clearly?

---

## 12.1 SOLID — The 5 Principles

SOLID is a set of design principles that make code **flexible, maintainable, and testable**.

### S — Single Responsibility Principle

> A class should have **only one reason to change**.

**Bad:**
```java
class Report {
    public String getContent() { }
    public void save(String path) { }         // file I/O
    public void email(String to) { }          // networking
}
```
Any change in file format, email provider, or content structure → same class edited. Fragile.

**Good:**
```java
class Report { public String getContent() { } }
class ReportSaver { public void save(Report r, String path) { } }
class ReportMailer { public void email(Report r, String to) { } }
```

**Analogy:** a chef cooks; a waiter serves; a cashier handles money. Don't make one person do all three.

### O — Open/Closed Principle

> Software should be **open for extension** but **closed for modification**.

**Bad:**
```java
class AreaCalc {
    double area(Object shape) {
        if (shape instanceof Circle)   return ...;
        else if (shape instanceof Square) return ...;
        // Adding Triangle? Modify this class!
    }
}
```

**Good — extend via polymorphism:**
```java
interface Shape { double area(); }
class Circle implements Shape { public double area() { return Math.PI * r * r; } }
class Square implements Shape { public double area() { return side * side; } }
class Triangle implements Shape { public double area() { return 0.5 * b * h; } }

class AreaCalc {
    double area(Shape s) { return s.area(); }
}
```
Now adding a new shape doesn't touch `AreaCalc`.

### L — Liskov Substitution Principle

> Subtypes must be **substitutable** for their base types without breaking behavior.

**Classic violation — Rectangle/Square:**
```java
class Rectangle {
    void setWidth(int w) { }
    void setHeight(int h) { }
}
class Square extends Rectangle {  // ❌
    // If I set width, height must also change to stay a square!
    // Client code expecting Rectangle semantics breaks.
}
```

**Fix:** Don't force an inheritance that violates behavior. Use composition, or make both implement a common `Shape` interface without shared mutation.

### I — Interface Segregation Principle

> Clients shouldn't be forced to depend on methods they don't use.

**Bad:**
```java
interface Worker {
    void work();
    void eat();
}

class Robot implements Worker {
    public void work() { }
    public void eat() { throw new UnsupportedOperationException(); }  // ❌
}
```

**Good — split:**
```java
interface Workable { void work(); }
interface Eatable  { void eat(); }

class Human implements Workable, Eatable { }
class Robot implements Workable { }
```

### D — Dependency Inversion Principle

> Depend on **abstractions**, not concretions.

**Bad:**
```java
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository();  // ❌
}
```

**Good:**
```java
interface OrderRepository { void save(Order o); }

class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }
}
```
Now you can swap `MySQLOrderRepository`, `MongoOrderRepository`, or a mock for tests.

DI frameworks (Spring) make this the default.

### SOLID cheat sheet

- **S**RP — one class, one reason to change
- **O**CP — extend without modifying (polymorphism)
- **L**SP — subtype must honor contract
- **I**SP — no fat interfaces
- **D**IP — depend on interfaces, not classes

---

## 12.2 LLD Approach — 6 Steps

When asked "design X":

1. **Clarify requirements** — features, constraints, scale
2. **Identify entities/classes** — nouns from requirements
3. **Define relationships** — is-a (inheritance) vs has-a (composition)
4. **Design interfaces & apply SOLID + patterns**
5. **Handle edge cases** — errors, concurrency, security
6. **Discuss extensions** — how it evolves

---

## 12.3 LLD — Parking Lot

**Requirements:**
- Multiple floors, multiple slot sizes (small, medium, large, disabled)
- Cars/motorcycles/trucks
- Entry/exit, ticket, payment
- Track availability

**Classes:**
```java
enum VehicleType { CAR, BIKE, TRUCK }
enum SlotSize { SMALL, MEDIUM, LARGE, DISABLED }

abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;
    public abstract SlotSize requiredSize();
}
class Car extends Vehicle { public SlotSize requiredSize() { return SlotSize.MEDIUM; } }
class Bike extends Vehicle { public SlotSize requiredSize() { return SlotSize.SMALL; } }
class Truck extends Vehicle { public SlotSize requiredSize() { return SlotSize.LARGE; } }

class Slot {
    private final int id;
    private final SlotSize size;
    private boolean occupied;
    // occupy / vacate
}

class Floor {
    private final int number;
    private final List<Slot> slots;
    public Optional<Slot> findFreeSlot(SlotSize size) { }
}

class Ticket {
    private final String id;
    private final Slot slot;
    private final Vehicle vehicle;
    private final Instant entryTime;
    private Instant exitTime;
}

interface PricingStrategy {
    double calculate(Ticket t);
}
class HourlyPricing implements PricingStrategy { }
class FlatRatePricing implements PricingStrategy { }

class ParkingLot {
    private final List<Floor> floors;
    private final PricingStrategy pricing;
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();

    public synchronized Ticket park(Vehicle v) {
        for (Floor f : floors) {
            Optional<Slot> s = f.findFreeSlot(v.requiredSize());
            if (s.isPresent()) {
                Slot slot = s.get();
                slot.occupy();
                Ticket t = new Ticket(slot, v);
                activeTickets.put(t.getId(), t);
                return t;
            }
        }
        throw new NoSlotAvailableException();
    }

    public synchronized double exit(String ticketId) {
        Ticket t = activeTickets.remove(ticketId);
        if (t == null) throw new InvalidTicketException();
        double fee = pricing.calculate(t);
        t.getSlot().vacate();
        return fee;
    }
}
```

**Patterns used:** Strategy (pricing), Factory (Vehicle creation), Singleton (ParkingLot).

**Concurrency:** synchronize / atomic slot state.

---

## 12.4 LLD — URL Shortener (bit.ly)

**Requirements:**
- Shorten a long URL to a short code
- Redirect from short → long
- 500M URLs/month
- Analytics, custom aliases, expiry

**API:**
```
POST /shorten
Body: { "longUrl": "...", "customAlias": "myLink", "expiryDate": "2026-01-01" }
Response: { "shortUrl": "https://x.co/abc123" }

GET  /{code}  →  302 redirect to longUrl
```

**DB schema:**
```sql
CREATE TABLE urls (
    short_code   VARCHAR(10) PRIMARY KEY,
    long_url     VARCHAR(2048) NOT NULL,
    user_id      BIGINT,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at   TIMESTAMP,
    click_count  BIGINT DEFAULT 0
);
CREATE INDEX idx_long_url ON urls(long_url);
```

**Short code generation — 3 approaches:**

1. **Random Base62** — `[a-zA-Z0-9]` → 62^7 = 3.5T combinations. Check collision; retry.
2. **Counter + Base62 encode** — auto-increment ID → encode. Guaranteed unique, short. Best choice.
3. **Hash (MD5/SHA)** — first 6-8 chars. Simple but collisions possible.

```java
public class Base62 {
    private static final String CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    public static String encode(long n) {
        StringBuilder sb = new StringBuilder();
        while (n > 0) { sb.append(CHARS.charAt((int)(n % 62))); n /= 62; }
        return sb.reverse().toString();
    }
}
```

**Components:**
```
    Client
      │
      ▼
  Load Balancer
      │
      ▼
  App Servers (stateless — horizontally scale)
      │
      ├── Redis Cache (hot mappings — 80% hit rate)
      │
      ├── DB (Postgres or Cassandra — sharded by short_code)
      │
      └── Kafka → Analytics consumer (async click counting)
```

**Read path:**
1. `GET /abc123`
2. Check Redis
3. Miss → hit DB, populate cache
4. Publish click event to Kafka (async)
5. 302 redirect

**Optimizations:**
- Read-heavy → aggressive caching
- CDN for redirects
- Consistent hashing for DB shards
- Rate limiting per user

---

## 12.5 LLD — Notification Service

**Requirements:**
- Multi-channel: Email, SMS, Push
- Scheduling
- Retries + DLQ
- User preferences

**Architecture:**
```
API → Kafka(notifications) → Workers → Provider (SendGrid/Twilio/FCM)
                                      → DLQ on repeated failure
```

**Class design:**
```java
enum Channel { EMAIL, SMS, PUSH }

public class Notification {
    private String id;
    private String userId;
    private Channel channel;
    private String template;
    private Map<String, Object> data;
    private Instant scheduleAt;
}

// Strategy pattern for channels
public interface NotificationSender {
    void send(Notification n);
}

@Component("email")
public class EmailSender implements NotificationSender { }

@Component("sms")
public class SmsSender implements NotificationSender { }

@Component("push")
public class PushSender implements NotificationSender { }

public class NotificationDispatcher {
    private final Map<Channel, NotificationSender> senders;   // injected

    public void dispatch(Notification n) {
        senders.get(n.getChannel()).send(n);
    }
}
```

**Retry with exponential backoff:**
- Attempt 1 fail → wait 1s
- Attempt 2 fail → wait 2s
- Attempt 3 fail → wait 4s
- Still fail → move to DLQ

**Templating:** Freemarker/Thymeleaf. Store templates in DB or file storage.

---

## 12.6 LLD — Logging System

Not just `log.info(...)` in code; a structured system:

```
App (SLF4J)
    │
    ▼
Filebeat / Fluentd (agent on each host)
    │
    ▼
Kafka (buffer)
    │
    ▼
Logstash (parse/enrich)
    │
    ▼
Elasticsearch (index, search)   +   S3 (long-term archive)
    │
    ▼
Kibana (visualize)
    │
    ▼
AlertManager (alert on patterns)
```

**Structured logging (JSON):**
```json
{
  "timestamp": "2025-10-07T14:00:00Z",
  "level": "ERROR",
  "service": "book-service",
  "traceId": "abc123",
  "spanId": "def456",
  "message": "Failed to save book",
  "bookId": 42
}
```

Enables searching by any field, correlation across services.

---

## 12.7 HLD — General Approach

Follow this framework in any HLD interview:

### Step 1 — Requirements (5 min)
- **Functional** — what should it do?
- **Non-functional** — scale, latency, availability, consistency
- Constraints (read-heavy? write-heavy? geographies?)

### Step 2 — Estimate scale (2 min)
- Users, QPS (queries per second), storage
- Example: 1B users, 10% active daily = 100M DAU. 10 queries/day/user = 1B queries/day = ~12K QPS avg, 5x peak = 60K QPS.

### Step 3 — API Design (3 min)
```
POST /api/orders          → OrderResponse
GET  /api/orders/{id}     → OrderResponse
GET  /api/orders?userId=  → paginated list
```

### Step 4 — Data Model (3 min)
- SQL / NoSQL choice + why
- Key schema decisions
- Partitioning / sharding

### Step 5 — High-Level Diagram (10 min)
```
Client → LB → API Gateway → Services → Cache → DB
                                      → Queue → Async workers
```

### Step 6 — Deep-Dive (10-15 min)
- Focus on 1-2 critical components (interviewer will hint)
- Show trade-offs

### Step 7 — Bottlenecks / Scaling (5 min)
- Where's the SPOF?
- How to scale further?
- Monitoring, security, backup

---

## 12.8 HLD — Chat System (WhatsApp)

**Requirements:**
- 1-1 & group chat (up to 200 members)
- Online/offline status
- Read receipts
- Media (image/video)
- ~1B users, 50B messages/day, low latency (<200ms)

**Capacity:**
- 50B msgs/day = ~580K msgs/sec average, 2-3M peak
- Message size ~200 bytes → ~10 TB/day
- Storage: keep 5 years → ~18 PB

**Components:**

```
    Mobile / Web
         │
         ▼
   Load Balancer
         │
         ▼
   ┌───────────────┐
   │  API Gateway  │
   └───────────────┘
         │
         ├────► Auth Service (JWT / OAuth)
         │
         ├────► Chat Service (WebSocket, sticky sessions)
         │           │
         │           ├───► Kafka (chat.messages)
         │           │        │
         │           │        ▼
         │           │      Message Store (Cassandra)
         │           │
         │           └───► Presence Cache (Redis)
         │
         ├────► Media Service → S3 + CDN
         │
         └────► Notification Service (FCM/APNs)
                   for offline users
```

### Why Cassandra for messages?
- Massive write throughput
- Time-series friendly
- Partition by `chatId + bucket(time)` for locality
- No cross-partition joins needed (each chat is self-contained)

### Message flow (User A → User B)
1. A sends over WebSocket to Chat Service instance holding A's connection
2. Chat Service publishes to Kafka topic `chat.messages`
3. Consumer stores in Cassandra
4. If B is online (in Redis) → routed to B's Chat Service instance → sent over WebSocket
5. If B is offline → push notification via FCM
6. When B opens app → fetches unread messages
7. B's read receipt → Kafka event → propagate to A

### Group chat
- Message published to group topic
- Fan-out to all members' WebSocket connections
- Persist once (with recipient list)

### Presence (online status)
- Heartbeat every 30s → Redis key with TTL 60s
- Query "who's online" → Redis get

### Storage estimate
- 10 TB/day compressed
- Cassandra clusters sized accordingly
- Cold data → object storage (S3 Glacier)

### Scaling levers
- Sticky WebSocket sessions with consistent hashing
- Sharded Redis for presence
- Cassandra replication factor 3, cross-DC for HA

---

## 12.9 HLD — Cache Design

### Why cache?
- DB is slow (disk IO)
- Cache (RAM) is 100-1000× faster
- Reduce DB load
- Enable higher QPS

### Cache patterns

1. **Cache-Aside (lazy loading)** — most common
   ```java
   V get(K k) {
       V v = redis.get(k);
       if (v == null) {
           v = db.get(k);
           if (v != null) redis.setex(k, ttl, v);
       }
       return v;
   }
   ```
   Pros: simple, cache only what's needed. Cons: cold cache misses.

2. **Write-Through** — write goes to cache AND DB synchronously
   Pros: cache always fresh. Cons: write latency doubled.

3. **Write-Back (Write-Behind)** — write to cache, DB updated async
   Pros: fast writes. Cons: data loss risk on crash.

4. **Read-Through** — cache library fetches from DB on miss transparently

### Eviction policies
- **LRU** (Least Recently Used) — most common, kicks out oldest
- **LFU** (Least Frequently Used)
- **FIFO**
- **TTL** — time-based

### Cache consistency strategies
- **TTL** — accept short staleness
- **Explicit invalidation** — on write, delete cache key
- **Pub/sub invalidation** — write publishes an invalidation event

### Cache stampede
Many clients miss cache at once → all hit DB simultaneously.

**Fixes:**
- **Mutex/lock** — only one thread refreshes; others wait
- **Request coalescing** — one request per key
- **Probabilistic early expiration** — some requests refresh before TTL

### Distributed cache
- **Redis** — most popular, single-threaded (fast), rich data types (strings, hashes, sets, sorted sets, streams)
- **Memcached** — simpler, multi-threaded

### Consistent hashing
Distributing keys across N cache servers. Adding/removing a server should re-hash only 1/N keys, not all.

---

## 12.10 HLD — Rate Limiter

**Requirements:**
- N req/sec per user/IP/API-key
- Distributed
- Low latency (<5ms)

### Algorithms

1. **Fixed Window**
   - Count requests per fixed time window (e.g., 100/minute)
   - Simple; but bursty at window edges (200 req possible in 1 sec if fall on boundary)

2. **Sliding Log**
   - Store timestamps of each request
   - Precise, but memory-heavy

3. **Sliding Window Counter**
   - Weighted between two adjacent fixed windows
   - Good balance

4. **Token Bucket**
   - Bucket has N tokens, refilled at R tokens/sec
   - Each request consumes 1 token; deny if empty
   - Allows bursts up to bucket size

5. **Leaky Bucket**
   - Requests fill a queue; drain at fixed rate
   - Smooth output; no bursts

### Token bucket in Redis

```
Key: rl:user:123
Fields: { tokens: 10, lastRefill: 1696700000 }

On each request:
  now = current_time
  tokens = min(capacity, tokens + (now - lastRefill) * refill_rate)
  lastRefill = now
  if tokens >= 1:
      tokens -= 1
      allow
  else:
      deny (429 Too Many Requests)
```

Use Redis Lua script for atomicity.

### Where to place
- **API Gateway** — first line of defense
- **Nginx** — `limit_req_zone`
- **App middleware** — Bucket4j (Java), express-rate-limit

---

## 12.11 HLD — File Upload (like Dropbox)

**Requirements:**
- Upload up to 5 GB files
- Resumable
- Secure
- Fast (parallel upload)

**Approach — Presigned URLs + Multipart Upload:**

```
1. Client → API: request upload (size, name)
2. API → S3: initiate multipart upload
3. API → returns list of presigned URLs (one per part)
4. Client → S3: uploads parts in parallel directly (no server bottleneck!)
5. Client → API: complete upload (ETags for each part)
6. API → S3: complete multipart
7. API → DB: save metadata (fileId, name, size, s3Key, ownerId)
8. Async: virus scan, thumbnail gen
```

**Why presigned URLs?**
- Client uploads directly to S3 — bypasses your server (no 5GB through your API!)
- URLs expire (short-lived) for security
- API stays lightweight

**DB schema:**
```sql
CREATE TABLE files (
    id UUID PRIMARY KEY,
    owner_id BIGINT,
    name VARCHAR(255),
    size BIGINT,
    s3_key VARCHAR(500),
    upload_status ENUM('IN_PROGRESS','COMPLETED','FAILED'),
    created_at TIMESTAMP
);
```

**Resumability:** track uploaded part numbers per upload session. Resume by re-requesting URLs for missing parts.

---

## 12.12 Scaling Concepts

### Vertical vs Horizontal

- **Vertical (scale up)** — bigger machine (more CPU/RAM). Limit: hardware max, cost.
- **Horizontal (scale out)** — more machines. Preferred for large systems (cheap commodity hardware, no limit).

### Load Balancing

Distributes traffic across instances.

- **Client-side LB** — client picks instance (from service discovery). No hop.
- **Server-side LB** — LB is a hop (Nginx, HAProxy, AWS ELB).

Algorithms: round-robin, least-connections, IP hash, weighted, geographic.

### DB Scaling

- **Replication (master/slave)** — writes to master, reads from replicas. HA + read scaling.
- **Sharding** — split data across nodes by hash/range/geo. Scales writes. Complexity: cross-shard queries.
- **Federation** — split by function (users DB, orders DB, products DB).

### Consistency Models

- **Strong consistency** — reads always see latest writes (single node, or Raft/Paxos)
- **Eventual consistency** — reads may be stale, but converge (DynamoDB, Cassandra)
- **Causal** — related events seen in order
- **Read-your-writes** — you always see your own writes

### CAP Theorem

In a distributed system with partitions:
- **C**onsistency — every read gets latest write
- **A**vailability — every request gets a (possibly stale) response

Pick one during a partition. **P is not optional** in real distributed systems.

- **CP systems** — MongoDB (primary), HBase, Zookeeper (refuse serving during partition)
- **AP systems** — Cassandra, DynamoDB, Couchbase (serve stale data)

### PACELC
"If **P**artition, choose **A** or **C**; **E**lse (normal), choose **L**atency or **C**onsistency."

### Message Queues — Enable async, decoupling
- Kafka — streaming, high throughput
- RabbitMQ — traditional queue, rich routing
- SQS — AWS managed

Benefits: buffer spikes, decouple services, retry logic, async processing.

---

## 12.13 Common HLD Building Blocks

| Component | Purpose | Tech |
|-----------|---------|------|
| Load balancer | Distribute traffic | AWS ELB, Nginx, HAProxy |
| API Gateway | Auth, routing, rate limit | Kong, AWS API GW, Spring Cloud Gateway |
| App servers | Business logic | K8s pods |
| Cache | Fast reads | Redis, Memcached |
| SQL DB | Transactional | Postgres, MySQL |
| NoSQL DB | Scale/flexibility | MongoDB, Cassandra, DynamoDB |
| Search | Full-text | Elasticsearch, Solr |
| Object storage | Files/media | S3, GCS |
| CDN | Global delivery | CloudFront, Akamai |
| Message queue | Async | Kafka, RabbitMQ, SQS |
| Search engine | Text search | Elasticsearch |
| Analytics | Big data | BigQuery, Redshift, Spark |
| Monitoring | Metrics/logs/traces | Prometheus + Grafana + ELK + Zipkin |

---

## 12.14 System Design Checklist (in interview)

- [ ] Clarify requirements (functional + non-functional)
- [ ] Estimate scale (QPS, storage, bandwidth)
- [ ] Sketch HLD (client → LB → services → cache → DB → queue)
- [ ] DB choice with justification (SQL vs NoSQL)
- [ ] Caching strategy
- [ ] Async processing (queues)
- [ ] Failure handling (retries, circuit breakers, DLQ)
- [ ] Monitoring / observability
- [ ] Security (auth, encryption, rate limit)
- [ ] Trade-offs and alternatives

---

## 12.15 Common Interview Questions

1. **Design a URL shortener.** — see 12.4
2. **Design WhatsApp / chat.** — see 12.8
3. **Design a rate limiter.** — see 12.10
4. **Design a notification service.** — see 12.5
5. **Design an e-commerce checkout.**
6. **Design Netflix / YouTube (streaming).**
7. **Design Uber (ride sharing).**
8. **Design a distributed cache.**
9. **Design an autocomplete/typeahead.**
10. **Design Dropbox / Google Drive.** — see 12.11
11. **Scale a system to 1M/100M users.**
12. **CAP theorem — real examples.**
13. **SQL vs NoSQL — when?**
14. **Eventual consistency — how to reason?**
15. **How to handle failures?** — retries, timeouts, circuit breakers, replicas, backups.
16. **How to improve your app's performance?** — profile first; then indexes, caching, async, DTO projections, connection pool tuning.
17. **Explain SOLID with examples.** — see 12.1
18. **What is horizontal vs vertical scaling?**
19. **What is a CDN?**
20. **How does DNS work?** — Recursive resolver → root → TLD → authoritative NS → IP. Cached at multiple levels.

---

## 12.16 Cheat Sheet

```
LLD approach:
  1. Clarify → 2. Entities → 3. Relationships → 4. SOLID/patterns
  5. Edge cases → 6. Extensibility

HLD approach:
  1. Requirements → 2. Estimate → 3. API → 4. Data model
  5. HLD diagram → 6. Deep dive → 7. Scale/bottleneck

SOLID:
  S — one reason to change
  O — extend, don't modify
  L — subtype substitutable
  I — no fat interfaces
  D — depend on abstractions

Building blocks:
  LB → API GW → Services → Cache → DB / Queue
  Scale horizontally; cache aggressively; async via queues
  CAP: pick C or A during partition
  Async + retries + circuit breakers = resilient
```

---

## Practical Assignments

1. Design a Parking Lot end-to-end with all SOLID applied. Draw the class diagram.
2. Design a URL Shortener; whiteboard the full flow (API, DB, cache, analytics).
3. Design an in-memory LRU cache class (interview classic).
4. Explain how you would scale your current project to 10x traffic.
5. Pick any app you use (Twitter, Instagram) and design its core feed feature.
6. Trace what happens when you type google.com in your browser (DNS → TCP → TLS → HTTP → server → DB → response).

Master these and system design interviews are yours. 🚀