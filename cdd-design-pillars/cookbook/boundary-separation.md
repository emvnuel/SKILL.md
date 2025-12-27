# Boundary Separation with DTOs

## Principle

> Separate external system boundaries from the domain core. Don't bind request parameters directly to domain objects. Don't serialize domain objects to responses.

## Request DTOs

### Simple Record-Based DTO

```java
public record CreateProductRequest(
    @NotBlank @Size(max = 100)
    String name,

    @NotNull @Positive
    BigDecimal price,

    @Size(max = 1000)
    String description,

    @NotNull
    Long categoryId
) {
    public Product toEntity(Category category) {
        return new Product(name, price, description, category);
    }
}
```

### DTO with Validation Logic

```java
public record TransferRequest(
    @NotBlank String fromAccountId,
    @NotBlank String toAccountId,
    @NotNull @Positive BigDecimal amount,
    String description
) {
    // Compact constructor for cross-field validation
    public TransferRequest {
        if (fromAccountId.equals(toAccountId)) {
            throw new IllegalArgumentException(
                "Source and destination must be different");
        }
    }
}
```

### Nested Request DTOs

```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @Valid AddressRequest shippingAddress
) {
    public Order toEntity(Customer customer) {
        return new Order(
            customer,
            items.stream().map(OrderItemRequest::toEntity).toList(),
            shippingAddress.toEntity()
        );
    }
}

public record OrderItemRequest(
    @NotBlank String productId,
    @Positive int quantity
) {
    public OrderItem toEntity() {
        return new OrderItem(productId, quantity);
    }
}

public record AddressRequest(
    @NotBlank String street,
    @NotBlank String city,
    @NotBlank String zipCode,
    @NotBlank String country
) {
    public Address toEntity() {
        return new Address(street, city, zipCode, country);
    }
}
```

## Response DTOs

### Static Factory Pattern

```java
public record ProductResponse(
    String id,
    String name,
    String priceFormatted,
    String category,
    boolean available
) {
    public static ProductResponse from(Product product) {
        return new ProductResponse(
            product.getId().toString(),
            product.getName(),
            formatPrice(product.getPrice()),
            product.getCategory().getName(),
            product.isAvailable()
        );
    }

    private static String formatPrice(BigDecimal price) {
        return NumberFormat.getCurrencyInstance().format(price);
    }
}
```

### List Responses

```java
public record ProductListResponse(
    List<ProductResponse> products,
    int total,
    int page,
    int pageSize
) {
    public static ProductListResponse from(
            List<Product> products, int page, int pageSize, long total) {
        return new ProductListResponse(
            products.stream().map(ProductResponse::from).toList(),
            (int) total,
            page,
            pageSize
        );
    }
}
```

### Partial/Projection Responses

```java
// Lightweight response for lists
public record ProductSummaryResponse(
    String id,
    String name,
    String price
) {
    public static ProductSummaryResponse from(Product product) {
        return new ProductSummaryResponse(
            product.getId().toString(),
            product.getName(),
            product.getPrice().toString()
        );
    }
}

// Full response for detail views
public record ProductDetailResponse(
    String id,
    String name,
    String description,
    String price,
    CategoryResponse category,
    List<ReviewResponse> recentReviews
) {
    public static ProductDetailResponse from(Product product) {
        // ...
    }
}
```

## JAX-RS Resource Pattern

```java
@Path("/products")
@RequestScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ProductResource {

    @Inject
    private ProductService productService;

    @POST
    public Response create(@Valid CreateProductRequest request) {
        Product product = productService.create(request);

        URI location = UriBuilder.fromResource(ProductResource.class)
            .path("{id}")
            .build(product.getId());

        return Response.created(location)
            .entity(ProductResponse.from(product))
            .build();
    }

    @GET
    @Path("/{id}")
    public ProductDetailResponse getById(@PathParam("id") Long id) {
        return productService.findById(id)
            .map(ProductDetailResponse::from)
            .orElseThrow(NotFoundException::new);
    }

    @GET
    public ProductListResponse list(
            @QueryParam("page") @DefaultValue("0") int page,
            @QueryParam("size") @DefaultValue("20") int size) {
        return productService.findAll(page, size);
    }
}
```

## Benefits

| Aspect        | Benefit                           |
| ------------- | --------------------------------- |
| Security      | Can't inject internal fields      |
| Versioning    | API changes independent of domain |
| Validation    | Centralized on DTOs               |
| Documentation | DTOs self-document API contract   |

## Related Entries

- [protect-boundaries](protect-boundaries.md) - Core principle
- [rest-resource-design](rest-resource-design.md) - Resource patterns
