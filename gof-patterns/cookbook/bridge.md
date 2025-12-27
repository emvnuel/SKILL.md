# Bridge Pattern - Jakarta EE Cookbook

## Intent

Decouple an abstraction from its implementation so that the two can vary independently.

## Jakarta EE Implementation

### 1. Abstraction and Implementor

```java
// Implementor interface
public interface MessageSender {
    void send(String message, String recipient);
}

// Abstraction
public abstract class Notification {
    protected final MessageSender sender;

    protected Notification(MessageSender sender) {
        this.sender = sender;
    }

    public abstract void notify(String message);
}
```

### 2. Concrete Implementors

```java
@ApplicationScoped
@Named("email")
public class EmailSender implements MessageSender {

    @Inject
    EmailService emailService;

    @Override
    public void send(String message, String recipient) {
        emailService.sendEmail(recipient, "Notification", message);
    }
}

@ApplicationScoped
@Named("sms")
public class SmsSender implements MessageSender {

    @Inject
    SmsGateway smsGateway;

    @Override
    public void send(String message, String recipient) {
        smsGateway.sendSms(recipient, message);
    }
}

@ApplicationScoped
@Named("push")
public class PushSender implements MessageSender {

    @Inject
    PushNotificationService pushService;

    @Override
    public void send(String message, String recipient) {
        pushService.sendPush(recipient, message);
    }
}
```

### 3. Refined Abstractions

```java
public class UrgentNotification extends Notification {

    private final String recipient;

    public UrgentNotification(MessageSender sender, String recipient) {
        super(sender);
        this.recipient = recipient;
    }

    @Override
    public void notify(String message) {
        sender.send("🚨 URGENT: " + message, recipient);
    }
}

public class RegularNotification extends Notification {

    private final String recipient;

    public RegularNotification(MessageSender sender, String recipient) {
        super(sender);
        this.recipient = recipient;
    }

    @Override
    public void notify(String message) {
        sender.send(message, recipient);
    }
}

public class ScheduledNotification extends Notification {

    private final String recipient;
    private final Instant scheduledTime;

    public ScheduledNotification(MessageSender sender, String recipient, Instant time) {
        super(sender);
        this.recipient = recipient;
        this.scheduledTime = time;
    }

    @Override
    public void notify(String message) {
        // Schedule for later
        sender.send("[Scheduled: " + scheduledTime + "] " + message, recipient);
    }
}
```

### 4. Factory for Bridge

```java
@ApplicationScoped
public class NotificationFactory {

    @Inject
    @Any
    Instance<MessageSender> senders;

    public Notification createUrgent(String channel, String recipient) {
        MessageSender sender = senders.select(new NamedLiteral(channel)).get();
        return new UrgentNotification(sender, recipient);
    }

    public Notification createRegular(String channel, String recipient) {
        MessageSender sender = senders.select(new NamedLiteral(channel)).get();
        return new RegularNotification(sender, recipient);
    }
}
```

### 5. Usage

```java
@ApplicationScoped
public class AlertService {

    @Inject
    NotificationFactory factory;

    public void sendAlert(User user, String message, AlertLevel level) {
        Notification notification = switch (level) {
            case CRITICAL -> factory.createUrgent(user.getPreferredChannel(), user.getContact());
            case NORMAL -> factory.createRegular(user.getPreferredChannel(), user.getContact());
        };
        notification.notify(message);
    }
}
```

## Real-World Example: Multi-Database Repository

```java
// Implementor
public interface DatabaseDriver {
    void connect(String connectionString);
    List<Map<String, Object>> query(String sql);
    int execute(String sql);
}

// Abstraction
public abstract class Repository<T> {
    protected final DatabaseDriver driver;

    protected Repository(DatabaseDriver driver) {
        this.driver = driver;
    }

    public abstract T findById(Long id);
    public abstract List<T> findAll();
    public abstract void save(T entity);
}

// Concrete implementors for different databases
@ApplicationScoped
@Named("postgresql")
public class PostgreSQLDriver implements DatabaseDriver { }

@ApplicationScoped
@Named("mysql")
public class MySQLDriver implements DatabaseDriver { }

// Refined abstraction
public class UserRepository extends Repository<User> {

    public UserRepository(DatabaseDriver driver) {
        super(driver);
    }

    @Override
    public User findById(Long id) {
        List<Map<String, Object>> results = driver.query(
            "SELECT * FROM users WHERE id = " + id
        );
        return mapToUser(results.get(0));
    }
}
```

## When to Use

✅ **Use Bridge when:**

- Abstraction and implementation should vary independently
- You want to avoid permanent binding between abstraction and implementation
- Changes in implementation shouldn't affect clients

❌ **Avoid Bridge when:**

- You have only one implementation
- The abstraction-implementation relationship is straightforward
