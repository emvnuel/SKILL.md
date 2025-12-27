# Separate Query from Modifier

## Intent

Split a method that returns a value and also has side effects into two methods: one for the query and one for the modification.

## Code Smells That Indicate This Refactoring

- Method returns a value but also changes state
- Hard to call method just for its return value
- Difficult to test because of hidden side effects
- Command-Query Separation (CQS) principle violation

## Mechanics

1. Copy the function and name it as a query
2. Remove side effects from the query
3. Run static checks
4. Remove the return value from the original (modifier)
5. Replace each call with separate query and modifier calls
6. Test

## Example

### Before

```java
public class SecuritySystem {
    private List<String> intruders = new ArrayList<>();

    public String alertAndReturnIntruder(List<String> people) {
        for (String person : people) {
            if (person.equals("Don")) {
                sendAlert();
                return "Don";
            }
            if (person.equals("John")) {
                sendAlert();
                return "John";
            }
        }
        return "";
    }

    private void sendAlert() {
        System.out.println("Intruder detected!");
    }
}

// Usage
String found = securitySystem.alertAndReturnIntruder(people);
if (!found.isEmpty()) {
    log("Found intruder: " + found);
}
```

### After

```java
public class SecuritySystem {
    private List<String> intruders = new ArrayList<>();

    public String findIntruder(List<String> people) {
        for (String person : people) {
            if (person.equals("Don")) {
                return "Don";
            }
            if (person.equals("John")) {
                return "John";
            }
        }
        return "";
    }

    public void alertIfIntruderFound(List<String> people) {
        if (!findIntruder(people).isEmpty()) {
            sendAlert();
        }
    }

    private void sendAlert() {
        System.out.println("Intruder detected!");
    }
}

// Usage
String found = securitySystem.findIntruder(people);
securitySystem.alertIfIntruderFound(people);
if (!found.isEmpty()) {
    log("Found intruder: " + found);
}
```

### Billing Example

```java
// Before
public class BillingService {
    public BigDecimal chargeCustomer(Customer customer, BigDecimal amount) {
        customer.deductFromBalance(amount);  // Side effect
        sendReceipt(customer, amount);        // Side effect
        return customer.getBalance();          // Query
    }
}

// After
public class BillingService {
    public BigDecimal getBalance(Customer customer) {
        return customer.getBalance();
    }

    public void chargeCustomer(Customer customer, BigDecimal amount) {
        customer.deductFromBalance(amount);
        sendReceipt(customer, amount);
    }
}

// Usage: separate query from modification
BigDecimal balanceBefore = billingService.getBalance(customer);
billingService.chargeCustomer(customer, amount);
BigDecimal balanceAfter = billingService.getBalance(customer);
```

## When to Use

✅ **Use Separate Query from Modifier when:**

- A method both reads and writes state
- You need to call the query without side effects
- Testing is difficult due to combined responsibilities
- You want to follow Command-Query Separation

❌ **Avoid Separate Query from Modifier when:**

- The combination is atomic (e.g., compareAndSet)
- Performance requires doing both at once
- The return value indicates the modification's success
- Separation would cause race conditions

## Concurrency Consideration

Sometimes the combination is intentional for atomicity:

```java
// This is correct - atomic operation
public synchronized Integer pollFirst() {
    if (queue.isEmpty()) return null;
    return queue.removeFirst();  // Both query and modify atomically
}

// Don't separate into:
// if (!queue.isEmpty()) { queue.removeFirst(); } // Race condition!
```

## Related Refactorings

- **Extract Function**: To create the separate methods
- **Remove Flag Argument**: Often related to query/command separation
- **Move Function**: May follow to organize queries and commands
