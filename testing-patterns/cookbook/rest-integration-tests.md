# REST Integration Tests

## Intent

Test full HTTP request/response cycle through REST endpoints. Validate status codes, response bodies, and side effects with real application context.

---

## Basic Test Structure

```java
@QuarkusTest
class ProductResourceTest {

    @Test
    void shouldReturnProduct() {
        given()
            .pathParam("id", "prod-123")
        .when()
            .get("/products/{id}")
        .then()
            .statusCode(200)
            .body("id", equalTo("prod-123"))
            .body("name", notNullValue());
    }
}
```

---

## POST with JSON Body

Use text blocks for raw JSON input:

```java
@Test
void shouldCreateProduct() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            {
                "name": "Widget",
                "price": 29.99,
                "category": "ELECTRONICS"
            }
            """)
    .when()
        .post("/products")
    .then()
        .statusCode(201)
        .header("Location", containsString("/products/"))
        .body("id", notNullValue())
        .body("name", equalTo("Widget"));
}
```

---

## Testing Error Responses

```java
@Test
void shouldReturn404WhenProductNotFound() {
    given()
        .pathParam("id", "non-existent")
    .when()
        .get("/products/{id}")
    .then()
        .statusCode(404)
        .body("message", containsString("not found"));
}

@Test
void shouldReturn400ForInvalidInput() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            {
                "name": "",
                "price": -10
            }
            """)
    .when()
        .post("/products")
    .then()
        .statusCode(400)
        .body("violations", hasSize(greaterThan(0)));
}
```

---

## Testing with Authentication

```java
@Test
void shouldRequireAuthentication() {
    given()
    .when()
        .get("/admin/users")
    .then()
        .statusCode(401);
}

@Test
void shouldAllowAuthenticatedAccess() {
    given()
        .header("Authorization", "Bearer " + getValidToken())
    .when()
        .get("/admin/users")
    .then()
        .statusCode(200);
}
```

---

## Response Body Assertions

### JsonPath Matchers

```java
@Test
void shouldReturnProductList() {
    given()
    .when()
        .get("/products")
    .then()
        .statusCode(200)
        .body("size()", greaterThan(0))
        .body("[0].id", notNullValue())
        .body("findAll { it.price > 100 }.size()", greaterThan(0))
        .body("name", hasItems("Widget", "Gadget"));
}
```

### Extract Response for Further Assertions

```java
@Test
void shouldCreateAndReturnNewProduct() {
    String id = given()
        .contentType(ContentType.JSON)
        .body("""
            {"name": "New Product", "price": 50.00}
            """)
    .when()
        .post("/products")
    .then()
        .statusCode(201)
        .extract()
        .path("id");

    // Verify created resource
    given()
        .pathParam("id", id)
    .when()
        .get("/products/{id}")
    .then()
        .statusCode(200)
        .body("name", equalTo("New Product"));
}
```

---

## Query Parameters

```java
@Test
void shouldFilterByCategory() {
    given()
        .queryParam("category", "ELECTRONICS")
        .queryParam("minPrice", 100)
        .queryParam("page", 0)
        .queryParam("size", 10)
    .when()
        .get("/products")
    .then()
        .statusCode(200)
        .body("every { it.category == 'ELECTRONICS' }", is(true));
}
```

---

## Related Recipes

- [JSON Request Bodies](./json-request-bodies.md): Advanced JSON payload patterns
- [Mocking Dependencies](./mocking-dependencies.md): Isolate external services
- [Test Structure](./test-structure.md): JUnit 5 organization
