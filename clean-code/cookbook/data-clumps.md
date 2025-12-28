# Data Clumps - Clean Code Cookbook

## Intent

Data clumps are groups of data that appear together repeatedly. When you see the same three or four data items together in multiple places, consider bundling them into their own class.

---

## Detection

You have a data clump when:

- Same parameters appear together in multiple method signatures
- Same fields appear together in multiple classes
- You frequently pass the same group of values together
- Changing one value often requires changing others

---

## Method Parameter Clumps

### Before

```java
public class CustomerService {

    public void createCustomer(
        String firstName,
        String lastName,
        String street,
        String city,
        String state,
        String zipCode
    ) { ... }

    public void updateAddress(
        Long customerId,
        String street,
        String city,
        String state,
        String zipCode
    ) { ... }

    public Money calculateShipping(
        String street,
        String city,
        String state,
        String zipCode,
        double weight
    ) { ... }
}
```

### After

```java
public record Address(
    String street,
    String city,
    String state,
    String zipCode
) {
    public boolean isSameZone(Address other) {
        return this.state.equals(other.state);
    }

    public String singleLine() {
        return String.format("%s, %s, %s %s", street, city, state, zipCode);
    }
}

public record CustomerName(String firstName, String lastName) {
    public String fullName() {
        return firstName + " " + lastName;
    }
}

public class CustomerService {

    public void createCustomer(CustomerName name, Address address) { ... }

    public void updateAddress(Long customerId, Address address) { ... }

    public Money calculateShipping(Address destination, double weight) { ... }
}
```

---

## Field Clumps

### Before

```java
public class Order {
    private String billingStreet;
    private String billingCity;
    private String billingState;
    private String billingZip;

    private String shippingStreet;
    private String shippingCity;
    private String shippingState;
    private String shippingZip;

    private String cardNumber;
    private String cardExpiry;
    private String cardCvv;
    private String cardholderName;
}
```

### After

```java
public class Order {
    private Address billingAddress;
    private Address shippingAddress;
    private PaymentCard paymentCard;
}

public record Address(
    String street,
    String city,
    String state,
    String zipCode
) {}

public record PaymentCard(
    String number,
    YearMonth expiry,
    String cvv,
    String holderName
) {
    public String maskedNumber() {
        return "****-****-****-" + number.substring(number.length() - 4);
    }

    public boolean isExpired() {
        return expiry.isBefore(YearMonth.now());
    }
}
```

---

## Date Range Clump

### Before

```java
public class ReportService {

    public Report generateSalesReport(LocalDate startDate, LocalDate endDate) { ... }

    public List<Order> findOrders(LocalDate startDate, LocalDate endDate) { ... }

    public BigDecimal calculateRevenue(LocalDate startDate, LocalDate endDate) { ... }

    public void exportData(LocalDate startDate, LocalDate endDate, Format format) { ... }
}
```

### After

```java
public record DateRange(LocalDate start, LocalDate end) {

    public DateRange {
        if (end.isBefore(start)) {
            throw new IllegalArgumentException("End date must be after start date");
        }
    }

    public static DateRange lastNDays(int days) {
        LocalDate end = LocalDate.now();
        LocalDate start = end.minusDays(days);
        return new DateRange(start, end);
    }

    public static DateRange thisMonth() {
        LocalDate now = LocalDate.now();
        return new DateRange(now.withDayOfMonth(1), now);
    }

    public boolean contains(LocalDate date) {
        return !date.isBefore(start) && !date.isAfter(end);
    }

    public long daysBetween() {
        return ChronoUnit.DAYS.between(start, end);
    }
}

public class ReportService {

    public Report generateSalesReport(DateRange period) { ... }

    public List<Order> findOrders(DateRange period) { ... }

    public BigDecimal calculateRevenue(DateRange period) { ... }

    public void exportData(DateRange period, Format format) { ... }
}
```

---

## Coordinate Clump

### Before

```java
public class MapService {

    public double calculateDistance(
        double lat1, double lon1,
        double lat2, double lon2
    ) { ... }

    public List<Place> findNearby(
        double latitude, double longitude,
        double radiusKm
    ) { ... }

    public String getAddress(double latitude, double longitude) { ... }
}
```

### After

```java
public record GeoLocation(double latitude, double longitude) {

    public GeoLocation {
        if (latitude < -90 || latitude > 90) {
            throw new IllegalArgumentException("Invalid latitude: " + latitude);
        }
        if (longitude < -180 || longitude > 180) {
            throw new IllegalArgumentException("Invalid longitude: " + longitude);
        }
    }

    public Distance distanceTo(GeoLocation other) {
        // Haversine formula
        double dLat = Math.toRadians(other.latitude - this.latitude);
        double dLon = Math.toRadians(other.longitude - this.longitude);
        // ... calculation
        return Distance.ofKilometers(distance);
    }

    public boolean isWithin(GeoLocation center, Distance radius) {
        return distanceTo(center).lessThan(radius);
    }
}

public class MapService {

    public Distance calculateDistance(GeoLocation from, GeoLocation to) {
        return from.distanceTo(to);  // Behavior moved to value object
    }

    public List<Place> findNearby(GeoLocation location, Distance radius) { ... }

    public String getAddress(GeoLocation location) { ... }
}
```

---

## Money and Currency Clump

### Before

```java
public class PaymentService {

    public void processPayment(
        BigDecimal amount,
        String currencyCode,
        String accountId
    ) { ... }

    public BigDecimal convert(
        BigDecimal amount,
        String fromCurrency,
        String toCurrency
    ) { ... }

    public boolean hasEnoughBalance(
        String accountId,
        BigDecimal amount,
        String currency
    ) { ... }
}
```

### After

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    public Money subtract(Money other) {
        requireSameCurrency(other);
        return new Money(amount.subtract(other.amount), currency);
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

public class PaymentService {

    public void processPayment(Money amount, AccountId accountId) { ... }

    public Money convert(Money amount, Currency targetCurrency) { ... }

    public boolean hasEnoughBalance(AccountId accountId, Money amount) { ... }
}
```

---

## Refactoring Steps

1. **Identify the clump**: Same data appearing 3+ times together
2. **Create a class/record**: Bundle the data with a meaningful name
3. **Add validation**: Enforce invariants in constructor
4. **Add behavior**: Methods that operate on the data
5. **Replace usages**: Update method signatures and fields

---

## Signs of Data Clumps

| Pattern                                  | Example                             |
| ---------------------------------------- | ----------------------------------- |
| Parameter lists > 3 related items        | `(street, city, state, zip)`        |
| Repeated field groups                    | `billingX`, `shippingX`             |
| Paired parameters always together        | `(startDate, endDate)`              |
| Parallel arrays/lists                    | `names[]`, `ages[]`, `addresses[]`  |
| Constructor with many related primitives | `new Order(s1, c1, st1, z1, s2...)` |

---

## Related Recipes

- [Primitive Obsession](./primitive-obsession.md): Single primitives → value objects
- [Function Arguments](./function-arguments.md): Parameter objects reduce arguments
- [Feature Envy](./feature-envy.md): Methods often envy the new class
