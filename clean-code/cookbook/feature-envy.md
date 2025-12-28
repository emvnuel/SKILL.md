# Feature Envy - Clean Code Cookbook

## Intent

A method that uses more features (data and methods) from another class than its own class has Feature Envy. Move the method to the class it envies.

---

## Detection

Feature Envy is present when:

- A method accesses data from another object more than its own
- Long chains of method calls on one object
- A method could be moved to another class with minimal changes
- Frequent getter calls on another object

---

## Before - Feature Envy

```java
public class OrderPrinter {

    public String formatOrder(Order order) {
        // Uses 8 methods from Order, 0 from OrderPrinter
        StringBuilder sb = new StringBuilder();
        sb.append("Order: ").append(order.getOrderNumber()).append("\n");
        sb.append("Customer: ").append(order.getCustomer().getName()).append("\n");
        sb.append("Date: ").append(order.getOrderDate()).append("\n");

        for (OrderItem item : order.getItems()) {
            sb.append("  - ").append(item.getProduct().getName())
              .append(" x ").append(item.getQuantity())
              .append(" @ ").append(item.getPrice())
              .append("\n");
        }

        sb.append("Subtotal: ").append(order.getSubtotal()).append("\n");
        sb.append("Tax: ").append(order.getTax()).append("\n");
        sb.append("Total: ").append(order.getTotal()).append("\n");

        return sb.toString();
    }
}
```

---

## After - Method Moved

```java
public class Order {
    private OrderNumber orderNumber;
    private Customer customer;
    private LocalDate orderDate;
    private List<OrderItem> items;
    private Money subtotal;
    private Money tax;
    private Money total;

    // Formatting logic now lives with the data it uses
    public String formatted() {
        StringBuilder sb = new StringBuilder();
        sb.append("Order: ").append(orderNumber).append("\n");
        sb.append("Customer: ").append(customer.getName()).append("\n");
        sb.append("Date: ").append(orderDate).append("\n");

        items.forEach(item -> sb.append(item.formatted()));

        sb.append("Subtotal: ").append(subtotal).append("\n");
        sb.append("Tax: ").append(tax).append("\n");
        sb.append("Total: ").append(total).append("\n");

        return sb.toString();
    }
}

public class OrderItem {
    private Product product;
    private Quantity quantity;
    private Money price;

    public String formatted() {
        return String.format("  - %s x %s @ %s\n",
            product.getName(), quantity, price);
    }
}
```

---

## Alternative - Strategy Pattern

If multiple formatting strategies are needed:

```java
public interface OrderFormatter {
    String format(Order order);
}

public class PlainTextOrderFormatter implements OrderFormatter {
    @Override
    public String format(Order order) {
        // Order provides data through a single method
        return order.toFormattingData().renderAsText();
    }
}

public class Order {
    // Only expose what formatters need
    public OrderFormattingData toFormattingData() {
        return new OrderFormattingData(
            orderNumber, customer.getName(), orderDate,
            items.stream().map(OrderItem::toFormattingData).toList(),
            subtotal, tax, total
        );
    }
}
```

---

## Calculation Feature Envy

### Before

```java
public class ShippingCalculator {

    public Money calculateShipping(Order order) {
        // Envies Order - uses its data extensively
        Money itemsValue = Money.ZERO;
        double totalWeight = 0;

        for (OrderItem item : order.getItems()) {
            itemsValue = itemsValue.add(item.getPrice().multiply(item.getQuantity()));
            totalWeight += item.getProduct().getWeight() * item.getQuantity();
        }

        if (itemsValue.greaterThan(Money.of(100))) {
            return Money.ZERO;  // Free shipping over $100
        }

        return Money.of(totalWeight * 0.5);
    }
}
```

### After

