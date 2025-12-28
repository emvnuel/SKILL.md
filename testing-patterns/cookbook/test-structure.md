# Test Structure & Lifecycle

## Intent

Organize integration tests with JUnit 5 features for readability, maintainability, and proper setup/teardown.

---

## Basic Structure

```java
@QuarkusTest
class OrderResourceTest {

    @Inject
    OrderRepository orderRepository;

    @BeforeEach
    @Transactional
    void setUp() {
        orderRepository.deleteAll();
    }

    @Test
    void shouldCreateOrder() {
        // Given-When-Then structure
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"customerId": "cust-1", "items": []}
                """)
        .when()
            .post("/orders")
        .then()
            .statusCode(201);
    }
}
```

---

## @Nested Classes

Group related scenarios:

```java
@QuarkusTest
class ProductResourceTest {

    @Nested
    class GetProduct {

        @Test
        void shouldReturnProduct() {
            given()
                .pathParam("id", existingProductId)
            .when()
                .get("/products/{id}")
            .then()
                .statusCode(200);
        }

        @Test
        void shouldReturn404WhenNotFound() {
            given()
                .pathParam("id", "non-existent")
            .when()
                .get("/products/{id}")
            .then()
                .statusCode(404);
        }
    }

    @Nested
    class CreateProduct {

        @Test
        void shouldCreateWithValidInput() {
            // ...
        }

        @Test
        void shouldRejectInvalidInput() {
            // ...
        }
    }
}
```

---

## Test Naming Conventions

Use descriptive method names:

```java
// Pattern: should[ExpectedBehavior]When[Condition]

@Test
void shouldReturnOrderWhenExists() { }

@Test
void shouldReturn404WhenOrderNotFound() { }

@Test
void shouldRejectOrderWhenInsufficientStock() { }

@Test
void shouldSendNotificationWhenOrderCompleted() { }
```

---

## Setup and Teardown

```java
@QuarkusTest
class IntegrationTest {

    @BeforeAll
    static void beforeAll() {
        // One-time setup (e.g., external service config)
    }

    @AfterAll
    static void afterAll() {
        // One-time cleanup
    }

    @BeforeEach
    void setUp() {
        // Per-test setup
    }

    @AfterEach
    void tearDown() {
        // Per-test cleanup
    }
}
```

---

## Test Data Builders

Create readable test data:

```java
class OrderBuilder {
    private String customerId = "default-customer";
    private List<OrderItem> items = new ArrayList<>();

    static OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    OrderBuilder forCustomer(String customerId) {
        this.customerId = customerId;
        return this;
    }

    OrderBuilder withItem(String productId, int quantity) {
        items.add(new OrderItem(productId, quantity));
        return this;
    }

    String asJson() {
        // Return JSON representation
        return """
            {
                "customerId": "%s",
                "items": %s
            }
            """.formatted(customerId, itemsToJson());
    }
}

// Usage
given()
    .contentType(ContentType.JSON)
    .body(anOrder()
        .forCustomer("cust-123")
        .withItem("prod-1", 2)
        .asJson())
.when()
    .post("/orders");
```

---

## Conditional Tests

```java
@Test
@EnabledIfSystemProperty(named = "integration", matches = "true")
void shouldConnectToExternalService() {
    // Only runs with -Dintegration=true
}

@Test
@DisabledOnOs(OS.WINDOWS)
void shouldHandleUnixPaths() {
    // Skip on Windows
}

@Test
@EnabledIfEnvironmentVariable(named = "CI", matches = "true")
void shouldRunOnlyInCI() {
    // CI-specific test
}
```

---

## Test Tagging

```java
@Tag("slow")
@Test
void shouldProcessLargeDataset() {
    // Excluded from quick test runs
}

@Tag("integration")
@QuarkusTest
class ExternalServiceTest {
    // Run with: mvn test -Dgroups=integration
}
```

---

## Related Recipes

- [REST Integration Tests](./rest-integration-tests.md): Full test patterns
- [Mocking Dependencies](./mocking-dependencies.md): Isolate external services
