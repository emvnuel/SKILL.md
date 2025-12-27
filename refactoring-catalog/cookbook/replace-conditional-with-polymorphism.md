# Replace Conditional with Polymorphism

## Intent

Replace a conditional that chooses different behavior based on type with polymorphism.

## Code Smells That Indicate This Refactoring

- Switch statements that select behavior based on type code
- If-else chains testing the same variable for different types
- Same conditional structure repeated in multiple places
- Adding a new type requires modifying multiple switch statements

## Mechanics

1. If the conditional logic is in a class that doesn't already have polymorphic behavior, create classes for each type with a common supertype
2. Add an abstract method to the supertype
3. Move the conditional logic for each leg to the appropriate subclass
4. Leave a default case in the supertype if one makes sense
5. Test after moving each leg

## Example

### Before

```java
public class Bird {
    private String type;
    private int numberOfCoconuts;
    private double voltage;
    private boolean isNailed;

    public double getSpeed() {
        switch (type) {
            case "european":
                return getBaseSpeed();
            case "african":
                return getBaseSpeed() - getLoadFactor() * numberOfCoconuts;
            case "norwegian_blue":
                return isNailed ? 0 : getBaseSpeed(voltage);
            default:
                throw new IllegalArgumentException("Unknown bird type: " + type);
        }
    }

    private double getBaseSpeed() { return 10.0; }
    private double getBaseSpeed(double voltage) { return voltage * 2; }
    private double getLoadFactor() { return 0.5; }
}
```

### After

```java
public abstract class Bird {
    public abstract double getSpeed();

    protected double getBaseSpeed() {
        return 10.0;
    }
}

public class EuropeanSwallow extends Bird {
    @Override
    public double getSpeed() {
        return getBaseSpeed();
    }
}

public class AfricanSwallow extends Bird {
    private final int numberOfCoconuts;

    public AfricanSwallow(int numberOfCoconuts) {
        this.numberOfCoconuts = numberOfCoconuts;
    }

    @Override
    public double getSpeed() {
        return getBaseSpeed() - getLoadFactor() * numberOfCoconuts;
    }

    private double getLoadFactor() {
        return 0.5;
    }
}

public class NorwegianBlueParrot extends Bird {
    private final double voltage;
    private final boolean isNailed;

    public NorwegianBlueParrot(double voltage, boolean isNailed) {
        this.voltage = voltage;
        this.isNailed = isNailed;
    }

    @Override
    public double getSpeed() {
        return isNailed ? 0 : voltage * 2;
    }
}
```

### CDI Version with Instance

```java
public interface Bird {
    double getSpeed();
    String getType();
}

@ApplicationScoped
@Named("european")
public class EuropeanSwallow implements Bird {
    @Override
    public double getSpeed() { return 10.0; }
    @Override
    public String getType() { return "european"; }
}

@ApplicationScoped
public class BirdFactory {
    @Inject @Any
    Instance<Bird> birds;

    public Bird create(String type) {
        return birds.select(new NamedLiteral(type)).get();
    }
}
```

## Using Variation Points (Strategy)

```java
// When you have multiple axes of variation
public class VoyageRating {
    private final Voyage voyage;
    private final HistoryDelegate historyDelegate;

    public VoyageRating(Voyage voyage, List<VoyageHistory> history) {
        this.voyage = voyage;
        this.historyDelegate = createHistoryDelegate(voyage, history);
    }

    private HistoryDelegate createHistoryDelegate(Voyage voyage, List<VoyageHistory> history) {
        if (voyage.getZone().equals("china") && hasChinaHistory(history)) {
            return new ExperiencedChinaHistoryDelegate(history);
        }
        return new DefaultHistoryDelegate(history);
    }

    public int rating() {
        return voyage.profitFactor() + historyDelegate.historyFactor();
    }
}
```

## When to Use

✅ **Use Replace Conditional with Polymorphism when:**

- Switch/if-else based on type code
- Same type-based conditional appears multiple places
- Adding types requires changing existing code
- You want Open-Closed principle compliance

❌ **Avoid Replace Conditional with Polymorphism when:**

- The conditional is simple and appears once
- Adding subclasses would be over-engineering
- The types aren't truly polymorphic
- You have very few variations

## Related Refactorings

- **Replace Type Code with Subclasses**: Often precedes this
- **Extract Class**: To create the new subclasses
- **Move Function**: To move behavior to subclasses
- **Introduce Special Case**: For null handling specifically
