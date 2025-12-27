# Extract Superclass

## Intent

Create a new superclass and move common features from existing classes into it.

## Code Smells That Indicate This Refactoring

- Multiple classes with duplicate fields and methods
- Similar classes that could share a common base
- Repeated code across unrelated classes
- Candidates for a type hierarchy

## Mechanics

1. Create a new superclass
2. Adjust subclass constructors to call super
3. Use **Pull Up Method** for common methods
4. Use **Pull Up Field** for common fields
5. Use **Pull Up Constructor Body** for common initialization
6. Check clients to see if they can use the superclass type

## Example

### Before

```java
public class Employee {
    private String name;
    private String id;
    private double monthlyCost;

    public Employee(String name, String id, double monthlyCost) {
        this.name = name;
        this.id = id;
        this.monthlyCost = monthlyCost;
    }

    public String getName() { return name; }
    public String getId() { return id; }
    public double getMonthlyCost() { return monthlyCost; }

    public double getAnnualCost() {
        return monthlyCost * 12;
    }
}

public class Department {
    private String name;
    private List<Employee> staff;

    public Department(String name) {
        this.name = name;
        this.staff = new ArrayList<>();
    }

    public String getName() { return name; }

    public double getMonthlyCost() {
        return staff.stream()
            .mapToDouble(Employee::getMonthlyCost)
            .sum();
    }

    public double getAnnualCost() {
        return getMonthlyCost() * 12;
    }
}
```

### After

```java
public abstract class Party {
    protected String name;

    protected Party(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public abstract double getMonthlyCost();

    public double getAnnualCost() {
        return getMonthlyCost() * 12;
    }
}

public class Employee extends Party {
    private String id;
    private double monthlyCost;

    public Employee(String name, String id, double monthlyCost) {
        super(name);
        this.id = id;
        this.monthlyCost = monthlyCost;
    }

    public String getId() { return id; }

    @Override
    public double getMonthlyCost() {
        return monthlyCost;
    }
}

public class Department extends Party {
    private List<Employee> staff;

    public Department(String name) {
        super(name);
        this.staff = new ArrayList<>();
    }

    @Override
    public double getMonthlyCost() {
        return staff.stream()
            .mapToDouble(Employee::getMonthlyCost)
            .sum();
    }

    public void addStaff(Employee employee) {
        staff.add(employee);
    }
}

// Client can now use Party type
public double calculateTotalCost(List<Party> parties) {
    return parties.stream()
        .mapToDouble(Party::getAnnualCost)
        .sum();
}
```

## When to Use

✅ **Use Extract Superclass when:**

- Multiple classes share common features
- You want to use polymorphism across classes
- Shared behavior is being duplicated
- Similar classes should be unified

❌ **Avoid Extract Superclass when:**

- The commonality is coincidental, not conceptual
- Composition would be more appropriate
- The classes already have different inheritance
- Creating a superclass forces unnatural relationships

## Related Refactorings

- **Pull Up Method**: Used during extraction
- **Pull Up Field**: Used during extraction
- **Pull Up Constructor Body**: Used during extraction
- **Extract Class**: Alternative for composition
- **Replace Superclass with Delegate**: Inverse when inheritance is wrong
