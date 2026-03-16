# Project Overview

<!-- Replace with your project's elevator pitch -->
This is a [describe your application] built with Eclipse Vert.x.
It provides [key features] for [target users/audience].

## Tech Stack

### Backend

- Java 21 (LTS) with Eclipse Vert.x 5.0.8
- Vert.x Web for HTTP routing and REST APIs
- Vert.x Web Client for outbound HTTP requests
- Vert.x SQL Client (Reactive PostgreSQL / MySQL client)
- Vert.x Auth for authentication and authorization
- Vert.x Config for externalized configuration
- Maven or Gradle for build management

### Infrastructure

- Docker and Docker Compose for local development
- GitHub Actions for CI/CD

### Testing

- JUnit 5 with `vertx-junit5` extension
- Testcontainers for database integration tests

## Coding Conventions

### General

- Always use explicit types — avoid `var` for non-obvious types
- Maximum line length: 120 characters
- Use `final` for variables that should not be reassigned
- Prefer immutable objects and Java records where possible
- Never swallow exceptions silently — always log or propagate with context
- Use `slf4j` for logging — never `System.out.println`

### Naming

- Classes: `PascalCase` (e.g., `OrderVerticle`, `PaymentHandler`)
- Methods and variables: `camelCase` (e.g., `findOrderById`, `totalAmount`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `MAX_RETRY_COUNT`)
- Packages: all lowercase, no underscores (e.g., `com.example.order`)
- REST endpoints: kebab-case (e.g., `/api/v1/order-items`)
- Database tables and columns: snake_case (e.g., `order_items`, `created_at`)
- Verticle classes: suffix with `Verticle` (e.g., `HttpServerVerticle`, `DatabaseVerticle`)

### Vert.x Patterns

- Use the `Future`-based API exclusively — never use callbacks (removed in Vert.x 5)
- Compose futures with `.compose()`, `.map()`, `.recover()` — avoid nested callbacks
- Use `Vertx.builder()` to create Vert.x instances (Vert.x 5 builder pattern)
- Use `vertx.executeBlocking(() -> result)` for blocking operations (Vert.x 5 lambda style)
- Deploy verticles with `vertx.deployVerticle()` returning `Future<String>`
- Use the EventBus for inter-verticle communication — keep verticles decoupled
- Use `JsonObject` and `JsonArray` for EventBus messages and configuration
- Use `@Suspendable` or virtual threads for sequential-looking async code (Java 21+)
- Keep handlers short — delegate to service methods that return `Future<T>`
- Use `DeploymentOptions` with `setInstances()` for scaling CPU-bound verticles
- Use `DeploymentOptions().setThreadingModel(ThreadingModel.WORKER)` for worker verticles (not `setWorker(true)`)

### REST API Design

- Use Vert.x Web `Router` for defining routes
- Use sub-routers for grouping related endpoints (e.g., `Router orderRouter = Router.router(vertx)`)
- Return appropriate HTTP status codes (201 for created, 204 for no content, 400 for bad request, 404 for not found)
- Use a consistent error response JSON format with `timestamp`, `status`, `message`, and `path`
- Use Vert.x Web Validation for request validation
- Use `BodyHandler.create()` before routes that read request bodies
- Use `routingContext.response().setStatusCode(200).putHeader("content-type", "application/json").end(json)` pattern

### Error Handling

- Use `Future.failedFuture(exception)` to propagate errors through the future chain
- Use `.recover()` or `.otherwise()` for fallback logic
- Create custom exception classes for domain-specific errors
- Use `router.errorHandler(statusCode, ctx -> ...)` for centralized HTTP error handling
- Always include meaningful error messages in exceptions
- Log exceptions at the appropriate level: `ERROR` for unexpected failures, `WARN` for recoverable issues

## Project Structure

```
project-root/
├── .github/
│   ├── copilot-instructions.md          # This file — repo-wide instructions
│   ├── instructions/                     # Path-specific instruction files
│   │   ├── java.instructions.md
│   │   ├── tests.instructions.md
│   │   └── dependencies.instructions.md
│   └── workflows/                        # CI/CD pipelines
├── src/
│   ├── main/
│   │   ├── java/com/example/app/
│   │   │   ├── verticle/                 # Verticle classes (entry points)
│   │   │   ├── handler/                  # Vert.x Web route handlers
│   │   │   ├── service/                  # Business logic (returns Future<T>)
│   │   │   ├── repository/               # Data access layer (SQL client queries)
│   │   │   ├── model/                    # Domain models, DTOs, records
│   │   │   ├── codec/                    # EventBus message codecs
│   │   │   └── util/                     # Utility and helper classes
│   │   └── resources/
│   │       ├── config.json               # Default Vert.x config
│   │       ├── config-dev.json           # Dev profile config
│   │       └── config-prod.json          # Production config
│   └── test/
│       └── java/com/example/app/
│           ├── verticle/                 # Verticle deployment tests
│           ├── handler/                  # HTTP handler integration tests
│           ├── service/                  # Service unit tests
│           └── repository/               # Repository tests with Testcontainers
├── docker-compose.yml                    # Local dev dependencies (DB, Redis, etc.)
├── Dockerfile                            # Multi-stage build for the app
├── pom.xml (or build.gradle)             # Build configuration
└── README.md
```

## Build, Test & Run

### Prerequisites

