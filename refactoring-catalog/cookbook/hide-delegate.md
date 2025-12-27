# Hide Delegate

## Intent

Encapsulate a delegate to hide the chain of method calls and reduce coupling.

## Code Smells That Indicate This Refactoring

- Client is calling through one object to reach another
- Client knows about the server's internal structure
- Changes to delegate structure break clients
- Law of Demeter violations

## Mechanics

1. For each method on the delegate called by the client, create a delegating method on the server
2. Change the client to call the server
3. Test after each change
4. If no client needs to access the delegate anymore, remove the accessor

## Example

### Before

```java
public class Person {
    private Department department;

    public Department getDepartment() {
        return department;
    }

    public void setDepartment(Department department) {
        this.department = department;
    }
}

public class Department {
    private Person manager;

    public Person getManager() {
        return manager;
    }

    public void setManager(Person manager) {
        this.manager = manager;
    }
}

// Client - knows about internal structure
Person manager = person.getDepartment().getManager();
```

### After

```java
public class Person {
    private Department department;

    public void setDepartment(Department department) {
        this.department = department;
    }

    public Person getManager() {
        return department.getManager();
    }
}

public class Department {
    private Person manager;

    public Person getManager() {
        return manager;
    }

    public void setManager(Person manager) {
        this.manager = manager;
    }
}

// Client - doesn't need to know about Department
Person manager = person.getManager();
```

### Service Layer Example

```java
// Before - controller reaches through service to repository
public class OrderController {
    @Inject
    private OrderService orderService;

    public List<Order> getRecentOrders() {
        return orderService.getRepository().findRecent();
    }
}

// After - service provides the operation directly
public class OrderService {
    private OrderRepository repository;

    public List<Order> findRecentOrders() {
        return repository.findRecent();
    }
}

public class OrderController {
    @Inject
    private OrderService orderService;

    public List<Order> getRecentOrders() {
        return orderService.findRecentOrders();
    }
}
```

## Law of Demeter

This refactoring helps follow the Law of Demeter: "Only talk to your immediate friends."

```java
// Violates Law of Demeter
invoice.getCustomer().getAddress().getCity()

// Better - hide the chain
invoice.getShippingCity()
```

## When to Use

✅ **Use Hide Delegate when:**

- Clients are reaching through objects
- You want to reduce coupling
- Internal structure might change
- Following Law of Demeter is important

❌ **Avoid Hide Delegate when:**

- It would create too many forwarding methods
- The delegate is stable and well-known
- Clients genuinely need full delegate access
- See **Remove Middle Man** for when delegation goes too far

## Related Refactorings

- **Remove Middle Man**: The inverse when there are too many delegates
- **Extract Class**: May follow to create a new delegate
- **Move Function**: To the appropriate place in the chain
