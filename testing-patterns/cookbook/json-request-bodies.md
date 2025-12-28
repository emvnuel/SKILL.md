# JSON Request Bodies

## Intent

Use raw JSON strings for request bodies instead of serializing Java objects. Provides exact control over payloads and makes tests readable and self-contained.

---

## Text Block Pattern

Java text blocks (triple quotes) for inline JSON:

```java
@Test
void shouldCreateOrder() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            {
                "customerId": "cust-123",
                "items": [
                    {"productId": "prod-1", "quantity": 2},
                    {"productId": "prod-2", "quantity": 1}
                ],
                "shippingAddress": {
                    "street": "123 Main St",
                    "city": "Springfield",
                    "zip": "12345"
                }
            }
            """)
    .when()
        .post("/orders")
    .then()
        .statusCode(201);
}
```

---

## String Interpolation

For dynamic values, use `String.formatted()`:

```java
@Test
void shouldCreateOrderForCustomer() {
    String customerId = "cust-" + UUID.randomUUID();
    String productId = existingProduct.getId();

    given()
        .contentType(ContentType.JSON)
        .body("""
            {
                "customerId": "%s",
                "items": [{"productId": "%s", "quantity": 1}]
            }
            """.formatted(customerId, productId))
    .when()
        .post("/orders")
    .then()
        .statusCode(201)
        .body("customerId", equalTo(customerId));
}
```

---

## Loading JSON from Files

For complex or reusable payloads:

### Test Resource

`src/test/resources/payloads/create-order.json`:

```json
{
  "customerId": "cust-123",
  "items": [{ "productId": "prod-1", "quantity": 2 }]
}
```

### Test Class

```java
@Test
void shouldCreateOrderFromFile() throws Exception {
    String json = Files.readString(
        Path.of("src/test/resources/payloads/create-order.json")
    );

    given()
        .contentType(ContentType.JSON)
        .body(json)
    .when()
        .post("/orders")
    .then()
        .statusCode(201);
}
```

---

## JsonPath Response Assertions

Assert on response JSON structure:

```java
@Test
void shouldReturnOrderDetails() {
    given()
        .pathParam("id", "order-123")
    .when()
        .get("/orders/{id}")
    .then()
        .statusCode(200)
        .body("id", equalTo("order-123"))
        .body("items.size()", equalTo(2))
        .body("items[0].productId", equalTo("prod-1"))
        .body("items.find { it.productId == 'prod-2' }.quantity", equalTo(1))
        .body("total", closeTo(99.99, 0.01));
}
```

---

## Complex JsonPath Queries

```java
@Test
void shouldFilterExpensiveItems() {
    given()
    .when()
        .get("/orders/order-123")
    .then()
        .body("items.findAll { it.price > 50 }.size()", greaterThan(0))
        .body("items.collect { it.price }.sum()", greaterThan(100.0))
        .body("items*.productId", hasItems("prod-1", "prod-2"));
}
```

---

## Testing Validation Errors

```java
@Test
void shouldRejectInvalidJson() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            {
                "customerId": "",
                "items": []
            }
            """)
    .when()
        .post("/orders")
    .then()
        .statusCode(400)
        .body("violations.field", hasItems("customerId", "items"))
        .body("violations.find { it.field == 'customerId' }.message",
              containsString("must not be blank"));
}
```

---

## Related Recipes

- [REST Integration Tests](./rest-integration-tests.md): Full test patterns
- [Test Structure](./test-structure.md): Organizing tests