```java
public class Order {

    public Money getTotalValue() {
        return items.stream()
            .map(OrderItem::getLineTotal)
            .reduce(Money.ZERO, Money::add);
    }

    public double getTotalWeight() {
        return items.stream()
            .mapToDouble(OrderItem::getWeight)
            .sum();
    }

    public boolean qualifiesForFreeShipping() {
        return getTotalValue().greaterThan(FREE_SHIPPING_THRESHOLD);
    }
}

public class OrderItem {

    public Money getLineTotal() {
        return price.multiply(quantity.value());
    }

    public double getWeight() {
        return product.getWeight() * quantity.value();
    }
}

public class ShippingCalculator {

    public Money calculateShipping(Order order) {
        if (order.qualifiesForFreeShipping()) {
            return Money.ZERO;
        }
        return Money.of(order.getTotalWeight() * RATE_PER_KG);
    }
}
```

---

## Validation Feature Envy

### Before

```java
public class OrderValidator {

    public List<String> validate(Order order) {
        List<String> errors = new ArrayList<>();

        // Validates by reaching into Order's internals
        if (order.getItems() == null || order.getItems().isEmpty()) {
            errors.add("Order must have at least one item");
        }

        if (order.getCustomer() == null) {
            errors.add("Customer is required");
        } else if (order.getCustomer().getEmail() == null) {
            errors.add("Customer email is required");
        }

        if (order.getShippingAddress() == null) {
            errors.add("Shipping address is required");
        } else if (order.getShippingAddress().getZipCode() == null) {
            errors.add("Zip code is required");
        }

        return errors;
    }
}
```

### After

```java
// Validation at construction - invalid orders can't exist
public class Order {

    public Order(Customer customer, Address shippingAddress, List<OrderItem> items) {
        this.customer = requireValid(customer);
        this.shippingAddress = requireValid(shippingAddress);
        this.items = requireNotEmpty(items, "Order must have items");
    }

    private Customer requireValid(Customer customer) {
        requireNonNull(customer, "Customer required");
        requireNonNull(customer.getEmail(), "Customer email required");
        return customer;
    }

    private Address requireValid(Address address) {
        requireNonNull(address, "Shipping address required");
        requireNonNull(address.getZipCode(), "Zip code required");
        return address;
    }
}

// Or use Bean Validation
public class Order {
    @NotNull
    @Valid
    private Customer customer;

    @NotNull
    @Valid
    private Address shippingAddress;

    @NotEmpty
    @Valid
    private List<OrderItem> items;
}
```

---

## Tell, Don't Ask

Instead of asking an object for data and making decisions, tell it what to do:

### Before - Ask

```java
// Asks for data, makes decision externally
if (account.getBalance().greaterThan(amount)) {
    account.setBalance(account.getBalance().subtract(amount));
}
```

### After - Tell

```java
// Tells account what to do, account makes decision
account.withdraw(amount);

// In Account class
public void withdraw(Money amount) {
    if (balance.lessThan(amount)) {
        throw new InsufficientFundsException(this, amount);
    }
    this.balance = balance.subtract(amount);
}
```

---

## When NOT to Move

Keep method in its current class when:

1. **Strategy pattern**: Different algorithms operate on same data
2. **Visitor pattern**: Operations across unrelated types
3. **Data transfer**: DTOs intentionally expose data
4. **Framework requirements**: Mappers, converters, validators

---

## Detection Checklist

| Signal                                     | Likely Feature Envy |
| ------------------------------------------ | ------------------- |
| Method uses > 3 getters from another class | Yes                 |
| Multiple dot chains (a.getB().getC())      | Yes                 |
| Calculation using mostly external data     | Yes                 |
| Validation reaching into object internals  | Yes                 |
| Method name suggests it belongs elsewhere  | Yes                 |

---

## Related Recipes

- [Single Responsibility](./single-responsibility.md): Behavior belongs with data
- [Cohesion](./cohesion.md): Methods should use their own fields
- [Data Clumps](./data-clumps.md): Groups of data often become classes
