# Introduce Special Case

_Also known as: Introduce Null Object_

## Intent

Create a special-case object that encapsulates behavior for a common case that requires checking.

## Code Smells That Indicate This Refactoring

- Multiple places checking for null or a special value
- Same default behavior duplicated after each check
- Conditional logic for "unknown" or "not found" cases
- Special handling for a particular value scattered throughout code

## Mechanics

1. Add a special-case checking property to the subject, returning false
2. Create a special-case class with the checking property returning true
3. Apply **Extract Function** to the special-case comparison
4. Introduce the special-case object at the point of creation
5. Change the condition to use the checking property
6. Test
7. Move special-case handling logic into the special-case class
8. Test

## Example

### Before

```java
public class Site {
    private Customer customer;

    public Customer getCustomer() {
        return customer;
    }
}

public class Customer {
    private String name;
    private BillingPlan billingPlan;
    private PaymentHistory paymentHistory;

    public String getName() { return name; }
    public BillingPlan getBillingPlan() { return billingPlan; }
    public PaymentHistory getPaymentHistory() { return paymentHistory; }
}

// Client code - null checks everywhere
public class BillingService {
    public void billSite(Site site) {
        Customer customer = site.getCustomer();
        String customerName;
        BillingPlan plan;
        int weeksDelinquent;

        if (customer == null) {
            customerName = "occupant";
            plan = BillingPlan.basic();
            weeksDelinquent = 0;
        } else {
            customerName = customer.getName();
            plan = customer.getBillingPlan();
            weeksDelinquent = customer.getPaymentHistory().getWeeksDelinquent();
        }
        // ... use these values
    }
}
```

### After

```java
public interface Customer {
    String getName();
    BillingPlan getBillingPlan();
    PaymentHistory getPaymentHistory();
    boolean isUnknown();
}

public class RealCustomer implements Customer {
    private String name;
    private BillingPlan billingPlan;
    private PaymentHistory paymentHistory;

    @Override
    public String getName() { return name; }
    @Override
    public BillingPlan getBillingPlan() { return billingPlan; }
    @Override
    public PaymentHistory getPaymentHistory() { return paymentHistory; }
    @Override
    public boolean isUnknown() { return false; }
}

public class UnknownCustomer implements Customer {
    @Override
    public String getName() { return "occupant"; }
    @Override
    public BillingPlan getBillingPlan() { return BillingPlan.basic(); }
    @Override
    public PaymentHistory getPaymentHistory() { return new NullPaymentHistory(); }
    @Override
    public boolean isUnknown() { return true; }
}

public class NullPaymentHistory extends PaymentHistory {
    @Override
    public int getWeeksDelinquent() { return 0; }
}

public class Site {
    private Customer customer;

    public Customer getCustomer() {
        return customer != null ? customer : new UnknownCustomer();
    }
}

// Client code - no null checks needed
public class BillingService {
    public void billSite(Site site) {
        Customer customer = site.getCustomer();
        String customerName = customer.getName();
        BillingPlan plan = customer.getBillingPlan();
        int weeksDelinquent = customer.getPaymentHistory().getWeeksDelinquent();
        // ... use these values
    }
}
```

### Using Optional with Special Case

```java
public class CustomerRepository {
    public Optional<Customer> findById(String id) {
        // ...
    }

    public Customer findByIdOrUnknown(String id) {
        return findById(id).orElseGet(UnknownCustomer::new);
    }
}
```

## When to Use

✅ **Use Introduce Special Case when:**

- You see repeated null checks with the same default behavior
- A special value has consistent handling throughout the code
- You want to eliminate conditional logic
- The special case has well-defined behavior

❌ **Avoid Introduce Special Case when:**

- The special case handling varies between call sites
- Creating a special class adds too much complexity
- The object is rarely null or special
- Optional/null handling is clearer

## Related Refactorings

- **Extract Class**: To create the special case class
- **Replace Conditional with Polymorphism**: Similar technique
- **Consolidate Conditional Expression**: May precede this
