# Error Handling - Clean Code Cookbook

## Intent

Use exceptions rather than return codes. Write try-catch-finally statements first. Provide context with exceptions. Don't return null, don't pass null.

---

## Use Exceptions, Not Return Codes

### Before - Error Codes

```java
public int withdraw(Account account, Money amount) {
    if (account == null) return -1;
    if (amount.isNegative()) return -2;
    if (account.getBalance().lessThan(amount)) return -3;

    account.debit(amount);
    return 0;  // Success
}

// Caller must check and interpret codes
int result = withdraw(account, amount);
if (result == -1) {
    // handle null account
} else if (result == -2) {
    // handle negative amount
} else if (result == -3) {
    // handle insufficient funds
}
```

### After - Exceptions

```java
public void withdraw(Account account, Money amount) {
    requireNonNull(account, "Account cannot be null");

    if (amount.isNegative()) {
        throw new InvalidAmountException("Amount must be positive: " + amount);
    }
    if (account.getBalance().lessThan(amount)) {
        throw new InsufficientFundsException(account, amount);
    }

    account.debit(amount);
}

// Caller handles exceptions clearly
try {
    withdraw(account, amount);
} catch (InsufficientFundsException e) {
    notifyInsufficientFunds(e.getAccount(), e.getRequestedAmount());
}
```

---

## Write Try-Catch-Finally First

When writing code that might throw exceptions, start with the try-catch-finally structure:

```java
public void processFile(Path filePath) {
    BufferedReader reader = null;
    try {
        reader = Files.newBufferedReader(filePath);
        processContents(reader);
    } catch (IOException e) {
        throw new FileProcessingException("Failed to process: " + filePath, e);
    } finally {
        closeQuietly(reader);
    }
}

// Better: try-with-resources
public void processFile(Path filePath) {
    try (BufferedReader reader = Files.newBufferedReader(filePath)) {
        processContents(reader);
    } catch (IOException e) {
        throw new FileProcessingException("Failed to process: " + filePath, e);
    }
}
```

---

## Use Unchecked Exceptions

Checked exceptions violate the Open/Closed Principle - a change in a low-level method forces signature changes all the way up.

### Before - Checked Exception Cascade

```java
// Low level throws checked
public void readConfig() throws ConfigurationException {
    // ...
}

// Every caller must declare or handle
public void initializeApp() throws ConfigurationException {
    readConfig();
}

public void startServer() throws ConfigurationException {
    initializeApp();
}

// All the way to main
public static void main(String[] args) throws ConfigurationException {
    startServer();
}
```

### After - Unchecked Exception

```java
// Use unchecked exception
public class ConfigurationException extends RuntimeException {
    public ConfigurationException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Callers don't need to declare
public void initializeApp() {
    readConfig();  // ConfigurationException can propagate
}

// Handle at appropriate level
public static void main(String[] args) {
    try {
        startServer();
    } catch (ConfigurationException e) {
        log.error("Failed to start: {}", e.getMessage());
        System.exit(1);
    }
}
```

---

## Provide Context in Exceptions

### Before - No Context

```java
throw new IllegalStateException();
throw new RuntimeException("Error");
```

### After - Rich Context

```java
throw new OrderProcessingException(
    String.format("Failed to process order %s for customer %s: %s",
        orderId, customerId, reason)
);

// Custom exception with structured data
public class InsufficientFundsException extends RuntimeException {
    private final Account account;
    private final Money requestedAmount;
    private final Money availableBalance;

    public InsufficientFundsException(Account account, Money requestedAmount) {
        super(String.format(
            "Account %s has insufficient funds. Requested: %s, Available: %s",
            account.getId(), requestedAmount, account.getBalance()
        ));
        this.account = account;
        this.requestedAmount = requestedAmount;
        this.availableBalance = account.getBalance();
    }

    // Getters for programmatic access
    public Account getAccount() { return account; }
    public Money getRequestedAmount() { return requestedAmount; }
    public Money getAvailableBalance() { return availableBalance; }
}
```

---

## Define Exception Classes by Caller Needs

