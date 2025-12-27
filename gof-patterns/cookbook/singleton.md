# Singleton Pattern - Jakarta EE Cookbook

## Intent

Ensure a class has only one instance and provide a global point of access to it.

## Jakarta EE Implementation

In Jakarta EE with CDI, the Singleton pattern is implemented through **scoped beans**.

### 1. CDI ApplicationScoped (Recommended)

```java
@ApplicationScoped
public class ConfigurationManager {

    @Inject
    @ConfigProperty(name = "app.name")
    String appName;

    private final Map<String, String> cache = new ConcurrentHashMap<>();

    @PostConstruct
    void init() {
        loadDefaults();
    }

    public String getProperty(String key) {
        return cache.get(key);
    }
}

// Usage - inject anywhere
@ApplicationScoped
public class MyService {
    @Inject
    ConfigurationManager config;  // Same instance everywhere
}
```

### 2. Anti-Pattern: Manual Singleton

```java
// BAD: Manual singleton - don't do this in Jakarta EE
public class ConfigurationManager {
    private static volatile ConfigurationManager instance;

    public static ConfigurationManager getInstance() {
        if (instance == null) {
            synchronized (ConfigurationManager.class) {
                if (instance == null) {
                    instance = new ConfigurationManager();
                }
            }
        }
        return instance;
    }
}
// Problems: No CDI injection, hard to test, lifecycle not managed
```

### 3. Eager vs Lazy Initialization

```java
// LAZY (default)
@ApplicationScoped
public class LazyService {
    @PostConstruct
    void init() { /* Called when first injected */ }
}

// EAGER - initialized at startup
@ApplicationScoped
@Startup
public class EagerService {
    @PostConstruct
    void init() { /* Called during startup */ }
}
```

## Real-World Examples

### 4. Connection Pool Singleton

```java
@ApplicationScoped
public class DatabaseConnectionPool {

    @Inject
    @ConfigProperty(name = "db.pool.size", defaultValue = "10")
    int poolSize;

    private HikariDataSource dataSource;

    @PostConstruct
    void init() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(poolSize);
        dataSource = new HikariDataSource(config);
    }

    @PreDestroy
    void cleanup() {
        if (dataSource != null) dataSource.close();
    }

    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

### 5. Producer Methods for Third-Party Singletons

```java
@ApplicationScoped
public class ThirdPartyProducers {

    @Produces
    @ApplicationScoped
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .setSerializationInclusion(JsonInclude.Include.NON_NULL);
    }

    @Produces
    @ApplicationScoped
    public S3Client s3Client(@ConfigProperty(name = "aws.region") String region) {
        return S3Client.builder().region(Region.of(region)).build();
    }

    public void disposeS3Client(@Disposes S3Client client) {
        client.close();
    }
}
```

## When to Use

✅ **Use CDI Singleton when:** Shared resources, connection pools, caches, global config

❌ **Avoid manual Singleton when:** Using CDI - use @ApplicationScoped instead

## Singleton vs Other Scopes

| Scope                | Instances       | Use Case            |
| -------------------- | --------------- | ------------------- |
| `@ApplicationScoped` | 1 per app       | Shared resources    |
| `@RequestScoped`     | 1 per request   | Request data        |
| `@SessionScoped`     | 1 per session   | User session        |
| `@Dependent`         | 1 per injection | Stateless utilities |
