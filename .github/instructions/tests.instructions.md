---
applyTo: "**/test/**/*.java,**/*Test.java,**/*Tests.java,**/*IT.java"
---

# Test Code Instructions

## Test Naming

- Use descriptive method names that explain the scenario and expected outcome
- Follow the pattern: `should_expectedBehavior_when_condition`
- Do not prefix test methods with `test` — the `@Test` annotation is sufficient

```java
@Test
void should_returnOrder_when_validIdProvided() { ... }

@Test
void should_throwNotFoundException_when_orderDoesNotExist() { ... }

@Test
void should_returnEmptyList_when_noOrdersMatchCriteria() { ... }
```

## Test Structure (Arrange-Act-Assert)

- Always structure tests with clear Arrange, Act, Assert sections
- Use comments (`// given`, `// when`, `// then`) or blank lines to separate sections
- Each test should verify exactly one behavior

```java
@Test
void should_calculateTotalPrice_when_multipleItemsInOrder() {
    // given
    var item1 = new OrderItem("SKU-001", 2, BigDecimal.valueOf(10.00));
    var item2 = new OrderItem("SKU-002", 1, BigDecimal.valueOf(25.00));
    var order = new Order(List.of(item1, item2));

    // when
    BigDecimal total = order.calculateTotal();

    // then
    assertThat(total).isEqualByComparingTo(BigDecimal.valueOf(45.00));
}
```

## Assertions

- Prefer AssertJ (`assertThat(...)`) over JUnit assertions (`assertEquals(...)`)
- Use fluent assertion chains for readability
- Use specific assertions rather than generic boolean checks

```java
// Preferred (AssertJ)
assertThat(result).isNotNull();
assertThat(result.getItems()).hasSize(3).extracting("name").contains("Widget");
assertThat(result.getTotal()).isGreaterThan(BigDecimal.ZERO);

// Avoid
assertTrue(result != null);
assertEquals(3, result.getItems().size());
```

## Exception Testing

- Use `assertThrows` or AssertJ `assertThatThrownBy` for exception assertions
- Always verify both the exception type and message

```java
assertThatThrownBy(() -> orderService.findById("invalid-id"))
    .isInstanceOf(OrderNotFoundException.class)
    .hasMessageContaining("invalid-id");
```

## Mocking

- Use `@ExtendWith(MockitoExtension.class)` at the class level
- Use `@Mock` for dependencies and `@InjectMocks` for the class under test
- Prefer `given(...).willReturn(...)` (BDD-style) over `when(...).thenReturn(...)`
- Use `verify(...)` sparingly — only to confirm important interactions, not every call
- Never mock the class under test — only its dependencies

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    @Test
    void should_returnOrder_when_validIdProvided() {
        // given
        var order = new Order("ORD-001", "customer-1", List.of());
        given(orderRepository.findById("ORD-001")).willReturn(Optional.of(order));

        // when
        var result = orderService.findById("ORD-001");

        // then
        assertThat(result.getId()).isEqualTo("ORD-001");
    }
}
```

## Integration Tests

- Annotate integration tests with `@SpringBootTest`
- Use `@Tag("integration")` to separate from unit tests
- Use Testcontainers for database tests — never rely on an external database
- Use `@Sql` or `@BeforeEach` to set up test data — never depend on data from other tests
- Clean up after tests to ensure isolation

```java
@SpringBootTest
@Tag("integration")
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

## Controller Tests

- Use `@WebMvcTest(ControllerName.class)` for controller-layer tests
- Use `MockMvc` to make HTTP requests and verify responses
- Mock the service layer with `@MockBean`
- Verify response status, content type, and body structure

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    void should_returnCreated_when_validOrderSubmitted() throws Exception {
        // given
        var request = new CreateOrderRequest("cust-1", List.of(), BigDecimal.TEN);
        var response = new OrderResponse("ORD-001", "cust-1", BigDecimal.TEN);
        given(orderService.createOrder(any())).willReturn(response);

        // when & then
        mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("ORD-001"));
    }
}
```

## General Rules

- Tests must be independent — no shared mutable state, no ordering dependencies
- Each test class tests one production class
- Keep test data minimal — only what is needed for the assertion
- Do not use `Thread.sleep()` in tests — use `Awaitility` for async assertions
- Avoid `@SpringBootTest` when `@WebMvcTest`, `@DataJpaTest`, or plain unit tests suffice — faster feedback
