# Builder Pattern - Jakarta EE Cookbook

## Intent

Separate the construction of a complex object from its representation so that the same construction process can create different representations.

## Jakarta EE Implementation

### 1. Basic Builder Pattern

```java
public class EmailMessage {
    private final String to;
    private final String from;
    private final String subject;
    private final String body;
    private final List<String> cc;
    private final List<String> bcc;
    private final Map<String, String> headers;
    private final List<Attachment> attachments;
    private final Priority priority;
    private final boolean requestReadReceipt;

    private EmailMessage(Builder builder) {
        this.to = Objects.requireNonNull(builder.to, "to is required");
        this.from = Objects.requireNonNull(builder.from, "from is required");
        this.subject = builder.subject;
        this.body = builder.body;
        this.cc = List.copyOf(builder.cc);
        this.bcc = List.copyOf(builder.bcc);
        this.headers = Map.copyOf(builder.headers);
        this.attachments = List.copyOf(builder.attachments);
        this.priority = builder.priority;
        this.requestReadReceipt = builder.requestReadReceipt;
    }

    public static Builder builder() {
        return new Builder();
    }

    // Getters only - immutable object
    public String getTo() { return to; }
    public String getFrom() { return from; }
    public String getSubject() { return subject; }
    public String getBody() { return body; }
    public List<String> getCc() { return cc; }
    public Priority getPriority() { return priority; }
    // ... other getters

    public static class Builder {
        private String to;
        private String from;
        private String subject = "";
        private String body = "";
        private List<String> cc = new ArrayList<>();
        private List<String> bcc = new ArrayList<>();
        private Map<String, String> headers = new HashMap<>();
        private List<Attachment> attachments = new ArrayList<>();
        private Priority priority = Priority.NORMAL;
        private boolean requestReadReceipt = false;

        private Builder() {}

        public Builder to(String to) {
            this.to = to;
            return this;
        }

        public Builder from(String from) {
            this.from = from;
            return this;
        }

        public Builder subject(String subject) {
            this.subject = subject;
            return this;
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder addCc(String cc) {
            this.cc.add(cc);
            return this;
        }

        public Builder addCc(List<String> cc) {
            this.cc.addAll(cc);
            return this;
        }

        public Builder addBcc(String bcc) {
            this.bcc.add(bcc);
            return this;
        }

        public Builder addHeader(String name, String value) {
            this.headers.put(name, value);
            return this;
        }

        public Builder attach(Attachment attachment) {
            this.attachments.add(attachment);
            return this;
        }

        public Builder priority(Priority priority) {
            this.priority = priority;
            return this;
        }

        public Builder requestReadReceipt() {
            this.requestReadReceipt = true;
            return this;
        }

        public EmailMessage build() {
            return new EmailMessage(this);
        }
    }
}
```

### 2. Usage Example

```java
@ApplicationScoped
public class NotificationService {

    @Inject
    EmailSender emailSender;

    public void sendWelcomeEmail(User user) {
        EmailMessage email = EmailMessage.builder()
            .to(user.getEmail())
            .from("welcome@example.com")
            .subject("Welcome to Our Platform!")
            .body(generateWelcomeBody(user))
            .priority(Priority.HIGH)
            .addHeader("X-Campaign", "welcome-series")
            .build();

        emailSender.send(email);
    }

    public void sendOrderConfirmation(Order order) {
        EmailMessage email = EmailMessage.builder()
            .to(order.getCustomerEmail())
            .from("orders@example.com")
            .subject("Order Confirmation #" + order.getId())
            .body(generateOrderBody(order))
            .attach(generateInvoicePdf(order))
            .addCc(order.getAccountManagerEmail())
            .requestReadReceipt()
            .build();

        emailSender.send(email);
    }
}
```

## Real-World Example: JAX-RS Request Builder

### 3. HTTP Request Builder with CDI Integration

