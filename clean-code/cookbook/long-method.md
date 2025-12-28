# Long Method - Clean Code Cookbook

## Intent

Long methods are hard to read, understand, test, and maintain. Extract smaller methods until each method does one thing at one level of abstraction.

---

## How Long is Too Long?

| Lines | Assessment           |
| ----- | -------------------- |
| 1-10  | Ideal                |
| 10-20 | Acceptable           |
| 20-50 | Consider refactoring |
| 50+   | Definitely refactor  |

---

## Detection Signals

1. Method needs comments to explain sections
2. Scrolling required to see entire method
3. Multiple levels of indentation
4. Variables declared far from use
5. Multiple try-catch blocks
6. Mix of abstraction levels

---

## Extract Method Refactoring

### Before

```java
public void processOrder(Order order) {
    // Validate order
    if (order == null) {
        throw new IllegalArgumentException("Order cannot be null");
    }
    if (order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Order must have items");
    }
    for (OrderItem item : order.getItems()) {
        if (item.getQuantity() <= 0) {
            throw new IllegalArgumentException("Invalid quantity for item: " + item.getSku());
        }
    }

    // Calculate subtotal
    Money subtotal = Money.ZERO;
    for (OrderItem item : order.getItems()) {
        Money itemTotal = item.getPrice().multiply(item.getQuantity());
        subtotal = subtotal.add(itemTotal);
    }

    // Apply discounts
    Money discount = Money.ZERO;
    if (order.getCustomer().isPremium()) {
        discount = subtotal.multiply(0.1);
    }
    if (order.hasPromoCode()) {
        PromoCode promo = promoCodeService.validate(order.getPromoCode());
        if (promo != null) {
            discount = discount.add(promo.calculateDiscount(subtotal));
        }
    }

    // Calculate tax
    Money taxableAmount = subtotal.subtract(discount);
    TaxRate rate = taxService.getRateForAddress(order.getShippingAddress());
    Money tax = taxableAmount.multiply(rate.getPercentage());

    // Set totals
    order.setSubtotal(subtotal);
    order.setDiscount(discount);
    order.setTax(tax);
    order.setTotal(subtotal.subtract(discount).add(tax));

    // Reserve inventory
    for (OrderItem item : order.getItems()) {
        inventoryService.reserve(item.getSku(), item.getQuantity());
    }

    // Save order
    orderRepository.save(order);

    // Send confirmation
    emailService.sendOrderConfirmation(order);
}
```

### After

```java
public void processOrder(Order order) {
    validateOrder(order);
    calculatePricing(order);
    reserveInventory(order);
    persistOrder(order);
    sendConfirmation(order);
}

private void validateOrder(Order order) {
    requireNonNull(order, "Order cannot be null");
    requireNotEmpty(order.getItems(), "Order must have items");
    order.getItems().forEach(this::validateItem);
}

private void validateItem(OrderItem item) {
    if (item.getQuantity() <= 0) {
        throw new InvalidQuantityException(item);
    }
}

private void calculatePricing(Order order) {
    Money subtotal = calculateSubtotal(order);
    Money discount = calculateDiscount(order, subtotal);
    Money tax = calculateTax(order, subtotal.subtract(discount));

    order.applyPricing(new OrderPricing(subtotal, discount, tax));
}

private Money calculateSubtotal(Order order) {
    return order.getItems().stream()
        .map(this::calculateItemTotal)
        .reduce(Money.ZERO, Money::add);
}

private Money calculateItemTotal(OrderItem item) {
    return item.getPrice().multiply(item.getQuantity());
}

private Money calculateDiscount(Order order, Money subtotal) {
    Money discount = Money.ZERO;

    if (order.getCustomer().isPremium()) {
        discount = discount.add(subtotal.multiply(PREMIUM_DISCOUNT));
    }
    if (order.hasPromoCode()) {
        discount = discount.add(calculatePromoDiscount(order, subtotal));
    }

    return discount;
}

private Money calculatePromoDiscount(Order order, Money subtotal) {
    return promoCodeService.validate(order.getPromoCode())
        .map(promo -> promo.calculateDiscount(subtotal))
        .orElse(Money.ZERO);
}

private Money calculateTax(Order order, Money taxableAmount) {
    TaxRate rate = taxService.getRateForAddress(order.getShippingAddress());
    return taxableAmount.multiply(rate.getPercentage());
}

private void reserveInventory(Order order) {
    order.getItems().forEach(item ->
        inventoryService.reserve(item.getSku(), item.getQuantity())
    );
}

private void persistOrder(Order order) {
    orderRepository.save(order);
}

private void sendConfirmation(Order order) {
    emailService.sendOrderConfirmation(order);
}
```

