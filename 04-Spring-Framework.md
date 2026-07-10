# 4. Spring Framework & Spring Boot — Deep Dive

> **Goal:** By the end of this, you'll understand *why* Spring exists, how IoC works internally, and be able to build any REST service with confidence.

---

## 4.0 Why Spring Was Created

### The problem before Spring (early 2000s)

Java web apps were dominated by **EJB (Enterprise JavaBeans)**:
- Heavy XML configuration
- Complex deployment (application servers like WebLogic, WebSphere)
- Tight coupling between business logic and infrastructure
- Hard to test — no way to run components without the whole container

### Rod Johnson's insight (Spring, 2003)

> "Business logic shouldn't depend on the framework or the infrastructure."

Spring provides:
- **Inversion of Control (IoC)** — framework creates and manages objects
- **Dependency Injection (DI)** — wires them together
- **AOP** — cross-cutting concerns (logging, transactions, security)
- **Modules** for every layer: MVC, Data, Security, Batch, Cloud, etc.

### Spring Boot (2014)

Solves Spring's biggest complaint: too much config. Adds:
- **Auto-configuration** — sensible defaults
- **Starter dependencies** — bundled sets of libraries
- **Embedded server** — Tomcat/Jetty inside your JAR
- **Production ready** — Actuator, metrics, health

---

## 4.1 Inversion of Control (IoC) — The Core Idea

### Traditional flow (without IoC)

```java
class OrderService {
    private OrderRepository repo = new OrderRepositoryImpl();  // hard-wired
    private EmailService email = new EmailServiceImpl();       // hard-wired

    public void checkout(Order o) {
        repo.save(o);
        email.send(o.getUser().getEmail(), "Order confirmed");
    }
}
```

**Problems:**
- OrderService is glued to specific implementations
- Hard to test (can't swap Email for a mock)
- Hard to change (want to switch to Kafka event? Rewrite it.)

### With IoC (Spring)

```java
@Service
class OrderService {
    private final OrderRepository repo;
    private final EmailService email;

    public OrderService(OrderRepository repo, EmailService email) {
        this.repo = repo;                    // Spring gives them to us
        this.email = email;
    }

    public void checkout(Order o) {
        repo.save(o);
        email.send(o.getUser().getEmail(), "Order confirmed");
    }
}
```

**Now:**
- OrderService doesn't know or care which implementation it gets
- Test with mocks: `new OrderService(mockRepo, mockEmail)`
- Swap impls in one place (a config bean)

### Analogy

- **Without IoC:** You cook by going shopping, buying ingredients, cooking. You control everything.
- **With IoC:** You go to a restaurant and order. The restaurant (Spring) fetches ingredients and prepares the meal. You just consume.

---

## 4.2 Dependency Injection — 3 Types

### 1) Constructor Injection (recommended ✅)

```java
@Service
public class BookService {
    private final BookRepository repo;

    public BookService(BookRepository repo) {   // Spring auto-wires
        this.repo = repo;
    }
}
```

**Why best?**
- Fields can be `final` → immutable
- Required dependencies enforced at compile time
- Easy to test (just pass mocks)
- No circular dependency hidden bugs

### 2) Setter Injection — for optional deps

```java
@Service
public class BookService {
    private BookRepository repo;

    @Autowired
    public void setRepo(BookRepository repo) { this.repo = repo; }
}
```

### 3) Field Injection — concise but avoid ❌

```java
@Service
public class BookService {
    @Autowired
    private BookRepository repo;
}
```

**Downsides:**
- Can't make field `final`
- Hard to test outside Spring
- Hides dependency count (may be tempting to have 20 fields)
- Depends on reflection

### Spring's evolution

Since Spring 4.3, if a class has one constructor, `@Autowired` is optional. Since 5.x, constructor injection is officially recommended.

---

## 4.3 Beans — Objects Managed by Spring

A **bean** is any object created and managed by the Spring container.

### Ways to declare a bean

**1) Annotations on classes** (auto-detected by component scan)
- `@Component` — generic
- `@Service` — business logic
- `@Repository` — data access (adds exception translation)
- `@Controller` / `@RestController` — MVC

