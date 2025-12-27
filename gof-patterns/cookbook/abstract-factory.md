# Abstract Factory Pattern - Jakarta EE Cookbook

## Intent

Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

## Jakarta EE Implementation

### 1. Define the Product Interfaces

```java
// Product family interfaces
public interface Button {
    void render();
    String getType();
}

public interface TextField {
    void render();
    String getValue();
    void setValue(String value);
}

public interface CheckBox {
    void render();
    boolean isChecked();
}
```

### 2. Define the Abstract Factory Interface

```java
public interface UIComponentFactory {
    Button createButton(String label);
    TextField createTextField(String placeholder);
    CheckBox createCheckBox(String label, boolean checked);
}
```

### 3. Create Concrete Product Families

```java
// Dark Theme Products
public class DarkButton implements Button {
    private final String label;

    public DarkButton(String label) {
        this.label = label;
    }

    @Override
    public void render() {
        // Render dark themed button
    }

    @Override
    public String getType() {
        return "dark";
    }
}

public class DarkTextField implements TextField {
    private String value;
    private final String placeholder;

    public DarkTextField(String placeholder) {
        this.placeholder = placeholder;
    }

    @Override
    public void render() {
        // Render dark themed text field
    }

    @Override
    public String getValue() { return value; }

    @Override
    public void setValue(String value) { this.value = value; }
}

// Light Theme Products
public class LightButton implements Button {
    private final String label;

    public LightButton(String label) {
        this.label = label;
    }

    @Override
    public void render() {
        // Render light themed button
    }

    @Override
    public String getType() {
        return "light";
    }
}

public class LightTextField implements TextField {
    // Similar implementation
}
```

### 4. Create Concrete Factories with CDI Qualifiers

```java
// Custom qualifier for theme selection
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface Theme {
    String value();
}

// Dark theme factory
@ApplicationScoped
@Theme("dark")
public class DarkUIFactory implements UIComponentFactory {

    @Override
    public Button createButton(String label) {
        return new DarkButton(label);
    }

    @Override
    public TextField createTextField(String placeholder) {
        return new DarkTextField(placeholder);
    }

    @Override
    public CheckBox createCheckBox(String label, boolean checked) {
        return new DarkCheckBox(label, checked);
    }
}

// Light theme factory
@ApplicationScoped
@Theme("light")
public class LightUIFactory implements UIComponentFactory {

    @Override
    public Button createButton(String label) {
        return new LightButton(label);
    }

    @Override
    public TextField createTextField(String placeholder) {
        return new LightTextField(placeholder);
    }

    @Override
    public CheckBox createCheckBox(String label, boolean checked) {
        return new LightCheckBox(label, checked);
    }
}
```

### 5. Factory Provider for Dynamic Selection

```java
@ApplicationScoped
public class UIFactoryProvider {

    @Inject
    @Any
    Instance<UIComponentFactory> factories;

    @Inject
    @ConfigProperty(name = "ui.theme", defaultValue = "light")
    String defaultTheme;

    public UIComponentFactory getFactory() {
        return getFactory(defaultTheme);
    }

    public UIComponentFactory getFactory(String theme) {
        return factories.select(new ThemeLiteral(theme)).get();
    }
}

// Qualifier literal for programmatic selection
public class ThemeLiteral extends AnnotationLiteral<Theme> implements Theme {
    private final String value;

    public ThemeLiteral(String value) {
        this.value = value;
    }

    @Override
    public String value() {
        return value;
    }
}
```

## Real-World Example: Multi-Database Support

### 6. Database Connection Factory

```java
// Product interfaces
public interface ConnectionPool {
    Connection getConnection() throws SQLException;
    void release(Connection connection);
}

public interface QueryBuilder {
    String buildSelect(String table, List<String> columns, String where);
    String buildInsert(String table, Map<String, Object> values);
}

public interface DatabaseMigrator {
    void migrate(String scriptsPath);
}

// Abstract Factory
public interface DatabaseFactory {
    ConnectionPool createConnectionPool(DatabaseConfig config);
    QueryBuilder createQueryBuilder();
    DatabaseMigrator createMigrator();
}
```

### 7. PostgreSQL Implementation