---

## Compose Method Pattern

Structure a method as a sequence of steps at the same abstraction level:

```java
// High level - reads like a story
public Invoice generateMonthlyInvoice(Customer customer, YearMonth month) {
    List<UsageRecord> usage = collectUsageForMonth(customer, month);
    List<LineItem> lineItems = calculateCharges(usage);
    Money total = sumLineItems(lineItems);
    Money tax = calculateTax(total, customer.getTaxInfo());

    return createInvoice(customer, month, lineItems, total, tax);
}
```

---

## Replace Temp with Query

Long methods often have many temporary variables. Extract to methods.

### Before

```java
public Money calculateTotal(Order order) {
    Money basePrice = order.getQuantity() * order.getItemPrice();

    double discountFactor;
    if (basePrice.greaterThan(Money.of(1000))) {
        discountFactor = 0.95;
    } else {
        discountFactor = 0.98;
    }

    return basePrice.multiply(discountFactor);
}
```

### After

```java
public Money calculateTotal(Order order) {
    return basePrice(order).multiply(discountFactor(order));
}

private Money basePrice(Order order) {
    return order.getQuantity() * order.getItemPrice();
}

private double discountFactor(Order order) {
    return basePrice(order).greaterThan(Money.of(1000)) ? 0.95 : 0.98;
}
```

---

## Decompose Conditional

Complex conditionals are a form of long method. Extract the condition and branches.

### Before

```java
if (date.before(SUMMER_START) || date.after(SUMMER_END)) {
    charge = quantity * winterRate + winterServiceCharge;
} else {
    charge = quantity * summerRate;
}
```

### After

```java
if (isWinter(date)) {
    charge = winterCharge(quantity);
} else {
    charge = summerCharge(quantity);
}

private boolean isWinter(LocalDate date) {
    return date.isBefore(SUMMER_START) || date.isAfter(SUMMER_END);
}

private Money winterCharge(int quantity) {
    return Money.of(quantity * winterRate + winterServiceCharge);
}

private Money summerCharge(int quantity) {
    return Money.of(quantity * summerRate);
}
```

---

## Extract Surrounding

When code has setup/teardown around interesting logic:

### Before

```java
public void processRecords() {
    Connection conn = null;
    try {
        conn = dataSource.getConnection();
        conn.setAutoCommit(false);

        // Interesting part
        List<Record> records = fetchRecords(conn);
        for (Record record : records) {
            processRecord(record, conn);
        }
        markAsProcessed(records, conn);

        conn.commit();
    } catch (SQLException e) {
        if (conn != null) {
            try { conn.rollback(); } catch (SQLException ex) { }
        }
        throw new ProcessingException(e);
    } finally {
        if (conn != null) {
            try { conn.close(); } catch (SQLException e) { }
        }
    }
}
```

### After - Extract Interesting Part

```java
@Transactional  // Container handles transaction
public void processRecords() {
    List<Record> records = recordRepository.fetchUnprocessed();
    records.forEach(this::processRecord);
    recordRepository.markAsProcessed(records);
}

private void processRecord(Record record) {
    // Just the interesting logic
}
```

---

## Metrics

| Metric                | Target | Warning |
| --------------------- | ------ | ------- |
| Lines of code         | < 20   | > 30    |
| Cyclomatic complexity | < 5    | > 10    |
| Nesting depth         | < 3    | > 4     |
| Number of parameters  | < 3    | > 4     |

---

## Related Recipes

- [Small Functions](./small-functions.md): Keep functions focused
- [Comments](./comments.md): Comments often indicate need to extract
- [Single Responsibility](./single-responsibility.md): Methods should do one thing
