# Side Effects - Clean Code Cookbook

## Intent

Functions should not have hidden side effects. A side effect is any modification of state that is not apparent from the function name or signature.

---

## What Are Side Effects?

Side effects include:

- Modifying input parameters
- Changing instance variables unexpectedly
- Modifying global/static state
- Making unexpected external calls (DB, network, files)
- Changing state of injected dependencies

---

## Hidden Side Effects

### Before - Hidden Side Effect

```java
public class UserValidator {
    private Session session;

    public boolean checkPassword(String userName, String password) {
        User user = userRepository.findByName(userName);
        if (user == null) {
            return false;
        }

        String encodedPassword = encoder.encode(password);
        if (encodedPassword.equals(user.getPassword())) {
            // HIDDEN SIDE EFFECT - not apparent from method name
            session.initialize(user);
            return true;
        }
        return false;
    }
}
```

### After - Explicit Behavior

```java
public class UserValidator {

    // Pure function - no side effects
    public boolean checkPassword(String userName, String password) {
        User user = userRepository.findByName(userName);
        if (user == null) {
            return false;
        }

        String encodedPassword = encoder.encode(password);
        return encodedPassword.equals(user.getPassword());
    }
}

// Caller handles session explicitly
public class LoginService {
    @Inject UserValidator validator;
    @Inject Session session;

    public LoginResult login(String userName, String password) {
        if (validator.checkPassword(userName, password)) {
            User user = userRepository.findByName(userName);
            session.initialize(user);  // Side effect is visible here
            return LoginResult.success(user);
        }
        return LoginResult.failure("Invalid credentials");
    }
}
```

---

## Temporal Coupling

Hidden side effects create temporal coupling - functions must be called in specific order.

### Before - Temporal Coupling

```java
public class OrderProcessor {
    private Order currentOrder;
    private Customer currentCustomer;

    public void setOrder(Order order) {
        this.currentOrder = order;  // Must call first
    }

    public void setCustomer(Customer customer) {
        this.currentCustomer = customer;  // Must call second
    }

    public void process() {  // Must call third
        // Uses currentOrder and currentCustomer
        validateOrder(currentOrder);
        chargeCustomer(currentCustomer, currentOrder.getTotal());
    }
}

// Caller must know the correct order
processor.setOrder(order);
processor.setCustomer(customer);
processor.process();
```

### After - Explicit Dependencies

```java
public class OrderProcessor {

    // All dependencies explicit in signature
    public ProcessingResult process(Order order, Customer customer) {
        validateOrder(order);
        chargeCustomer(customer, order.getTotal());
        return ProcessingResult.success();
    }
}

// Caller has no temporal coupling
processor.process(order, customer);
```

---

## Modifying Input Parameters

### Before - Mutating Input

```java
public void addTransaction(Account account, Transaction tx) {
    // Mutates the input - caller may not expect this
    account.getTransactions().add(tx);
    account.setBalance(account.getBalance().add(tx.getAmount()));
    account.setLastModified(Instant.now());
}
```

### After - Return New State

```java
// Option 1: Return updated object (immutable style)
public Account addTransaction(Account account, Transaction tx) {
    List<Transaction> newTransactions = new ArrayList<>(account.getTransactions());
    newTransactions.add(tx);

    return new Account(
        account.getId(),
        account.getBalance().add(tx.getAmount()),
        newTransactions,
        Instant.now()
    );
}

// Option 2: Make mutation explicit via command object
public class AddTransactionCommand {
    private final Account account;
    private final Transaction transaction;

    public void execute() {
        account.addTransaction(transaction);  // Account manages its own state
    }
}

// Option 3: Account manages its own mutations (encapsulation)
public class Account {
    public void addTransaction(Transaction tx) {
        this.transactions.add(tx);
        this.balance = this.balance.add(tx.getAmount());
        this.lastModified = Instant.now();
    }
}
```

---

## Pure Functions

Prefer pure functions when possible - same input always produces same output, no side effects.

### Pure Function Example

```java
// Pure - no side effects, deterministic
public Money calculateDiscount(Order order, DiscountPolicy policy) {
    Money total = order.getTotal();
    Percentage discount = policy.getDiscountFor(order.getCustomerType());
    return total.multiply(discount);
}

// Pure - transforms input to output
public OrderDTO toDTO(Order order) {
    return new OrderDTO(
        order.getId(),
        order.getTotal().toString(),
        order.getItems().stream()
            .map(this::toItemDTO)
            .toList()
    );
}
```

### Impure But Necessary

```java
// Impure but clearly named - side effect is expected
public void saveOrder(Order order) {
    repository.save(order);  // Side effect: database write
    eventBus.publish(new OrderSavedEvent(order));  // Side effect: event
}
```

---

## Functional Core, Imperative Shell

Separate pure logic from side effects:

```java
// Core: Pure business logic
public class PricingCalculator {

    public OrderPricing calculatePricing(Order order, PricingRules rules) {
        Money subtotal = calculateSubtotal(order);
        Money discount = applyDiscounts(subtotal, order.getCustomer(), rules);
        Money tax = calculateTax(subtotal.subtract(discount), rules);
        Money total = subtotal.subtract(discount).add(tax);

        return new OrderPricing(subtotal, discount, tax, total);
    }
}

// Shell: Handles side effects
@ApplicationScoped
public class OrderService {

    @Inject
    OrderRepository repository;

    @Inject
    PricingCalculator calculator;

    @Inject
    Event<OrderPlacedEvent> orderPlacedEvent;

    @Transactional
    public Order placeOrder(Order order) {
        PricingRules rules = repository.loadPricingRules();  // Side effect: DB read

        OrderPricing pricing = calculator.calculatePricing(order, rules);  // Pure
        order.applyPricing(pricing);  // Mutation

        Order saved = repository.save(order);  // Side effect: DB write
        orderPlacedEvent.fire(new OrderPlacedEvent(saved));  // Side effect: event

        return saved;
    }
}
```

---

## Side Effects Checklist

| Side Effect Type       | How to Handle                                   |
| ---------------------- | ----------------------------------------------- |
| Modify input params    | Return new value or make caller do mutation     |
| Change instance vars   | Make mutation methods explicit (verb names)     |
| Global/static state    | Use dependency injection instead                |
| Database operations    | Separate repository layer, name methods clearly |
| External service calls | Isolate in dedicated service classes            |
| File I/O               | Separate I/O concerns from business logic       |

---

## Related Recipes

- [Command Query Separation](./command-query-separation.md): Separate state changes from queries
- [Small Functions](./small-functions.md): One thing includes one side effect
