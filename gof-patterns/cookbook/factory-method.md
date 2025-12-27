# Factory Method Pattern - Jakarta EE Cookbook

## Intent

Define an interface for creating an object, but let subclasses or CDI decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses or the container.

## Jakarta EE Implementation

### 1. Basic CDI Factory Method

```java
// Product interface
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
}

// Concrete products
@ApplicationScoped
@Named("credit-card")
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // Credit card processing logic
        return new PaymentResult(true, "CC-TXN-" + UUID.randomUUID());
    }
}

@ApplicationScoped
@Named("paypal")
public class PayPalProcessor implements PaymentProcessor {
    @Inject
    @ConfigProperty(name = "paypal.api.key")
    String apiKey;

    @Override
    public PaymentResult process(PaymentRequest request) {
        // PayPal processing logic
        return new PaymentResult(true, "PP-TXN-" + UUID.randomUUID());
    }
}

@ApplicationScoped
@Named("pix")
public class PixProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // PIX (Brazilian instant payment) logic
        return new PaymentResult(true, "PIX-TXN-" + UUID.randomUUID());
    }
}
```

### 2. Factory Service Using CDI Instance

```java
@ApplicationScoped
public class PaymentProcessorFactory {

    @Inject
    @Any
    Instance<PaymentProcessor> processors;

    public PaymentProcessor getProcessor(String type) {
        return processors.select(new NamedLiteral(type)).get();
    }

    public PaymentProcessor getProcessor(PaymentMethod method) {
        return processors.select(new NamedLiteral(method.getProcessorName())).get();
    }

    public boolean isSupported(String type) {
        return processors.select(new NamedLiteral(type)).isResolvable();
    }

    public List<String> getSupportedProcessors() {
        return processors.stream()
            .map(p -> p.getClass().getAnnotation(Named.class).value())
            .toList();
    }
}

// NamedLiteral helper for programmatic qualifier lookup
public class NamedLiteral extends AnnotationLiteral<Named> implements Named {
    private final String value;

    public NamedLiteral(String value) {
        this.value = value;
    }

    @Override
    public String value() {
        return value;
    }
}
```

### 3. Using the Factory

```java
@Path("/payments")
@ApplicationScoped
public class PaymentResource {

    @Inject
    PaymentProcessorFactory factory;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public Response processPayment(PaymentRequest request) {
        try {
            PaymentProcessor processor = factory.getProcessor(request.getPaymentMethod());
            PaymentResult result = processor.process(request);
            return Response.ok(result).build();
        } catch (UnsatisfiedResolutionException e) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(new ErrorResponse("Unsupported payment method: " + request.getPaymentMethod()))
                    .build();
        }
    }

    @GET
    @Path("/methods")
    @Produces(MediaType.APPLICATION_JSON)
    public List<String> getSupportedMethods() {
        return factory.getSupportedProcessors();
    }
}
```

## Alternative: Custom Qualifier Approach

### 4. Using Custom Qualifiers

```java
// Custom qualifier
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface PaymentType {
    PaymentMethod value();
}

// Enum for type safety
public enum PaymentMethod {
    CREDIT_CARD, PAYPAL, PIX, BANK_TRANSFER
}

// Annotated implementations
@ApplicationScoped
@PaymentType(PaymentMethod.CREDIT_CARD)
public class CreditCardProcessor implements PaymentProcessor {
    // ...
}

@ApplicationScoped
@PaymentType(PaymentMethod.PAYPAL)
public class PayPalProcessor implements PaymentProcessor {
    // ...
}
```

### 5. Qualifier Literal for Programmatic Lookup

```java
public class PaymentTypeLiteral extends AnnotationLiteral<PaymentType> implements PaymentType {
    private final PaymentMethod method;

    public PaymentTypeLiteral(PaymentMethod method) {
        this.method = method;
    }

    @Override
    public PaymentMethod value() {
        return method;
    }
}

@ApplicationScoped
public class PaymentProcessorFactory {

    @Inject
    @Any
    Instance<PaymentProcessor> processors;

    public PaymentProcessor getProcessor(PaymentMethod method) {
        return processors.select(new PaymentTypeLiteral(method)).get();
    }
}
```

## MicroProfile Extensions

### 6. Factory with Config-Driven Defaults

```java
@ApplicationScoped
public class PaymentProcessorFactory {

    @Inject
    @ConfigProperty(name = "payment.default.processor", defaultValue = "credit-card")
    String defaultProcessorType;

    @Inject
    @Any
    Instance<PaymentProcessor> processors;

    public PaymentProcessor getDefaultProcessor() {
        return getProcessor(defaultProcessorType);
    }

    public PaymentProcessor getProcessor(String type) {
        Instance<PaymentProcessor> selected = processors.select(new NamedLiteral(type));
        if (!selected.isResolvable()) {
            throw new ProcessorNotFoundException("No processor found for type: " + type);
        }
        return selected.get();
    }
}
```

### 7. Factory with Fallback

```java
@ApplicationScoped
public class ResilientPaymentProcessorFactory {

    @Inject
    @Any
    Instance<PaymentProcessor> processors;

    @Inject
    @Named("fallback")
    PaymentProcessor fallbackProcessor;

    @Retry(maxRetries = 3, delay = 100)
    @Fallback(fallbackMethod = "getFallbackProcessor")
    public PaymentProcessor getProcessor(String type) {
        return processors.select(new NamedLiteral(type)).get();
    }

    public PaymentProcessor getFallbackProcessor(String type) {
        return fallbackProcessor;
    }
}
```

## Testing

### 8. Unit Testing with Mocks

```java
@QuarkusTest
class PaymentProcessorFactoryTest {

    @Inject
    PaymentProcessorFactory factory;

    @Test
    void shouldReturnCreditCardProcessor() {
        PaymentProcessor processor = factory.getProcessor("credit-card");
        assertThat(processor).isInstanceOf(CreditCardProcessor.class);
    }

    @Test
    void shouldThrowForUnknownProcessor() {
        assertThrows(UnsatisfiedResolutionException.class,
            () -> factory.getProcessor("unknown"));
    }

    @Test
    void shouldListAllSupportedProcessors() {
        List<String> supported = factory.getSupportedProcessors();
        assertThat(supported).containsExactlyInAnyOrder("credit-card", "paypal", "pix");
    }
}
```

## When to Use

✅ **Use Factory Method when:**

- You need to create objects based on runtime conditions
- Adding new product types should not require modifying existing code
- You want to leverage CDI for dependency injection in products
- You need a central place to handle product creation logic

❌ **Avoid Factory Method when:**

- You have only 2-3 types that rarely change
- Direct instantiation is clearer and simpler
- Performance is critical and the factory overhead matters

## Related Patterns

- **Abstract Factory**: Creates families of related objects
- **Builder**: Constructs complex objects step by step
- **Prototype**: Creates objects by cloning existing instances
