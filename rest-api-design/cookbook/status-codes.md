# Status Codes

## Intent

Return appropriate HTTP status codes to clearly communicate request outcomes.

---

## Success Codes (2xx)

| Code | Name       | When to Use                     |
| ---- | ---------- | ------------------------------- |
| 200  | OK         | Successful GET, PUT, PATCH      |
| 201  | Created    | Successful POST (resource made) |
| 204  | No Content | Successful DELETE               |

### Examples

```java
// 200 OK - Successful retrieval
@GET
@Path("/{id}")
public Product get(@PathParam("id") Long id) {
    return productRepository.findByIdOptional(id)
        .orElseThrow(NotFoundException::new);  // Returns 200 with body
}

// 201 Created - New resource
@POST
public Response create(Product product) {
    productRepository.persist(product);
    return Response.status(Status.CREATED)
        .header("Location", "/products/" + product.getId())
        .entity(product)
        .build();
}

// 204 No Content - Successful delete
@DELETE
@Path("/{id}")
public Response delete(@PathParam("id") Long id) {
    productRepository.deleteById(id);
    return Response.noContent().build();
}
```

---

## Client Error Codes (4xx)

| Code | Name                 | When to Use                        |
| ---- | -------------------- | ---------------------------------- |
| 400  | Bad Request          | Malformed JSON, validation errors  |
| 401  | Unauthorized         | Missing or invalid authentication  |
| 403  | Forbidden            | Authenticated but not authorized   |
| 404  | Not Found            | Resource doesn't exist             |
| 409  | Conflict             | Duplicate resource, state conflict |
| 422  | Unprocessable Entity | Business rule violation            |

### Examples

```java
// 400 Bad Request - Validation error
@POST
public Response create(@Valid CreateProductRequest request) {
    // Bean Validation automatically returns 400 on constraint violations
}

// 404 Not Found
@GET
@Path("/{id}")
public Product get(@PathParam("id") Long id) {
    return productRepository.findByIdOptional(id)
        .orElseThrow(() -> new NotFoundException("Product not found: " + id));
}

// 409 Conflict - Duplicate
@POST
public Response create(Product product) {
    if (productRepository.existsByName(product.getName())) {
        return Response.status(Status.CONFLICT)
            .entity(new ErrorResponse("Product already exists"))
            .build();
    }
    // ...
}

// 422 Unprocessable Entity - Business rule
@POST
@Path("/{id}/ship")
public Response ship(@PathParam("id") Long id) {
    Order order = orderRepository.findById(id);
    if (order.getStatus() != OrderStatus.PAID) {
        return Response.status(422)
            .entity(new ErrorResponse("Cannot ship unpaid order"))
            .build();
    }
    // ...
}
```

---

## Server Error Codes (5xx)

| Code | Name                  | When to Use             |
| ---- | --------------------- | ----------------------- |
| 500  | Internal Server Error | Unexpected server error |
| 503  | Service Unavailable   | Temporary maintenance   |

---

## Error Response Structure

Consistent error format:

```java
public record ErrorResponse(
    String message,
    String code,
    Instant timestamp
) {
    public ErrorResponse(String message) {
        this(message, null, Instant.now());
    }
}

public record ValidationErrorResponse(
    String message,
    List<FieldError> violations
) {}

public record FieldError(
    String field,
    String message
) {}
```

### Exception Mapper

```java
@Provider
public class NotFoundExceptionMapper implements ExceptionMapper<NotFoundException> {

    @Override
    public Response toResponse(NotFoundException e) {
        return Response.status(Status.NOT_FOUND)
            .entity(new ErrorResponse(e.getMessage()))
            .build();
    }
}
```

---

## Quick Reference by Operation

| Operation    | Success | Common Errors |
| ------------ | ------- | ------------- |
| GET one      | 200     | 404           |
| GET list     | 200     | -             |
| POST create  | 201     | 400, 409, 422 |
| PUT replace  | 200     | 400, 404, 422 |
| PATCH update | 200     | 400, 404, 422 |
| DELETE       | 204     | 404           |

---

## Related Recipes

- [HTTP Methods](./http-methods.md): Method semantics
- [OpenAPI Documentation](./openapi-documentation.md): Document status codes
