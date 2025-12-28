# Mocking External Dependencies

## Intent

Isolate integration tests from external services (HTTP clients, message queues, third-party APIs) by using Mockito mocks. Test your code's behavior without network dependencies.

---

## @InjectMock Pattern

Replace a CDI bean with a mock for all tests in the class:

```java
@QuarkusTest
class OrderServiceTest {

    @InjectMock
    PaymentGatewayClient paymentClient;

    @Test
    void shouldProcessOrderWhenPaymentSucceeds() {
        // Arrange
        when(paymentClient.charge(any(PaymentRequest.class)))
            .thenReturn(new PaymentResponse("txn-123", PaymentStatus.SUCCESS));

        // Act & Assert
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"customerId": "cust-1", "amount": 100.00}
                """)
        .when()
            .post("/orders")
        .then()
            .statusCode(201)
            .body("paymentStatus", equalTo("SUCCESS"));
    }

    @Test
    void shouldRejectOrderWhenPaymentFails() {
        when(paymentClient.charge(any()))
            .thenReturn(new PaymentResponse(null, PaymentStatus.DECLINED));

        given()
            .contentType(ContentType.JSON)
            .body("""
                {"customerId": "cust-1", "amount": 100.00}
                """)
        .when()
            .post("/orders")
        .then()
            .statusCode(422)
            .body("error", containsString("Payment declined"));
    }
}
```

---

## Verify Interactions

Confirm the mock was called with expected arguments:

```java
@Test
void shouldSendCorrectPaymentRequest() {
    when(paymentClient.charge(any()))
        .thenReturn(new PaymentResponse("txn-123", PaymentStatus.SUCCESS));

    given()
        .contentType(ContentType.JSON)
        .body("""
            {"customerId": "cust-1", "amount": 150.00}
            """)
    .when()
        .post("/orders")
    .then()
        .statusCode(201);

    // Verify the payment client was called correctly
    ArgumentCaptor<PaymentRequest> captor = ArgumentCaptor.forClass(PaymentRequest.class);
    verify(paymentClient).charge(captor.capture());

    PaymentRequest sentRequest = captor.getValue();
    assertThat(sentRequest.getAmount()).isEqualTo(new BigDecimal("150.00"));
    assertThat(sentRequest.getCustomerId()).isEqualTo("cust-1");
}
```

---

## Mock External HTTP Client

For REST clients (Quarkus REST Client, MicroProfile Rest Client):

```java
@RegisterRestClient(configKey = "inventory-api")
public interface InventoryClient {
    @GET
    @Path("/stock/{productId}")
    StockLevel getStock(@PathParam("productId") String productId);
}
```

```java
@QuarkusTest
class ProductAvailabilityTest {

    @InjectMock
    @RestClient
    InventoryClient inventoryClient;

    @Test
    void shouldShowInStockWhenAvailable() {
        when(inventoryClient.getStock("prod-1"))
            .thenReturn(new StockLevel("prod-1", 50));

        given()
            .pathParam("id", "prod-1")
        .when()
            .get("/products/{id}/availability")
        .then()
            .statusCode(200)
            .body("inStock", is(true))
            .body("quantity", equalTo(50));
    }

    @Test
    void shouldHandleInventoryServiceDown() {
        when(inventoryClient.getStock(any()))
            .thenThrow(new WebApplicationException("Service unavailable", 503));

        given()
            .pathParam("id", "prod-1")
        .when()
            .get("/products/{id}/availability")
        .then()
            .statusCode(200)
            .body("inStock", equalTo(null))
            .body("message", containsString("unavailable"));
    }
}
```

---

## Mock Async/Reactive Clients

```java
@InjectMock
NotificationService notificationService;

@Test
void shouldNotifyCustomerOnOrderComplete() {
    when(notificationService.sendEmail(any()))
        .thenReturn(Uni.createFrom().item(true));

    given()
        .pathParam("id", "order-123")
    .when()
        .post("/orders/{id}/complete")
    .then()
        .statusCode(200);

    verify(notificationService).sendEmail(argThat(email ->
        email.getSubject().contains("Order Complete")
    ));
}
```

---

## @InjectMock vs @Mock

| Annotation    | Use Case                       | Scope            |
| ------------- | ------------------------------ | ---------------- |
| `@InjectMock` | Replace CDI bean in container  | All test methods |
| `@Mock`       | Standalone mock, manual inject | Single test      |

Use `@InjectMock` for integration tests where you need the mock wired into the CDI container.

---

## Related Recipes

- [REST Integration Tests](./rest-integration-tests.md): Full endpoint testing
- [Testcontainers Setup](./testcontainers-setup.md): Real external dependencies
