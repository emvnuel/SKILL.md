# Small Classes - Clean Code Cookbook

## Intent

Classes should be small. We measure class size by counting responsibilities, not lines of code. A class should have one responsibility and a small number of instance variables.

---

## How Small is Small?

Rules of thumb:

| Metric             | Guideline |
| ------------------ | --------- |
| Lines of code      | < 200     |
| Methods            | < 10      |
| Instance variables | < 5-7     |
| Responsibilities   | Exactly 1 |

---

## The Single Responsibility Test

Can you describe the class in about 25 words without using:

- "and"
- "or"
- "but"
- "if"

### Fails the Test

> SuperDashboard handles version management **and** display state **and** focus traversal **and** component management.

### Passes the Test

> Version tracks the major, minor, and build versions of a software component.

---

## Refactoring Large Classes

### Before - Large Class

```java
public class SuperDashboard extends JFrame implements MetaDataUser {
    private Component lastFocusedComponent;
    private String currentVersion;
    private String buildNumber;
    private String versionDate;
    private int majorVersion;
    private int minorVersion;
    private int buildVersion;
    private List<Component> focusList = new ArrayList<>();
    private Map<String, Widget> widgets = new HashMap<>();

    // Focus management methods
    public Component getLastFocusedComponent() { /*...*/ }
    public void setLastFocus(Component focus) { /*...*/ }
    public int getFocusableCount() { /*...*/ }
    public void setFocusList(List<Component> list) { /*...*/ }

    // Version management methods
    public String getMajorVersion() { /*...*/ }
    public String getMinorVersion() { /*...*/ }
    public String getBuildVersion() { /*...*/ }
    public String getVersion() { /*...*/ }
    public String getBuildDate() { /*...*/ }

    // Widget management methods
    public Widget getWidget(String name) { /*...*/ }
    public void addWidget(String name, Widget widget) { /*...*/ }
    public void removeWidget(String name) { /*...*/ }

    // Display methods
    public void refreshDisplay() { /*...*/ }
    public void updateLayout() { /*...*/ }
}
```

### After - Small, Focused Classes

```java
// Focus tracking only
public class FocusTracker {
    private Component lastFocusedComponent;
    private List<Component> focusableComponents = new ArrayList<>();

    public void setLastFocus(Component component) {
        this.lastFocusedComponent = component;
    }

    public Component getLastFocusedComponent() {
        return lastFocusedComponent;
    }

    public int getFocusableCount() {
        return focusableComponents.size();
    }

    public void setFocusableComponents(List<Component> components) {
        this.focusableComponents = new ArrayList<>(components);
    }
}

// Version information only
public record Version(int major, int minor, int build, LocalDate releaseDate) {

    public String formatted() {
        return String.format("%d.%d.%d", major, minor, build);
    }

    public boolean isNewerThan(Version other) {
        if (major != other.major) return major > other.major;
        if (minor != other.minor) return minor > other.minor;
        return build > other.build;
    }
}

// Widget registry only
public class WidgetRegistry {
    private final Map<String, Widget> widgets = new HashMap<>();

    public void register(String name, Widget widget) {
        widgets.put(name, widget);
    }

    public Optional<Widget> get(String name) {
        return Optional.ofNullable(widgets.get(name));
    }

    public void remove(String name) {
        widgets.remove(name);
    }
}

// Dashboard composes the small classes
public class Dashboard extends JFrame {
    private final FocusTracker focusTracker = new FocusTracker();
    private final WidgetRegistry widgets = new WidgetRegistry();
    private final Version version;

    public Dashboard(Version version) {
        this.version = version;
    }

    public void refreshDisplay() {
        // Display logic only
    }
}
```

---

## Organizing for Change

### Before - Class Changes for Multiple Reasons

```java
public class Sql {
    public Sql(String table, Column[] columns) { }

    public String create() { /*...*/ }
    public String insert(Object[] fields) { /*...*/ }
    public String selectAll() { /*...*/ }
    public String select(Column column, String pattern) { /*...*/ }
    public String select(Criteria criteria) { /*...*/ }
    public String findByKey(String key) { /*...*/ }
    public String preparedInsert() { /*...*/ }
    // Adding UPDATE requires modifying this class
    // Adding new SELECT variation requires modifying this class
}
```

