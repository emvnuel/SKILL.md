# Pull Up Constructor Body

## Intent

Move common constructor code from subclass constructors to the superclass constructor.

## Code Smells That Indicate This Refactoring

- Subclass constructors have duplicate code
- Same initialization logic in multiple subclasses
- Common fields being set in subclass constructors

## Mechanics

1. Define a superclass constructor if one doesn't exist
2. Move common statements from subclass constructor to the superclass constructor via `super()`
3. Move any common statements that can't be in the super call to a method and call from both
4. Test

## Example

### Before

```java
public abstract class Party {
    protected String name;
}

public class Employee extends Party {
    private final String id;
    private final int monthlyCost;

    public Employee(String name, String id, int monthlyCost) {
        this.name = name;  // Common initialization
        this.id = id;
        this.monthlyCost = monthlyCost;
    }
}

public class Department extends Party {
    private final Staff staff;

    public Department(String name, Staff staff) {
        this.name = name;  // Common initialization
        this.staff = staff;
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
}

public class Employee extends Party {
    private final String id;
    private final int monthlyCost;

    public Employee(String name, String id, int monthlyCost) {
        super(name);
        this.id = id;
        this.monthlyCost = monthlyCost;
    }
}

public class Department extends Party {
    private final Staff staff;

    public Department(String name, Staff staff) {
        super(name);
        this.staff = staff;
    }
}
```

### With Post-Super Initialization

```java
// Before
public class Manager extends Employee {
    private List<Employee> directReports;

    public Manager(String name, String id, int monthlyCost) {
        super(name, id, monthlyCost);
        this.directReports = new ArrayList<>();
        initializeReports();
    }
}

public class Executive extends Employee {
    private List<Employee> directReports;

    public Executive(String name, String id, int monthlyCost) {
        super(name, id, monthlyCost);
        this.directReports = new ArrayList<>();
        initializeReports();
    }
}

// After - extract common post-super logic
public abstract class ManagerialEmployee extends Employee {
    protected List<Employee> directReports;

    protected ManagerialEmployee(String name, String id, int monthlyCost) {
        super(name, id, monthlyCost);
        this.directReports = new ArrayList<>();
        initializeReports();
    }

    protected void initializeReports() {
        // Common initialization
    }
}

public class Manager extends ManagerialEmployee {
    public Manager(String name, String id, int monthlyCost) {
        super(name, id, monthlyCost);
    }
}
```

### Builder Pattern Alternative

```java
// When constructor logic is complex, consider builder
public abstract class Party {
    protected final String name;
    protected final LocalDate createdAt;
    protected final String createdBy;

    protected Party(PartyBuilder<?> builder) {
        this.name = builder.name;
        this.createdAt = builder.createdAt;
        this.createdBy = builder.createdBy;
    }

    public static abstract class PartyBuilder<T extends PartyBuilder<T>> {
        protected String name;
        protected LocalDate createdAt = LocalDate.now();
        protected String createdBy = "system";

        public T name(String name) {
            this.name = name;
            return self();
        }

        protected abstract T self();
    }
}
```

## When to Use

✅ **Use Pull Up Constructor Body when:**

- Subclass constructors have common initialization code
- Common fields are initialized the same way
- You want to enforce initialization in superclass
- DRY principle is being violated

❌ **Avoid Pull Up Constructor Body when:**

- Initialization differs significantly between subclasses
- Superclass constructor would become too complex
- The common code depends on subclass-specific state
- Some subclasses don't need the common initialization

## Related Refactorings

- **Pull Up Field**: Often done together
- **Pull Up Method**: For common post-construction logic
- **Extract Superclass**: When creating a new superclass
