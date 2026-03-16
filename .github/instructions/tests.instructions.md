---
applyTo: "**/test/**/*.java,**/*Test.java,**/*Tests.java,**/*IT.java"
---

# Test Code Instructions (Vert.x)

## Test Setup

- Use `vertx-junit5` extension — never `vertx-unit` (sunsetted in Vert.x 5)
- Annotate test classes with `@ExtendWith(VertxExtension.class)`
- Inject `Vertx` and `VertxTestContext` as method parameters
- Use `VertxTestContext` to manage async test completion

```java
@ExtendWith(VertxExtension.class)
class OrderServiceTest {

    @Test
    void should_returnOrder_when_validIdProvided(Vertx vertx, VertxTestContext testContext) {
        // test body
    }
}
```

## Test Naming

- Use descriptive method names that explain the scenario and expected outcome
- Follow the pattern: `should_expectedBehavior_when_condition`
- Do not prefix test methods with `test` — the `@Test` annotation is sufficient

```java
@Test
void should_returnOrder_when_validIdProvided(Vertx vertx, VertxTestContext testContext) { ... }

@Test
void should_failWithNotFound_when_orderDoesNotExist(Vertx vertx, VertxTestContext testContext) { ... }

@Test
void should_returnEmptyList_when_noOrdersMatchCriteria(Vertx vertx, VertxTestContext testContext) { ... }
```

## Async Test Completion

- Always use `testContext.succeedingThenComplete()` or `testContext.failingThenComplete()` (Vert.x 5)
- Never use the deprecated `testContext.succeeding()` or `testContext.failing()` (Vert.x 4)
- Use `testContext.verify(() -> { assertions })` for inline assertions
- Use `testContext.completeNow()` after all assertions pass
- Use `Checkpoint` for tests that need multiple async events to complete

```java
// Preferred — Vert.x 5 style
@Test
void should_deployVerticle(Vertx vertx, VertxTestContext testContext) {
    vertx.deployVerticle(new MainVerticle())
        .onComplete(testContext.succeedingThenComplete());
}

// With assertions
@Test
void should_respondWithOk(Vertx vertx, VertxTestContext testContext) {
    WebClient client = WebClient.create(vertx);

    vertx.deployVerticle(new HttpServerVerticle())
        .compose(id -> client.get(8080, "localhost", "/health").send())
        .onComplete(testContext.succeeding(response -> testContext.verify(() -> {
            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.bodyAsString()).isEqualTo("OK");
            testContext.completeNow();
        })));
}
```

```java
// Avoid — deprecated Vert.x 4 style
vertx.deployVerticle(new MainVerticle())
    .onComplete(testContext.succeeding(id -> testContext.completeNow()));
```

## Using Checkpoints

- Use checkpoints when a test requires multiple async events to complete
- Call `checkpoint.flag()` each time an expected event occurs

```java
@Test
void should_processAllMessages(Vertx vertx, VertxTestContext testContext) {
    Checkpoint messagesReceived = testContext.checkpoint(3);

    vertx.eventBus().<String>consumer("test.address", msg -> {
        messagesReceived.flag();
    });

    for (int i = 0; i < 3; i++) {
        vertx.eventBus().send("test.address", "message-" + i);
    }
}
```

## Assertions

- Prefer AssertJ (`assertThat(...)`) over JUnit assertions (`assertEquals(...)`)
- Use fluent assertion chains for readability
- Wrap assertions in `testContext.verify(() -> { ... })` to properly report failures

```java
// Preferred (AssertJ inside testContext.verify)
testContext.verify(() -> {
    assertThat(result).isNotNull();
    assertThat(result.getString("status")).isEqualTo("CREATED");
    assertThat(result.getJsonArray("items")).hasSize(3);
    testContext.completeNow();
});

// Avoid
assertTrue(result != null);
assertEquals("CREATED", result.getString("status"));
```

## Exception Testing

- Use `testContext.failingThenComplete()` to verify a future fails
- Use `testContext.succeeding()` with manual verification for checking exception types

```java
@Test
void should_failWithNotFound_when_invalidId(Vertx vertx, VertxTestContext testContext) {
    OrderService service = new OrderService(repository);

    service.findById("non-existent")
        .onComplete(testContext.failing(err -> testContext.verify(() -> {
            assertThat(err).isInstanceOf(OrderNotFoundException.class);
            assertThat(err.getMessage()).contains("non-existent");
            testContext.completeNow();
        })));
}
```