### After - Open for Extension, Closed for Modification

```java
public abstract class Sql {
    protected final String table;
    protected final Column[] columns;

    protected Sql(String table, Column[] columns) {
        this.table = table;
        this.columns = columns;
    }

    public abstract String generate();
}

public class CreateSql extends Sql {
    public CreateSql(String table, Column[] columns) {
        super(table, columns);
    }

    @Override
    public String generate() {
        return "CREATE TABLE " + table + " (...)";
    }
}

public class InsertSql extends Sql {
    private final Object[] fields;

    public InsertSql(String table, Column[] columns, Object[] fields) {
        super(table, columns);
        this.fields = fields;
    }

    @Override
    public String generate() {
        return "INSERT INTO " + table + " VALUES (...)";
    }
}

public class SelectSql extends Sql {
    // Different select variations as subclasses
}

// Adding UPDATE: just add new class, no modification needed
public class UpdateSql extends Sql {
    private final Object[] values;
    private final Criteria criteria;

    public UpdateSql(String table, Column[] columns,
                     Object[] values, Criteria criteria) {
        super(table, columns);
        this.values = values;
        this.criteria = criteria;
    }

    @Override
    public String generate() {
        return "UPDATE " + table + " SET ...";
    }
}
```

---

## Detecting Large Classes

### Warning Signs

```java
// Too many imports
import java.sql.*;
import javax.mail.*;
import java.io.*;
import com.company.payment.*;
import com.company.shipping.*;

// Too many fields
public class OrderProcessor {
    private OrderRepository orderRepo;
    private CustomerRepository customerRepo;
    private PaymentGateway paymentGateway;
    private ShippingService shippingService;
    private EmailService emailService;
    private AuditLog auditLog;
    private InventoryService inventory;
    private TaxCalculator taxCalculator;
    // 8+ dependencies is a smell
}
```

### Split by Domain

```java
// Payment concerns
@ApplicationScoped
public class PaymentProcessor {
    @Inject PaymentGateway gateway;
    @Inject TaxCalculator taxCalculator;

    public PaymentResult processPayment(Order order) {
        Money total = taxCalculator.calculateWithTax(order);
        return gateway.charge(order.getPaymentMethod(), total);
    }
}

// Fulfillment concerns
@ApplicationScoped
public class OrderFulfillment {
    @Inject ShippingService shipping;
    @Inject InventoryService inventory;

    public FulfillmentResult fulfill(Order order) {
        inventory.reserve(order.getItems());
        return shipping.scheduleDelivery(order);
    }
}

// Notification concerns
@ApplicationScoped
public class OrderNotification {
    @Inject EmailService emailService;

    public void notifyCustomer(Order order) {
        emailService.sendConfirmation(order);
    }
}

// Orchestrator keeps minimal dependencies
@ApplicationScoped
public class OrderService {
    @Inject PaymentProcessor payment;
    @Inject OrderFulfillment fulfillment;
    @Inject OrderNotification notification;
    @Inject OrderRepository repository;

    @Transactional
    public Order placeOrder(Order order) {
        payment.processPayment(order);
        fulfillment.fulfill(order);
        Order saved = repository.save(order);
        notification.notifyCustomer(saved);
        return saved;
    }
}
```

---

## Class Size Checklist

| Check                                   | Action if Failed             |
| --------------------------------------- | ---------------------------- |
| Can describe in 25 words without "and"? | Split responsibilities       |
| < 10 public methods?                    | Extract cohesive groups      |
| < 7 instance variables?                 | Extract value objects        |
| < 200 lines?                            | Look for hidden classes      |
| All methods use most fields?            | Low cohesion - split         |
| Name describes single concept?          | Find hidden responsibilities |

---

## Related Recipes

- [Single Responsibility](./single-responsibility.md): One reason to change
- [Cohesion](./cohesion.md): Related methods together
- [Divergent Change](./divergent-change.md): Class changes for different reasons