**2) Java config `@Bean` methods** (for 3rd-party classes)
```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper().findAndRegisterModules();
    }
}
```

**3) XML** (legacy — you'll rarely see this in new projects)
```xml
<bean id="bookService" class="com.example.BookService"/>
```

### `@Component` vs `@Bean`

| `@Component` | `@Bean` |
|--------------|---------|
| On class | On method (inside `@Configuration`) |
| For classes YOU write | For 3rd-party / configurable objects |
| Auto-scanned | Explicit |

---

## 4.4 Bean Lifecycle — The Journey of a Bean

```
1. Instantiation
   └─ Spring calls the constructor
2. Populate properties
   └─ @Autowired fields set
3. Aware callbacks
   ├─ BeanNameAware.setBeanName
   ├─ BeanFactoryAware
   └─ ApplicationContextAware
4. BeanPostProcessor.postProcessBeforeInitialization
5. Initialization
   ├─ @PostConstruct   ← common
   ├─ InitializingBean.afterPropertiesSet
   └─ Custom init-method
6. BeanPostProcessor.postProcessAfterInitialization
7. Bean is READY (returned by getBean)
   ...
8. Container shutdown
   ├─ @PreDestroy      ← common
   ├─ DisposableBean.destroy
   └─ Custom destroy-method
```

Practical:
```java
@Component
public class MyBean {
    @PostConstruct
    public void init() {
        System.out.println("Bean ready — DB pool warmed up");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Bean shutting down — closing connections");
    }
}
```

Interview: Why not use the constructor for init logic?  
→ Because `@Autowired` fields aren't set until *after* the constructor runs. `@PostConstruct` runs after DI is complete.

---

## 4.5 Bean Scopes

By default, all beans are **singleton** (one instance per container).

| Scope | When new instance? |
|-------|---------------------|
| **singleton** (default) | Once per container |
| **prototype** | Every injection or `getBean` call |
| **request** (web) | Per HTTP request |
| **session** (web) | Per HTTP session |
| **application** (web) | Per ServletContext |
| **websocket** | Per WebSocket session |

```java
@Component
@Scope("prototype")
public class RequestHandler { }
```

**Interview trap:** Injecting a prototype bean into a singleton — you get the same prototype instance every time. Use `ObjectFactory<T>`, `Provider<T>`, or `@Lookup` for a fresh instance each call.

```java
@Service
public class Service {
    @Autowired ObjectFactory<RequestHandler> handlerFactory;
    public void handle() {
        RequestHandler h = handlerFactory.getObject();  // new instance
    }
}
```

---

## 4.6 @Qualifier and @Primary — Multiple Beans of Same Type

```java
public interface Payment { void pay(double amt); }

@Service("stripe")
public class StripePayment implements Payment { ... }

@Service("paypal")
public class PaypalPayment implements Payment { ... }

// Now Spring is confused: which one to inject?
```

**Solution 1: `@Qualifier`**
```java
@Service
public class CheckoutService {
    private final Payment payment;
    public CheckoutService(@Qualifier("stripe") Payment payment) {
        this.payment = payment;
    }
}
```

**Solution 2: `@Primary`** — mark one as default
```java
@Service
@Primary
public class StripePayment implements Payment { ... }
```

**Solution 3: inject all as a list**
```java
public CheckoutService(List<Payment> allPayments) { ... }
```

---

## 4.7 Configuration & Profiles — Environment-Specific Beans

### Profiles

Different beans per environment (`dev`, `staging`, `prod`).

```java
@Component
@Profile("dev")
public class InMemoryUserRepo implements UserRepo { }

@Component
@Profile("prod")
public class PostgresUserRepo implements UserRepo { }
```

Activate:
```properties
spring.profiles.active=dev
```
Or CLI:
```bash
java -jar app.jar --spring.profiles.active=prod
```
Or env var:
```bash
export SPRING_PROFILES_ACTIVE=prod
```

### Property injection

```java
@Value("${server.port}")
private int port;

@Value("${app.retry-count:3}")   // default value if missing
private int retry;

@Value("#{systemProperties['user.home']}")   // SpEL expression
private String home;
```

### Type-safe config with `@ConfigurationProperties`

**application.yml:**
```yaml
app:
  name: BookApp
  version: 1.0
  cache:
    ttl-seconds: 300
    max-size: 1000
```

**Java:**
```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String version;
    private Cache cache = new Cache();

    public static class Cache {
        private int ttlSeconds;
        private int maxSize;
        // getters/setters
    }
    // getters/setters
}
```

Bonus with Boot 2.2+:
```java
@ConfigurationProperties(prefix = "app")
@ConstructorBinding
public record AppProperties(String name, String version, Cache cache) {
    public record Cache(int ttlSeconds, int maxSize) {}
}
```

---

## 4.8 Spring Boot — Magic Made Explicit

### `@SpringBootApplication` breakdown

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

Equivalent to:
```java
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class MyApp { }
```

- `@Configuration` — this class defines beans (via `@Bean` methods).
- `@EnableAutoConfiguration` — turn on auto-configuration magic.
- `@ComponentScan` — find all `@Component`, `@Service`, etc. in this package and sub-packages.

### How auto-configuration works

Behind the scenes:
1. Boot scans classpath for **starter jars** (`spring-boot-starter-web`, etc.)
2. For each starter, checks `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3) — a list of `@Configuration` classes
3. Each config class is annotated with **`@Conditional…`** to only activate under specific conditions:
   - `@ConditionalOnClass(DataSource.class)` — only if `DataSource` is on classpath
   - `@ConditionalOnMissingBean` — only if you haven't provided your own
   - `@ConditionalOnProperty("app.feature.x=true")`

So if you add `spring-boot-starter-data-jpa`, Spring Boot sees `EntityManager` on classpath → auto-configures a DataSource, EntityManagerFactory, TransactionManager, etc.

### Starter dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

`starter-web` transitively pulls in:
- Spring MVC
- Jackson (JSON)
- Embedded Tomcat
- Validation
- Logging (Logback)

You don't manage versions — Spring Boot BOM does it.

### Common starters

- `spring-boot-starter-web` — REST APIs
- `spring-boot-starter-data-jpa` — Hibernate + Spring Data
- `spring-boot-starter-security` — Spring Security
- `spring-boot-starter-actuator` — production monitoring
- `spring-boot-starter-test` — JUnit + Mockito + AssertJ + MockMvc
- `spring-boot-starter-validation` — Bean Validation

---

## 4.9 REST APIs — The Complete Guide

### REST principles (recap)

1. **Client-Server**
2. **Stateless** — no session on server (each request self-contained)
3. **Cacheable**
4. **Uniform Interface** — URIs + HTTP verbs
5. **Layered**

### Resource naming

**Good:**
```
GET    /api/books              (list)
GET    /api/books/1            (single)
POST   /api/books              (create)
PUT    /api/books/1            (full update)
PATCH  /api/books/1            (partial update)
DELETE /api/books/1            (delete)
GET    /api/books/1/reviews    (sub-resource)
```

**Bad:**
```
GET /api/getAllBooks
POST /api/deleteBook?id=1
```

### Complete Controller Example

```java
@RestController
@RequestMapping("/api/books")
@Validated
public class BookController {

    private final BookService service;

    public BookController(BookService service) { this.service = service; }

    // GET /api/books?page=0&size=10&sort=title,asc
    @GetMapping
    public Page<BookDto> list(@PageableDefault(size = 20) Pageable pageable) {
        return service.findAll(pageable);
    }

    // GET /api/books/1
    @GetMapping("/{id}")
    public ResponseEntity<BookDto> byId(@PathVariable Long id) {
        return service.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // GET /api/books/search?title=Java
    @GetMapping("/search")
    public List<BookDto> search(@RequestParam String title,
                                @RequestParam(defaultValue = "0") @Min(0) int minPrice) {
        return service.search(title, minPrice);
    }

    // POST /api/books
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public BookDto create(@Valid @RequestBody CreateBookRequest req) {
        return service.create(req);
    }

    // PUT /api/books/1
    @PutMapping("/{id}")
    public BookDto update(@PathVariable Long id,
                          @Valid @RequestBody UpdateBookRequest req) {
        return service.update(id, req);
    }

    // PATCH /api/books/1
    @PatchMapping("/{id}")
    public BookDto patch(@PathVariable Long id,
                         @RequestBody Map<String, Object> updates) {
        return service.patch(id, updates);
    }

    // DELETE /api/books/1
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        service.delete(id);
    }

    // Handle domain exception locally
    @ExceptionHandler(BookNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(BookNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("BOOK_NOT_FOUND", ex.getMessage()));
    }
}
```

### Annotations reference

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@PathVariable` | URL segment | `/books/{id}` |
| `@RequestParam` | Query param | `?title=x` |
| `@RequestBody` | JSON body → object | POST/PUT body |
| `@RequestHeader` | Header value | `Authorization` |
| `@ResponseStatus` | Set status code | `HttpStatus.CREATED` |
| `@CookieValue` | Cookie value | – |

### ResponseEntity — full control

```java
return ResponseEntity
    .status(HttpStatus.CREATED)
    .header("Location", "/api/books/" + book.getId())
    .header("X-App-Version", "1.0")
    .body(book);
```

### HTTP status codes cheat sheet

| Category | Common codes |
|----------|--------------|
| 2xx Success | 200 OK, 201 Created, 204 No Content |
| 3xx Redirect | 301 Moved, 304 Not Modified |
| 4xx Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| 5xx Server error | 500 Internal, 502 Bad Gateway, 503 Service Unavailable |

### Pagination the Spring Data way

```java
@GetMapping
public Page<Book> list(Pageable pageable) {
    return repo.findAll(pageable);
}
// GET /books?page=0&size=10&sort=title,asc&sort=price,desc
```

Response:
```json
{
  "content": [ ... ],
  "totalElements": 100,
  "totalPages": 10,
  "number": 0,
  "size": 10
}
```

### API Versioning approaches

1. **URI:** `/api/v1/books` — simplest, most common
2. **Header:** `X-API-Version: 1`
3. **Query param:** `?version=1`
4. **Content negotiation:** `Accept: application/vnd.myapp.v1+json`

---

## 4.10 Validation — Bean Validation (JSR-380)

Add: `spring-boot-starter-validation`

### DTO with constraints

```java
public record CreateBookRequest(
    @NotBlank                            String title,
    @NotBlank @Size(min=3,max=50)        String author,
    @Min(0) @Max(1_000_000)              double price,
    @Pattern(regexp="\\d{13}")           String isbn,
    @Email                               String contactEmail,
    @NotNull @Future                     LocalDate publishDate,
    @NotEmpty                            List<@NotBlank String> tags
) {}
```

Controller:
```java
@PostMapping
public Book create(@Valid @RequestBody CreateBookRequest req) { ... }
```

### Common built-in constraints

| Annotation | Purpose |
|------------|---------|
| `@NotNull` | must not be null |
| `@NotEmpty` | not null + not empty (String/collection) |
| `@NotBlank` | not null + not just whitespace (String) |
| `@Size(min=,max=)` | length range |
| `@Min` / `@Max` | numeric bounds |
| `@Positive` / `@PositiveOrZero` | numeric > 0 / ≥ 0 |
| `@Email` | email format |
| `@Pattern(regexp=)` | regex |
| `@Past` / `@Future` | date bounds |

### Custom validator

```java
@Constraint(validatedBy = IsbnValidator.class)
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidIsbn {
    String message() default "Invalid ISBN";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class IsbnValidator implements ConstraintValidator<ValidIsbn, String> {
    public boolean isValid(String v, ConstraintValidatorContext ctx) {
        return v != null && v.matches("\\d{13}");
    }
}

// Usage
public record CreateBookRequest(@ValidIsbn String isbn) { }
```

### Path/param validation

```java
@GetMapping("/{id}")
public Book byId(@PathVariable @Min(1) Long id) { ... }
```

Requires class-level `@Validated`.

---

## 4.11 Exception Handling — Global with `@RestControllerAdvice`

Instead of `try-catch` in every controller, handle exceptions globally.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BookNotFoundException.class)
    public ResponseEntity<ErrorResponse> notFound(BookNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("BOOK_NOT_FOUND", ex.getMessage()));
    }

    // Validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String,String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        return ResponseEntity.badRequest()
            .body(new ValidationErrorResponse("VALIDATION_FAILED", errors));
    }

    // Constraint violations (path/param)
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraint(ConstraintViolationException ex) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("INVALID_PARAM", ex.getMessage()));
    }

    // Unknown exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception at {}", req.getRequestURI(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
    }
}

