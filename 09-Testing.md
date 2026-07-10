# 9. Testing — Deep Dive

> **Goal:** Learn the testing pyramid, master JUnit 5 & Mockito, know when to use each Spring Boot test slice, and write tests that give real confidence.

---

## 9.0 Why Test?

- **Catch bugs early** — cheaper to fix at dev time than in production
- **Regression protection** — don't accidentally break working features
- **Documentation** — tests show how the code is meant to be used
- **Enable refactoring** — you can rewrite code confidently if tests pass
- **Design pressure** — hard-to-test code is often poorly designed

### The Testing Pyramid

```
              /\
             /  \    E2E (UI/Playwright) — few, slow, brittle
            /----\
           /      \  Integration (real DB, real services) — some
          /--------\
         /   Unit   \ Unit (Mockito, no external deps) — many, fast
        /____________\
```

**Rule:** many fast unit tests, fewer integration tests, minimal E2E.

### F.I.R.S.T principles

- **F**ast
- **I**ndependent (no shared state)
- **R**epeatable (same result every time)
- **S**elf-validating (pass/fail, no manual inspection)
- **T**imely (written just before or with the code)

---

## 9.1 JUnit 5 — The Modern Standard

### Setup (Spring Boot)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Includes: JUnit 5, Mockito, AssertJ, Hamcrest, JsonPath, MockMvc.

### Anatomy — AAA pattern

**A**rrange → **A**ct → **A**ssert

```java
class CalculatorTest {

    @Test
    void add_shouldReturnSum() {
        // Arrange
        Calculator calc = new Calculator();

        // Act
        int result = calc.add(2, 3);

        // Assert
        assertEquals(5, result);
    }
}
```

### Assertions

**JUnit assertions:**
```java
assertEquals(expected, actual);
assertNotEquals(x, y);
assertTrue(x > 0);
assertFalse(list.isEmpty());
assertNull(obj);
assertNotNull(obj);
assertSame(a, b);                      // reference equality
assertArrayEquals(new int[]{1,2}, arr);

// Grouped — reports ALL failures
assertAll("book",
    () -> assertEquals("Java", book.getTitle()),
    () -> assertEquals(500, book.getPrice()),
    () -> assertNotNull(book.getAuthor())
);

// Exception assertion
IllegalArgumentException ex = assertThrows(
    IllegalArgumentException.class,
    () -> service.deposit(-100)
);
assertEquals("Amount must be positive", ex.getMessage());
```

**AssertJ — fluent, preferred:**
```java
import static org.assertj.core.api.Assertions.*;

assertThat(result).isEqualTo(5);
assertThat(name).isEqualTo("Alice").isNotBlank().startsWith("A");
assertThat(list).hasSize(3).contains("Java", "Python").doesNotContain("Ruby");
assertThat(map).containsEntry("key", "value").hasSize(2);
assertThat(book.getPrice()).isBetween(100.0, 1000.0);
assertThat(exception).isInstanceOf(BookNotFoundException.class)
                     .hasMessage("Book 1 not found");
assertThatThrownBy(() -> service.deposit(-100))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("positive");
```

AssertJ chains beautifully — prefer it for readability.

### Test Lifecycle

```java
class MyTest {

    @BeforeAll static void beforeAll() {}       // once before all tests
    @BeforeEach void beforeEach() {}             // before each test
    @Test void test1() {}
    @Test void test2() {}
    @AfterEach void afterEach() {}               // after each test
    @AfterAll static void afterAll() {}          // once after all
}
```

Each `@Test` method gets a **fresh instance** of the test class by default.

### Test naming — communicate intent

Three common conventions:

- `methodName_scenario_expectedResult()` — `deposit_negativeAmount_throwsException()`
- `should_expected_when_scenario()` — `shouldThrowException_whenDepositIsNegative()`
- `given_when_then()` — `givenNegativeAmount_whenDeposit_thenThrow()`

Pick one and be consistent.

### Displaying friendly names

```java
@Test
@DisplayName("Deposit should reject negative amounts")
void depositRejectsNegative() { }
```

### Disabling tests

```java
@Disabled("Fix after Kafka upgrade — TICKET-1234")
@Test
void broken() { }

@DisabledOnOs(OS.WINDOWS)                                // conditional
@EnabledOnJre(JRE.JAVA_17)
@EnabledIf("java.version.startsWith('17')")
```