- Java 21 (verify with `java --version`)
- Maven 3.9+ (verify with `mvn --version`) or Gradle 8+
- Docker and Docker Compose (for local database and services)

### Setup & Run

```bash
# Start local infrastructure (database, etc.)
docker-compose up -d

# Build the project (skip tests for quick builds)
mvn clean install -DskipTests

# Run the application using Vert.x Application Launcher
mvn exec:java -Dexec.mainClass="io.vertx.launcher.application.VertxApplication" -Dexec.args="com.example.app.verticle.MainVerticle"

# Or using the Vert.x Maven Plugin (dev mode with hot reload)
mvn vertx:run

# Or with Gradle:
# ./gradlew run
```

### Testing

```bash
# Run all tests
mvn test

# Run only unit tests (excludes integration tests)
mvn test -Dgroups="unit"

# Run only integration tests (requires Docker for Testcontainers)
mvn test -Dgroups="integration"

# Run tests with coverage report
mvn test jacoco:report
# Report at: target/site/jacoco/index.html
```

### Common Issues

- If Testcontainers tests fail, ensure Docker daemon is running
- If the app fails with port conflict, check if the configured port is already in use
- Always run `docker-compose up -d` before running integration tests locally
- If you see `ClassNotFoundException: io.vertx.core.Launcher`, you need `vertx-launcher-application` dependency and `VertxApplication` as main class

## Migration Rules

### Vert.x 4.x → 5.x Migration

#### Core API Changes

- Replace `new VertxOptions().setMetricsOptions(...)` → `Vertx.builder().with(options).withMetrics(factory).build()`
- Replace `Vertx.clusteredVertx(options)` → `Vertx.builder().with(options).buildClustered()`
- Replace `vertx.executeBlocking(promise -> { promise.complete(result); })` → `vertx.executeBlocking(() -> result)`
- Replace `CompositeFuture.all(list)` → `Future.all(list)`
- Replace `future.eventually(v -> someFuture())` → `future.eventually(() -> someFuture())`
- Replace `new DeploymentOptions().setWorker(true)` → `new DeploymentOptions().setThreadingModel(ThreadingModel.WORKER)`
- Remove all callback-based API usages — only Future-based APIs exist in 5.x

#### HTTP Changes

- Replace `client.request(options, handler)` → `client.request(options)` (returns `Future`)
- Replace `request.setTimeout(timeout)` → `request.setIdleTimeout(timeout)`
- Replace `request.setHost(host).setPort(port)` → `request.authority(HostAndPort.create(host, port))`
- Replace `request.host()` → `request.authority()`
- Replace `request.cookieMap()` → `request.cookies()`
- Replace `connection.shutdown(5000)` → `connection.shutdown(5, TimeUnit.SECONDS)`
- WebSocket is now separate: `vertx.createWebSocketClient()` instead of `httpClient.webSocket(...)`
- Pool options: `new HttpClientOptions().setMaxPoolSize(n)` → `new PoolOptions().setHttp1MaxSize(n)`

#### EventBus Changes

- Replace `eventBus.consumer(ADDRESS, handler).setMaxBufferedMessages(n)` → `eventBus.consumer(new MessageConsumerOptions().setAddress(ADDRESS).setMaxBufferedMessages(n), handler)`

#### Web Changes

- Replace `routingContext.clearUser()` → `routingContext.userContext().logout()`
- Replace `StaticHandler.create().setAllowRootFileSystemAccess(true).setWebRoot(root)` → `StaticHandler.create(FileSystemAccess.ROOT, root)`
- Replace `CorsHandler.addRelativeOrigin(".*")` → `CorsHandler.addOriginWithRegex(".*")`
- Replace `ResponsePredicate.SC_SUCCESS` → `HttpResponseExpectation.SC_SUCCESS` (Web Client)

#### SQL Client Changes

- Replace `PgPool.pool(vertx, connectOptions, poolOptions)` → `PgBuilder.pool().with(poolOptions).connectingTo(connectOptions).using(vertx).build()`

#### Auth Changes

- Replace `AuthProvider` → `AuthenticationProvider`

#### Launcher Changes

- Replace `io.vertx.core.Launcher` → `io.vertx.launcher.application.VertxApplication`
- Add `vertx-launcher-application` dependency
- The `vertx` CLI tool is removed — use Maven/Gradle plugins or fat JARs

#### JUnit 5 Changes

- Replace `testContext.succeeding()` → `testContext.succeedingThenComplete()`
- Replace `testContext.failing()` → `testContext.failingThenComplete()`

### Java Version Upgrades (if applicable)

- Prefer records for immutable data carriers (Java 16+)
- Use sealed classes/interfaces for closed type hierarchies (Java 17+)
- Use pattern matching in `switch` expressions (Java 21+)
- Use virtual threads with Vert.x for simplified async code (Java 21+)
- Replace `text.stripLeading()` + manual null checks with pattern matching where possible

## Resources

- `docker-compose.yml` — starts all local dependencies
- `Dockerfile` — multi-stage build for containerized deployment
- `.github/workflows/` — CI/CD pipeline definitions
- `README.md` — project documentation, setup instructions, and API reference
- Vert.x 5 Migration Guide: https://vertx.io/docs/guides/vertx-5-migration-guide/
- Vert.x 5 Breaking Changes Wiki: https://github.com/eclipse-vertx/vert.x/wiki/Vert.x-5
