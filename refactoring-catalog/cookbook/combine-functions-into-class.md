# Combine Functions into Class

## Intent

Combine a group of functions that operate on the same data into a class.

## Code Smells That Indicate This Refactoring

- Functions that always operate together on the same data
- Same data passed to many related functions
- Functions that form a coherent concept
- Data and its related functions are scattered

## Mechanics

1. Apply **Encapsulate Record** to group the data together if not already
2. Use **Move Function** to move each function into the new class
3. Replace the data argument with the class field
4. Expose the functions as methods

## Example

### Before

```java
// Functions scattered in a utility class
public class MeasurementUtils {
    public static double baseCharge(MeasurementData data) {
        return baseRate(data.month(), data.year()) * data.quantity();
    }

    public static double taxableCharge(MeasurementData data) {
        return Math.max(0, baseCharge(data) - taxThreshold(data.year()));
    }

    public static double calculateBill(MeasurementData data) {
        return baseCharge(data) + taxableCharge(data) * taxRate(data.year());
    }

    private static double baseRate(int month, int year) { /* ... */ }
    private static double taxThreshold(int year) { /* ... */ }
    private static double taxRate(int year) { /* ... */ }
}

// Usage
MeasurementData data = new MeasurementData(month, year, quantity);
double bill = MeasurementUtils.calculateBill(data);
```

### After

```java
public class MeasurementReading {
    private final int month;
    private final int year;
    private final double quantity;

    public MeasurementReading(int month, int year, double quantity) {
        this.month = month;
        this.year = year;
        this.quantity = quantity;
    }

    public double getBaseCharge() {
        return baseRate() * quantity;
    }

    public double getTaxableCharge() {
        return Math.max(0, getBaseCharge() - taxThreshold());
    }

    public double calculateBill() {
        return getBaseCharge() + getTaxableCharge() * taxRate();
    }

    private double baseRate() {
        return RateService.getBaseRate(month, year);
    }

    private double taxThreshold() {
        return RateService.getTaxThreshold(year);
    }

    private double taxRate() {
        return RateService.getTaxRate(year);
    }
}

// Usage
MeasurementReading reading = new MeasurementReading(month, year, quantity);
double bill = reading.calculateBill();
```

### Order Calculation Example

```java
// Before - scattered functions
double subtotal = calculateSubtotal(items);
double discount = calculateDiscount(items, customer);
double shipping = calculateShipping(items, address);
double tax = calculateTax(items, address);
double total = subtotal - discount + shipping + tax;

// After - grouped in class
public class OrderCalculation {
    private final List<LineItem> items;
    private final Customer customer;
    private final Address address;

    public OrderCalculation(List<LineItem> items, Customer customer, Address address) {
        this.items = items;
        this.customer = customer;
        this.address = address;
    }

    public double getSubtotal() {
        return items.stream()
            .mapToDouble(LineItem::getTotal)
            .sum();
    }

    public double getDiscount() {
        return customer.getDiscountRate() * getSubtotal();
    }

    public double getShipping() {
        return ShippingCalculator.calculate(items, address);
    }

    public double getTax() {
        return TaxCalculator.calculate(getSubtotal() - getDiscount(), address);
    }

    public double getTotal() {
        return getSubtotal() - getDiscount() + getShipping() + getTax();
    }
}
```

## When to Use

✅ **Use Combine Functions into Class when:**

- Functions operate on the same data
- Functions are logically related
- The data and functions form a concept
- You want to encapsulate the logic

❌ **Avoid Combine Functions into Class when:**

- Functions are used independently
- The grouping would be artificial
- Functions need different instantiation lifetimes
- A transform function would be more appropriate

## Related Refactorings

- **Combine Functions into Transform**: Alternative approach
- **Extract Class**: Similar grouping from existing class
- **Move Function**: Used during the combination
- **Encapsulate Record**: May be done first
