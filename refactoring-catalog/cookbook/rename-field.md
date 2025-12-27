# Rename Field

## Intent

Change a field name to better communicate its purpose.

## Code Smells That Indicate This Refactoring

- Field name is cryptic or abbreviated
- Name doesn't match what the field actually represents
- Name uses outdated domain terminology
- Name doesn't follow naming conventions
- Name is inconsistent with similar fields in other classes

## Mechanics

1. If the field has limited scope, rename all usages and test
2. If the field is widely used:
   - Encapsulate the field if not already
   - Rename the private field
   - Update accessors if they follow the field name
   - Test

## Example

### Before

```java
public class Organization {
    private String nm;
    private String ctry;

    public Organization(String name) {
        this.nm = name;
    }

    public String getNm() {
        return nm;
    }

    public void setNm(String nm) {
        this.nm = nm;
    }

    public String getCtry() {
        return ctry;
    }
}
```

### After

```java
public class Organization {
    private String name;
    private String country;

    public Organization(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getCountry() {
        return country;
    }
}
```

### Record Example (Java 16+)

```java
// Before
public record Person(String nm, int yob) {}

// After
public record Person(String name, int yearOfBirth) {}
```

## When to Use

✅ **Use Rename Field when:**

- Field name doesn't describe its content
- Domain terminology has evolved
- Name uses abbreviations that aren't universally understood
- Field was auto-generated with poor naming
- Team conventions have changed

❌ **Avoid renaming when:**

- The name is part of a stable API
- The name follows a serialization contract (JSON/XML mapping)
- It's a well-understood abbreviation in your domain
- The scope is small and the name is clear enough

## Related Refactorings

- **Rename Variable**: Similar for local variables
- **Encapsulate Variable**: Often applied before renaming public fields
- **Change Function Declaration**: For renaming associated accessors
