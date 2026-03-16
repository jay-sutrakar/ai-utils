---
applyTo: "**/*.java"
---

# Java Source Code Instructions (Vert.x)

## Class Structure

- Order class members consistently:
  1. Static fields (constants first)
  2. Instance fields
  3. Constructors
  4. Public methods
  5. Package-private / protected methods
  6. Private methods
  7. Inner classes / enums

## Verticle Design

- Each verticle should have a single responsibility
- Override `start()` returning `Future<Void>` — never use the callback-based `start(Promise<Void>)`
- Use `vertx.deployVerticle()` to compose verticle startup in the main verticle
- Pass configuration via `DeploymentOptions().setConfig(jsonObject)`
- Access config inside a verticle with `config()`

```java
// Preferred — Future-based start (Vert.x 5)
public class HttpServerVerticle extends AbstractVerticle {

    @Override
    public Future<Void> start() {
        Router router = Router.router(vertx);
        router.get("/health").handler(ctx -> ctx.response().end("OK"));

        return vertx.createHttpServer()
            .requestHandler(router)
            .listen(config().getInteger("http.port", 8080))
            .mapEmpty();
    }
}
```

```java
// Avoid — callback-based start (removed in Vert.x 5)
@Override
public void start(Promise<Void> startPromise) { ... }
```

## Future Composition

- Use `.compose()` for sequential async operations — never nest callbacks
- Use `.map()` to transform the result of a Future
- Use `.recover()` or `.otherwise()` for error fallback
- Use `Future.all()`, `Future.any()`, `Future.join()` for parallel operations (not `CompositeFuture`)
- Use `.eventually(() -> cleanup())` for finally-like cleanup
- Use `Future.succeededFuture()` and `Future.failedFuture()` to create resolved futures

```java
// Preferred — flat future chain
public Future<Order> createOrder(JsonObject data) {
    return validateOrder(data)
        .compose(valid -> repository.save(valid))
        .compose(saved -> notifyOrderCreated(saved).map(saved))
        .recover(err -> {
            log.error("Order creation failed", err);
            return Future.failedFuture(err);
        });
}
```

```java
// Avoid — nested callbacks / futures
validateOrder(data).onSuccess(valid -> {
    repository.save(valid).onSuccess(saved -> {
        notifyOrderCreated(saved).onSuccess(v -> { ... });
    });
});
```

## Route Handlers

- Keep handlers short — extract business logic into service classes
- Use `RoutingContext` to read request and write response
- Always call `ctx.response().end()` or `ctx.fail()` — never leave a request hanging
- Use `ctx.request().getParam("id")` for path/query params
- Use `ctx.body().asJsonObject()` for request body (requires `BodyHandler`)
- Set response headers explicitly: `ctx.response().putHeader("content-type", "application/json")`

```java
public class OrderHandler {

    private final OrderService orderService;

    public OrderHandler(OrderService orderService) {
        this.orderService = orderService;
    }

    public void getOrder(RoutingContext ctx) {
        String orderId = ctx.pathParam("id");

        orderService.findById(orderId)
            .onSuccess(order -> ctx.response()
                .setStatusCode(200)
                .putHeader("content-type", "application/json")
                .end(JsonObject.mapFrom(order).encode()))
            .onFailure(ctx::fail);
    }
}
```

## Service Layer

- Service classes contain business logic and return `Future<T>`
- Accept and return domain objects or `JsonObject` — not `RoutingContext`
- Use constructor injection for dependencies (Vert.x instance, config, repositories)
- Throw domain-specific exceptions wrapped in `Future.failedFuture()`

```java
public class OrderService {

    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public Future<Order> findById(String id) {
        return repository.findById(id)
            .compose(opt -> opt
                .map(Future::succeededFuture)
                .orElse(Future.failedFuture(new OrderNotFoundException(id))));
    }
}
```

## Repository / Data Access Layer

- Use Vert.x SQL Client (reactive, non-blocking) — not JDBC
- Use `PgBuilder.pool()` or `MySQLBuilder.pool()` to create connection pools (Vert.x 5 builder pattern)
- Use `PreparedQuery` with `Tuple` for parameterized queries — never concatenate SQL strings
- Use `SqlClient.withTransaction(conn -> ...)` for transactional operations
- Map `Row` results to domain objects explicitly

```java
public class OrderRepository {

    private final Pool pool;

    public OrderRepository(Pool pool) {
        this.pool = pool;
    }

    public Future<Optional<Order>> findById(String id) {
        return pool.preparedQuery("SELECT * FROM orders WHERE id = $1")
            .execute(Tuple.of(id))
            .map(rows -> {
                RowIterator<Row> it = rows.iterator();
                return it.hasNext() ? Optional.of(mapRow(it.next())) : Optional.empty();
            });
    }

    private Order mapRow(Row row) {
        return new Order(
            row.getString("id"),
            row.getString("customer_id"),
            row.getBigDecimal("total_amount"),
            row.getLocalDateTime("created_at")
        );
    }
}
```

## DTOs and Records

- Use Java records for immutable request/response DTOs
- Use `JsonObject.mapFrom(record)` and `record.toJson()` for serialization
- Never expose internal domain objects directly in API responses — map to DTOs

```java
public record CreateOrderRequest(
    String customerId,
    List<OrderItemRequest> items,
    BigDecimal totalAmount
) {
    public static CreateOrderRequest fromJson(JsonObject json) {
        return new CreateOrderRequest(
            json.getString("customerId"),
            json.getJsonArray("items").stream()
                .map(o -> OrderItemRequest.fromJson((JsonObject) o))
                .toList(),
            new BigDecimal(json.getString("totalAmount"))
        );
    }
}
```

## EventBus Communication

- Use the EventBus for communication between verticles
- Register `MessageCodec` for custom object types
- Use specific addresses with a naming convention: `service.entity.action` (e.g., `order.service.create`)
- Always handle failures from `eventBus.request()` with `.onFailure()`

```java
// Sender
vertx.eventBus().<JsonObject>request("order.service.create", orderJson)
    .onSuccess(reply -> log.info("Order created: {}", reply.body()))
    .onFailure(err -> log.error("Failed to create order", err));

// Consumer
vertx.eventBus().<JsonObject>consumer("order.service.create", msg -> {
    orderService.createOrder(msg.body())
        .onSuccess(result -> msg.reply(JsonObject.mapFrom(result)))
        .onFailure(err -> msg.fail(500, err.getMessage()));
});
```

## Null Safety

- Use `Optional` for return types that may be absent
- Use `Objects.requireNonNull()` for defensive checks on constructor parameters
- Never pass `null` as a method argument — use overloaded methods or builder patterns
- Use `JsonObject.getString("key", "default")` to provide defaults when reading config

## Streams and Functional Style

- Prefer streams over manual loops for collection transformations
- Keep stream pipelines readable — break into multiple lines
- Avoid side effects inside `map()` or `filter()` — use `forEach()` for side effects
- Use `.toList()` (Java 16+) for immutable results
