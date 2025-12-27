# Remove Setting Method

## Intent

Remove a setter method when a field should only be set during construction.

## Code Smells That Indicate This Refactoring

- Field should be immutable after object creation
- Setter exists but should never be called after construction
- Field contains identity or configuration data
- Defensive programming against incorrect state changes

## Mechanics

1. Check that the setter is only called during construction
2. Modify the constructor to set the field directly
3. Remove the setter
4. Make the field final if possible
5. Test

## Example

### Before

```java
public class Person {
    private String id;
    private String name;

    public Person() {
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

// Usage
Person person = new Person();
person.setId("123");      // Should only happen at creation
person.setName("John");
```

### After

```java
public class Person {
    private final String id;
    private String name;

    public Person(String id, String name) {
        this.id = id;
        this.name = name;
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

// Usage
Person person = new Person("123", "John");
```

### With Builder Pattern

```java
public class Order {
    private final String orderId;
    private final LocalDateTime createdAt;
    private List<LineItem> items;
    private OrderStatus status;

    private Order(Builder builder) {
        this.orderId = builder.orderId;
        this.createdAt = LocalDateTime.now();
        this.items = new ArrayList<>(builder.items);
        this.status = OrderStatus.PENDING;
    }

    public static Builder builder(String orderId) {
        return new Builder(orderId);
    }

    // No setters for orderId and createdAt
    public String getOrderId() { return orderId; }
    public LocalDateTime getCreatedAt() { return createdAt; }

    // Status can change
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }

    public static class Builder {
        private final String orderId;
        private List<LineItem> items = new ArrayList<>();

        private Builder(String orderId) {
            this.orderId = orderId;
        }

        public Builder addItem(LineItem item) {
            items.add(item);
            return this;
        }

        public Order build() {
            return new Order(this);
        }
    }
}
```

### Collection Without Setter

```java
public class Team {
    private final String name;
    private final List<Player> players = new ArrayList<>();

    public Team(String name) {
        this.name = name;
    }

    // No setter for players list - provide add/remove instead
    public void addPlayer(Player player) {
        players.add(player);
    }

    public void removePlayer(Player player) {
        players.remove(player);
    }

    public List<Player> getPlayers() {
        return Collections.unmodifiableList(players);
    }
}
```

## When to Use

✅ **Use Remove Setting Method when:**

- The field should be immutable after creation
- The setter is only called during initialization
- The field represents identity (ID, creation time)
- You want to enforce immutability

❌ **Avoid Remove Setting Method when:**

- The field legitimately changes over time
- Framework requires setter (some ORMs, dependency injection)
- Backward compatibility is required
- The object has valid states with unset fields

## Framework Considerations

Some frameworks require property access:

```java
// JPA entity - may need setters for framework
@Entity
public class Customer {
    @Id
    private String id;

    // JPA needs no-arg constructor
    protected Customer() {}

    public Customer(String id) {
        this.id = id;
    }

    // Setter may be needed for JPA, make it protected
    protected void setId(String id) {
        this.id = id;
    }
}
```

## Related Refactorings

- **Encapsulate Variable**: Often precedes this (if field was public)
- **Change Reference to Value**: May lead to removing all setters
- **Replace Constructor with Factory Function**: Alternative for complex construction
