# HTTP Methods

## Intent

Use HTTP methods correctly for CRUD operations. Understand idempotency and when to use each method.

---

## Method Overview

| Method   | CRUD    | Idempotent | Safe | Request Body |
| -------- | ------- | ---------- | ---- | ------------ |
| `GET`    | Read    | Yes        | Yes  | No           |
| `POST`   | Create  | No         | No   | Yes          |
| `PUT`    | Replace | Yes        | No   | Yes          |
| `PATCH`  | Update  | Yes        | No   | Yes          |
| `DELETE` | Delete  | Yes        | No   | No           |

- **Idempotent**: Same request multiple times = same result
- **Safe**: Doesn't modify server state

---

## GET - Retrieve Resources

```java
@GET
@Path("/products")
public List<Product> list() {
    return productRepository.listAll();
}

@GET
@Path("/products/{id}")
public Product getById(@PathParam("id") Long id) {
    return productRepository.findByIdOptional(id)
        .orElseThrow(NotFoundException::new);
}
```

**Response**: 200 OK with body, or 404 Not Found

---

## POST - Create Resource

```java
@POST
@Path("/products")
public Response create(CreateProductRequest request) {
    Product product = productService.create(request);

    return Response.status(Status.CREATED)
        .header("Location", "/products/" + product.getId())
        .entity(product)
        .build();
}
```

**Response**: 201 Created with Location header

---

## PUT - Replace Entire Resource

Replace the entire resource. All fields required.

```java
@PUT
@Path("/products/{id}")
public Product replace(
    @PathParam("id") Long id,
    Product product
) {
    Product existing = productRepository.findByIdOptional(id)
        .orElseThrow(NotFoundException::new);

    existing.setName(product.getName());
    existing.setPrice(product.getPrice());
    existing.setCategory(product.getCategory());

    return existing;
}
```

**Response**: 200 OK with updated resource

---

## PATCH - Partial Update

Update only provided fields.

```java
@PATCH
@Path("/products/{id}")
public Product partialUpdate(
    @PathParam("id") Long id,
    Map<String, Object> updates
) {
    Product product = productRepository.findByIdOptional(id)
        .orElseThrow(NotFoundException::new);

    if (updates.containsKey("name")) {
        product.setName((String) updates.get("name"));
    }
    if (updates.containsKey("price")) {
        product.setPrice(new BigDecimal(updates.get("price").toString()));
    }

    return product;
}
```

**Response**: 200 OK with updated resource

---

## DELETE - Remove Resource

```java
@DELETE
@Path("/products/{id}")
public Response delete(@PathParam("id") Long id) {
    boolean deleted = productRepository.deleteById(id);

    if (!deleted) {
        throw new NotFoundException();
    }

    return Response.noContent().build();
}
```

**Response**: 204 No Content, or 404 Not Found

---

## Idempotency

**Idempotent** means calling the same request multiple times produces the same result.

```java
// PUT is idempotent - same result each time
PUT /products/123 {"name": "Widget", "price": 10}

// POST is NOT idempotent - creates new resource each time
POST /products {"name": "Widget", "price": 10}
```

Use idempotency keys for safe retries on POST:

```java
@POST
@Path("/orders")
public Response create(
    @HeaderParam("Idempotency-Key") String idempotencyKey,
    CreateOrderRequest request
) {
    // Check if order with this key already exists
    Optional<Order> existing = orderRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return Response.ok(existing.get()).build();
    }
    // Create new order
}
```

---

## Related Recipes

- [Status Codes](./status-codes.md): Response codes for each method
- [Endpoint Design](./endpoint-design.md): URL structure