public record ErrorResponse(String code, String message, Instant timestamp) {
    public ErrorResponse(String code, String message) {
        this(code, message, Instant.now());
    }
}
```

### Best practice — standardized error format

```json
{
  "code": "BOOK_NOT_FOUND",
  "message": "Book with id 42 not found",
  "timestamp": "2025-10-07T14:00:00Z",
  "path": "/api/books/42",
  "traceId": "abc123"
}
```

---

## 4.12 Spring Security — Basics

Add: `spring-boot-starter-security`

By default, Spring Security:
- Adds a login form
- Generates a random password (in logs)
- Requires authentication on all endpoints

### Modern config (Spring Security 6)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                       // for APIs
            .cors(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/books/**").hasAuthority("BOOK_WRITE")
                .anyRequest().authenticated())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### JWT — the modern REST auth

**Flow:**
```
1. Client POSTs credentials → /auth/login
2. Server validates, returns JWT token
3. Client sends: Authorization: Bearer <token>  on every subsequent request
4. Server validates signature + expiration, extracts user info from claims
```

**JWT structure:** `header.payload.signature`
- Header: `{"alg":"HS256","typ":"JWT"}`
- Payload: `{"sub":"user123","roles":["USER"],"exp":1700000000}`
- Signature: HMAC or RSA sign

**Generate a token:**
```java
String token = Jwts.builder()
    .setSubject(user.getUsername())
    .claim("roles", user.getRoles())
    .setIssuedAt(new Date())
    .setExpiration(new Date(System.currentTimeMillis() + 3600_000))
    .signWith(SignatureAlgorithm.HS256, secretKey)
    .compact();
