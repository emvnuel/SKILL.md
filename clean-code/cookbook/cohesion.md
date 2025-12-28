# Cohesion - Clean Code Cookbook

## Intent

Cohesion measures how related the responsibilities of a class are. High cohesion means the class does one thing well. Low cohesion indicates the class is doing too many unrelated things.

---

## What is Cohesion?

A class has high cohesion when:

- Each method uses most instance variables
- Methods are functionally related
- The class represents a single concept
- All methods contribute to a single purpose

---

## Measuring Cohesion

### High Cohesion (Good)

```java
// Every method uses most instance variables
public class Rectangle {
    private final double width;
    private final double height;

    public double area() {
        return width * height;  // Uses both
    }

    public double perimeter() {
        return 2 * (width + height);  // Uses both
    }

    public double diagonal() {
        return Math.sqrt(width * width + height * height);  // Uses both
    }
}
```

### Low Cohesion (Bad)

```java
// Methods use different subsets of variables
public class Utility {
    private Connection dbConnection;
    private EmailClient emailClient;
    private FileSystem fileSystem;

    public void saveToDatabase(Entity entity) {
        dbConnection.save(entity);  // Only uses dbConnection
    }

    public void sendEmail(String to, String message) {
        emailClient.send(to, message);  // Only uses emailClient
    }

    public void readFile(String path) {
        fileSystem.read(path);  // Only uses fileSystem
    }
}
```

---

## Lack of Cohesion of Methods (LCOM)

LCOM measures cohesion. Higher LCOM = lower cohesion = class should be split.

### Signs of Low Cohesion

1. Methods form clusters using different fields
2. Some methods don't use instance state at all
3. Class name is generic (Manager, Processor, Handler)
4. Class does multiple unrelated things

---

## Refactoring for Cohesion

### Before - Low Cohesion

```java
public class UserService {
    @Inject UserRepository userRepository;
    @Inject PasswordEncoder encoder;
    @Inject EmailService emailService;
    @Inject AuditLog auditLog;

    // Cluster 1: User CRUD
    public User createUser(UserRequest request) {
        User user = new User(request.getEmail());
        return userRepository.save(user);
    }

    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    // Cluster 2: Authentication
    public boolean authenticate(String email, String password) {
        User user = userRepository.findByEmail(email).orElse(null);
        return user != null && encoder.matches(password, user.getPassword());
    }

    public void resetPassword(String email) {
        User user = userRepository.findByEmail(email).orElseThrow();
        String token = generateResetToken();
        emailService.sendPasswordReset(email, token);
    }

    // Cluster 3: Audit
    public List<AuditEntry> getAuditHistory(Long userId) {
        return auditLog.getEntriesFor(userId);
    }
}
```

### After - High Cohesion Classes

```java
// User lifecycle operations
@ApplicationScoped
public class UserRepository {
    @PersistenceContext EntityManager em;

    public User save(User user) {
        em.persist(user);
        return user;
    }

    public Optional<User> findById(Long id) {
        return Optional.ofNullable(em.find(User.class, id));
    }

    public Optional<User> findByEmail(String email) {
        // ...
    }
}

// Authentication concerns
@ApplicationScoped
public class AuthenticationService {
    @Inject UserRepository userRepository;
    @Inject PasswordEncoder encoder;

    public AuthResult authenticate(String email, String password) {
        return userRepository.findByEmail(email)
            .filter(user -> encoder.matches(password, user.getPassword()))
            .map(AuthResult::success)
            .orElse(AuthResult.failure());
    }
}

// Password reset flow
@ApplicationScoped
public class PasswordResetService {
    @Inject UserRepository userRepository;
    @Inject EmailService emailService;
    @Inject TokenGenerator tokenGenerator;

    public void initiateReset(String email) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(UserNotFoundException::new);
        String token = tokenGenerator.generate();
        storeResetToken(user, token);
        emailService.sendPasswordReset(email, token);
    }
}

// Audit operations
@ApplicationScoped
public class UserAuditService {
    @Inject AuditLog auditLog;

    public List<AuditEntry> getHistory(Long userId) {
        return auditLog.getEntriesFor(userId);
    }
}
```

---

## Static Methods Indicate Low Cohesion

Static methods don't use instance state, suggesting they belong elsewhere.

### Before - Static Methods

```java
public class Order {
    private List<OrderItem> items;
    private Customer customer;

    // Instance method - uses state
    public Money getTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // Static method - doesn't use instance state
    public static boolean isValidEmail(String email) {
        return email != null && email.matches(".*@.*\\..*");
    }

    // Static method - doesn't belong here
    public static Money convertCurrency(Money amount, Currency target) {
        return CurrencyExchange.convert(amount, target);
    }
}
```

### After - Moved to Appropriate Classes

```java
public class Order {
    private List<OrderItem> items;
    private Customer customer;

    public Money getTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Email validation moved to value object or validator
public record Email(String value) {
    public Email {
        if (!isValid(value)) {
            throw new InvalidEmailException(value);
        }
    }

    public static boolean isValid(String email) {
        return email != null && email.matches(".*@.*\\..*");
    }
}

// Currency conversion in domain service
@ApplicationScoped
public class CurrencyExchangeService {
    public Money convert(Money amount, Currency target) {
        // ...
    }
}
```

---

## Cohesion in Record Classes

Java records naturally promote high cohesion:

```java
// High cohesion - all methods related to address
public record Address(
    String street,
    String city,
    String state,
    String zipCode,
    String country
) {
    public String singleLine() {
        return String.format("%s, %s, %s %s, %s",
            street, city, state, zipCode, country);
    }

    public boolean isInternational() {
        return !country.equals("US");
    }

    public String shippingLabel() {
        return String.format("%s\n%s, %s %s\n%s",
            street, city, state, zipCode, country);
    }
}
```

---

## Cohesion Checklist

| Question                                     | Low Cohesion Sign     |
| -------------------------------------------- | --------------------- |
| Do all methods use most instance variables?  | No → Split class      |
| Can you describe the class in one sentence?  | No → Split class      |
| Do methods form unrelated clusters?          | Yes → Split class     |
| Are there many static methods?               | Yes → Move elsewhere  |
| Is the class name generic (Manager, Helper)? | Yes → Find real names |

---

## Related Recipes

- [Single Responsibility](./single-responsibility.md): One reason to change
- [Small Classes](./small-classes.md): Small, focused classes
- [Feature Envy](./feature-envy.md): Move behavior to data owner