## Mocking

- Use `@ExtendWith(MockitoExtension.class)` alongside `@ExtendWith(VertxExtension.class)`
- Use `@Mock` for dependencies
- Manually inject mocks via constructor — no `@InjectMocks` with Vert.x classes
- Prefer `given(...).willReturn(...)` (BDD-style) over `when(...).thenReturn(...)`
- Mock methods that return `Future<T>` by returning `Future.succeededFuture(value)` or `Future.failedFuture(err)`

```java
@ExtendWith(VertxExtension.class)
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Test
    void should_returnOrder_when_validIdProvided(Vertx vertx, VertxTestContext testContext) {
        var order = new Order("ORD-001", "customer-1", BigDecimal.TEN, LocalDateTime.now());
        given(orderRepository.findById("ORD-001"))
            .willReturn(Future.succeededFuture(Optional.of(order)));

        OrderService service = new OrderService(orderRepository);

        service.findById("ORD-001")
            .onComplete(testContext.succeeding(result -> testContext.verify(() -> {
                assertThat(result.id()).isEqualTo("ORD-001");
                testContext.completeNow();
            })));
    }
}
```

## Integration Tests (HTTP)

- Use `WebClient` from `vertx-web-client` to make HTTP requests in tests
- Deploy the verticle first, then send requests
- Use `@Tag("integration")` to separate from unit tests
- Always undeploy verticles or use `@AfterEach` to clean up

```java
@ExtendWith(VertxExtension.class)
@Tag("integration")
class OrderHandlerIT {

    @BeforeEach
    void setup(Vertx vertx, VertxTestContext testContext) {
        vertx.deployVerticle(new HttpServerVerticle())
            .onComplete(testContext.succeedingThenComplete());
    }

    @Test
    void should_createOrder_when_validRequest(Vertx vertx, VertxTestContext testContext) {
        WebClient client = WebClient.create(vertx);
        JsonObject body = new JsonObject()
            .put("customerId", "cust-1")
            .put("totalAmount", "99.99");

        client.post(8080, "localhost", "/api/v1/orders")
            .sendJsonObject(body)
            .onComplete(testContext.succeeding(response -> testContext.verify(() -> {
                assertThat(response.statusCode()).isEqualTo(201);
                assertThat(response.bodyAsJsonObject().getString("id")).isNotBlank();
                testContext.completeNow();
            })));
    }
}
```

## Database Integration Tests (Testcontainers)

- Use Testcontainers for database tests — never rely on an external database
- Use `@Tag("integration")` to separate from unit tests
- Create the SQL client pool pointing to the Testcontainer's dynamic port

```java
@ExtendWith(VertxExtension.class)
@Tag("integration")
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    private Pool pool;

    @BeforeEach
    void setup(Vertx vertx) {
        PgConnectOptions connectOptions = new PgConnectOptions()
            .setHost(postgres.getHost())
            .setPort(postgres.getMappedPort(5432))
            .setDatabase(postgres.getDatabaseName())
            .setUser(postgres.getUsername())
            .setPassword(postgres.getPassword());

        pool = PgBuilder.pool()
            .with(new PoolOptions().setMaxSize(5))
            .connectingTo(connectOptions)
            .using(vertx)
            .build();
    }

    @Test
    void should_findOrder_when_existsInDatabase(Vertx vertx, VertxTestContext testContext) {
        OrderRepository repository = new OrderRepository(pool);

        repository.findById("ORD-001")
            .onComplete(testContext.succeeding(result -> testContext.verify(() -> {
                assertThat(result).isPresent();
                assertThat(result.get().id()).isEqualTo("ORD-001");
                testContext.completeNow();
            })));
    }
}
```

## General Rules

- Tests must be independent — no shared mutable state, no ordering dependencies
- Each test class tests one production class
- Keep test data minimal — only what is needed for the assertion
- Do not use `Thread.sleep()` in tests — use `VertxTestContext` with timeouts or Checkpoints
- Always set a test timeout to avoid hanging tests: `@Timeout(value = 5, timeUnit = TimeUnit.SECONDS)`
