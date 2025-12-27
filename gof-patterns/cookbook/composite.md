# Composite Pattern - Jakarta EE Cookbook

## Intent

Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

## Jakarta EE Implementation

### 1. Component Interface

```java
public interface MenuComponent {
    String getName();
    BigDecimal getPrice();
    void print(int depth);

    // Default implementations for leaf nodes
    default void add(MenuComponent component) {
        throw new UnsupportedOperationException();
    }

    default void remove(MenuComponent component) {
        throw new UnsupportedOperationException();
    }

    default List<MenuComponent> getChildren() {
        return Collections.emptyList();
    }
}
```

### 2. Leaf Node

```java
@Entity
public class MenuItem implements MenuComponent {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private String description;
    private BigDecimal price;

    @Override
    public String getName() { return name; }

    @Override
    public BigDecimal getPrice() { return price; }

    @Override
    public void print(int depth) {
        System.out.println("  ".repeat(depth) + name + ": $" + price);
    }
}
```

### 3. Composite Node

```java
@Entity
public class MenuCategory implements MenuComponent {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderColumn
    private List<MenuComponent> children = new ArrayList<>();

    @Override
    public String getName() { return name; }

    @Override
    public BigDecimal getPrice() {
        // Aggregate price of all children
        return children.stream()
            .map(MenuComponent::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    @Override
    public void add(MenuComponent component) {
        children.add(component);
    }

    @Override
    public void remove(MenuComponent component) {
        children.remove(component);
    }

    @Override
    public List<MenuComponent> getChildren() {
        return Collections.unmodifiableList(children);
    }

    @Override
    public void print(int depth) {
        System.out.println("  ".repeat(depth) + "== " + name + " ==");
        children.forEach(c -> c.print(depth + 1));
    }
}
```

### 4. Organization Hierarchy Example

```java
public interface OrganizationUnit {
    String getName();
    BigDecimal getBudget();
    int getHeadcount();
    List<OrganizationUnit> getSubunits();
}

@Entity
public class Employee implements OrganizationUnit {
    @Id private Long id;
    private String name;
    private BigDecimal salary;

    public String getName() { return name; }
    public BigDecimal getBudget() { return salary; }
    public int getHeadcount() { return 1; }
    public List<OrganizationUnit> getSubunits() { return List.of(); }
}

@Entity
public class Department implements OrganizationUnit {
    @Id private Long id;
    private String name;

    @OneToMany(cascade = CascadeType.ALL)
    private List<OrganizationUnit> members = new ArrayList<>();

    public String getName() { return name; }

    public BigDecimal getBudget() {
        return members.stream()
            .map(OrganizationUnit::getBudget)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public int getHeadcount() {
        return members.stream()
            .mapToInt(OrganizationUnit::getHeadcount)
            .sum();
    }

    public List<OrganizationUnit> getSubunits() { return members; }

    public void add(OrganizationUnit unit) { members.add(unit); }
}
```

### 5. Usage with CDI

```java
@ApplicationScoped
public class MenuService {

    @Inject
    MenuRepository repository;

    public MenuComponent getFullMenu() {
        MenuCategory root = new MenuCategory("Main Menu");

        MenuCategory appetizers = new MenuCategory("Appetizers");
        appetizers.add(new MenuItem("Soup", new BigDecimal("5.99")));
        appetizers.add(new MenuItem("Salad", new BigDecimal("7.99")));

        MenuCategory mains = new MenuCategory("Main Courses");
        mains.add(new MenuItem("Steak", new BigDecimal("24.99")));
        mains.add(new MenuItem("Fish", new BigDecimal("19.99")));

        root.add(appetizers);
        root.add(mains);

        return root;
    }

    public BigDecimal calculateTotalPrice(MenuComponent menu) {
        return menu.getPrice();  // Works uniformly for leaf or composite
    }
}
```

## When to Use

✅ **Use Composite when:**

- You have tree/hierarchy structures
- Clients should treat leaves and composites uniformly
- You need recursive operations over the structure

❌ **Avoid Composite when:**

- Structure is flat (no hierarchy)
- Leaves and composites have very different behaviors