### Parameterized tests

Run the same test with different inputs:

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
void isPositive(int n) {
    assertTrue(n > 0);
}

@ParameterizedTest
@CsvSource({
    "1, 2, 3",
    "5, 5, 10",
    "-5, 10, 5",
    "0, 0, 0"
})
void add(int a, int b, int expected) {
    assertEquals(expected, calc.add(a, b));
}

@ParameterizedTest
@MethodSource("bookProvider")
void testBook(Book b) {
    assertNotNull(b.getTitle());
}

static Stream<Book> bookProvider() {
    return Stream.of(new Book("A", 100), new Book("B", 200));
}

@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {" ", "\t"})
void blankStrings(String s) {
    assertTrue(s == null || s.isBlank());
}
```

### Nested tests — organize by scenario

```java
@DisplayName("BookService")
class BookServiceTest {

    @Nested
    @DisplayName("when book exists")
    class WhenBookExists {
        @BeforeEach void setup() { /* stub found */ }
        @Test void findByIdReturnsIt() { }
        @Test void deleteRemovesIt() { }
    }

    @Nested
    @DisplayName("when book missing")
    class WhenBookMissing {
        @BeforeEach void setup() { /* stub empty */ }
        @Test void findByIdThrows() { }
        @Test void deleteThrows() { }
    }
}
```

### Timeouts & Repetition

```java
@Test
@Timeout(value = 500, unit = TimeUnit.MILLISECONDS)
void mustBeFast() { }

@RepeatedTest(5)
void flaky() { }        // for detecting non-determinism
```

---

## 9.2 Mockito — Isolating What You Test

### Why mock?

A unit test should test **one** unit (class/method) in isolation. But your class depends on others (repository, HTTP client, email sender). **Mocks** replace those dependencies with fakes you fully control.

### Core concepts

- **Mock** — empty fake; you tell it how to respond
- **Spy** — real object; you can selectively override methods
- **Stub** — telling a mock what to return
- **Verify** — asserting an interaction happened

### Setup

```java
@ExtendWith(MockitoExtension.class)     // auto-init @Mock fields
class BookServiceTest {

    @Mock
    BookRepository repo;                 // fake

    @InjectMocks
    BookService service;                 // real, with repo injected
}
```

Or manually:
```java
BookRepository repo = mock(BookRepository.class);
BookService service = new BookService(repo);
```

### Stubbing — teach the mock

```java
// Return specific value
when(repo.findById(1L)).thenReturn(Optional.of(new Book("Java")));

// Throw exception
when(repo.findById(999L)).thenThrow(new EntityNotFoundException());

// Different result on subsequent calls
when(repo.findAll())
    .thenReturn(List.of(bookA))
    .thenReturn(List.of(bookA, bookB));

// Answer with dynamic logic
when(repo.save(any())).thenAnswer(invocation -> {
    Book b = invocation.getArgument(0);
    b.setId(42L);
    return b;
});
```

### Argument matchers

```java
when(repo.findByTitle(anyString())).thenReturn(list);
when(service.method(eq("A"), any(Book.class))).thenReturn(result);

// Custom matcher
when(repo.findByPrice(argThat(p -> p > 100))).thenReturn(expensiveBooks);
```

**⚠️ Rule:** if you use one matcher, use them for ALL arguments!
```java
when(x.method(eq("A"), any())).thenReturn(...);      // ✅
when(x.method("A", any())).thenReturn(...);          // ❌ error
```

### Stubbing void methods

```java
doNothing().when(repo).delete(any());
doThrow(new RuntimeException()).when(repo).delete(1L);
doAnswer(inv -> {
    log.info("saving " + inv.getArgument(0));
    return null;
}).when(repo).save(any());
```

### Verify — check interactions

```java
verify(repo).save(book);                             // called exactly once
verify(repo, times(2)).findAll();
verify(repo, never()).delete(any());
verify(repo, atLeastOnce()).findById(anyLong());
verify(repo, atMost(3)).save(any());
verifyNoInteractions(emailSender);
verifyNoMoreInteractions(repo);                      // no other calls beyond verified
```

### ArgumentCaptor — inspect actual arguments passed

```java
ArgumentCaptor<Book> captor = ArgumentCaptor.forClass(Book.class);
service.create(new CreateBookRequest("Java", 500));
verify(repo).save(captor.capture());