### Before - Too Many Specific Exceptions

```java
try {
    port.open();
} catch (DeviceResponseException e) {
    reportPortError(e);
    log.error("Device response problem", e);
} catch (ATM1212UnlockedException e) {
    reportPortError(e);
    log.error("Unlock exception", e);
} catch (GMXError e) {
    reportPortError(e);
    log.error("Device response exception", e);
}
```

### After - Wrapped Exception

```java
public class LocalPort {
    private ACMEPort innerPort;

    public void open() {
        try {
            innerPort.open();
        } catch (DeviceResponseException | ATM1212UnlockedException | GMXError e) {
            throw new PortDeviceException("Failed to open port", e);
        }
    }
}

// Caller only catches one type
try {
    localPort.open();
} catch (PortDeviceException e) {
    reportPortError(e);
    log.error("Port error", e);
}
```

---

## Special Case Pattern

Instead of handling exceptional cases with conditionals, encapsulate in a class.

### Before - Null Checks Everywhere

```java
public void processMealExpenses(Employee employee) {
    MealExpenses expenses = expenseReportDAO.getMeals(employee.getId());

    if (expenses != null) {
        total += expenses.getTotal();
    } else {
        total += getMealPerDiem();
    }
}
```

### After - Special Case Object

```java
public interface MealExpenses {
    Money getTotal();
}

public class ActualMealExpenses implements MealExpenses {
    private final List<Expense> expenses;

    public Money getTotal() {
        return expenses.stream()
            .map(Expense::getAmount)
            .reduce(Money.ZERO, Money::add);
    }
}

public class PerDiemMealExpenses implements MealExpenses {
    public Money getTotal() {
        return MealExpenses.PER_DIEM;
    }
}

// DAO returns special case instead of null
public MealExpenses getMeals(Long employeeId) {
    List<Expense> expenses = findExpenses(employeeId);
    return expenses.isEmpty()
        ? new PerDiemMealExpenses()
        : new ActualMealExpenses(expenses);
}

// No null check needed
public void processMealExpenses(Employee employee) {
    MealExpenses expenses = expenseReportDAO.getMeals(employee.getId());
    total = total.add(expenses.getTotal());
}
```

---

## Exception Hierarchy Example

```java
// Base exception for domain
public abstract class DomainException extends RuntimeException {
    protected DomainException(String message) {
        super(message);
    }
    protected DomainException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Business rule violations
public class BusinessRuleException extends DomainException {
    public BusinessRuleException(String rule) {
        super("Business rule violated: " + rule);
    }
}

// Entity not found
public class EntityNotFoundException extends DomainException {
    public EntityNotFoundException(String entityType, Object id) {
        super(String.format("%s not found with id: %s", entityType, id));
    }
}

// Validation errors
public class ValidationException extends DomainException {
    private final List<String> errors;

    public ValidationException(List<String> errors) {
        super("Validation failed: " + String.join(", ", errors));
        this.errors = errors;
    }

    public List<String> getErrors() {
        return errors;
    }
}
```

---

## Jakarta EE Exception Mapping

```java
@Provider
public class DomainExceptionMapper implements ExceptionMapper<DomainException> {

    @Override
    public Response toResponse(DomainException e) {
        if (e instanceof EntityNotFoundException) {
            return Response.status(Status.NOT_FOUND)
                .entity(new ErrorResponse(e.getMessage()))
                .build();
        }
        if (e instanceof ValidationException ve) {
            return Response.status(Status.BAD_REQUEST)
                .entity(new ValidationErrorResponse(ve.getErrors()))
                .build();
        }
        if (e instanceof BusinessRuleException) {
            return Response.status(Status.UNPROCESSABLE_ENTITY)
                .entity(new ErrorResponse(e.getMessage()))
                .build();
        }
        return Response.status(Status.INTERNAL_SERVER_ERROR)
            .entity(new ErrorResponse("Internal error"))
            .build();
    }
}
```

---

## Related Recipes

- [Null Handling](./null-handling.md): Avoid null return values
- [Small Functions](./small-functions.md): Try blocks should be small
