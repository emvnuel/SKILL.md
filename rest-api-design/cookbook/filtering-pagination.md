# Filtering, Sorting & Pagination

## Intent

Use query parameters to filter, sort, and paginate collections. Return only the data clients need.

---

## Query Parameter Filtering

```java
@GET
@Path("/products")
public List<Product> list(
    @QueryParam("category") String category,
    @QueryParam("minPrice") BigDecimal minPrice,
    @QueryParam("maxPrice") BigDecimal maxPrice,
    @QueryParam("inStock") Boolean inStock
) {
    return productRepository.find(category, minPrice, maxPrice, inStock);
}
```

**Request**: `GET /products?category=electronics&minPrice=100&inStock=true`

---

## Sorting

```java
@GET
@Path("/products")
public List<Product> list(
    @QueryParam("sort") @DefaultValue("name") String sortField,
    @QueryParam("order") @DefaultValue("asc") String sortOrder
) {
    Sort sort = sortOrder.equalsIgnoreCase("desc")
        ? Sort.descending(sortField)
        : Sort.ascending(sortField);

    return productRepository.listAll(sort);
}
```

**Request**: `GET /products?sort=price&order=desc`

### Multiple Sort Fields

```java
// sort=price,-name (price asc, name desc)
@QueryParam("sort") List<String> sortFields

// Parse: + or no prefix = asc, - prefix = desc
```

---

## Offset Pagination

Simple page/size pagination:

```java
@GET
@Path("/products")
public PaginatedResponse<Product> list(
    @QueryParam("page") @DefaultValue("0") int page,
    @QueryParam("size") @DefaultValue("20") int size
) {
    List<Product> items = productRepository.findAll()
        .page(page, size)
        .list();

    long totalItems = productRepository.count();
    int totalPages = (int) Math.ceil((double) totalItems / size);

    return new PaginatedResponse<>(items, page, size, totalItems, totalPages);
}
```

### Response Structure

```java
public record PaginatedResponse<T>(
    List<T> items,
    int page,
    int size,
    long totalItems,
    int totalPages
) {}
```

```json
{
    "items": [...],
    "page": 0,
    "size": 20,
    "totalItems": 150,
    "totalPages": 8
}
```

---

## Cursor-Based Pagination

Better for large datasets and real-time data:

```java
@GET
@Path("/products")
public CursorResponse<Product> list(
    @QueryParam("cursor") String cursor,
    @QueryParam("limit") @DefaultValue("20") int limit
) {
    Long afterId = cursor != null ? decodeCursor(cursor) : 0L;

    List<Product> items = productRepository
        .find("id > ?1", afterId)
        .page(0, limit + 1)  // Fetch one extra to check hasMore
        .list();

    boolean hasMore = items.size() > limit;
    if (hasMore) {
        items = items.subList(0, limit);
    }

    String nextCursor = hasMore
        ? encodeCursor(items.get(items.size() - 1).getId())
        : null;

    return new CursorResponse<>(items, nextCursor, hasMore);
}
```

### Response Structure

```java
public record CursorResponse<T>(
    List<T> items,
    String nextCursor,
    boolean hasMore
) {}
```

---

## Combined Example

```java
@GET
@Path("/products")
public PaginatedResponse<Product> list(
    // Filtering
    @QueryParam("category") String category,
    @QueryParam("minPrice") BigDecimal minPrice,
    // Sorting
    @QueryParam("sort") @DefaultValue("name") String sort,
    @QueryParam("order") @DefaultValue("asc") String order,
    // Pagination
    @QueryParam("page") @DefaultValue("0") int page,
    @QueryParam("size") @DefaultValue("20") int size
) {
    // Build query with all parameters
}
```

**Request**: `GET /products?category=electronics&sort=price&order=desc&page=2&size=10`

---

## Limit Maximum Page Size

```java
private static final int MAX_PAGE_SIZE = 100;

@GET
public List<Product> list(
    @QueryParam("size") @DefaultValue("20") int size
) {
    int safeSize = Math.min(size, MAX_PAGE_SIZE);
    // ...
}
```

---

## Related Recipes

- [Endpoint Design](./endpoint-design.md): URL structure
- [OpenAPI Documentation](./openapi-documentation.md): Document query params
