# Comments - Clean Code Cookbook

## Intent

Comments are not inherently bad, but they are often used as a deodorant for bad code. The best comment is the one you don't have to write because the code explains itself.

---

## The Truth About Comments

> Don't comment bad code—rewrite it.

Comments lie over time. Code changes, comments don't. The only source of truth is the code itself.

---

## Good Comments

### Legal Comments

```java
// Copyright (C) 2024 Company Name. All rights reserved.
// Licensed under the Apache License, Version 2.0
```

### Informative Comments

```java
// Format: kk:mm:ss EEE, MMM dd, yyyy
Pattern timeMatcher = Pattern.compile(
    "\\d*:\\d*:\\d* \\w*, \\w* \\d*, \\d*"
);
```

### Explanation of Intent

```java
// We use a custom comparator because natural ordering doesn't account
// for the business rule that premium customers always appear first
public int compareTo(Customer other) {
    if (this.isPremium() && !other.isPremium()) return -1;
    if (!this.isPremium() && other.isPremium()) return 1;
    return this.name.compareTo(other.name);
}
```

### Clarification (when code can't be changed)

```java
// Third-party API returns null instead of empty list
List<Item> items = externalApi.fetchItems();
if (items == null) {
    items = Collections.emptyList();
}
```

### Warning of Consequences

```java
// WARNING: Running this test takes 30 minutes and requires database
// @Disabled("Long-running integration test - run manually")
void testFullDataMigration() {
    // ...
}
```

### TODO Comments (with ticket reference)

```java
// TODO(JIRA-1234): Extract discount calculation to separate service
// when pricing module is refactored in Q2
public Money calculateDiscount(Order order) {
    // ...
}
```

### Javadoc for Public APIs

```java
/**
 * Calculates the shipping cost based on destination and package weight.
 *
 * @param destination the shipping address (must not be null)
 * @param weight package weight in kilograms
 * @return shipping cost in the merchant's default currency
 * @throws ShippingNotAvailableException if destination is not serviceable
 */
public Money calculateShipping(Address destination, BigDecimal weight) {
    // ...
}
```

---

## Bad Comments

### Mumbling

```java
// Bad - what does this even mean?
try {
    loadProperties();
} catch (IOException e) {
    // No properties files means all defaults are loaded
}

// Better - fail fast or log properly
try {
    loadProperties();
} catch (IOException e) {
    log.warn("Could not load properties file, using defaults: {}", e.getMessage());
}
```

### Redundant Comments

```java
// Bad - comment adds nothing
// Returns the day of the month
public int getDayOfMonth() {
    return dayOfMonth;
}

// Bad - repeats code
// Increment counter by one
counter++;

// Bad - states the obvious
// Default constructor
public Customer() {
}
```

### Misleading Comments

```java
// Bad - comment is wrong
// Returns true if the customer has orders
public boolean isActive() {
    // Actually checks last login date!
    return lastLogin.isAfter(Instant.now().minus(Duration.ofDays(90)));
}
```

### Mandated Comments

```java
// Bad - forced comment policy with no value
/**
 * @param id the id
 * @return the customer
 */
public Customer getCustomer(Long id) {
    return repository.findById(id);
}
```

### Journal Comments

```java
// Bad - use version control instead
/**
 * Changes:
 * 2024-01-15 - Added validation for email
 * 2024-01-10 - Fixed NPE in getName
 * 2024-01-05 - Initial implementation
 */
public class Customer {
    // ...
}
```

### Noise Comments

```java
// Bad - adds nothing
// The day of the month
private int dayOfMonth;

// Bad - separates natural groupings
///////////// Getters //////////////
public String getName() { return name; }
///////////// Setters //////////////
public void setName(String name) { this.name = name; }
```

### Commented-Out Code

```java
// Bad - just delete it, version control remembers
public void processOrder(Order order) {
    validateOrder(order);
    // calculateDiscount(order);
    // applyTax(order);
    calculateTotal(order);
    // notifyWarehouse(order);
}
```

### Position Markers

```java
// Bad - creates noise
////////////////////////////// ACTIONS //////////////////////////////
public void save() { }
public void delete() { }
////////////////////////////// QUERIES //////////////////////////////
public Order find() { }
```

### Closing Brace Comments

```java
// Bad - indicates method is too long
public void processLargeOrder() {
    // ... 100 lines of code ...
    if (condition) {
        // ... 50 more lines ...
    } // end if
} // end processLargeOrder
```

---

## Refactor Instead of Comment

### Before - Comment Explains Logic

```java
// Check to see if employee is eligible for full benefits
if ((employee.flags & HOURLY_FLAG) && (employee.age > 65)) {
    // ...
}
```

### After - Code Explains Itself

```java
if (employee.isEligibleForFullBenefits()) {
    // ...
}
```

### Before - Comment Describes Block

```java
public void addToCart(Product product) {
    // Check if product is in stock
    if (product.getInventory() <= 0) {
        throw new OutOfStockException(product);
    }

    // Calculate the price including discounts
    Money price = product.getPrice();
    if (customer.isPremium()) {
        price = price.multiply(0.9);
    }

    // Add to cart
    cart.addItem(new CartItem(product, price));
}
```

### After - Extract Methods

```java
public void addToCart(Product product) {
    validateInStock(product);
    Money price = calculatePriceWithDiscounts(product);
    cart.addItem(new CartItem(product, price));
}

private void validateInStock(Product product) {
    if (product.getInventory() <= 0) {
        throw new OutOfStockException(product);
    }
}

private Money calculatePriceWithDiscounts(Product product) {
    Money price = product.getPrice();
    if (customer.isPremium()) {
        price = price.applyDiscount(PREMIUM_DISCOUNT);
    }
    return price;
}
```

---

## Comment Decision Tree

```
Do I need a comment?
│
├── Is the code unclear without it?
│   └── YES → Can I rename or extract to clarify?
│       ├── YES → Refactor instead of comment
│       └── NO → Add clarifying comment
│
├── Is it a legal requirement?
│   └── YES → Add required legal header
│
├── Is it explaining WHY (not WHAT)?
│   └── YES → Intent comment is acceptable
│
├── Is it a public API?
│   └── YES → Add Javadoc
│
├── Is it a warning or TODO with ticket?
│   └── YES → Add with reference
│
└── Otherwise
    └── Don't add the comment
```

---

## Related Recipes

- [Meaningful Names](./meaningful-names.md): Good names reduce need for comments
- [Small Functions](./small-functions.md): Short methods are self-documenting
