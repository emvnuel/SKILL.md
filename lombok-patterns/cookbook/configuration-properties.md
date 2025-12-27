# Configuration Properties with Lombok

## Intent

Use Lombok with Spring Boot `@ConfigurationProperties` for type-safe configuration binding.

## Basic Pattern

```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Getter
@Setter
public class AppProperties {
    private String name;
    private String version;
    private boolean debugEnabled;
    private Duration timeout;
}
```

```yaml
# application.yml
app:
  name: My Application
  version: 1.0.0
  debug-enabled: true
  timeout: 30s
```

## Immutable Configuration (Recommended)

```java
@ConfigurationProperties(prefix = "mail")
@RequiredArgsConstructor
@Getter
public class MailProperties {
    private final String host;
    private final int port;
    private final String username;
    private final String password;
    private final boolean enabled;
}
```

Enable in main class:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## Nested Properties

```java
@ConfigurationProperties(prefix = "app")
@Getter
@Setter
public class AppProperties {
    private String name;
    private Server server = new Server();
    private Database database = new Database();

    @Getter
    @Setter
    public static class Server {
        private String host = "localhost";
        private int port = 8080;
    }

    @Getter
    @Setter
    public static class Database {
        private String url;
        private String username;
        private String password;
        private int poolSize = 10;
    }
}
```

```yaml
app:
  name: MyApp
  server:
    host: api.example.com
    port: 443
  database:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: secret
    pool-size: 20
```

## With Validation

```java
@ConfigurationProperties(prefix = "app.security")
@Validated
@Getter
@Setter
public class SecurityProperties {
    @NotBlank
    private String jwtSecret;

    @Min(60)
    @Max(86400)
    private int tokenExpirationSeconds = 3600;

    @NotEmpty
    private List<String> allowedOrigins;
}
```

## Using in Services

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EmailService {
    private final MailProperties mailProperties;

    public void sendEmail(String to, String subject, String body) {
        if (!mailProperties.isEnabled()) {
            log.info("Email disabled, skipping: {}", subject);
            return;
        }

        // Use mailProperties.getHost(), getPort(), etc.
    }
}
```

## Common Patterns

### List and Map Configuration

```java
@ConfigurationProperties(prefix = "features")
@Getter
@Setter
public class FeatureProperties {
    private List<String> enabledModules = new ArrayList<>();
    private Map<String, String> settings = new HashMap<>();
}
```

```yaml
features:
  enabled-modules:
    - auth
    - payments
    - notifications
  settings:
    max-retries: "3"
    timeout: "5000"
```

### Duration and DataSize

```java
@ConfigurationProperties(prefix = "cache")
@Getter
@Setter
public class CacheProperties {
    private Duration ttl = Duration.ofMinutes(30);
    private DataSize maxSize = DataSize.ofMegabytes(100);
}
```

## Best Practices

| Do ✅                                                | Don't ❌                     |
| ---------------------------------------------------- | ---------------------------- |
| Use immutable config with `@RequiredArgsConstructor` | Mutable config in production |
| Add `@Validated` for validation                      | Trust unvalidated input      |
| Provide sensible defaults                            | Require all properties       |
| Use type-safe Duration, DataSize                     | String parsing               |

## Related Annotations

- **@RequiredArgsConstructor**: Immutable configuration
- **@Getter/@Setter**: Mutable configuration
