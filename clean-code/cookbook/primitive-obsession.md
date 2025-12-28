# Primitive Obsession - Clean Code Cookbook

## Intent

Avoid using primitives (String, int, double) to represent domain concepts. Create small objects for small things like money, phone numbers, and email addresses.

---

## The Problem

Primitives:

- Don't convey meaning
- Don't enforce constraints
- Don't provide behavior
- Lead to scattered validation
- Allow invalid states

---

## Common Examples

| Primitive Use        | Better Domain Type  |
| -------------------- | ------------------- |
| `String email`       | `Email`             |
| `String phoneNumber` | `PhoneNumber`       |
| `double amount`      | `Money`             |
| `int quantity`       | `Quantity`          |
| `String orderId`     | `OrderId`           |
| `long timestamp`     | `Instant` or custom |
| `String status`      | `OrderStatus` enum  |

---

## Before - Primitive Obsession

```java
public class Customer {
    private String id;
    private String email;
    private String phoneNumber;
    private String country;
    private int age;
}

public class Order {
    private String orderId;
    private double amount;
    private String currencyCode;
    private String status;
}

public class OrderService {
    public void processOrder(String orderId, double amount, String currency,
                            String email, String phone) {
        // Validation scattered everywhere
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        if (amount < 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (!isValidCurrency(currency)) {
            throw new IllegalArgumentException("Invalid currency");
        }
        // Same validation repeated in other services...
    }
}
```

---

## After - Value Objects

### Email Value Object

```java
public record Email(String value) {

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[A-Za-z0-9+_.-]+@(.+)$");

    public Email {
        if (value == null || !EMAIL_PATTERN.matcher(value).matches()) {
            throw new InvalidEmailException(value);
        }
        value = value.toLowerCase().trim();
    }

    public String domain() {
        return value.substring(value.indexOf('@') + 1);
    }

    @Override
    public String toString() {
        return value;
    }
}
```

### Money Value Object

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        requireNonNull(amount, "Amount required");
        requireNonNull(currency, "Currency required");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new NegativeAmountException(amount);
        }
        amount = amount.setScale(2, RoundingMode.HALF_UP);
    }

    public static Money of(double amount, String currencyCode) {
        return new Money(BigDecimal.valueOf(amount), Currency.getInstance(currencyCode));
    }

    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    public Money subtract(Money other) {
        requireSameCurrency(other);
        return new Money(amount.subtract(other.amount), currency);
    }

    public Money multiply(double factor) {
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }

    public boolean greaterThan(Money other) {
        requireSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new CurrencyMismatchException(currency, other.currency);
        }
    }
}
```

### Phone Number Value Object

```java
public record PhoneNumber(String countryCode, String number) {

    private static final Pattern PHONE_PATTERN = Pattern.compile("^\\d{10,15}$");

    public PhoneNumber {
        requireNonNull(countryCode, "Country code required");
        requireNonNull(number, "Number required");

        // Normalize
        number = number.replaceAll("[^0-9]", "");

        if (!PHONE_PATTERN.matcher(number).matches()) {
            throw new InvalidPhoneNumberException(number);
        }
    }

    public static PhoneNumber parse(String fullNumber) {
        // Parse and extract country code
        return new PhoneNumber("+1", fullNumber);
    }

    public String formatted() {
        return String.format("%s (%s) %s-%s",
            countryCode,
            number.substring(0, 3),
            number.substring(3, 6),
            number.substring(6)
        );
    }

    public String e164() {
        return countryCode + number;
    }
}
```

### Entity ID Value Object

```java
public record OrderId(String value) {

    private static final Pattern ID_PATTERN = Pattern.compile("^ORD-[A-Z0-9]{8}$");

    public OrderId {
        requireNonNull(value, "Order ID required");
        if (!ID_PATTERN.matcher(value).matches()) {
            throw new InvalidOrderIdException(value);
        }
    }

    public static OrderId generate() {
        String random = UUID.randomUUID().toString()
            .substring(0, 8)
            .toUpperCase();
        return new OrderId("ORD-" + random);
    }
}
```

---

## Refactored Domain Classes

```java
public class Customer {
    private final CustomerId id;
    private final Email email;
    private final PhoneNumber phone;
    private final Country country;
    private final Age age;
}

public class Order {
    private final OrderId id;
    private final Money total;
    private final OrderStatus status;
}

public class OrderService {
    // No validation needed - value objects are always valid
    public void processOrder(OrderId orderId, Money amount,
                            Email email, PhoneNumber phone) {
        // Just business logic, no primitive validation
    }
}
```

---

## Enum Over String Constants

### Before

```java
public class Order {
    private String status;  // "PENDING", "CONFIRMED", "SHIPPED", etc.

    public void setStatus(String status) {
        // No validation, any string accepted
        this.status = status;
    }
}

// Caller can pass invalid values
order.setStatus("PNDING");  // Typo, no compile error
order.setStatus("invalid");  // Runtime error later
```

### After

```java
public enum OrderStatus {
    PENDING,
    CONFIRMED,
    PROCESSING,
    SHIPPED,
    DELIVERED,
    CANCELLED;

    public boolean canTransitionTo(OrderStatus next) {
        return switch (this) {
            case PENDING -> next == CONFIRMED || next == CANCELLED;
            case CONFIRMED -> next == PROCESSING || next == CANCELLED;
            case PROCESSING -> next == SHIPPED;
            case SHIPPED -> next == DELIVERED;
            case DELIVERED, CANCELLED -> false;
        };
    }
}

public class Order {
    private OrderStatus status = OrderStatus.PENDING;

    public void transitionTo(OrderStatus newStatus) {
        if (!status.canTransitionTo(newStatus)) {
            throw new InvalidStatusTransitionException(status, newStatus);
        }
        this.status = newStatus;
    }
}
```

---

## Quantity Value Object

```java
public record Quantity(int value) {

    public Quantity {
        if (value < 0) {
            throw new NegativeQuantityException(value);
        }
    }

    public static final Quantity ZERO = new Quantity(0);
    public static final Quantity ONE = new Quantity(1);

    public Quantity add(Quantity other) {
        return new Quantity(value + other.value);
    }

    public Quantity subtract(Quantity other) {
        return new Quantity(value - other.value);
    }

    public boolean isZero() {
        return value == 0;
    }

    public boolean isPositive() {
        return value > 0;
    }
}
```

---

## JPA Mapping

```java
@Entity
public class Order {

    @EmbeddedId
    private OrderId id;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "currency_code"))
    })
    private Money total;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;
}

@Embeddable
public record Money(BigDecimal amount, Currency currency) {
    // Hibernate can embed this directly
}

@Embeddable
public record OrderId(String value) {
    // Used as @EmbeddedId
}
```

---

## Benefits

| Benefit          | Description                           |
| ---------------- | ------------------------------------- |
| Self-validating  | Invalid objects cannot exist          |
| Type-safe        | Compiler prevents mixing up arguments |
| Self-documenting | Method signatures are clear           |
| Behavior-rich    | Domain logic lives in the right place |
| DRY              | Validation defined once               |

---

## Related Recipes

- [Data Clumps](./data-clumps.md): Groups of primitives → objects
- [Meaningful Names](./meaningful-names.md): Types provide meaning
- [Single Responsibility](./single-responsibility.md): Value objects focus on one concept
