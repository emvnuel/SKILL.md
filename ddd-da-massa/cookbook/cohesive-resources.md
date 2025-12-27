# 100% Cohesive JAX-RS Resources

## Principle

> Every method in a resource must use ALL injected dependencies.

## Problem

Resources with partially used dependencies are not cohesive.

```java
// ❌ Not cohesive - different methods use different deps
@Path("/orders")
@RequestScoped
public class OrderResource {

    @Inject private OrderRepository orders;      // Used by 3 methods
    @Inject private CustomerRepository customers; // Used by 1 method
    @Inject private ShippingService shipping;     // Used by 1 method
    @Inject private ReportService reports;        // Used by 1 method

    @GET
    public List<OrderResponse> list() {
        return orders.findAll().stream()/*...*/;  // orders only
    }

    @POST
    public Response create(@Valid CreateOrderRequest req) {
        // Uses orders AND customers
    }

    @POST @Path("/{id}/ship")
    public Response ship(@PathParam("id") Long id) {
        // Uses orders AND shipping
    }

    @GET @Path("/report")
    public ReportResponse report() {
        // Uses orders AND reports
    }
}
```

## Solution

Split into focused, 100% cohesive resources.

```java
@Path("/orders")
@RequestScoped
public class OrderCrudResource {

    @Inject private OrderRepository orders;
    @Inject private CustomerRepository customers;

    @GET
    public List<OrderResponse> list() {
        return orders.findAll().stream()
            .map(OrderResponse::from)
            .toList();
    }

    @POST
    public Response create(@Valid CreateOrderRequest req) {
        Customer customer = customers.findById(req.customerId())
            .orElseThrow(NotFoundException::new);
        Order order = req.toEntity(customer);
        orders.save(order);
        return Response.created(/*...*/).build();
    }
}
// Both methods use BOTH dependencies ✓

@Path("/orders/{orderId}/shipping")
@RequestScoped
public class OrderShippingResource {

    @Inject private OrderRepository orders;
    @Inject private ShippingService shipping;

    @POST
    public Response ship(@PathParam("orderId") Long orderId) {
        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);
        shipping.ship(order);
        return Response.ok().build();
    }
}
// Single method uses BOTH dependencies ✓

@Path("/orders/reports")
@RequestScoped
public class OrderReportResource {

    @Inject private OrderRepository orders;
    @Inject private ReportService reports;

    @GET
    public ReportResponse generate() {
        List<Order> all = orders.findAll();
        return reports.generateOrderReport(all);
    }
}
// Single method uses BOTH dependencies ✓
```

## Tradeoff

**More resource classes** — This is OK! More focused files that distribute cognitive load properly is better than fewer bloated files.

## Cognitive Load Check

Each focused resource should stay within 7 points:

| Element              | Points         |
| -------------------- | -------------- |
| OrderRepository      | +1             |
| CustomerRepository   | +1             |
| orElseThrow (branch) | +1             |
| req.toEntity()       | +1             |
| **Total**            | **4 points ✓** |

## Package Organization

```
resources/
├── orders/
│   ├── OrderCrudResource.java
│   ├── OrderShippingResource.java
│   └── OrderReportResource.java
└── customers/
    └── CustomerResource.java
```

## Related Entries

- [domain-service-controller](domain-service-controller.md) - When to merge controller + service
- [cohesive-services](cohesive-services.md) - Same principle for services
