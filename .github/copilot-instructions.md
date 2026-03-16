# Project Overview

<!-- Replace with your project's elevator pitch -->
This is a [describe your application] built with Java and Spring Boot.
It provides [key features] for [target users/audience].

## Tech Stack

### Backend

- Java 21 (LTS) with Spring Boot 3.x
- Spring Web for REST APIs
- Spring Data JPA with Hibernate for persistence
- PostgreSQL as the primary database
- Spring Security for authentication and authorization
- Spring AI for LLM integrations (if applicable)
- Maven or Gradle for build management

### Frontend (if applicable)

- React.js with TypeScript
- Vite as the build tool

### Infrastructure

- Docker and Docker Compose for local development
- GitHub Actions for CI/CD

### Testing

- JUnit 5 for unit tests
- Mockito for mocking dependencies
- Spring Boot Test for integration tests
- Testcontainers for database integration tests

## Coding Conventions

### General

- Always use type hints and explicit types — avoid `var` for non-obvious types
- Follow Google Java Style Guide for formatting
- Maximum line length: 120 characters
- Use `final` for variables that should not be reassigned
- Prefer immutable objects and records where possible
- Never swallow exceptions silently — always log or rethrow with context
- Use `Optional` for return types that may be absent — never return `null` from public methods
- Use `slf4j` with `@Slf4j` (Lombok) for logging — never `System.out.println`

### Naming

- Classes: `PascalCase` (e.g., `OrderService`, `PaymentController`)
- Methods and variables: `camelCase` (e.g., `findOrderById`, `totalAmount`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `MAX_RETRY_COUNT`)
- Packages: all lowercase, no underscores (e.g., `com.example.orderservice`)
- REST endpoints: kebab-case (e.g., `/api/v1/order-items`)
- Database tables and columns: snake_case (e.g., `order_items`, `created_at`)

### Spring Boot Patterns

- Use constructor injection — never field injection with `@Autowired`
- Annotate service classes with `@Service`, repositories with `@Repository`
- Use `@RestController` for API controllers, `@Controller` for view controllers
- Keep controllers thin — delegate business logic to service layer
- Use `@Transactional` at the service layer, not the controller
- Externalize configuration with `@ConfigurationProperties` — avoid hardcoded values
- Use profiles (`application-dev.yml`, `application-prod.yml`) for environment-specific config

### REST API Design

- Follow RESTful conventions: `GET` for reads, `POST` for creates, `PUT` for full updates, `PATCH` for partial updates, `DELETE` for removals
- Version APIs with URL prefix: `/api/v1/...`
- Return appropriate HTTP status codes (201 for created, 204 for no content, 400 for bad request, 404 for not found)
- Use a consistent error response format with `timestamp`, `status`, `message`, and `path`
- Use `@Valid` with Bean Validation annotations for request body validation
- Use pagination for list endpoints: `Pageable` with `page`, `size`, `sort` parameters

### Error Handling

- Use `@RestControllerAdvice` with `@ExceptionHandler` for global exception handling
- Create custom exception classes extending `RuntimeException` for domain-specific errors
- Always include meaningful error messages in exceptions
- Log exceptions at the appropriate level: `ERROR` for unexpected failures, `WARN` for recoverable issues

## Project Structure

```
project-root/
├── .github/
│   ├── copilot-instructions.md          # This file — repo-wide instructions
│   ├── instructions/                     # Path-specific instruction files
│   │   ├── java.instructions.md
│   │   └── tests.instructions.md
│   └── workflows/                        # CI/CD pipelines
├── src/
│   ├── main/
│   │   ├── java/com/example/app/
│   │   │   ├── config/                   # Spring configuration classes
│   │   │   ├── controller/               # REST controllers (thin, delegation only)
│   │   │   ├── dto/                      # Data Transfer Objects and request/response models
│   │   │   ├── entity/                   # JPA entity classes
│   │   │   ├── exception/                # Custom exception classes and global handler
│   │   │   ├── mapper/                   # MapStruct or manual mappers (Entity <-> DTO)
│   │   │   ├── repository/               # Spring Data JPA repositories
│   │   │   ├── service/                  # Business logic layer
│   │   │   └── util/                     # Utility and helper classes
│   │   └── resources/
│   │       ├── application.yml           # Default config
│   │       ├── application-dev.yml       # Dev profile
│   │       ├── application-prod.yml      # Production profile
│   │       └── db/migration/             # Flyway or Liquibase migration scripts
│   └── test/
│       └── java/com/example/app/
│           ├── controller/               # Controller integration tests
│           ├── service/                   # Service unit tests
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

# Run the application with dev profile
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Or with Gradle:
# ./gradlew bootRun --args='--spring.profiles.active=dev'
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

### Linting & Formatting

```bash
# Check code style (if using Checkstyle)
mvn checkstyle:check

# Spot bugs (if using SpotBugs)
mvn spotbugs:check
```

### Common Issues

- If Testcontainers tests fail, ensure Docker daemon is running
- If `mvn spring-boot:run` fails with port conflict, check if port 8080 is already in use
- Always run `docker-compose up -d` before running integration tests locally

## Migration Rules

<!-- Customize this section for your specific migration -->

### Spring Boot 2.x → 3.x Migration

- Replace `javax.*` imports with `jakarta.*` (Jakarta EE 9+ namespace)
- Update Spring Security: `WebSecurityConfigurerAdapter` is removed — use `SecurityFilterChain` bean instead
- Update `@ConstructorBinding` — no longer required on single-constructor classes in Spring Boot 3
- Replace `spring.redis.*` properties with `spring.data.redis.*`
- Use `spring.docker.compose.enabled=true` for Docker Compose integration (Spring Boot 3.1+)

### Java Version Upgrades

- Prefer records for immutable data carriers (Java 16+)
- Use sealed classes/interfaces for closed type hierarchies (Java 17+)
- Use pattern matching in `switch` expressions (Java 21+)
- Use `SequencedCollection` methods like `getFirst()`, `getLast()` (Java 21+)
- Replace `text.stripLeading()` + manual null checks with pattern matching where possible

### Vert.x 4.x → 5.x Migration (if applicable)

- Replace `new VertxOptions().setWorkerPoolSize(n)` → `Vertx.builder().withWorkerPoolSize(n).build()`
- Replace `vertx.createHttpClient(options)` → `vertx.httpClientBuilder().with(options).build()`
- Replace callback-based handlers with `Future`/`Promise` composition
- Replace `vertx.executeBlocking(promise -> {...}, handler)` → `vertx.executeBlocking(() -> {...})`
- Consult official migration guide: https://vertx.io/docs/guides/vertx-5-migration-guide/

## Resources

- `docker-compose.yml` — starts all local dependencies
- `Dockerfile` — multi-stage build for containerized deployment
- `.github/workflows/` — CI/CD pipeline definitions
- `README.md` — project documentation, setup instructions, and API reference
