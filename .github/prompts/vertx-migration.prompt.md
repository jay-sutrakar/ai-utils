---
mode: agent
description: Migrate a Vert.x 4.x project to Vert.x 5.0.8
---

Migrate this entire project from Vert.x 4.x to Vert.x 5.0.8.

Follow the instructions defined in:
- `.github/copilot-instructions.md`
- `.github/instructions/dependencies.instructions.md`
- `.github/instructions/java.instructions.md`
- `.github/instructions/tests.instructions.md`

## Migration Steps

### Phase 1: Dependencies (pom.xml / build.gradle)

1. Analyze all dependencies using the rules in `dependencies.instructions.md`
2. Update the Vert.x BOM version to 5.0.8
3. Remove any artifacts that were removed in Vert.x 5 (vertx-sync, service factories, etc.)
4. Replace sunsetted artifacts with their recommended replacements
5. Add `vertx-launcher-application` if the project uses `io.vertx.core.Launcher`
6. Update the main class from `io.vertx.core.Launcher` to `io.vertx.launcher.application.VertxApplication`
7. Update Java version to 21 if currently below 11
8. Update build plugins (vertx-maven-plugin, shade plugin main class, etc.)

### Phase 2: Java Source Files

1. Remove all callback-based API usages — replace with Future-based APIs
2. Replace `new VertxOptions().setXxx(...)` with `Vertx.builder().withXxx(...).build()`
3. Replace `vertx.executeBlocking(promise -> { promise.complete(result); })` with `vertx.executeBlocking(() -> result)`
4. Replace `CompositeFuture.all/any/join(list)` with `Future.all/any/join(list)`
5. Replace `future.eventually(v -> someFuture())` with `future.eventually(() -> someFuture())`
6. Replace `new DeploymentOptions().setWorker(true)` with `new DeploymentOptions().setThreadingModel(ThreadingModel.WORKER)`
7. Update HTTP client/server APIs:
   - `request.setTimeout()` → `request.setIdleTimeout()`
   - `request.setHost().setPort()` → `request.authority(HostAndPort.create(host, port))`
   - `request.cookieMap()` → `request.cookies()`
   - `connection.shutdown(millis)` → `connection.shutdown(value, TimeUnit)`
   - WebSocket split: `httpClient.webSocket(...)` → `vertx.createWebSocketClient().connect(...)`
   - Pool options: `HttpClientOptions.setMaxPoolSize()` → `PoolOptions.setHttp1MaxSize()`
8. Update EventBus API: use `MessageConsumerOptions` for consumer configuration
9. Update SQL client: `PgPool.pool(vertx, opts, pool)` → `PgBuilder.pool().with(...).connectingTo(...).using(vertx).build()`
10. Update Auth: `AuthProvider` → `AuthenticationProvider`
11. Update Web APIs:
    - `routingContext.clearUser()` → `routingContext.userContext().logout()`
    - `StaticHandler.create().setAllowRootFileSystemAccess(true)` → `StaticHandler.create(FileSystemAccess.ROOT, root)`
    - `CorsHandler.addRelativeOrigin()` → `CorsHandler.addOriginWithRegex()`
    - `ResponsePredicate.SC_SUCCESS` → `HttpResponseExpectation.SC_SUCCESS`

### Phase 3: Test Files

1. Replace `vertx-unit` patterns with `vertx-junit5`:
   - Add `@ExtendWith(VertxExtension.class)` to test classes
   - Inject `Vertx` and `VertxTestContext` as method parameters
2. Replace `testContext.succeeding()` with `testContext.succeedingThenComplete()`
3. Replace `testContext.failing()` with `testContext.failingThenComplete()`
4. Update integration tests to use `WebClient` for HTTP testing
5. Update database tests to use `PgBuilder.pool()` with Testcontainers

### Phase 4: Validate

1. Run `mvn clean compile` (or `./gradlew build`) and fix any compilation errors
2. Run `mvn test` and fix any test failures
3. Report a summary of all changes made
