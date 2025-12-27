# Domain Service Controller Pattern

## Principle

> When a controller is small and cohesive, it can also act as the domain service — no separate service layer needed.

## When to Use

- Simple CRUD operations
- Single-step business flows
- When adding a service would just be a pass-through

## Example

```java
@Path("/payments/pagseguro/{orderId}")
@RequestScoped
public class PagseguroCallbackResource {

    @Inject
    private OrderRepository orders;

    @Inject
    private PaymentRepository payments;

    @Inject
    private Event<NewPaymentEvent> paymentEvents;

    @POST
    @Transactional
    public void processCallback(
            @PathParam("orderId") Long orderId,
            @Valid PagseguroCallbackRequest request) {

        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);

        // Validation can be done via Validator on Binder
        order.validateForPayment();

        Payment payment = request.toPayment(order);
        payments.save(payment);

        paymentEvents.fire(new NewPaymentEvent(payment));
    }
}
```

## Cognitive Load Analysis

| Element                  | Points       |
| ------------------------ | ------------ |
| OrderRepository          | +1           |
| PaymentRepository        | +1           |
| Event<NewPaymentEvent>   | +1           |
| PagseguroCallbackRequest | +1           |
| Order                    | +1           |
| orElseThrow (branch)     | +1           |
| Payment                  | +1           |
| NewPaymentEvent          | +1           |
| **Total**                | **8 points** |

Slightly above 7, but acceptable for this pattern.

## When to Extract a Service

Extract when:

- Multiple controllers need the same logic
- Cognitive load exceeds 9 points
- Complex orchestration required

```java
// If you need to extract, the service is just the extracted part
@ApplicationScoped
public class PaymentProcessor {

    @Inject private PaymentRepository payments;
    @Inject private Event<NewPaymentEvent> paymentEvents;

    @Transactional
    public void process(Order order, PaymentRequest request) {
        Payment payment = request.toPayment(order);
        payments.save(payment);
        paymentEvents.fire(new NewPaymentEvent(payment));
    }
}

// Controller becomes simpler
@Path("/payments/pagseguro/{orderId}")
@RequestScoped
public class PagseguroCallbackResource {

    @Inject private OrderRepository orders;
    @Inject private PaymentProcessor processor;

    @POST
    @Transactional
    public void processCallback(
            @PathParam("orderId") Long orderId,
            @Valid PagseguroCallbackRequest request) {
        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);
        processor.process(order, request);
    }
}
```

## Benefits

- Less indirection
- Fewer files to navigate
- Clear request-to-action mapping
- Framework handles HTTP, you handle business

## Related Entries

- [cohesive-resources](cohesive-resources.md) - 100% cohesion rule
- [splitting-load](splitting-load.md) - When to extract