```java
@ApplicationScoped
@Named("postgresql")
public class PostgreSQLFactory implements DatabaseFactory {

    @Override
    public ConnectionPool createConnectionPool(DatabaseConfig config) {
        return new PostgreSQLConnectionPool(config);
    }

    @Override
    public QueryBuilder createQueryBuilder() {
        return new PostgreSQLQueryBuilder();
    }

    @Override
    public DatabaseMigrator createMigrator() {
        return new FlywayMigrator("postgresql");
    }
}
```

### 8. MySQL Implementation

```java
@ApplicationScoped
@Named("mysql")
public class MySQLFactory implements DatabaseFactory {

    @Override
    public ConnectionPool createConnectionPool(DatabaseConfig config) {
        return new MySQLConnectionPool(config);
    }

    @Override
    public QueryBuilder createQueryBuilder() {
        return new MySQLQueryBuilder();
    }

    @Override
    public DatabaseMigrator createMigrator() {
        return new FlywayMigrator("mysql");
    }
}
```

### 9. Using the Database Factory

```java
@ApplicationScoped
public class DatabaseService {

    @Inject
    @ConfigProperty(name = "database.type", defaultValue = "postgresql")
    String databaseType;

    @Inject
    @Any
    Instance<DatabaseFactory> factories;

    private DatabaseFactory factory;
    private ConnectionPool connectionPool;

    @PostConstruct
    void init() {
        factory = factories.select(new NamedLiteral(databaseType)).get();
        connectionPool = factory.createConnectionPool(loadConfig());
    }

    public Connection getConnection() throws SQLException {
        return connectionPool.getConnection();
    }

    public QueryBuilder getQueryBuilder() {
        return factory.createQueryBuilder();
    }
}
```

## MicroProfile Integration

### 10. Config-Driven Factory Selection

```properties
# application.properties
cloud.provider=aws
# or
cloud.provider=azure
# or
cloud.provider=gcp
```

```java
// Cloud provider factory
public interface CloudFactory {
    StorageService createStorageService();
    ComputeService createComputeService();
    MessagingService createMessagingService();
}

@ApplicationScoped
@Named("aws")
public class AWSFactory implements CloudFactory {

    @Inject
    @ConfigProperty(name = "aws.region")
    String region;

    @Override
    public StorageService createStorageService() {
        return new S3StorageService(region);
    }

    @Override
    public ComputeService createComputeService() {
        return new EC2ComputeService(region);
    }

    @Override
    public MessagingService createMessagingService() {
        return new SQSMessagingService(region);
    }
}

@ApplicationScoped
@Named("azure")
public class AzureFactory implements CloudFactory {
    // Azure implementations
}

@ApplicationScoped
public class CloudServices {

    @Inject
    @ConfigProperty(name = "cloud.provider")
    String provider;

    @Inject
    @Any
    Instance<CloudFactory> factories;

    @Produces
    @ApplicationScoped
    public StorageService storageService() {
        return factories.select(new NamedLiteral(provider)).get().createStorageService();
    }

    @Produces
    @ApplicationScoped
    public ComputeService computeService() {
        return factories.select(new NamedLiteral(provider)).get().createComputeService();
    }
}
```

## Testing

### 11. Testing with Alternative Implementations

```java
@QuarkusTest
@TestProfile(TestDatabaseProfile.class)
class DatabaseServiceTest {

    @Inject
    DatabaseService databaseService;

    @Test
    void shouldUseTestDatabase() {
        QueryBuilder builder = databaseService.getQueryBuilder();
        assertThat(builder).isInstanceOf(H2QueryBuilder.class);
    }
}

// Test profile that overrides the database type
public class TestDatabaseProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of("database.type", "h2");
    }
}
```

## When to Use

✅ **Use Abstract Factory when:**

- You need to create families of related objects
- Products from one family should be used together
- You want to provide a library of products without exposing implementations
- The system needs to support multiple product families

❌ **Avoid Abstract Factory when:**

- You have only one type of product
- Product families are unlikely to change
- The added abstraction doesn't provide value

## Related Patterns

- **Factory Method**: Often used to implement Abstract Factory methods
- **Builder**: Can combine with Abstract Factory for complex object creation
- **Prototype**: Alternative when configuring factory is simpler than subclassing
