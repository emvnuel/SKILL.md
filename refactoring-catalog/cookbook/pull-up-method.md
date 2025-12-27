# Pull Up Method

## Intent

Move identical methods from subclasses to the superclass to eliminate duplication.

## Code Smells That Indicate This Refactoring

- Identical or very similar methods in sibling classes
- Duplicate code in subclasses
- Same behavior implemented multiple times
- Changes need to be made in multiple places

## Mechanics

1. Inspect methods to ensure they are identical (if not, consider **Parameterize Function** first)
2. Check that all method calls and field references are available in the superclass
3. If methods have different signatures, use **Change Function Declaration** to match
4. Create a new method in the superclass with the implementation
5. Delete one subclass method
6. Test
7. Continue deleting subclass methods until all are removed

## Example

### Before

```java
public abstract class Employee {
    protected String name;
    protected double baseSalary;
}

public class Engineer extends Employee {
    public double getAnnualCost() {
        return baseSalary * 12;
    }
}

public class Salesperson extends Employee {
    public double getAnnualCost() {
        return baseSalary * 12;
    }
}

public class Manager extends Employee {
    public double getAnnualCost() {
        return baseSalary * 12;
    }
}
```

### After

```java
public abstract class Employee {
    protected String name;
    protected double baseSalary;

    public double getAnnualCost() {
        return baseSalary * 12;
    }
}

public class Engineer extends Employee {
    // getAnnualCost() inherited from Employee
}

public class Salesperson extends Employee {
    // getAnnualCost() inherited from Employee
}

public class Manager extends Employee {
    // getAnnualCost() inherited from Employee
}
```

### With Slight Differences

```java
// Before - methods differ slightly
public class Engineer extends Employee {
    public String getDescription() {
        return "Engineer: " + name;
    }
}

public class Manager extends Employee {
    public String getDescription() {
        return "Manager: " + name;
    }
}

// After - extract the varying part
public abstract class Employee {
    protected String name;

    public String getDescription() {
        return getJobTitle() + ": " + name;
    }

    protected abstract String getJobTitle();
}

public class Engineer extends Employee {
    @Override
    protected String getJobTitle() {
        return "Engineer";
    }
}

public class Manager extends Employee {
    @Override
    protected String getJobTitle() {
        return "Manager";
    }
}
```

## When to Use

✅ **Use Pull Up Method when:**

- Subclasses have identical methods
- The method uses only data available in the superclass
- Changes to the method would need to happen in all subclasses
- You want to reduce code duplication

❌ **Avoid Pull Up Method when:**

- Methods have different behavior that should remain different
- The method relies on subclass-specific state
- Pulling up would violate the Liskov Substitution Principle
- Subclasses might need different implementations in the future

## Related Refactorings

- **Pull Up Field**: Often done together
- **Form Template Method**: When methods differ only in specific steps
- **Extract Superclass**: When creating a new superclass for the pulled-up method
- **Push Down Method**: The inverse operation