Book saved = captor.getValue();
assertThat(saved.getTitle()).isEqualTo("Java");
assertThat(saved.getPrice()).isEqualTo(500);
```

Great for asserting on complex arguments constructed by the code under test.

### Complete example

```java
@ExtendWith(MockitoExtension.class)
class BookServiceTest {

    @Mock BookRepository repo;
    @Mock EmailService emailService;
    @InjectMocks BookService service;

    @Test
    void createBook_shouldSaveAndNotify() {
        // Arrange
        Book input = new Book("Java", 500);
        Book saved = new Book(1L, "Java", 500);
        when(repo.save(any(Book.class))).thenReturn(saved);

        // Act
        Book result = service.create(input);

        // Assert
        assertThat(result.getId()).isEqualTo(1L);
        verify(repo).save(input);
        verify(emailService).sendConfirmation("Java");
        verifyNoMoreInteractions(emailService);
    }

    @Test
    void findById_notFound_shouldThrow() {
        when(repo.findById(anyLong())).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.findById(99L))
            .isInstanceOf(BookNotFoundException.class)
            .hasMessageContaining("99");
    }

    @Test
    void findById_shouldReturnBook() {
        Book book = new Book(1L, "Java");
        when(repo.findById(1L)).thenReturn(Optional.of(book));

        Book result = service.findById(1L);

        assertThat(result).isEqualTo(book);
    }
}
```

### Spy — partial mock

Real object; you selectively override methods.

```java
List<String> list = new ArrayList<>();
List<String> spy = spy(list);

spy.add("a");                              // real method runs — list now has "a"
verify(spy).add("a");
assertThat(spy.get(0)).isEqualTo("a");

// Override selectively
doReturn(10).when(spy).size();             // now size() returns 10
```

Careful: `when(spy.method()).thenReturn(...)` calls the real method! Use `doReturn().when(spy).method()` for spies.

### Mock vs Spy

| | Mock | Spy |
|-|------|-----|
| What is it? | Empty fake object | Real object with selective stubs |
| Unmocked calls | Return default (null/0/empty) | Call real method |
| Use case | Isolate dependencies | Real behavior + override a few methods |

### Mocking static methods (Mockito 3.4+)

```java
try (MockedStatic<Utils> mocked = mockStatic(Utils.class)) {
    mocked.when(() -> Utils.now()).thenReturn(Instant.parse("2025-01-01T00:00:00Z"));
    // code that calls Utils.now() now gets the fixed value
}
```

---

## 9.3 Spring Boot Testing — Test Slices

Spring Boot provides "test slices" — load only the parts of the context you need.

### @SpringBootTest — full context

Loads entire Spring context. Slowest but most realistic.

```java
@SpringBootTest
class ApplicationTests {
    @Autowired BookService service;

    @Test
    void contextLoads() { }