```

**Verify in a filter:**
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                Claims claims = Jwts.parser().setSigningKey(secret)
                    .parseClaimsJws(token).getBody();
                String user = claims.getSubject();
                List<String> roles = claims.get("roles", List.class);
                var auth = new UsernamePasswordAuthenticationToken(
                    user, null, roles.stream().map(SimpleGrantedAuthority::new).toList());
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException e) { /* invalid token — do nothing */ }
        }
        chain.doFilter(req, res);
    }
}
```

### Method-level security

```java
@EnableMethodSecurity                // enable

@Service
public class BookService {
    @PreAuthorize("hasRole('ADMIN')")
    public void delete(Long id) { ... }

    @PreAuthorize("#book.ownerId == authentication.principal.id")
    public void update(Book book) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Book get(Long id) { ... }
}
```

### API security best practices

1. **HTTPS everywhere**
2. **Never log tokens/passwords**
3. **Short-lived access token + refresh token**
4. **Input validation + parameterized queries** (prevent SQL injection)
5. **Rate limiting** (per user/IP)
6. **CORS** — configure explicitly:
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://myapp.com"));
    cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    cfg.setAllowedHeaders(List.of("*"));
    cfg.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", cfg);
    return src;
}
```
7. **Store passwords hashed** (BCrypt/Argon2)
8. **CSRF for browser forms** (not needed for stateless APIs with tokens)

---

## 4.13 Spring Boot Actuator — Production Monitoring

Add: `spring-boot-starter-actuator`

Enable endpoints:
```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,env,beans,loggers,httptrace
management.endpoint.health.show-details=when_authorized
management.info.env.enabled=true
info.app.name=BookApp
info.app.version=1.0
```

**Key endpoints:**

| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Overall health (UP/DOWN) with sub-components (DB, disk, etc.) |
| `/actuator/info` | App info |
| `/actuator/metrics` | JVM & app metrics (memory, GC, HTTP request counts) |
| `/actuator/env` | Environment variables & properties |
| `/actuator/beans` | List of all beans |
| `/actuator/loggers` | View & change log levels at runtime |
| `/actuator/prometheus` | Prometheus scrape endpoint |
| `/actuator/threaddump` | JVM threads |
| `/actuator/heapdump` | Heap dump |
| `/actuator/mappings` | HTTP endpoint mappings |

### Custom health check

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    public Health health() {
        try {
            // ping external API
            return Health.up().withDetail("api", "reachable").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

### Custom metrics

```java
@Service
public class OrderService {
    private final Counter ordersCreated;

