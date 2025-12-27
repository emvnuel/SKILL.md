# JAX-RS Resource Design with CDD

## Principle

Resources are entry points — they should be simple (≤7 cognitive load points).

## Standard Resource Pattern

```java
@Path("/orders")
@RequestScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {

    @Inject
    private OrderService orderService;  // +1

    @POST
    public Response create(@Valid CreateOrderRequest request) {  // +1
        Order order = orderService.create(request);  // +1

        URI location = UriBuilder.fromResource(OrderResource.class)
            .path("{id}")
            .build(order.getId());

        return Response.created(location)
            .entity(OrderResponse.from(order))  // +1
            .build();
    }
    // Total: 4 points ✓

    @GET
    @Path("/{id}")
    public OrderResponse getById(@PathParam("id") Long id) {
        return orderService.findById(id)  // +1
            .map(OrderResponse::from)  // +1
            .orElseThrow(NotFoundException::new);  // +1
    }
    // Total: 3 points ✓
}
```

## Resource Responsibilities

| Do ✓                       | Don't ✗                  |
| -------------------------- | ------------------------ |
| Receive and validate input | Business logic           |
| Call service layer         | Direct repository access |
| Transform to response      | Complex conditionals     |
| Set HTTP status/headers    | Multi-step orchestration |

## Error Handling Pattern

Keep resources clean by using exception mappers:

```java
// Resource stays simple
@POST
public Response create(@Valid CreateOrderRequest request) {
    Order order = orderService.create(request);  // May throw
    return Response.created(/*...*/).entity(OrderResponse.from(order)).build();
}

// Exception mapper handles errors
@Provider
public class BusinessExceptionMapper
    implements ExceptionMapper<BusinessRuleException> {

    @Override
    public Response toResponse(BusinessRuleException e) {
        return Response.status(422)
            .entity(new ErrorResponse(e.getCode(), e.getMessage()))
            .build();
    }
}
```

## Query Parameters

```java
@GET
public ProductListResponse search(
        @QueryParam("q") String query,
        @QueryParam("category") Long categoryId,
        @QueryParam("minPrice") BigDecimal minPrice,
        @QueryParam("maxPrice") BigDecimal maxPrice,
        @QueryParam("page") @DefaultValue("0") int page,
        @QueryParam("size") @DefaultValue("20") int size) {

    // Encapsulate search criteria
    ProductSearchCriteria criteria = new ProductSearchCriteria(
        query, categoryId, minPrice, maxPrice, page, size);

    return productService.search(criteria);  // +1
}
// Total: 2 points ✓
```

## Subresources for Nested Paths

```java
@Path("/orders/{orderId}/items")
@RequestScoped
public class OrderItemResource {

    @PathParam("orderId")
    private Long orderId;

    @Inject
    private OrderItemService itemService;

    @POST
    public Response addItem(@Valid AddItemRequest request) {
        OrderItem item = itemService.addToOrder(orderId, request);
        return Response.ok(OrderItemResponse.from(item)).build();
    }

    @DELETE
    @Path("/{itemId}")
    public Response removeItem(@PathParam("itemId") Long itemId) {
        itemService.removeFromOrder(orderId, itemId);
        return Response.noContent().build();
    }
}
```

## Async Operations

```java
@POST
@Path("/{id}/process")
public Response processAsync(@PathParam("id") Long id) {
    String jobId = orderService.startProcessing(id);  // Returns job ID immediately

    URI statusUri = UriBuilder.fromResource(JobResource.class)
        .path("{jobId}")
        .build(jobId);

    return Response.accepted()
        .header("Location", statusUri)
        .entity(new JobStartedResponse(jobId))
        .build();
}
```

## Cognitive Load Check

```java
// ❌ Too much logic in resource (8+ points)
@POST
public Response create(@Valid CreateOrderRequest request) {
    Customer customer = customerRepo.findById(request.customerId())  // +1
        .orElseThrow(NotFoundException::new);  // +1

    if (!customer.isActive()) {  // +1
        return Response.status(422).entity("Inactive customer").build();
    }

    for (OrderItemRequest item : request.items()) {  // +1
        if (!inventory.hasStock(item.productId(), item.quantity())) {  // +1
            return Response.status(422).entity("Out of stock").build();
        }
    }
    // ... more logic
}

// ✓ Move logic to service
@POST
public Response create(@Valid CreateOrderRequest request) {
    Order order = orderService.create(request);  // +1, throws if invalid
    return Response.created(/*...*/).entity(OrderResponse.from(order)).build();
}
```

## Related Entries

- [boundary-separation](boundary-separation.md) - DTO patterns
- [cdi-bean-design](cdi-bean-design.md) - Service layer design
