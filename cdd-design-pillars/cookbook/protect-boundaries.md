# Protect System Boundaries

## Principle

> We protect system boundaries as if there's no tomorrow. The more external the boundary, the more protection it needs.

## Problem

Binding external data directly to domain objects creates security vulnerabilities and couples your domain to external formats.

```java
// ❌ BAD: Domain object directly bound to request
@POST
public Response create(Order order) {  // Attacker can set id, status, etc.
    return Response.ok(orderRepository.save(order)).build();
}

// ❌ BAD: Domain object serialized to response
@GET
@Path("/{id}")
public Order get(@PathParam("id") Long id) {
    return orderRepository.findById(id);  // Exposes internal structure
}
```

## Solution

Separate external boundaries from the domain core using DTOs.

```java
// Request DTO - validates and sanitizes input
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @Size(max = 500) String notes
) {
    public Order toEntity(Customer customer) {
        return new Order(customer, mapItems(), notes);
    }

    private List<OrderItem> mapItems() {
        return items.stream()
            .map(OrderItemRequest::toEntity)
            .toList();
    }
}

// Response DTO - controls what's exposed
public record OrderResponse(
    String orderId,
    String status,
    List<OrderItemResponse> items,
    String totalFormatted
) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId().toString(),
            order.getStatus().name(),
            order.getItems().stream()
                .map(OrderItemResponse::from)
                .toList(),
            order.getFormattedTotal()
        );
    }
}
```

### JAX-RS Resource

```java
@Path("/orders")
@RequestScoped
public class OrderResource {

    @Inject
    private OrderService orderService;

    @Inject
    private CustomerRepository customers;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public Response create(@Valid CreateOrderRequest request) {
        Customer customer = customers.findById(request.customerId())
            .orElseThrow(CustomerNotFoundException::new);

        Order order = request.toEntity(customer);
        Order saved = orderService.create(order);

        URI location = UriBuilder.fromResource(OrderResource.class)
            .path("{id}").build(saved.getId());

        return Response.created(location)
            .entity(OrderResponse.from(saved))
            .build();
    }

    @GET
    @Path("/{id}")
    public OrderResponse get(@PathParam("id") Long id) {
        return orderRepository.findById(id)
            .map(OrderResponse::from)
            .orElseThrow(OrderNotFoundException::new);
    }
}
```

## Cognitive Load Analysis

| Element              | Points         |
| -------------------- | -------------- |
| OrderService         | +1             |
| CustomerRepository   | +1             |
| request.toEntity()   | +1             |
| if orElseThrow       | +1             |
| OrderResponse.from() | +1             |
| **Total**            | **5 points ✓** |

## Benefits

- **Security**: Attackers can't inject internal fields
- **Flexibility**: API format independent of domain model
- **Versioning**: Change API without changing domain
- **Validation**: Centralized input validation on DTOs

## Related Entries

- [boundary-separation](boundary-separation.md) - Detailed DTO patterns
- [valid-state-only](valid-state-only.md) - Entity construction