    @Test
    void serviceWorks() {
        assertThat(service.findAll()).isNotNull();
    }
}
```

Use for end-to-end integration tests.

### @WebMvcTest — controller layer only

Loads MVC only. No service beans, no DB. Fast.

```java
@WebMvcTest(BookController.class)
class BookControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean BookService service;                  // replaces real bean
    @Autowired ObjectMapper mapper;

    @Test
    void getBooks_shouldReturnList() throws Exception {
        when(service.findAll()).thenReturn(List.of(new Book(1L, "Java")));

        mockMvc.perform(get("/api/books"))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$", hasSize(1)))
            .andExpect(jsonPath("$[0].title").value("Java"))
            .andExpect(jsonPath("$[0].id").value(1));
    }

    @Test
    void createBook_shouldReturn201() throws Exception {
        Book input = new Book(null, "New");
        Book saved = new Book(1L, "New");
        when(service.save(any())).thenReturn(saved);

        mockMvc.perform(post("/api/books")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(input)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(header().string("Location", "/api/books/1"));
    }

    @Test
    void getBookNotFound_shouldReturn404() throws Exception {
        when(service.findById(999L)).thenThrow(new BookNotFoundException(999L));

        mockMvc.perform(get("/api/books/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("BOOK_NOT_FOUND"));
    }

    @Test
    void createBookInvalid_shouldReturn400() throws Exception {
        String bad = "{\"title\":\"\",\"price\":-10}";
        mockMvc.perform(post("/api/books")
                .contentType(MediaType.APPLICATION_JSON)
                .content(bad))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.title").value("must not be blank"));
    }
}
```

### @DataJpaTest — repository layer only

Rolls back after each test. Uses in-memory H2 by default.

```java
@DataJpaTest
class BookRepositoryTest {

    @Autowired BookRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void findByTitle_shouldFindMatch() {
        em.persistAndFlush(new Book("Java Book"));
        em.persistAndFlush(new Book("Python Book"));

        List<Book> found = repo.findByTitleContainingIgnoreCase("java");

        assertThat(found).hasSize(1);
        assertThat(found.get(0).getTitle()).isEqualTo("Java Book");
    }

    @Test
    void save_shouldGenerateId() {
        Book saved = repo.save(new Book("New"));
        assertThat(saved.getId()).isNotNull();
    }
}
```

### Other slices

- `@JsonTest` — Jackson serialization
- `@RestClientTest` — RestTemplate/WebClient callers
- `@DataMongoTest` — MongoDB repositories
- `@WebFluxTest` — reactive controllers

### @MockBean vs @Mock

| `@MockBean` | `@Mock` |
|-------------|---------|
| Replaces the bean in Spring context | Plain Mockito mock (no Spring) |
| Use in `@SpringBootTest`, `@WebMvcTest` | Use in pure Mockito tests |
| Slower (context) | Faster |

### MockMvc — testing controllers

```java
mockMvc.perform(get("/api/books/1")
        .accept(MediaType.APPLICATION_JSON)
        .header("Authorization", "Bearer xyz"))
    .andDo(print())                                          // debug output
    .andExpect(status().isOk())
    .andExpect(content().contentType(MediaType.APPLICATION_JSON))
    .andExpect(jsonPath("$.title").value("Java"))
    .andExpect(jsonPath("$.author.name").exists())
    .andExpect(jsonPath("$.tags", hasSize(2)));

// Test with query params
mockMvc.perform(get("/api/books")
        .param("author", "Bloch")
        .param("sort", "price,desc"))
    .andExpect(status().isOk());
```

### JSONPath cheat sheet
```
$              root
$.title        field
$[0]           first array element
$.items[*]     all array elements
$.author.name  nested field
$..author      recursive (any depth)
```

---

## 9.4 Integration Testing with TestContainers

Real databases in Docker for tests. Way more realistic than H2.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
@AutoConfigureMockMvc
class BookIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url",      postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired MockMvc mvc;
    @Autowired BookRepository repo;

    @Test
    void endToEnd() throws Exception {
        // Given
        repo.save(new Book("Java"));

        // When / Then
        mvc.perform(get("/api/books"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].title").value("Java"));
    }
}
```

**Supports** almost anything: PostgreSQL, MySQL, MongoDB, Redis, Kafka, RabbitMQ, Elasticsearch, LocalStack (AWS mock).

**Best practice**: use TestContainers instead of H2 — H2 behaves differently from real Postgres/MySQL and can hide bugs.

---

## 9.5 Testing REST Clients (RestTemplate / WebClient)

### With `@RestClientTest`

```java
@RestClientTest(BookClient.class)
class BookClientTest {

    @Autowired BookClient client;
    @Autowired MockRestServiceServer server;

    @Test
    void getBook_shouldParseResponse() {
        server.expect(requestTo("/books/1"))
              .andRespond(withSuccess(
                  "{\"id\":1,\"title\":\"Java\"}",
                  MediaType.APPLICATION_JSON));

        Book b = client.getBook(1L);

        assertThat(b.getTitle()).isEqualTo("Java");
    }
}
```

Or with WireMock:
```java
@Test
void withWireMock() {
    stubFor(get("/books/1")
        .willReturn(okJson("{\"id\":1,\"title\":\"Java\"}")));
    // call your code
}
```

---

## 9.6 UI Testing — Playwright

Modern E2E from Microsoft. Runs actual browsers.

```bash
npm init playwright@latest
```

```typescript
import { test, expect } from '@playwright/test';

test('login flow', async ({ page }) => {
  await page.goto('http://localhost:4200/login');
  await page.fill('#username', 'user');
  await page.fill('#password', 'pass');
  await page.click('button[type=submit]');

  await expect(page).toHaveURL(/dashboard/);
  await expect(page.locator('h1')).toContainText('Welcome');
});

test('add a book', async ({ page }) => {
  await page.goto('/books');
  await page.click('text=Add Book');
  await page.fill('input[name=title]', 'Java');
  await page.click('text=Save');
  await expect(page.locator('td:has-text("Java")')).toBeVisible();
});
```

Features:
- Multi-browser (Chromium, Firefox, WebKit)
- Auto-waits (no `sleep`)
- Screenshots & videos on failure
- Network interception (mock backend)
- Parallel execution
- Trace viewer

### Playwright vs Selenium vs Cypress

| | Playwright | Selenium | Cypress |
|-|------------|----------|---------|
| Speed | Fast | Slow | Very fast |
| Browsers | All | All | Chrome-family only |
| API | Modern async | Classic | Modern |
| Cross-origin | Yes | Yes | Limited |

---

## 9.7 Code Coverage — JaCoCo

Measures what code your tests execute.

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Run `mvn test` → open `target/site/jacoco/index.html` for the report.

**Target**: 70-80% line + branch coverage. Beware: 100% coverage ≠ bug-free!

---

## 9.8 SonarQube — Static Analysis

Scans code for bugs, vulnerabilities, code smells, duplication.

```bash
mvn sonar:sonar \
    -Dsonar.host.url=http://localhost:9000 \
    -Dsonar.login=<token>
```

Common quality gate: block PR if coverage < 80% or new critical bugs.

---

## 9.9 Best Practices

1. **AAA structure** — Arrange, Act, Assert (or Given, When, Then)
2. **One logical assertion per test** — makes failures obvious
3. **Descriptive test names** — `deposit_negativeAmount_throwsException`
4. **Test behavior, not implementation** — don't over-mock internal calls
5. **Don't test private methods directly** — test them via public API
6. **Fixtures for common setup** — but avoid over-generic base classes
7. **Test edge cases** — nulls, empty, boundaries, invalid inputs
8. **Mock only what you own** — mock your own dependencies, not framework classes
9. **Fast tests** — < 1 second per unit test; slow tests get skipped
10. **Delete useless tests** — flaky or tautological tests are worse than nothing

---

## 9.10 Testing Async / Reactive Code

### CompletableFuture

```java
@Test
void asyncTest() throws Exception {
    CompletableFuture<String> cf = service.processAsync("input");
    String result = cf.get(5, TimeUnit.SECONDS);      // wait
    assertThat(result).isEqualTo("done");
}
```

### Awaitility — for eventual conditions

```java
@Test
void kafkaEventProcessed() {
    kafkaProducer.send("orders", order);

    await().atMost(5, TimeUnit.SECONDS)
           .until(() -> orderRepo.existsById(order.getId()));
}
```

### Reactor testing

```java
@Test
void webFluxTest() {
    Flux<Integer> flux = Flux.just(1, 2, 3).map(i -> i * 2);
    StepVerifier.create(flux)
        .expectNext(2, 4, 6)
        .verifyComplete();
}
```

---

## 9.11 Complete Test Suite Example

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("BookService")
class BookServiceTest {

    @Mock BookRepository repo;
    @Mock EmailService emailService;
    @InjectMocks BookService service;

    @Nested
    @DisplayName("create()")
    class Create {

        @Test @DisplayName("saves book and sends email")
        void happyPath() {
            Book input = new Book("Java", 500);
            when(repo.save(any())).thenAnswer(inv -> {
                Book b = inv.getArgument(0); b.setId(1L); return b;
            });

            Book saved = service.create(input);

            assertThat(saved.getId()).isEqualTo(1L);
            verify(emailService).sendConfirmation(saved);
        }

        @ParameterizedTest
        @ValueSource(doubles = {0, -1, -100})
        @DisplayName("rejects non-positive price")
        void rejectsBadPrice(double price) {
            Book bad = new Book("X", price);
            assertThatThrownBy(() -> service.create(bad))
                .isInstanceOf(IllegalArgumentException.class);
            verifyNoInteractions(repo);
        }
    }

    @Nested
    @DisplayName("findById()")
    class FindById {

        @Test @DisplayName("returns book when found")
        void found() {
            Book book = new Book(1L, "Java");
            when(repo.findById(1L)).thenReturn(Optional.of(book));

            Book result = service.findById(1L);

            assertThat(result).isEqualTo(book);
        }

        @Test @DisplayName("throws BookNotFoundException when missing")
        void notFound() {
            when(repo.findById(anyLong())).thenReturn(Optional.empty());

            assertThatThrownBy(() -> service.findById(99L))
                .isInstanceOf(BookNotFoundException.class);
        }
    }
}
```

---

## 9.12 Interview Question Bank

1. **Why unit test?**  
   Catch bugs early, enable refactoring, document behavior, faster feedback.

2. **Unit vs Integration vs E2E test?**  
   Unit: one class in isolation. Integration: several components + real DB/services. E2E: full flow through UI.

3. **Mock vs Spy?**  
   Mock: empty fake, you tell it what to do. Spy: real object, override selectively.

4. **@MockBean vs @Mock?**  
   `@MockBean` replaces bean in Spring context. `@Mock` is plain Mockito (no Spring).

5. **How to test REST controllers?**  
   `@WebMvcTest` + `MockMvc` + `@MockBean` for service.

6. **How to test repositories?**  
   `@DataJpaTest` — in-memory or TestContainers.

7. **What is TestContainers?**  
   Library that runs Docker containers (real DBs, Kafka, etc.) for integration tests.

8. **What is code coverage? Ideal %?**  
   Metric — % of lines/branches executed by tests. Aim for 70-80%.

9. **How to test async code?**  
   `CompletableFuture.get(timeout)`, Awaitility for eventual conditions, StepVerifier for Reactor.

10. **What is the test pyramid?**  
    Many unit tests, some integration, few E2E.

11. **JUnit 4 vs JUnit 5?**  
    JUnit 5 is modular (Jupiter, Vintage, Platform), extension model, better parameterized tests, nested classes.

12. **How to mock static methods?**  
    `Mockito.mockStatic(Utils.class)` in try-with-resources (Mockito 3.4+).

13. **How to handle flaky tests?**  
    Identify root cause (time, order, shared state). Use `@RepeatedTest`, `Awaitility`, deterministic mocks.

14. **Difference between BDD and TDD?**  
    TDD: red → green → refactor. BDD: focuses on behavior, Given/When/Then, often with Cucumber.

15. **When would you not test?**  
    Trivial getters/setters, framework internals, ad-hoc scripts. But be honest — most code should be tested.

16. **What is a test double?**  
    Umbrella term: dummy (unused arg), stub (returns canned data), fake (working simplified impl), mock (with expectations), spy (records calls).

17. **How to test exception scenarios?**  
    `assertThatThrownBy(...).isInstanceOf(...)` or `assertThrows`.

18. **@BeforeEach vs @BeforeAll?**  
    BeforeEach: before every test. BeforeAll: once for the whole class (static).

19. **Why use AssertJ over JUnit assertions?**  
    Fluent, more readable, chainable, better error messages.

20. **How to test Kafka producers/consumers?**  
    Embedded Kafka (`spring-kafka-test`) or Testcontainers with Kafka image.

---

## 9.13 Cheat Sheet

```
Pyramid: many unit, some integration, few E2E
AAA: Arrange → Act → Assert
Prefer AssertJ (assertThat…)
@Mock + @InjectMocks + MockitoExtension for pure unit tests
@WebMvcTest for controllers; @DataJpaTest for repos; @SpringBootTest for full
TestContainers > H2 for realistic integration tests
Mock only what you own; don't mock value objects
ArgumentCaptor to inspect complex args
JaCoCo for coverage; SonarQube for quality gates
Playwright for modern E2E
```

---

## Practical Assignments

1. Write unit tests for a `PriceCalculator` covering happy path, boundary values, and exceptions using JUnit 5 + AssertJ + Mockito.
2. Write a `@WebMvcTest` for a `BookController` covering GET (200), POST (201, 400 for invalid), DELETE (204, 404).
3. Write a `@DataJpaTest` for repo with derived queries.
4. Set up a TestContainers-based integration test with PostgreSQL.
5. Configure JaCoCo with 80% coverage gate; make the build fail if under.
6. Write a Playwright E2E test for a login → dashboard flow.

Master these and testing interviews are yours. 🚀