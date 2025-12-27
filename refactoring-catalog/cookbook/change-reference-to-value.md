# Change Reference to Value

## Intent

Replace a reference object with a value object, making it immutable and allowing equality comparison by content rather than identity.

## Code Smells That Indicate This Refactoring

- Reference object is never mutated after creation
- Reference management (repository/factory) adds complexity
- You want to compare objects by their content
- Object would be simpler if immutable
- Need to use object as a map key or in a set

## Mechanics

1. Check that the candidate class is immutable or can become immutable
2. Remove any setters/mutating methods
3. Provide a value-based equality method (equals/hashCode)
4. Test
5. Consider if you also need to remove the reference repository

## Example

### Before

```java
public class TelephoneNumber {
    private String areaCode;
    private String number;

    public String getAreaCode() {
        return areaCode;
    }

    public void setAreaCode(String areaCode) {
        this.areaCode = areaCode;
    }

    public String getNumber() {
        return number;
    }

    public void setNumber(String number) {
        this.number = number;
    }
}

public class Person {
    private TelephoneNumber telephoneNumber;

    public TelephoneNumber getTelephoneNumber() {
        return telephoneNumber;
    }
}

// Problematic: external mutation possible
person.getTelephoneNumber().setAreaCode("312");
```

### After

```java
public final class TelephoneNumber {
    private final String areaCode;
    private final String number;

    public TelephoneNumber(String areaCode, String number) {
        this.areaCode = areaCode;
        this.number = number;
    }

    public String getAreaCode() {
        return areaCode;
    }

    public String getNumber() {
        return number;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        TelephoneNumber that = (TelephoneNumber) o;
        return Objects.equals(areaCode, that.areaCode)
            && Objects.equals(number, that.number);
    }

    @Override
    public int hashCode() {
        return Objects.hash(areaCode, number);
    }

    @Override
    public String toString() {
        return String.format("(%s) %s", areaCode, number);
    }
}

public class Person {
    private TelephoneNumber telephoneNumber;

    public TelephoneNumber getTelephoneNumber() {
        return telephoneNumber;
    }

    public void setTelephoneNumber(TelephoneNumber telephoneNumber) {
        this.telephoneNumber = telephoneNumber;
    }
}

// To change, create a new value
person.setTelephoneNumber(new TelephoneNumber("312", "555-1234"));
```

### Using Java Record (Java 16+)

```java
public record TelephoneNumber(String areaCode, String number) {
    public TelephoneNumber {
        Objects.requireNonNull(areaCode, "areaCode cannot be null");
        Objects.requireNonNull(number, "number cannot be null");
    }

    @Override
    public String toString() {
        return String.format("(%s) %s", areaCode, number);
    }
}
```

## When to Use

✅ **Use Change Reference to Value when:**

- The object is naturally immutable (dates, money, coordinates)
- You need to use it as a map key or set element
- You want to avoid accidental mutation
- Thread safety is important
- The object is small and cheap to copy

❌ **Avoid Change Reference to Value when:**

- The object is large and copying is expensive
- Multiple clients need to see updates to the same object
- Identity matters more than equality
- Mutation is a core part of the object's nature

## Related Refactorings

- **Change Value to Reference**: The inverse refactoring
- **Replace Primitive with Object**: Often creates candidates for value objects
- **Remove Setting Method**: Part of making the object immutable
