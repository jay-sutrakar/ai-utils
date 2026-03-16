---
applyTo: "**/*.java"
---

# Java Source Code Instructions

## Class Structure

- Order class members consistently:
  1. Static fields (constants first)
  2. Instance fields
  3. Constructors
  4. Public methods
  5. Package-private / protected methods
  6. Private methods
  7. Inner classes / enums

## Constructor Injection

- Always use constructor injection for dependencies
- Mark constructor parameters as `final`
- Use Lombok `@RequiredArgsConstructor` to reduce boilerplate, or write explicit constructors

```java
// Preferred
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
}
```

```java
// Avoid
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}
```

## DTOs and Records

- Use Java records for immutable request/response DTOs
- Use Bean Validation annotations on record components
- Never expose JPA entities directly in API responses — always map to DTOs

```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<OrderItemRequest> items,
    @NotNull @Positive BigDecimal totalAmount
) {}
```

## Entity Design

- Annotate entities with `@Entity` and always specify `@Table(name = "...")`
- Use `@GeneratedValue(strategy = GenerationType.IDENTITY)` for auto-increment IDs
- Add `@Column(nullable = false)` for required fields
- Include `createdAt` and `updatedAt` audit fields with `@CreationTimestamp` and `@UpdateTimestamp`
- Override `equals()` and `hashCode()` based on the entity's business key or ID — not all fields

## Service Layer

- Service methods should have a single clear responsibility
- Wrap database writes in `@Transactional`
- Throw domain-specific exceptions (e.g., `OrderNotFoundException`) — not generic ones
- Log entry/exit of important operations at `DEBUG` level
- Return `Optional<T>` when a result may not exist — never return `null`

## Controller Layer

- Keep controllers thin — max 5-10 lines per handler method
- Use `ResponseEntity<T>` for explicit control over status codes and headers
- Use `@Valid` on `@RequestBody` parameters to trigger validation
- Document endpoints with meaningful method names and Javadoc or OpenAPI annotations

```java
@PostMapping
public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
    OrderResponse response = orderService.createOrder(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

## Exception Handling

- Define custom exceptions in the `exception/` package
- Use `@ResponseStatus` on exceptions or handle via `@RestControllerAdvice`
- Always include the original cause when wrapping exceptions

```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String orderId) {
        super("Order not found with id: " + orderId);
    }
}
```

## Streams and Functional Style

- Prefer streams over manual loops for collection transformations
- Keep stream pipelines readable — break into multiple lines
- Avoid side effects inside `map()` or `filter()` — use `forEach()` for side effects
- Use `Collectors.toUnmodifiableList()` or `.toList()` (Java 16+) for immutable results

## Null Safety

- Use `Optional` for return types that may be absent
- Use `@NonNull` / `@Nullable` annotations from `jakarta.annotation` to document intent
- Use `Objects.requireNonNull()` for defensive checks on constructor parameters
- Never pass `null` as a method argument — use overloaded methods or builder patterns instead