```java
public class HttpRequest {
    private final String method;
    private final URI uri;
    private final Map<String, String> headers;
    private final Map<String, String> queryParams;
    private final Object body;
    private final Duration timeout;
    private final int retries;

    private HttpRequest(Builder builder) {
        this.method = builder.method;
        this.uri = buildUri(builder);
        this.headers = Map.copyOf(builder.headers);
        this.queryParams = Map.copyOf(builder.queryParams);
        this.body = builder.body;
        this.timeout = builder.timeout;
        this.retries = builder.retries;
    }

    public static Builder get(String url) {
        return new Builder("GET", url);
    }

    public static Builder post(String url) {
        return new Builder("POST", url);
    }

    public static Builder put(String url) {
        return new Builder("PUT", url);
    }

    public static Builder delete(String url) {
        return new Builder("DELETE", url);
    }

    // Getters

    public static class Builder {
        private final String method;
        private final String baseUrl;
        private Map<String, String> headers = new HashMap<>();
        private Map<String, String> queryParams = new HashMap<>();
        private Object body;
        private Duration timeout = Duration.ofSeconds(30);
        private int retries = 0;

        private Builder(String method, String baseUrl) {
            this.method = method;
            this.baseUrl = baseUrl;
        }

        public Builder header(String name, String value) {
            this.headers.put(name, value);
            return this;
        }

        public Builder bearerAuth(String token) {
            return header("Authorization", "Bearer " + token);
        }

        public Builder basicAuth(String username, String password) {
            String credentials = Base64.getEncoder()
                .encodeToString((username + ":" + password).getBytes());
            return header("Authorization", "Basic " + credentials);
        }

        public Builder contentType(String contentType) {
            return header("Content-Type", contentType);
        }

        public Builder accept(String accept) {
            return header("Accept", accept);
        }

        public Builder queryParam(String name, String value) {
            this.queryParams.put(name, value);
            return this;
        }

        public Builder queryParams(Map<String, String> params) {
            this.queryParams.putAll(params);
            return this;
        }

        public Builder body(Object body) {
            this.body = body;
            return this;
        }

        public Builder timeout(Duration timeout) {
            this.timeout = timeout;
            return this;
        }

        public Builder retries(int retries) {
            this.retries = retries;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }
}
```

### 4. HTTP Client Using the Builder

```java
@ApplicationScoped
public class ExternalApiClient {

    @Inject
    @ConfigProperty(name = "external.api.base-url")
    String baseUrl;

    @Inject
    @ConfigProperty(name = "external.api.token")
    String apiToken;

    @Inject
    HttpExecutor executor;

    public User getUser(Long userId) {
        HttpRequest request = HttpRequest.get(baseUrl + "/users/" + userId)
            .bearerAuth(apiToken)
            .accept("application/json")
            .timeout(Duration.ofSeconds(10))
            .retries(3)
            .build();

        return executor.execute(request, User.class);
    }

    public List<User> searchUsers(String query, int page, int size) {
        HttpRequest request = HttpRequest.get(baseUrl + "/users")
            .bearerAuth(apiToken)
            .accept("application/json")
            .queryParam("q", query)
            .queryParam("page", String.valueOf(page))
            .queryParam("size", String.valueOf(size))
            .build();

        return executor.executeList(request, User.class);
    }

    public User createUser(CreateUserRequest payload) {
        HttpRequest request = HttpRequest.post(baseUrl + "/users")
            .bearerAuth(apiToken)
            .contentType("application/json")
            .accept("application/json")
            .body(payload)
            .timeout(Duration.ofSeconds(30))
            .build();

        return executor.execute(request, User.class);
    }
}
```

## Builder with Validation

### 5. Builder with Jakarta Validation

```java
public class Order {
    @NotNull
    private final String customerId;

    @NotEmpty
    private final List<OrderItem> items;

    @NotNull
    private final ShippingAddress shippingAddress;

    @Positive
    private final BigDecimal total;

    private Order(Builder builder) {
        this.customerId = builder.customerId;
        this.items = List.copyOf(builder.items);
        this.shippingAddress = builder.shippingAddress;
        this.total = calculateTotal(builder.items);
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String customerId;
        private List<OrderItem> items = new ArrayList<>();
        private ShippingAddress shippingAddress;

        public Builder customerId(String customerId) {
            this.customerId = customerId;
            return this;
        }

        public Builder addItem(OrderItem item) {
            this.items.add(item);
            return this;
        }

        public Builder addItem(String productId, int quantity, BigDecimal price) {
            return addItem(new OrderItem(productId, quantity, price));
        }

        public Builder shippingAddress(ShippingAddress address) {
            this.shippingAddress = address;
            return this;
        }

        public Order build() {
            validate();
            return new Order(this);
        }

        private void validate() {
            List<String> errors = new ArrayList<>();

            if (customerId == null || customerId.isBlank()) {
                errors.add("customerId is required");
            }
            if (items.isEmpty()) {
                errors.add("at least one item is required");
            }
            if (shippingAddress == null) {
                errors.add("shippingAddress is required");
            }

            if (!errors.isEmpty()) {
                throw new IllegalStateException(
                    "Order validation failed: " + String.join(", ", errors)
                );
            }
        }
    }
}
```

