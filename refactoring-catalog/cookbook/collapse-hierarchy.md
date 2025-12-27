# Collapse Hierarchy

## Intent

Merge a subclass into its superclass when the subclass provides no significant value.

## Code Smells That Indicate This Refactoring

- Subclass adds almost nothing to superclass
- Subclass is rarely distinguished from superclass in code
- The distinction doesn't justify the complexity
- Subclass was created "just in case" but never diverged

## Mechanics

1. Check which class you want to keep (usually the more used one)
2. Use **Pull Up Field** and **Pull Up Method** to move everything to the survivor (or **Push Down** to combine into subclass)
3. Adjust references to the removed class
4. Delete the empty class
5. Test

## Example

### Before

```java
public class Employee {
    protected String name;
    protected double salary;

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() { return name; }
    public double getSalary() { return salary; }

    public double getAnnualCost() {
        return salary * 12;
    }
}

public class Salesperson extends Employee {
    // Originally expected to add commission logic
    // but it never happened

    public Salesperson(String name, double salary) {
        super(name, salary);
    }

    // No new methods or fields
}
```

### After

```java
public class Employee {
    private String name;
    private double salary;
    private boolean isSalesperson;  // If distinction still needed

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
        this.isSalesperson = false;
    }

    public static Employee createSalesperson(String name, double salary) {
        Employee emp = new Employee(name, salary);
        emp.isSalesperson = true;
        return emp;
    }

    public String getName() { return name; }
    public double getSalary() { return salary; }
    public boolean isSalesperson() { return isSalesperson; }

    public double getAnnualCost() {
        return salary * 12;
    }
}
```

### Merging Into Subclass

```java
// Before - abstract class with one implementation
public abstract class Shape {
    protected double x, y;

    public abstract double getArea();
}

public class Rectangle extends Shape {
    private double width;
    private double height;

    @Override
    public double getArea() {
        return width * height;
    }
}

// After - if Rectangle is the only Shape
public class Rectangle {
    private double x, y;
    private double width;
    private double height;

    public double getArea() {
        return width * height;
    }
}
```

## When to Use

✅ **Use Collapse Hierarchy when:**

- Subclass adds nothing meaningful
- The distinction causes maintenance overhead
- No other subclasses exist or are planned
- The subclass was speculative and unused

❌ **Avoid Collapse Hierarchy when:**

- Subclass will diverge in the future
- The hierarchy provides useful polymorphism
- Other subclasses exist
- The distinction is semantically important

## Related Refactorings

- **Pull Up Method**: Used during collapse
- **Pull Up Field**: Used during collapse
- **Remove Subclass**: Similar simplification
- **Extract Superclass**: The inverse operation