    public OrderService(MeterRegistry registry) {
        this.ordersCreated = registry.counter("orders.created");
    }

    public void create(Order o) {
        // ...
        ordersCreated.increment();
    }
}
```

---

## 4.14 Testing Spring — Quick Overview

(Full details in `09-Testing.md`)

```java
// Full integration test
@SpringBootTest
class ApplicationTests { @Test void contextLoads() { } }

// Controller layer only
@WebMvcTest(BookController.class)
class BookControllerTest {
    @Autowired MockMvc mvc;
    @MockBean BookService service;
    // ...
}

// Repository layer only
@DataJpaTest
class BookRepositoryTest { }

// Service layer — plain Mockito
@ExtendWith(MockitoExtension.class)
class BookServiceTest {
    @Mock BookRepository repo;
    @InjectMocks BookService service;
}
```

---

## 4.15 Layered Architecture — Standard Structure

```
com.example.bookapp
├── BookApplication.java          @SpringBootApplication
├── controller/                    (Web layer)
│   └── BookController.java        @RestController
├── service/                       (Business layer)
│   ├── BookService.java           interface (optional)
│   └── impl/BookServiceImpl.java  @Service
├── repository/                    (Data layer)
│   └── BookRepository.java        extends JpaRepository
├── entity/                        (Persistence models)
│   └── Book.java                  @Entity
├── dto/                           (Data Transfer Objects)
│   ├── BookDto.java
│   ├── CreateBookRequest.java
│   └── UpdateBookRequest.java
├── mapper/                        (Entity ↔ DTO)
│   └── BookMapper.java            MapStruct or manual
├── exception/                     (Custom exceptions + GlobalExceptionHandler)
├── config/                        (@Configuration classes)
└── security/                      (Security config, JWT filter)
```

**Why DTOs and not expose entities?**
- Entities have JPA lazy proxies (LazyInitializationException outside session)
- Entities may have fields you don't want to expose (passwords)
- API contract shouldn't change every time you refactor the DB

---

## 4.16 Real Mini Project — Full CRUD Book Service

You already have this in your `bookapp` project. Here's the essential structure:

**Entity:**
```java
@Entity
@Table(name = "books")
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private String title;
    private String author;
    private double price;
    // getters/setters/ctors
}
```

**Repository:**
```java
public interface BookRepository extends JpaRepository<Book, Long> {
    List<Book> findByTitleContainingIgnoreCase(String kw);
}
```

**Service:**
```java
@Service
@Transactional
public class BookService {
    private final BookRepository repo;
    public BookService(BookRepository repo) { this.repo = repo; }

