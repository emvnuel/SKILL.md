# Remove Middle Man

## Intent

Remove excessive delegation methods that don't add value by letting clients call the delegate directly.

## Code Smells That Indicate This Refactoring

- Class has too many simple delegating methods
- Forwarding methods add no value
- Changes to delegate require changes to the middleman
- Middle man just passes requests through

## Mechanics

1. Create a getter for the delegate
2. For each client use of a delegating method, remove the method and have the client call the delegate directly
3. Test after each removal

## Example

### Before

```java
public class Person {
    private Department department;

    public Person getManager() {
        return department.getManager();
    }

    public String getDepartmentName() {
        return department.getName();
    }

    public List<Person> getColleagues() {
        return department.getMembers();
    }

    public String getDepartmentLocation() {
        return department.getLocation();
    }

    public Budget getDepartmentBudget() {
        return department.getBudget();
    }
    // ... many more delegating methods
}

// Client
Person manager = john.getManager();
String location = john.getDepartmentLocation();
```

### After

```java
public class Person {
    private Department department;

    public Department getDepartment() {
        return department;
    }
}

// Client
Department dept = john.getDepartment();
Person manager = dept.getManager();
String location = dept.getLocation();
```

### Balanced Approach

```java
// Keep the most common/meaningful delegations
public class Person {
    private Department department;

    // Keep frequently used delegation
    public Person getManager() {
        return department.getManager();
    }

    // Expose department for less common access
    public Department getDepartment() {
        return department;
    }
}

// Client uses either based on needs
Person manager = john.getManager();  // Common case
List<Person> allMembers = john.getDepartment().getMembers();  // Less common
```

## Finding the Balance

Hide Delegate and Remove Middle Man are opposing forces:

| Hide Delegate            | Remove Middle Man          |
| ------------------------ | -------------------------- |
| Reduces coupling         | Reduces forwarding methods |
| Hides internal structure | Exposes delegate           |
| More methods on server   | Fewer methods on server    |

The right balance depends on:

- How often delegating methods are used
- How stable the delegate is
- How much clients need to know about the delegate

## When to Use

✅ **Use Remove Middle Man when:**

- Class has become a passthrough
- Too many simple forwarding methods
- Clients need access to the full delegate API
- Delegation doesn't add value

❌ **Avoid Remove Middle Man when:**

- Forwarding methods add validation/transformation
- You want to hide the delegate implementation
- The delegate might change
- Law of Demeter is important

## Related Refactorings

- **Hide Delegate**: The inverse operation
- **Inline Function**: Similar simplification for methods
- **Inline Class**: When the whole class becomes a middleman
