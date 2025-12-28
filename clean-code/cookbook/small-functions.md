# Small Functions - Clean Code Cookbook

## Intent

Functions should be small, do one thing, and operate at one level of abstraction. Smaller functions are easier to read, test, and maintain.

---

## The First Rule: Small

Functions should be **small**. Then they should be **smaller than that**.

### Guidelines

- Max 20 lines (ideally 4-10)
- Max 2-3 levels of indentation
- One level of abstraction per function
- Function does ONE thing

---

## One Level of Abstraction

### Before - Mixed Abstractions

```java
public String renderPage(PageData pageData) {
    StringBuilder result = new StringBuilder();

    // High level: getting page name
    String pageName = pageData.getName();

    // Low level: string manipulation
    result.append("<html>");
    result.append("<head>");
    result.append("<title>").append(pageName).append("</title>");
    result.append("</head>");

    // Medium level: getting content
    String content = pageData.getContent();

    // Low level: more string manipulation
    result.append("<body>");
    result.append(content);
    result.append("</body>");
    result.append("</html>");

    return result.toString();
}
```

### After - Single Level

```java
public String renderPage(PageData pageData) {
    return buildHtmlDocument(pageData.getName(), pageData.getContent());
}

private String buildHtmlDocument(String title, String bodyContent) {
    return wrapInHtml(
        buildHead(title),
        buildBody(bodyContent)
    );
}

private String wrapInHtml(String head, String body) {
    return "<html>" + head + body + "</html>";
}

private String buildHead(String title) {
    return "<head><title>" + title + "</title></head>";
}

private String buildBody(String content) {
    return "<body>" + content + "</body>";
}
```

---

## The Stepdown Rule

Code should read like a top-down narrative. Each function should be followed by functions at the next level of abstraction.

```java
// Level 1: High-level story
public void processOrder(Order order) {
    validateOrder(order);
    calculateTotals(order);
    applyDiscounts(order);
    persistOrder(order);
    notifyCustomer(order);
}

// Level 2: Details of validation
private void validateOrder(Order order) {
    validateCustomer(order.getCustomer());
    validateItems(order.getItems());
    validatePaymentMethod(order.getPayment());
}

// Level 3: Details of customer validation
private void validateCustomer(Customer customer) {
    requireNonNull(customer, "Customer is required");
    requireNonBlank(customer.getEmail(), "Email is required");
}
```

---

## Extract Till You Drop

### Before - Long Method

```java
public void pay(Employee employee) {
    if (employee.isHourly()) {
        double overtime = 0;
        double regularHours = employee.getHoursWorked();
        if (regularHours > 40) {
            overtime = regularHours - 40;
            regularHours = 40;
        }
        double regularPay = regularHours * employee.getHourlyRate();
        double overtimePay = overtime * employee.getHourlyRate() * 1.5;
        employee.setPay(regularPay + overtimePay);
    } else if (employee.isSalaried()) {
        employee.setPay(employee.getSalary() / 24);
    } else if (employee.isCommissioned()) {
        double basePay = employee.getSalary() / 24;
        double commission = employee.getSales() * employee.getCommissionRate();
        employee.setPay(basePay + commission);
    }
}
```

### After - Extracted Methods

```java
public void pay(Employee employee) {
    Money payment = calculatePayment(employee);
    employee.setPay(payment);
}

private Money calculatePayment(Employee employee) {
    return switch (employee.getPaymentType()) {
        case HOURLY -> calculateHourlyPay(employee);
        case SALARIED -> calculateSalariedPay(employee);
        case COMMISSIONED -> calculateCommissionedPay(employee);
    };
}

private Money calculateHourlyPay(Employee employee) {
    Hours hours = employee.getHoursWorked();
    Money regularPay = hours.getRegularHours().times(employee.getHourlyRate());
    Money overtimePay = hours.getOvertimeHours().times(employee.getOvertimeRate());
    return regularPay.add(overtimePay);
}

private Money calculateSalariedPay(Employee employee) {
    return employee.getSalary().divideBy(PAYMENTS_PER_YEAR);
}

private Money calculateCommissionedPay(Employee employee) {
    Money basePay = calculateSalariedPay(employee);
    Money commission = employee.getSales().times(employee.getCommissionRate());
    return basePay.add(commission);
}
```

---

## Switch Statements

Switch statements violate Single Responsibility. Bury them in low-level classes using polymorphism.

### Before - Exposed Switch

```java
public Money calculatePay(Employee employee) {
    return switch (employee.getType()) {
        case COMMISSIONED -> calculateCommissionedPay(employee);
        case HOURLY -> calculateHourlyPay(employee);
        case SALARIED -> calculateSalariedPay(employee);
        default -> throw new InvalidEmployeeType(employee.getType());
    };
}
```

### After - Factory + Polymorphism

```java
public interface PaymentStrategy {
    Money calculatePay(Employee employee);
}

@ApplicationScoped
@Named("hourly")
public class HourlyPaymentStrategy implements PaymentStrategy {
    @Override
    public Money calculatePay(Employee employee) {
        Hours hours = employee.getHoursWorked();
        return hours.getRegularHours().times(employee.getHourlyRate())
                    .add(hours.getOvertimeHours().times(employee.getOvertimeRate()));
    }
}

@ApplicationScoped
public class PayrollService {
    @Inject
    @Any
    Instance<PaymentStrategy> strategies;

    public Money calculatePay(Employee employee) {
        return strategies.select(new NamedLiteral(employee.getPaymentType()))
                         .get()
                         .calculatePay(employee);
    }
}
```

---

## Function Structure

### Ideal Structure

```java
// Guard clauses at top
// Main logic in middle
// Single return at bottom (or early returns for guards)

public Order processOrder(Order order) {
    // Guard clauses
    if (order == null) {
        throw new IllegalArgumentException("Order cannot be null");
    }
    if (order.getItems().isEmpty()) {
        throw new InvalidOrderException("Order must have items");
    }

    // Main logic - one thing
    Order validatedOrder = validateInventory(order);
    Order pricedOrder = calculatePrices(validatedOrder);

    return pricedOrder;
}
```

---

## Signs You Need to Extract

1. **Comments explaining a block** → Extract to named method
2. **Nested conditionals** → Extract to predicates
3. **Loop with logic** → Extract loop body
4. **Try/catch blocks** → Extract try body
5. **Repeated patterns** → Extract common behavior

### Comment to Method

```java
// Before - comment explaining what code does
public void sendNotification(User user, Event event) {
    // Check if user wants to receive notifications
    if (user.getPreferences().isEmailEnabled()
        && user.getEmail() != null
        && user.isVerified()) {
        emailService.send(user.getEmail(), event);
    }
}

// After - method name explains intent
public void sendNotification(User user, Event event) {
    if (userWantsEmailNotifications(user)) {
        emailService.send(user.getEmail(), event);
    }
}

private boolean userWantsEmailNotifications(User user) {
    return user.getPreferences().isEmailEnabled()
           && user.getEmail() != null
           && user.isVerified();
}
```

---

## Metrics

| Metric                | Good | Acceptable | Needs Refactoring |
| --------------------- | ---- | ---------- | ----------------- |
| Lines per function    | < 10 | 10-20      | > 20              |
| Nesting depth         | 1-2  | 3          | > 3               |
| Cyclomatic complexity | 1-4  | 5-10       | > 10              |
| Parameters            | 0-2  | 3          | > 3               |

---

## Related Recipes

- [Function Arguments](./function-arguments.md): Keep argument count low
- [Long Method](./long-method.md): Detecting and fixing long methods
- [Single Responsibility](./single-responsibility.md): One reason to change
