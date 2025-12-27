# Strategy Pattern - Jakarta EE Cookbook

## Intent

Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

## Jakarta EE Implementation

### 1. Strategy Interface and Implementations

```java
// Strategy interface
public interface PricingStrategy {
    BigDecimal calculatePrice(Order order);
    CustomerType getSupportedType();
}

// Concrete strategies
@ApplicationScoped
public class RegularPricingStrategy implements PricingStrategy {

    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getSubtotal();  // Full price
    }

    @Override
    public CustomerType getSupportedType() {
        return CustomerType.REGULAR;
    }
}

@ApplicationScoped
public class PremiumPricingStrategy implements PricingStrategy {

    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getSubtotal().multiply(new BigDecimal("0.90"));  // 10% off
    }

    @Override
    public CustomerType getSupportedType() {
        return CustomerType.PREMIUM;
    }
}

@ApplicationScoped
public class VIPPricingStrategy implements PricingStrategy {

    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getSubtotal().multiply(new BigDecimal("0.80"));  // 20% off
    }

    @Override
    public CustomerType getSupportedType() {
        return CustomerType.VIP;
    }
}
```

### 2. Context - Strategy Selector

```java
@ApplicationScoped
public class PricingService {

    @Inject
    @Any
    Instance<PricingStrategy> strategies;

    public BigDecimal calculatePrice(Order order) {
        PricingStrategy strategy = getStrategy(order.getCustomer().getType());
        return strategy.calculatePrice(order);
    }

    private PricingStrategy getStrategy(CustomerType type) {
        return strategies.stream()
            .filter(s -> s.getSupportedType() == type)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No strategy for: " + type));
    }
}
```

### 3. Alternative: CDI Qualifiers

```java
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface CustomerPricing {
    CustomerType value();
}

@ApplicationScoped
@CustomerPricing(CustomerType.VIP)
public class VIPPricingStrategy implements PricingStrategy {
    // ...
}

@ApplicationScoped
public class PricingService {

    @Inject
    @Any
    Instance<PricingStrategy> strategies;

    public BigDecimal calculatePrice(Order order) {
        return strategies
            .select(new CustomerPricingLiteral(order.getCustomer().getType()))
            .get()
            .calculatePrice(order);
    }
}
```

## Real-World Examples

### 4. Payment Processing Strategy

```java
public interface PaymentStrategy {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentMethod method);
}

@ApplicationScoped
public class CreditCardStrategy implements PaymentStrategy {

    @Inject
    @RestClient
    StripeClient stripe;

    @Override
    public PaymentResult process(PaymentRequest request) {
        return stripe.charge(request.toStripeRequest());
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.CREDIT_CARD;
    }
}

@ApplicationScoped
public class PaymentService {

    @Inject
    @Any
    Instance<PaymentStrategy> strategies;

    public PaymentResult pay(PaymentRequest request) {
        return strategies.stream()
            .filter(s -> s.supports(request.getMethod()))
            .findFirst()
            .orElseThrow()
            .process(request);
    }
}
```

### 5. Export Strategy

```java
public interface ExportStrategy {
    byte[] export(List<Record> data);
    String getContentType();
    String getFileExtension();
}

@ApplicationScoped
@Named("csv")
public class CsvExportStrategy implements ExportStrategy {
    public byte[] export(List<Record> data) { /* CSV conversion */ }
    public String getContentType() { return "text/csv"; }
    public String getFileExtension() { return "csv"; }
}

@ApplicationScoped
@Named("excel")
public class ExcelExportStrategy implements ExportStrategy {
    public byte[] export(List<Record> data) { /* Excel conversion */ }
    public String getContentType() {
        return "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";
    }
    public String getFileExtension() { return "xlsx"; }
}
```

## When to Use

✅ **Use Strategy when:**

- You have multiple algorithms for a specific task
- You need to switch algorithms at runtime
- You want to eliminate conditional statements for algorithm selection

❌ **Avoid Strategy when:**

- You have only 2-3 algorithms that rarely change
- Algorithms don't share a common interface
