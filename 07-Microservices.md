# 7. Microservices — Deep Dive

> **Goal:** Understand *why* microservices exist, master the essential patterns (Circuit Breaker, Saga, Event-Driven), and design distributed systems confidently.

---

## 7.0 Monolith vs Microservices — The Big Trade-off

### Monolith — one big app

Imagine a **shopping mall** with everything under one roof: groceries, electronics, clothes, food court.

Pros:
- Simple to develop and deploy (one app, one DB)
- Easy transactions (all in same DB)
- Fast internal calls (in-process)
- Simpler debugging (one stack trace)

Cons:
- Whole thing must be redeployed for any change
- Everyone shares the same tech stack
- Scaling means scaling everything (can't scale just cart)
- Slow build/test
- One bug can bring down everything

### Microservices — many small independent apps

Imagine a **shopping street** with many specialized shops, each independently owned.

Pros:
- **Independent deployment** — deploy the cart service without touching payments
- **Independent scaling** — scale hot services separately
- **Team autonomy** — one team per service
- **Fault isolation** — payment failure doesn't crash the whole site
- **Technology diversity** — Java for one, Python for another
- **Faster CI/CD**

Cons:
- **Complex distributed system** — network calls fail, latency exists
- **Distributed transactions** are hard (no more ACID across services)
- **Service discovery** needed
- **Observability** harder (traces span many services)
- **Deployment complexity** (Docker, K8s, service mesh)
- **Data duplication / consistency** challenges

### When to choose?

**Start with a monolith** unless you have a specific reason. Microservices are worth the complexity only when:
- Multiple teams need to deploy independently
- Different scaling needs per component
- Different tech stacks needed
- Very high scale (millions of users)

**Modular monolith** — a middle ground. Well-modularized single app; can split later if needed.

---

## 7.1 Communication — How Services Talk

### Synchronous — REST (HTTP)

Simple, ubiquitous, works everywhere.

#### RestTemplate (legacy, still common)

```java
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Usage
Book book = restTemplate.getForObject(
    "http://book-service/books/{id}", Book.class, 1L);

Book created = restTemplate.postForObject(
    "http://book-service/books", newBook, Book.class);

// With headers / full response
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Bearer " + token);
HttpEntity<Book> req = new HttpEntity<>(book, headers);
ResponseEntity<Book> res = restTemplate.exchange(
    "http://book-service/books/{id}", HttpMethod.PUT, req, Book.class, 1L);
```

Downside: **synchronous, blocking**. If book-service is slow, your thread waits.

#### WebClient — reactive, non-blocking (Spring 5+)

```java
WebClient client = WebClient.builder()
    .baseUrl("http://book-service")
    .defaultHeader("Content-Type", "application/json")
    .build();

// Async
Mono<Book> bookMono = client.get().uri("/books/{id}", 1L)
    .retrieve()
    .bodyToMono(Book.class);

// Blocking (for simple cases)
Book book = bookMono.block();

// POST
Book created = client.post().uri("/books")
    .bodyValue(newBook)
    .retrieve()
    .bodyToMono(Book.class)
    .block();
```

#### Feign Client — declarative HTTP (most popular)

Write an interface — Spring generates the HTTP client.

```java
@FeignClient(name = "book-service", url = "${book-service.url}")
public interface BookClient {

    @GetMapping("/books/{id}")
    Book getBook(@PathVariable("id") Long id);

    @GetMapping("/books")
    List<Book> searchBooks(@RequestParam("title") String title);

    @PostMapping("/books")
    Book create(@RequestBody Book book);
}

// Enable
@SpringBootApplication
@EnableFeignClients
public class App { }

// Use
@Service
public class OrderService {
    private final BookClient bookClient;
    public Order createOrder(Long bookId) {
        Book b = bookClient.getBook(bookId);        // as easy as a method call
        // ...
    }
}
```

### Asynchronous — Messaging (Kafka, RabbitMQ)

Fire-and-forget. Decouples services, better resilience.

```
Producer  →  Message Broker  →  Consumers (many)
```

Perfect for:
- Long-running work (video encoding, email sending)
- Events (`OrderCreated`, `PaymentCompleted`)
- Broadcasting to multiple consumers
- Buffering spikes

Trade-offs:
- Eventual consistency (not immediate)
- More complex infrastructure
- Ordering & duplication challenges

---

## 7.2 Reliability Patterns — Handling Failures

**Network calls fail. Downstream services are slow. Services restart.** You must design for this.

### 7.2.1 Retry

Automatic retry on transient failures.

```java
@Retry(name = "bookService", fallbackMethod = "fallback")
public Book getBook(Long id) {
    return bookClient.getBook(id);
}
```

Application.yml:
```yaml
resilience4j:
  retry:
    instances:
      bookService:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.io.IOException
```

**Best practice:** exponential backoff with jitter:
```
Attempt 1: fail
Wait 500ms + rand(0..100ms)
Attempt 2: fail
Wait 1000ms + rand(0..200ms)
Attempt 3: fail → fallback
```

### 7.2.2 Circuit Breaker — Prevent Cascading Failure

**Analogy:** Just like an electrical circuit breaker trips to protect your house from a fire, a software circuit breaker trips to protect your app from a downstream failure storm.

**States:**

```
     CLOSED (normal) ────failures > threshold────► OPEN
        ▲                                            │
        │                                            │
        │                                    wait duration
        │                                            │
        └──── successes in HALF_OPEN ────── HALF_OPEN
                                                     │
                                    single test call │
                                                     ▼
                                          success? → CLOSED
                                          fail?    → OPEN
```

**Resilience4j:**
```java
@CircuitBreaker(name = "bookService", fallbackMethod = "fallback")
@Retry(name = "bookService")
@TimeLimiter(name = "bookService")
public CompletableFuture<Book> getBook(Long id) {
    return CompletableFuture.supplyAsync(() -> bookClient.getBook(id));
}

public CompletableFuture<Book> fallback(Long id, Throwable t) {
    log.warn("book-service failed, returning default", t);
    return CompletableFuture.completedFuture(Book.defaultBook(id));
}
```

Config:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      bookService:
        slidingWindowSize: 10                # last 10 calls
        failureRateThreshold: 50             # 50% fails → OPEN
        waitDurationInOpenState: 10s          # then try HALF_OPEN
        permittedNumberOfCallsInHalfOpenState: 3
```

### 7.2.3 Rate Limiter

Cap requests per second.

```java
@RateLimiter(name = "bookService")
public Book get(Long id) { ... }
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      bookService:
        limitForPeriod: 100         # 100 requests
        limitRefreshPeriod: 1s      # per second
        timeoutDuration: 500ms
```

### 7.2.4 Bulkhead

Isolate resource pools so one slow service can't consume all threads.

```java
@Bulkhead(name = "bookService", type = Bulkhead.Type.THREADPOOL)
public Book get(Long id) { ... }
```

Analogy: ships have watertight compartments — flooding one doesn't sink the ship.

### 7.2.5 Timeout

Never wait forever.

```java
@TimeLimiter(name = "bookService")
public CompletableFuture<Book> get(Long id) { ... }
```

Combine all patterns:
```
Client → Rate Limiter → Bulkhead → Circuit Breaker → Retry → Timeout → Downstream
```

### 7.2.6 Idempotency

Same request, same result — even if called twice.

**Naturally idempotent:** GET, PUT, DELETE (mostly).
**Not idempotent:** POST (default).

**Making POST idempotent:**
```
POST /payments
Headers: Idempotency-Key: <uuid>
```

Server stores processed keys in Redis. Second call with same key returns the same result.

---

## 7.3 Kafka — The Streaming Champion

### Concepts

- **Topic** — a named stream of messages (like a folder for events)
- **Partition** — subset of a topic (allows parallelism); ordered within a partition
- **Producer** — publishes messages
- **Consumer** — reads messages
- **Consumer Group** — parallel consumers sharing the load
- **Offset** — pointer to position in partition
- **Broker** — a Kafka server node
- **ZooKeeper / KRaft** — cluster coordination

### Visual

```
Producer ─┐
Producer ─┼────► Topic: orders
Producer ─┘        │
                   ├── Partition 0  ─┐
                   ├── Partition 1  ─┼──► Consumer Group A (3 consumers, 1 per partition)
                   └── Partition 2  ─┘

                                    ─┬──► Consumer Group B (email notifications)
                                     └──► Consumer Group C (analytics)
```

Each consumer group reads independently. Same message → many consumers.

### Producer in Spring Boot

```java
@Service
public class OrderPublisher {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafka;

    public void publish(OrderEvent event) {
        kafka.send("orders", event.getOrderId(), event);
    }
}
```

**Key insight:** the key (2nd arg) decides partition. Same key → same partition → preserves order for that key.

### Consumer

```java
@Component
public class EmailNotifier {

    @KafkaListener(topics = "orders", groupId = "email-service")
    public void onOrder(OrderEvent event) {
        emailService.sendConfirmation(event);
    }
}
```

### application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                          # wait for all replicas → durability
      retries: 3
    consumer:
      group-id: email-service
      auto-offset-reset: earliest         # earliest | latest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
```

### Guarantees & Trade-offs

- **At-most-once** — may lose messages (fastest)
- **At-least-once** — may duplicate (default)
- **Exactly-once** — hardest to achieve (needs transactional producer + idempotent consumer)

**Idempotent consumer pattern:**
```java
@KafkaListener(topics = "orders")
public void handle(OrderEvent e) {
    if (processedRepo.existsById(e.getEventId())) return;   // already processed
    processOrder(e);
    processedRepo.save(new ProcessedEvent(e.getEventId()));
}
```

### Dead Letter Queue (DLQ)

After N retries, send poison message to a DLQ topic for inspection.

```java
@Bean
public DeadLetterPublishingRecoverer recoverer(KafkaTemplate<Object,Object> template) {
    return new DeadLetterPublishingRecoverer(template);
}

@Bean
public DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer r) {
    return new DefaultErrorHandler(r, new FixedBackOff(1000L, 3));   // retry 3× with 1s
}
```

---

## 7.4 RabbitMQ — Traditional Message Broker

Uses **AMQP** protocol.

Components:
- **Producer** → **Exchange** → routes to → **Queue(s)** → **Consumer**

Exchange types:
- **Direct** — routes by exact routing key
- **Fanout** — broadcasts to all bound queues
- **Topic** — routes by pattern (e.g., `order.*`)
- **Headers** — routes by header values

Spring AMQP example:
```java
@Bean
public Queue queue() { return new Queue("orders"); }
@Bean
public TopicExchange exchange() { return new TopicExchange("app-exchange"); }
@Bean
public Binding binding(Queue q, TopicExchange e) {
    return BindingBuilder.bind(q).to(e).with("order.*");
}

@Service
public class Producer {
    @Autowired RabbitTemplate template;
    public void send(Order o) {
        template.convertAndSend("app-exchange", "order.created", o);
    }
}

@RabbitListener(queues = "orders")
public void receive(Order o) { }
```

### Kafka vs RabbitMQ

| Kafka | RabbitMQ |
|-------|----------|
| Log-based (retains messages) | Queue-based (deletes after ack) |
| High throughput (millions/sec) | Moderate throughput |
| Complex routing? Not really | Rich routing rules |
| Ordering per partition | Ordering per queue |
| Long retention & replay | Not designed for replay |
| Use: streaming, event sourcing | Use: task queues, complex routing |

---

## 7.5 API Gateway

**Single entry point** for all clients. All external traffic goes through it before reaching internal services.

**Responsibilities:**
- **Routing** — `/api/books/**` → book-service
- **Authentication** — validate JWT once, at edge
- **Rate limiting** — protect backend
- **Aggregation** — combine multiple service calls into one client response
- **SSL termination** — decrypt HTTPS at edge
- **Request/response transformation**
- **Logging & metrics**

```
[Mobile] ─┐
[Web]    ─┼──► [API Gateway] ──► [Auth Service, Book Service, Order Service, ...]
[Partner]─┘
```

### Spring Cloud Gateway

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: book-service
          uri: lb://book-service               # load-balanced via discovery
          predicates:
            - Path=/api/books/**
          filters:
            - StripPrefix=1                     # strips /api
            - name: CircuitBreaker
              args:
                name: bookCB
                fallbackUri: forward:/fallback
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

**Popular gateways:** Spring Cloud Gateway, Kong, AWS API Gateway, Zuul (deprecated).

---

## 7.6 Service Discovery — Finding Services

In dynamic environments, services move around. Hardcoding hostnames fails.

**Pattern:**
1. Services **register** themselves with a discovery server
2. Clients **query** discovery to find a service
3. Health checks remove dead instances

### Eureka (Netflix)

**Server:**
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp { ... }
```

**Client (each service):**
```java
@SpringBootApplication
@EnableDiscoveryClient
public class BookServiceApp { ... }
```

```yaml
spring.application.name: book-service
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka
```

Then, in another service:
```java
@FeignClient(name = "book-service")     // looked up in Eureka, no URL!
public interface BookClient { ... }
```

**Alternatives:** Consul (HashiCorp), Kubernetes DNS (in K8s environments), AWS Cloud Map.

---

## 7.7 Load Balancing

Distribute traffic across service instances.

**Client-side (in-process):**
- Spring Cloud LoadBalancer (successor to Ribbon)
- Client picks an instance from discovery
- No extra hop

**Server-side (dedicated):**
- Nginx, HAProxy, AWS ELB
- Client talks to LB; LB picks instance
- Extra hop, but simpler client

Algorithms: **round-robin**, **least connections**, **IP hash**, **weighted**, **random**.

---

## 7.8 Distributed Transactions & Saga Pattern

### Problem
In monolith: `@Transactional` covers everything. In microservices: each service has its own DB. Can't span a transaction across services!

### 2PC (Two-Phase Commit) — old solution

Coordinator asks all participants:
1. "Can you commit?" (prepare phase)
2. If all say yes → commit; else rollback

**Downsides:** slow, blocking, coordinator is SPOF. Rarely used in microservices.

### Saga Pattern — modern solution

Break the big transaction into a sequence of **local transactions**, each with a **compensating action** if something fails downstream.

**Example — Order flow:**

```
1. Order Service    → CreateOrder(status=pending)
   compensation:      CancelOrder
2. Payment Service  → ChargeCustomer
   compensation:      Refund
3. Inventory Service → ReserveStock
   compensation:      ReleaseStock
4. Shipping Service → CreateShipment
   compensation:      CancelShipment

If step 3 fails: run compensation of 2, then 1.
```

Two implementation styles:

### Choreography — event-driven

Each service reacts to events; no central coordinator.

```
Order Service → publishes OrderCreated
Payment Service listens → charges → publishes PaymentCompleted
Inventory Service listens → reserves → publishes InventoryReserved
Shipping Service listens → creates shipment → done

On any failure → services publish compensation events → previous services react.
```

Pros: loose coupling, simple.
Cons: hard to track full flow, order dependencies implicit.

### Orchestration — central coordinator

An **orchestrator** service directs each step.

```
Orchestrator:
  step 1: call Payment
  if success:
     step 2: call Inventory
     if success:
        step 3: call Shipping
        else: undo Inventory, undo Payment, mark Order failed
     else:
        undo Payment
        mark Order failed
```

Pros: explicit workflow, easier to debug.
Cons: central point (though can be scaled), more infrastructure.

**Popular libraries:** Camunda, Temporal.io, Netflix Conductor, AWS Step Functions.

---

## 7.9 Configuration Management

Configs shouldn't live in each service's git. Use a central config server.

### Spring Cloud Config

**Server:**
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApp { }
```

Configs stored in git (or Vault):
```
config-repo/
├── book-service.yml
├── book-service-prod.yml
├── order-service.yml
└── application.yml       (defaults for all)
```

**Client:**
```yaml
spring:
  application.name: book-service
  config:
    import: configserver:http://config-server:8888
  profiles.active: prod
```

Alternatives: **HashiCorp Vault**, **AWS SSM Parameter Store**, **Kubernetes ConfigMaps/Secrets**, **etcd**.

---

## 7.10 Observability — Logs, Metrics, Traces

The "three pillars":

### 1) Distributed Logging — ELK Stack

```
App logs → Filebeat → Logstash → Elasticsearch → Kibana
```

Include **traceId** and **spanId** in every log for correlation:
```
[book-service][traceId=abc123][spanId=def456] Processing book id=1
```

Alternatives: Splunk, Datadog, Grafana Loki.

### 2) Metrics — Prometheus + Grafana

Actuator exposes `/actuator/prometheus`. Prometheus scrapes. Grafana dashboards visualize.

Key metrics: **RED** (Rate, Errors, Duration) or **USE** (Utilization, Saturation, Errors).

### 3) Distributed Tracing — Zipkin / Jaeger

Each request gets a `traceId`. Each service adds its own `spanId` — creates a call tree.

Spring Cloud Sleuth (now Micrometer Tracing in newer versions) does this automatically:
```
[book-service, traceId=abc123, spanId=x1] Received request
  ↓ HTTP call to author-service
[author-service, traceId=abc123, spanId=y2] Fetching author
  ↓ SQL query
```

Zipkin UI shows the full flow with timings — invaluable for debugging.

---

## 7.11 CAP Theorem — The Distributed Systems Trilemma

In a distributed system, when a network partition happens (P), you must choose between:

- **C**onsistency — every read gets the latest write
- **A**vailability — every request gets a response (may be stale)

You **cannot** have both C and A during a partition. And partitions **will** happen.

Examples:
- **CP** — MongoDB (primary), HBase, Zookeeper → refuse reads/writes on partition
- **AP** — Cassandra, DynamoDB, Couchbase → serve stale data, sync later

**PACELC** — extends CAP:
> If Partition (P), choose Availability (A) or Consistency (C); Else (E), choose Latency (L) or Consistency (C).

---

## 7.12 Domain-Driven Design (DDD) Concepts

Helps decide **service boundaries**.

- **Bounded Context** — a self-contained model with its own language (e.g., "Order" means different things in Sales vs Shipping)
- **Aggregate Root** — the entry point to a cluster of related entities
- **Ubiquitous Language** — shared vocabulary between devs and business
- **Event Storming** — a workshop to discover events and boundaries

**Rule:** one service per bounded context. Don't split too finely (nano-services) — you'll drown in orchestration.

---

## 7.13 12-Factor App Principles

A checklist for cloud-native apps:

1. **Codebase** — one repo per app
2. **Dependencies** — declared explicitly (`pom.xml`, `requirements.txt`)
3. **Config** — stored in env vars, not code
4. **Backing services** — treated as attached resources
5. **Build, release, run** — separate stages
6. **Processes** — stateless; state in DB/cache
7. **Port binding** — self-contained (embed Tomcat)
8. **Concurrency** — scale by process/thread count
9. **Disposability** — fast startup, graceful shutdown
10. **Dev/prod parity** — same tech in all environments
11. **Logs** — treated as event streams (write to stdout)
12. **Admin processes** — run as one-off tasks

---

## 7.14 Complete Microservices Architecture (Reference)

```
                             ┌────────────┐
                             │    Users   │
                             └─────┬──────┘
                                   │
                                   ▼
                           ┌─────────────────┐
                           │  API Gateway    │  ← auth, rate limit, routing
                           └─────┬───────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
   ┌─────────┐             ┌──────────┐            ┌──────────┐
   │ Book Svc│             │ Order Svc│            │Payment Svc│
   └────┬────┘             └────┬─────┘            └────┬─────┘
        │                       │                       │
        ▼                       ▼                       ▼
   [Postgres]              [Postgres]              [Postgres]

                Kafka (topics: orders, payments, notifications)
        ├───────────────►◄─────────────────►◄───────────
        │
        ▼
   ┌────────────────┐
   │Notification Svc│
   └────────────────┘

Infrastructure:
   Eureka / Consul → service discovery
   Spring Cloud Config → configuration
   Zipkin + Sleuth → tracing
   ELK / Loki → logging
   Prometheus + Grafana → metrics
   Jenkins → CI/CD
   K8s + Helm → orchestration
```

---

## 7.15 Interview Question Bank

1. **Monolith vs Microservices — trade-offs?**  
   Monolith: simple, single deploy, transactions easy, hard to scale/change parts. Micro: independent deploy/scale/tech, complex distributed.

2. **How do microservices communicate?**  
   Sync (REST/gRPC) or async (Kafka/RabbitMQ). Sync = simple but coupled; async = loose coupling but eventual consistency.

3. **What is Service Discovery? Why needed?**  
   Registry where services register/find each other. Needed because IPs change (containers, autoscaling). Eureka, Consul, K8s DNS.

4. **What is API Gateway?**  
   Single entry point. Handles auth, routing, rate limiting, aggregation.

5. **Explain Circuit Breaker states.**  
   CLOSED (normal) → OPEN (failing, block calls) → HALF_OPEN (test) → CLOSED or OPEN.

6. **How to handle distributed transactions?**  
   Saga pattern (orchestration or choreography) with compensating actions.

7. **Explain Kafka architecture.**  
   Topic split into partitions (ordered). Producers write, consumers in groups share partitions. Offsets track progress.

8. **How to secure microservices?**  
   JWT/OAuth2 at gateway, mTLS between services, secrets in Vault, no plaintext.

9. **How to trace requests across services?**  
   Traceparent header (W3C). Sleuth/Micrometer + Zipkin/Jaeger.

10. **What is CAP theorem?**  
    In partition, choose Consistency or Availability. Can't have both.

11. **What if a service is down?**  
    Circuit breaker → fallback, retry with backoff, degraded mode, DLQ.

12. **Implementing idempotency in POST?**  
    Client sends Idempotency-Key header; server stores processed keys.

13. **Kafka vs RabbitMQ?**  
    Kafka: high throughput streaming, ordered per partition, log retention. RabbitMQ: queue-based, complex routing, low latency.

14. **How to version microservices APIs?**  
    URI (`/v1/`), header, or content negotiation. URI most common.

15. **Explain Saga pattern.**  
    Sequence of local tx with compensating actions on failure. Choreography (events) or Orchestration (coordinator).

16. **What is Dead Letter Queue?**  
    Where poison messages go after N failed retries. Preserves for manual inspection.

17. **How do you handle configuration in microservices?**  
    Centralized (Spring Cloud Config, Vault). Env vars for secrets.

18. **What is 12-Factor App?**  
    Best practices for cloud-native apps (config in env, stateless, logs as streams, etc.).

19. **What is Bulkhead?**  
    Isolate resource pools so slow service can't drain everything.

20. **How would you migrate a monolith to microservices?**  
    **Strangler Fig pattern**: build new services around the monolith; route traffic incrementally; retire old code as replaced.

---

## 7.16 Cheat Sheet

```
Communication: REST (Feign) sync; Kafka/RabbitMQ async
Reliability: Retry + Circuit Breaker + Timeout + Bulkhead
API Gateway: single entry (routing, auth, rate limit, aggregation)
Service Discovery: Eureka / Consul / K8s DNS
Distributed transactions: Saga (choreography or orchestration)
Kafka: topic → partitions → consumer group → offsets
CAP: pick 2 of C, A, P (P is not optional!)
Idempotency: Idempotency-Key header + processed-keys store
Observability: logs (ELK) + metrics (Prometheus) + tracing (Zipkin)
Config: Spring Cloud Config / Vault, env vars for secrets
```

---

## Practical Assignments

1. Build 2 microservices (Book + Order) that talk via Feign + Eureka.
2. Add Circuit Breaker to Order → Book call; simulate Book down and see fallback.
3. Publish an `OrderCreated` event to Kafka; consume in a Notification service.
4. Implement idempotent POST with `Idempotency-Key`.
5. Set up Zipkin tracing and view a request spanning 2 services.
6. Design a Saga for `Order → Payment → Inventory` with compensations.

Master these and microservices interviews are yours. 🚀