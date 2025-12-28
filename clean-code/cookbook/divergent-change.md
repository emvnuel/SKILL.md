# Divergent Change - Clean Code Cookbook

## Intent

Divergent Change occurs when one class is commonly changed for different reasons. This indicates the class has multiple responsibilities and should be split.

---

## The Problem

A class suffering from Divergent Change:

- Changes for multiple, unrelated reasons
- Has clusters of methods that change together
- Different stakeholders request changes to different parts
- Violates Single Responsibility Principle

---

## Detection

You have Divergent Change when:

- "Every time we add a new X, we have to change this class"
- "Every time the Y rules change, we modify this class"
- Class has methods grouped by different concerns
- Different teams modify different parts of the same class

---

## Before - Divergent Change

```java
public class Order {

    // Changes when: database schema changes (DBA)
    public void save() {
        Connection conn = DriverManager.getConnection(URL);
        PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO orders (customer_id, total) VALUES (?, ?)"
        );
        stmt.setLong(1, customerId);
        stmt.setBigDecimal(2, total);
        stmt.executeUpdate();
    }

    // Changes when: business rules change (Business Analyst)
    public Money calculateTotal() {
        Money subtotal = items.stream()
            .map(OrderItem::getLineTotal)
            .reduce(Money.ZERO, Money::add);

        Money discount = applyDiscounts(subtotal);
        Money tax = calculateTax(subtotal.subtract(discount));

        return subtotal.subtract(discount).add(tax);
    }

    // Changes when: tax rules change (Accounting)
    private Money calculateTax(Money taxableAmount) {
        if (shippingAddress.getState().equals("CA")) {
            return taxableAmount.multiply(0.0875);
        }
        // ... more state-specific rules
        return taxableAmount.multiply(0.06);
    }

    // Changes when: discount policies change (Marketing)
    private Money applyDiscounts(Money subtotal) {
        Money discount = Money.ZERO;
        if (customer.isPremium()) {
            discount = subtotal.multiply(0.1);
        }
        if (promoCode != null) {
            discount = discount.add(promoCode.calculateDiscount(subtotal));
        }
        return discount;
    }

    // Changes when: report format changes (Finance)
    public String generateInvoice() {
        return String.format(
            "Invoice for Order %s\nCustomer: %s\nTotal: %s",
            orderId, customer.getName(), total
        );
    }

    // Changes when: notification rules change (Customer Experience)
    public void sendConfirmation() {
        emailService.send(
            customer.getEmail(),
            "Order Confirmation",
            "Your order " + orderId + " has been placed."
        );
    }
}
```

---

## After - Separated Responsibilities

```java
// Pure domain entity - changes only for domain model changes
public class Order {
    private final OrderId id;
    private final Customer customer;
    private final List<OrderItem> items;
    private final Address shippingAddress;
    private Money total;
    private OrderStatus status;

    public void applyPricing(OrderPricing pricing) {
        this.total = pricing.total();
    }

    public Money getSubtotal() {
        return items.stream()
            .map(OrderItem::getLineTotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Changes when: pricing rules change
@ApplicationScoped
public class OrderPricingService {

    @Inject
    DiscountCalculator discountCalculator;

    @Inject
    TaxCalculator taxCalculator;

    public OrderPricing calculatePricing(Order order) {
        Money subtotal = order.getSubtotal();
        Money discount = discountCalculator.calculate(order);
        Money tax = taxCalculator.calculate(subtotal.subtract(discount), order.getShippingAddress());
        return new OrderPricing(subtotal, discount, tax);
    }
}

// Changes when: discount policies change
@ApplicationScoped
public class DiscountCalculator {

    public Money calculate(Order order) {
        Money discount = Money.ZERO;

        if (order.getCustomer().isPremium()) {
            discount = order.getSubtotal().multiply(PREMIUM_DISCOUNT);
        }

        if (order.hasPromoCode()) {
            discount = discount.add(
                order.getPromoCode().calculateDiscount(order.getSubtotal())
            );
        }

        return discount;
    }
}

// Changes when: tax rules change
@ApplicationScoped
public class TaxCalculator {

    private final Map<String, BigDecimal> taxRates;

    public Money calculate(Money taxableAmount, Address address) {
        BigDecimal rate = taxRates.getOrDefault(
            address.getState(),
            DEFAULT_TAX_RATE
        );
        return taxableAmount.multiply(rate);
    }
}

// Changes when: persistence layer changes
@ApplicationScoped
public class OrderRepository {

    @PersistenceContext
    EntityManager em;

    public void save(Order order) {
        em.persist(order);
    }

    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(em.find(Order.class, id));
    }
}

// Changes when: invoice format changes
@ApplicationScoped
public class InvoiceGenerator {

    public Invoice generate(Order order) {
        return Invoice.builder()
            .orderId(order.getId())
            .customerName(order.getCustomer().getName())
            .items(order.getItems())
            .total(order.getTotal())
            .build();
    }
}

// Changes when: notification rules change
@ApplicationScoped
public class OrderNotificationService {

    @Inject
    EmailService emailService;

    public void sendConfirmation(Order order) {
        emailService.send(
            order.getCustomer().getEmail(),
            "Order Confirmation",
            buildConfirmationBody(order)
        );
    }
}
```

---

## Orchestration Layer

After splitting, use a service to coordinate:

```java
@ApplicationScoped
public class OrderService {

    @Inject OrderRepository repository;
    @Inject OrderPricingService pricingService;
    @Inject OrderNotificationService notificationService;
    @Inject InvoiceGenerator invoiceGenerator;

    @Transactional
    public Order placeOrder(CreateOrderCommand command) {
        Order order = Order.create(command);

        OrderPricing pricing = pricingService.calculatePricing(order);
        order.applyPricing(pricing);

        Order saved = repository.save(order);
        notificationService.sendConfirmation(saved);

        return saved;
    }

    public Invoice generateInvoice(OrderId orderId) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        return invoiceGenerator.generate(order);
    }
}
```

---

## Comparison with Shotgun Surgery

| Divergent Change                  | Shotgun Surgery                    |
| --------------------------------- | ---------------------------------- |
| One class, many reasons to change | One change, many classes to modify |
| Split the class                   | Consolidate into one class         |
| Too many responsibilities         | Responsibility too scattered       |

---

## Refactoring Steps

1. **Identify change reasons**: List why this class changes
2. **Group related methods**: Cluster methods that change together
3. **Extract classes**: Create class per responsibility
4. **Define interfaces**: Clear contracts between new classes
5. **Inject dependencies**: Use CDI for loose coupling
6. **Add orchestrator**: Coordinate the new classes

---

## Related Recipes

- [Single Responsibility](./single-responsibility.md): One reason to change
- [Shotgun Surgery](./shotgun-surgery.md): Opposite problem
- [Small Classes](./small-classes.md): Result of proper splitting
