# Change Value to Reference

## Intent

Replace multiple copies of the same conceptual data with a single shared reference object, ensuring consistency across the system.

## Code Smells That Indicate This Refactoring

- Multiple objects represent the same real-world entity
- Updates to one copy don't reflect in others
- Inconsistent data across different parts of the system
- Need to compare identity, not just equality

## Mechanics

1. Create a repository (if one doesn't exist) for instances of the related object
2. Ensure the constructor has access to the repository
3. Modify the constructor to look up the canonical instance in the repository
4. Test

## Example

### Before

```java
public class Order {
    private Customer customer;

    public Order(String customerName) {
        this.customer = new Customer(customerName);
    }

    public Customer getCustomer() {
        return customer;
    }
}

public class Customer {
    private String name;

    public Customer(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

// Usage - creates separate Customer instances
Order order1 = new Order("Acme Corp");
Order order2 = new Order("Acme Corp");
// order1.getCustomer() != order2.getCustomer() (different objects)
```

### After

```java
public class CustomerRepository {
    private static final Map<String, Customer> instances = new HashMap<>();

    public static Customer get(String name) {
        return instances.computeIfAbsent(name, Customer::new);
    }

    public static void register(Customer customer) {
        instances.put(customer.getName(), customer);
    }
}

public class Customer {
    private String name;

    public Customer(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

public class Order {
    private Customer customer;

    public Order(String customerName) {
        this.customer = CustomerRepository.get(customerName);
    }

    public Customer getCustomer() {
        return customer;
    }
}

// Usage - shares Customer instances
Order order1 = new Order("Acme Corp");
Order order2 = new Order("Acme Corp");
// order1.getCustomer() == order2.getCustomer() (same object)
```

### CDI/Jakarta EE Version

```java
@ApplicationScoped
public class CustomerRepository {
    private final Map<String, Customer> instances = new ConcurrentHashMap<>();

    public Customer findOrCreate(String name) {
        return instances.computeIfAbsent(name, Customer::new);
    }
}

@ApplicationScoped
public class OrderFactory {
    @Inject
    CustomerRepository customerRepository;

    public Order createOrder(String customerName) {
        Customer customer = customerRepository.findOrCreate(customerName);
        return new Order(customer);
    }
}
```

## When to Use

✅ **Use Change Value to Reference when:**

- Multiple objects represent the same real-world entity
- Changes to one should be visible to all users
- You need identity comparison (==) not just equality
- Memory efficiency matters with many duplicate values

❌ **Avoid Change Value to Reference when:**

- Immutability is important
- Different contexts need different views of the same data
- Shared reference would cause threading issues
- The simplicity of value objects is preferred

## Related Refactorings

- **Change Reference to Value**: The inverse refactoring
- **Replace Primitive with Object**: Often precedes this refactoring
- **Extract Class**: May be needed to create the reference type
