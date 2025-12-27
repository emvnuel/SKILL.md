# Adapter Pattern - Jakarta EE Cookbook

## Intent

Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

## Jakarta EE Implementation

### 1. Object Adapter with CDI

```java
// Target interface (what our application expects)
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
    RefundResult refund(String transactionId, BigDecimal amount);
}

// Adaptee (legacy/external system with different interface)
public class LegacyPaymentSystem {
    public String processPayment(String cardNumber, String amount, String currency) { }
    public boolean reverseTransaction(String txnId, String amt) { }
}

// Adapter - bridges the gap
@ApplicationScoped
public class LegacyPaymentAdapter implements PaymentGateway {

    private final LegacyPaymentSystem legacy;

    @Inject
    public LegacyPaymentAdapter() {
        this.legacy = new LegacyPaymentSystem();
    }

    @Override
    public PaymentResult charge(PaymentRequest request) {
        String result = legacy.processPayment(
            request.getCardNumber(),
            request.getAmount().toString(),
            request.getCurrency().getCode()
        );
        return parseResult(result);
    }

    @Override
    public RefundResult refund(String transactionId, BigDecimal amount) {
        boolean success = legacy.reverseTransaction(transactionId, amount.toString());
        return new RefundResult(success);
    }

    private PaymentResult parseResult(String legacyResult) {
        // Convert legacy format to domain format
    }
}
```

### 2. Client Using the Adapter

```java
@ApplicationScoped
public class OrderService {

    @Inject
    PaymentGateway paymentGateway;  // Adapter injected

    public Order processOrder(OrderRequest request) {
        PaymentResult result = paymentGateway.charge(
            new PaymentRequest(request.getPaymentDetails())
        );

        if (result.isSuccessful()) {
            return createOrder(request, result);
        }
        throw new PaymentFailedException(result.getError());
    }
}
```

## Real-World Examples

### 3. REST Client Adapter

```java
// Target interface
public interface UserService {
    User findById(Long id);
    List<User> findByEmail(String email);
}

// External REST API has different structure
@ApplicationScoped
public class ExternalUserApiAdapter implements UserService {

    @Inject
    @RestClient
    ExternalUserApi api;  // MicroProfile REST Client

    @Override
    public User findById(Long id) {
        ExternalUserDto dto = api.getUser(id.toString());
        return mapToUser(dto);
    }

    @Override
    public List<User> findByEmail(String email) {
        ExternalSearchResponse response = api.searchUsers(
            Map.of("email", email)
        );
        return response.getData().stream()
            .map(this::mapToUser)
            .toList();
    }

    private User mapToUser(ExternalUserDto dto) {
        return User.builder()
            .id(Long.parseLong(dto.getUserId()))
            .email(dto.getEmailAddress())
            .name(dto.getFirstName() + " " + dto.getLastName())
            .build();
    }
}

// MicroProfile REST Client interface
@RegisterRestClient(configKey = "external-user-api")
@Path("/v2/users")
public interface ExternalUserApi {
    @GET
    @Path("/{userId}")
    ExternalUserDto getUser(@PathParam("userId") String userId);

    @GET
    @Path("/search")
    ExternalSearchResponse searchUsers(@QueryParam("filters") Map<String, String> filters);
}
```

### 4. Logger Adapter

```java
// Target interface
public interface AppLogger {
    void info(String message, Object... args);
    void error(String message, Throwable t);
    void audit(String action, String userId, Map<String, Object> details);
}

// Adapter for SLF4J + custom audit system
@ApplicationScoped
public class Slf4jLoggerAdapter implements AppLogger {

    private static final Logger log = LoggerFactory.getLogger("app");

    @Inject
    AuditService auditService;

    @Override
    public void info(String message, Object... args) {
        log.info(message, args);
    }

    @Override
    public void error(String message, Throwable t) {
        log.error(message, t);
    }

    @Override
    public void audit(String action, String userId, Map<String, Object> details) {
        log.info("AUDIT: {} by {} - {}", action, userId, details);
        auditService.record(new AuditEntry(action, userId, details));
    }
}
```

## When to Use

✅ **Use Adapter when:**

- You need to use an existing class with an incompatible interface
- You want to create a reusable class that cooperates with unrelated classes
- You need to integrate with third-party/legacy systems

❌ **Avoid Adapter when:**

- Interfaces are similar enough that direct use is clearer
- You can modify the source class to implement the target interface
