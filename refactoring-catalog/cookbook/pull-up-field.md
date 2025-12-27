# Pull Up Field

## Intent

Move identical fields from subclasses to the superclass to eliminate duplication.

## Code Smells That Indicate This Refactoring

- Same field declared in multiple subclasses
- Subclasses use the field in identical ways
- Field has the same meaning across subclasses

## Mechanics

1. Inspect all uses of the field to ensure they are the same
2. If the fields have different names, use **Rename Field** to give them the same name
3. Create a new field in the superclass (make it protected or private with accessors)
4. Delete the subclass fields
5. Test

## Example

### Before

```java
public abstract class Employee {
    protected String name;
}

public class Engineer extends Employee {
    private LocalDate hireDate;

    public LocalDate getHireDate() {
        return hireDate;
    }

    public int getYearsOfService() {
        return Period.between(hireDate, LocalDate.now()).getYears();
    }
}

public class Salesperson extends Employee {
    private LocalDate hireDate;

    public LocalDate getHireDate() {
        return hireDate;
    }

    public boolean isEligibleForBonus() {
        return Period.between(hireDate, LocalDate.now()).getYears() >= 2;
    }
}
```

### After

```java
public abstract class Employee {
    protected String name;
    protected LocalDate hireDate;

    public LocalDate getHireDate() {
        return hireDate;
    }

    public int getYearsOfService() {
        return Period.between(hireDate, LocalDate.now()).getYears();
    }
}

public class Engineer extends Employee {
    // hireDate inherited from Employee
}

public class Salesperson extends Employee {
    public boolean isEligibleForBonus() {
        return getYearsOfService() >= 2;
    }
}
```

### With Encapsulation

```java
public abstract class Employee {
    private String name;
    private LocalDate hireDate;

    protected Employee(String name, LocalDate hireDate) {
        this.name = name;
        this.hireDate = hireDate;
    }

    public String getName() {
        return name;
    }

    public LocalDate getHireDate() {
        return hireDate;
    }
}

public class Engineer extends Employee {
    private String specialty;

    public Engineer(String name, LocalDate hireDate, String specialty) {
        super(name, hireDate);
        this.specialty = specialty;
    }
}

public class Salesperson extends Employee {
    private String region;

    public Salesperson(String name, LocalDate hireDate, String region) {
        super(name, hireDate);
        this.region = region;
    }
}
```

## When to Use

✅ **Use Pull Up Field when:**

- The same field appears in multiple subclasses
- The field has the same meaning and usage
- You want to reduce duplication
- The field logically belongs to the parent concept

❌ **Avoid Pull Up Field when:**

- Fields have different meanings despite same name
- Subclasses use the field differently
- Not all subclasses need the field
- Pulling up would expose internal details

## Related Refactorings

- **Pull Up Method**: Often done together
- **Extract Superclass**: When creating a new superclass
- **Pull Up Constructor Body**: For constructor field initialization
- **Push Down Field**: The inverse operation
