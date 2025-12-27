# Inline Class

## Intent

Move all features from a class into another class and delete the empty class.

## Code Smells That Indicate This Refactoring

- Class has too little behavior to justify its existence
- Class has become redundant after other refactorings
- Two classes are too tightly coupled to be separate
- Class was left empty after moving features elsewhere

## Mechanics

1. In the target class, create functions for all public functions of the source class (delegate)
2. Change all references of source methods to target methods
3. Move all functions and data from source to target
4. Delete the source class

## Example

### Before

```java
public class TrackingInformation {
    private String shippingCompany;
    private String trackingNumber;

    public String getShippingCompany() { return shippingCompany; }
    public void setShippingCompany(String company) { this.shippingCompany = company; }

    public String getTrackingNumber() { return trackingNumber; }
    public void setTrackingNumber(String number) { this.trackingNumber = number; }

    public String display() {
        return shippingCompany + ": " + trackingNumber;
    }
}

public class Shipment {
    private TrackingInformation trackingInformation;

    public TrackingInformation getTrackingInformation() {
        return trackingInformation;
    }

    public void setTrackingInformation(TrackingInformation info) {
        this.trackingInformation = info;
    }

    public String getTrackingInfo() {
        return trackingInformation.display();
    }
}

// Usage
shipment.getTrackingInformation().setShippingCompany("UPS");
```

### After

```java
public class Shipment {
    private String shippingCompany;
    private String trackingNumber;

    public String getShippingCompany() { return shippingCompany; }
    public void setShippingCompany(String company) { this.shippingCompany = company; }

    public String getTrackingNumber() { return trackingNumber; }
    public void setTrackingNumber(String number) { this.trackingNumber = number; }

    public String getTrackingInfo() {
        return shippingCompany + ": " + trackingNumber;
    }
}

// Usage - simpler
shipment.setShippingCompany("UPS");
```

### After Refactoring Leaves Class Empty

```java
// Before - Person only holds TelephoneNumber
public class Person {
    private TelephoneNumber phone;

    public String getAreaCode() { return phone.getAreaCode(); }
    public String getNumber() { return phone.getNumber(); }
}

// After Extract Class moves everything
// TelephoneNumber is now used directly
// Person class can be inlined or removed
```

## When to Use

✅ **Use Inline Class when:**

- Class has too little functionality
- The abstraction no longer carries its weight
- Two classes should really be one
- Simplification outweighs separation

❌ **Avoid Inline Class when:**

- The class has a clear responsibility
- It provides valuable abstraction
- Many clients depend on the class directly
- The receiving class would become too large

## Related Refactorings

- **Extract Class**: The inverse operation
- **Inline Function**: Similar for functions
- **Collapse Hierarchy**: For subclass inlining