    public List<Book> findAll() { return repo.findAll(); }
    public Optional<Book> findById(Long id) { return repo.findById(id); }
    public Book save(Book b) { return repo.save(b); }
    public void delete(Long id) {
        if (!repo.existsById(id)) throw new BookNotFoundException(id);
        repo.deleteById(id);
    }
}
```

**Controller:** (see 4.9)

**Exception:**
```java
public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(Long id) { super("Book " + id + " not found"); }
}
```

**GlobalExceptionHandler:** (see 4.11)

Run it:
```bash
mvn spring-boot:run
```
Test:
```bash
curl http://localhost:8080/api/books
curl -X POST http://localhost:8080/api/books \
     -H "Content-Type: application/json" \
     -d '{"title":"Java","author":"X","price":499}'
```

---

## 4.17 Interview Question Bank

1. **What is IoC and DI?**  
   IoC: framework controls object creation/lifecycle instead of your code. DI: providing an object's dependencies from outside.

2. **Constructor vs setter vs field injection — which is best?**  
   Constructor — makes fields final, immutable, forces required dependencies, easiest to test.

3. **`@Component` vs `@Service` vs `@Repository` vs `@Controller`?**  
   All create beans; semantic markers. `@Repository` also translates persistence exceptions.

4. **`@Component` vs `@Bean`?**  
   `@Component` on class (for your code); `@Bean` in `@Configuration` method (for 3rd-party).

5. **Bean scopes?**  
   Singleton (default), prototype, request, session, application, websocket.

6. **Bean lifecycle?**  
   Instantiate → populate → aware callbacks → BPP before → init callbacks (`@PostConstruct`) → BPP after → ready → destroy (`@PreDestroy`).

7. **How does `@SpringBootApplication` work?**  
   Meta-annotation: `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.

