# Function Arguments - Clean Code Cookbook

## Intent

The ideal number of arguments for a function is zero (niladic). Next comes one (monadic), followed by two (dyadic). Three arguments (triadic) should be avoided. More than three requires special justification.

---

## Argument Count Guidelines

| Count | Name     | Recommendation                    |
| ----- | -------- | --------------------------------- |
| 0     | Niladic  | Ideal                             |
| 1     | Monadic  | Good - common and acceptable      |
| 2     | Dyadic   | Acceptable - harder to understand |
| 3     | Triadic  | Avoid - consider refactoring      |
| 4+    | Polyadic | Refactor into parameter object    |

---

## Monadic Forms (One Argument)

### Common Reasons for One Argument

```java
// 1. Asking a question about the argument
boolean fileExists(File file);
boolean isValid(Order order);

// 2. Transforming the argument
InputStream openFile(String path);
Customer mapToCustomer(CustomerDTO dto);

// 3. Event - argument alters state
void passwordAttemptFailed(User user);
void orderPlaced(Order order);
```

---

## Dyadic Functions (Two Arguments)

### Naturally Dyadic

```java
// Natural ordering makes these acceptable
Point(int x, int y);
assertEquals(expected, actual);
Money.of(amount, currency);
```

### Convert to Monadic

```java
// Before - dyadic
writeField(outputStream, name);

// After - method on object
outputStream.writeField(name);

// Before - dyadic
assertEqualsIgnoreOrder(expected, actual);

// After - use assertion library fluent API
assertThat(actual).containsExactlyInAnyOrder(expected);
```

---

## Triadic Functions (Three Arguments)

Triadic functions are significantly harder to understand. Consider alternatives.

### Before - Three Arguments

```java
Circle makeCircle(double x, double y, double radius);
```

### After - Parameter Object

```java
Circle makeCircle(Point center, double radius);
```

---

## Parameter Objects

When a function needs more than 2-3 arguments, wrap them in a class.

### Before - Too Many Arguments

```java
public Order createOrder(
    String customerId,
    String productId,
    int quantity,
    String shippingAddress,
    String billingAddress,
    String paymentMethod,
    String discountCode
) {
    // ...
}
```

### After - Parameter Object

```java
public record CreateOrderRequest(
    String customerId,
    String productId,
    int quantity,
    Address shippingAddress,
    Address billingAddress,
    PaymentMethod paymentMethod,
    Optional<String> discountCode
) {}

public Order createOrder(CreateOrderRequest request) {
    // ...
}
```

---

## Builder for Complex Construction

When construction requires many optional parameters:

```java
public class EmailBuilder {
    private String to;
    private String from;
    private String subject;
    private String body;
    private List<String> cc = new ArrayList<>();
    private List<Attachment> attachments = new ArrayList<>();

    public EmailBuilder to(String to) {
        this.to = to;
        return this;
    }

    public EmailBuilder from(String from) {
        this.from = from;
        return this;
    }

    public EmailBuilder subject(String subject) {
        this.subject = subject;
        return this;
    }

    public EmailBuilder body(String body) {
        this.body = body;
        return this;
    }

    public EmailBuilder cc(String... addresses) {
        this.cc.addAll(Arrays.asList(addresses));
        return this;
    }

    public EmailBuilder attach(Attachment attachment) {
        this.attachments.add(attachment);
        return this;
    }

    public Email build() {
        validate();
        return new Email(to, from, subject, body, cc, attachments);
    }
}

// Usage
Email email = new EmailBuilder()
    .to("user@example.com")
    .from("noreply@company.com")
    .subject("Order Confirmation")
    .body("Your order has been placed.")
    .build();
```

---

## Flag Arguments

**Avoid boolean arguments** - they indicate the function does more than one thing.

### Before - Flag Argument

```java
render(boolean isSuite);

// Caller has no idea what true means
render(true);
```

### After - Separate Functions

```java
renderForSuite();
renderForSingleTest();

// Or with enum if need to pass around
render(RenderType.SUITE);
render(RenderType.SINGLE_TEST);
```

---

## Output Arguments

Avoid output arguments. If function must change state, have it change the state of its owning object.

### Before - Output Argument

```java
// Confusing: is buffer an input or output?
void appendFooter(StringBuilder buffer);

// Caller
StringBuilder report = new StringBuilder();
appendFooter(report);
```

### After - Method on Object

```java
// Clear: footer is appended to report
report.appendFooter();

// Or return new value
String withFooter = addFooter(report);
```

---

## Argument Lists

When multiple arguments of the same type form a logical group, use varargs:

```java
// Good - variable number of same type
String format(String pattern, Object... args);
void log(LogLevel level, String message, Object... params);

// Counts as monadic/dyadic:
public String format(String pattern, Object... args)  // dyadic
void monad(Integer... args);  // monadic
void dyad(String name, Integer... args);  // dyadic
```

---

## Method Naming with Arguments

Verb/noun pair should form a clear phrase:

```java
// Good - reads naturally
writeField(name);
assertEqualsExpectedAndActual(expected, actual);  // Too verbose, but clear

// Better with keyword form
assertExpectedEqualsActual(expected, actual);

// Best - use fluent assertions
assertThat(actual).isEqualTo(expected);
```

---

## CDI Injection Alternative

In Jakarta EE, consider injection over method parameters:

### Before - Many Service Dependencies

```java
public void processOrder(
    Order order,
    InventoryService inventory,
    PaymentService payment,
    NotificationService notification
) {
    // ...
}
```

### After - CDI Injection

```java
@ApplicationScoped
public class OrderProcessor {

    @Inject
    InventoryService inventory;

    @Inject
    PaymentService payment;

    @Inject
    NotificationService notification;

    public void processOrder(Order order) {
        // ...
    }
}
```

---

## Summary

| Problem              | Solution                         |
| -------------------- | -------------------------------- |
| > 3 arguments        | Parameter object or builder      |
| Boolean argument     | Separate methods or enum         |
| Output argument      | Return value or method on object |
| Related arguments    | Group into object                |
| Service dependencies | CDI injection                    |

---

## Related Recipes

- [Small Functions](./small-functions.md): Keep functions focused
- [Data Clumps](./data-clumps.md): Identify parameter groupings