## Step Builder Pattern

### 6. Enforcing Build Order with Step Interfaces

```java
// Step interfaces enforce the build order
public interface RequireCustomer {
    RequireItems customer(String customerId);
}

public interface RequireItems {
    RequireShipping addItem(OrderItem item);
}

public interface RequireShipping {
    RequireShipping addItem(OrderItem item);  // Can add more items
    FinalBuild shippingAddress(ShippingAddress address);
}

public interface FinalBuild {
    FinalBuild priority(Priority priority);  // Optional
    FinalBuild note(String note);            // Optional
    Order build();
}

public class OrderBuilder implements RequireCustomer, RequireItems, RequireShipping, FinalBuild {

    private String customerId;
    private List<OrderItem> items = new ArrayList<>();
    private ShippingAddress shippingAddress;
    private Priority priority = Priority.NORMAL;
    private String note;

    private OrderBuilder() {}

    public static RequireCustomer newOrder() {
        return new OrderBuilder();
    }

    @Override
    public RequireItems customer(String customerId) {
        this.customerId = customerId;
        return this;
    }

    @Override
    public RequireShipping addItem(OrderItem item) {
        this.items.add(item);
        return this;
    }

    @Override
    public FinalBuild shippingAddress(ShippingAddress address) {
        this.shippingAddress = address;
        return this;
    }

    @Override
    public FinalBuild priority(Priority priority) {
        this.priority = priority;
        return this;
    }

    @Override
    public FinalBuild note(String note) {
        this.note = note;
        return this;
    }

    @Override
    public Order build() {
        return new Order(customerId, items, shippingAddress, priority, note);
    }
}

// Usage - compiler enforces the order!
Order order = OrderBuilder.newOrder()
    .customer("cust-123")                    // Required first
    .addItem(new OrderItem("prod-1", 2))     // Required at least one
    .addItem(new OrderItem("prod-2", 1))     // Can add more
    .shippingAddress(address)                // Required before build
    .priority(Priority.HIGH)                 // Optional
    .build();
```

## Director Pattern (Optional)

### 7. Director for Common Configurations

```java
@ApplicationScoped
public class EmailDirector {

    @Inject
    @ConfigProperty(name = "email.from.noreply")
    String noReplyAddress;

    @Inject
    @ConfigProperty(name = "email.from.support")
    String supportAddress;

    @Inject
    TemplateEngine templateEngine;

    public EmailMessage.Builder welcomeEmail(User user) {
        return EmailMessage.builder()
            .from(noReplyAddress)
            .to(user.getEmail())
            .subject("Welcome to Our Platform!")
            .body(templateEngine.render("welcome", Map.of("user", user)))
            .priority(Priority.HIGH)
            .addHeader("X-Campaign", "onboarding");
    }

    public EmailMessage.Builder passwordReset(User user, String resetToken) {
        return EmailMessage.builder()
            .from(noReplyAddress)
            .to(user.getEmail())
            .subject("Password Reset Request")
            .body(templateEngine.render("password-reset",
                Map.of("user", user, "token", resetToken)))
            .priority(Priority.URGENT)
            .addHeader("X-Security", "password-reset");
    }

    public EmailMessage.Builder supportTicket(User user, Ticket ticket) {
        return EmailMessage.builder()
            .from(supportAddress)
            .to(user.getEmail())
            .subject("Support Ticket #" + ticket.getId())
            .body(templateEngine.render("support-ticket",
                Map.of("ticket", ticket)))
            .addCc(ticket.getAssignee().getEmail());
    }
}

// Usage
@ApplicationScoped
public class UserService {

    @Inject
    EmailDirector emailDirector;

    @Inject
    EmailSender emailSender;

    public void registerUser(User user) {
        // Director provides pre-configured builder
        EmailMessage email = emailDirector.welcomeEmail(user)
            .addHeader("X-User-Tier", user.getTier().name())  // Customize further
            .build();

        emailSender.send(email);
    }
}
```

## When to Use

✅ **Use Builder when:**

- Object has many constructor parameters (4+)
- Object has optional parameters with defaults
- Object should be immutable after creation
- You need readable, fluent object construction
- Different representations need the same construction process

❌ **Avoid Builder when:**

- Object has only 2-3 required parameters
- Object is mutable with setters
- Performance is critical (creates extra object)
- Simple constructors suffice

## Related Patterns

- **Abstract Factory**: Can create builders for different product families
- **Prototype**: Alternative when cloning is simpler than building
- **Composite**: Builders often build composite structures