8. **How does auto-configuration work?**  
   Boot scans starter jars, reads `AutoConfiguration.imports`, each `@Configuration` uses `@Conditional…` to activate only when appropriate.

9. **What is `@Autowired`? Optional?**  
   Injects dependencies. Since Spring 4.3 optional on single-constructor classes.

10. **Difference between `@RestController` and `@Controller`?**  
    `@RestController` = `@Controller` + `@ResponseBody` → returns JSON directly.

11. **How to secure REST APIs?**  
    JWT/OAuth2, HTTPS, `@PreAuthorize`, input validation, rate limiting, CORS.

12. **What is `@Transactional`?**  
    Wraps method in DB transaction. Rollback on runtime exception.

13. **Global exception handling?**  
    `@RestControllerAdvice` + `@ExceptionHandler` methods.

14. **What is Actuator?**  
    Production endpoints (health, metrics, info, etc.) for monitoring.

15. **What are Spring Profiles?**  
    Env-specific bean groups. `@Profile("prod")`, activate via `spring.profiles.active`.

16. **Circular dependency in Spring — how to resolve?**  
    Prefer constructor injection redesign; use setter injection or `@Lazy` if unavoidable.

17. **What are the layers of a typical Spring Boot app?**  
    Controller → Service → Repository → DB. Plus DTOs, exceptions, config.

18. **What is Spring AOP?**  
    Aspect-Oriented Programming — cross-cutting concerns (logging, tx, security). Uses proxies.

19. **Filter vs Interceptor?**  
    Filter — servlet-level, runs for all requests. Interceptor — Spring MVC-level, only for controller requests.

20. **Explain `@RequestBody` vs `@RequestParam` vs `@PathVariable`.**  
    Body (JSON) vs query param vs URL segment.

---

## 4.18 Cheat Sheet

```
@SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
Constructor injection > setter > field
Default scope: singleton
@RestController = @Controller + @ResponseBody
Use DTOs, not entities, in API contracts
Global errors: @RestControllerAdvice + @ExceptionHandler
Validate with @Valid + JSR-380 annotations
Auth: JWT filter + Spring Security stateless config
Monitor: Actuator (/health, /metrics, /prometheus)
Profiles: @Profile("prod"), spring.profiles.active=prod
```

---

## Practical Assignments

1. Build a `PaymentService` with 3 strategy implementations (`Stripe`, `Paypal`, `Cash`) using `@Qualifier` and `@Primary`.
2. Create a `@RestControllerAdvice` that returns a standardized error object with traceId.
3. Set up JWT authentication for a REST endpoint from scratch.
4. Add a custom Actuator health check for an external dependency.
5. Enable pagination and sorting on a repository via `Pageable`.
6. Refactor a controller that returns entities to use DTOs with MapStruct.

Master these and Spring interviews are yours! 🚀