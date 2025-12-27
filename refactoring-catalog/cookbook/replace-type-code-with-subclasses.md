# Replace Type Code with Subclasses

_Also known as: Extract Subclass, Replace Type Code with State/Strategy_

## Intent

Replace a type code field with subclasses to enable polymorphic behavior.

## Code Smells That Indicate This Refactoring

- Field that categorizes the object type
- Switch/if statements based on type code
- Behavior varies based on type field
- Type code is enum or string constant

## Mechanics

1. Self-encapsulate the type code field if it isn't already
2. Pick one type code value and make a subclass for it
3. Create a factory method to return the appropriate subclass
4. Change callers to use the factory method
5. Move type-specific behavior from conditionals to overriding methods
6. Repeat for each type code value
7. Remove the type code field

## Example

### Before

```java
public class Employee {
    private String name;
    private String type; // "engineer", "manager", "salesperson"

    public Employee(String name, String type) {
        this.name = name;
        this.type = type;
    }

    public int getBonus() {
        switch (type) {
            case "engineer": return 1000;
            case "manager": return 2000;
            case "salesperson": return 1500;
            default: throw new IllegalArgumentException();
        }
    }

    public String getTitle() {
        switch (type) {
            case "engineer": return "Software Engineer";
            case "manager": return "Department Manager";
            case "salesperson": return "Sales Representative";
            default: throw new IllegalArgumentException();
        }
    }
}
```

### After

```java
public abstract class Employee {
    protected String name;

    protected Employee(String name) {
        this.name = name;
    }

    public static Employee create(String name, String type) {
        switch (type) {
            case "engineer": return new Engineer(name);
            case "manager": return new Manager(name);
            case "salesperson": return new Salesperson(name);
            default: throw new IllegalArgumentException("Unknown type: " + type);
        }
    }

    public abstract int getBonus();
    public abstract String getTitle();
}

public class Engineer extends Employee {
    public Engineer(String name) {
        super(name);
    }

    @Override
    public int getBonus() {
        return 1000;
    }

    @Override
    public String getTitle() {
        return "Software Engineer";
    }
}

public class Manager extends Employee {
    public Manager(String name) {
        super(name);
    }

    @Override
    public int getBonus() {
        return 2000;
    }

    @Override
    public String getTitle() {
        return "Department Manager";
    }
}

public class Salesperson extends Employee {
    public Salesperson(String name) {
        super(name);
    }

    @Override
    public int getBonus() {
        return 1500;
    }

    @Override
    public String getTitle() {
        return "Sales Representative";
    }
}
```

### Using CDI for Type Resolution

```java
@ApplicationScoped
public class EmployeeFactory {
    @Inject @Any
    Instance<Employee> employees;

    public Employee create(String name, String type) {
        Employee template = employees.select(new EmployeeTypeLiteral(type)).get();
        return template.withName(name);
    }
}
```

## When to Use

✅ **Use Replace Type Code with Subclasses when:**

- Behavior varies based on type code
- Multiple switch statements use the same type code
- New types might be added in the future
- You want polymorphic instead of conditional logic

❌ **Avoid Replace Type Code with Subclasses when:**

- The type can change during object lifetime (use Strategy)
- There's only one switch and it's simple
- Adding classes would over-engineer the solution
- The type code is just data with no behavior

## Related Refactorings

- **Replace Conditional with Polymorphism**: The core technique
- **Remove Subclass**: The inverse operation
- **Replace Subclass with Delegate**: When type can change
- **Extract Superclass**: May be needed first
