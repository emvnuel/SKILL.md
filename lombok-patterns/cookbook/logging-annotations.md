# Logging Annotations

## Intent

Automatically generate a static logger field for the class.

## Available Annotations

| Annotation    | Logger Framework       | Generated Field                         |
| ------------- | ---------------------- | --------------------------------------- |
| `@Slf4j`      | SLF4J (recommended)    | `private static final Logger log`       |
| `@Log4j2`     | Log4j 2                | `private static final Logger log`       |
| `@Log4j`      | Log4j 1.x              | `private static final Logger log`       |
| `@Log`        | java.util.logging      | `private static final Logger log`       |
| `@CommonsLog` | Apache Commons Logging | `private static final Log log`          |
| `@JBossLog`   | JBoss Logging          | `private static final Logger log`       |
| `@Flogger`    | Google Flogger         | `private static final FluentLogger log` |
| `@CustomLog`  | Custom logger          | Configurable                            |

## @Slf4j (Recommended)

```java
@Slf4j
@Service
public class OrderService {

    public Order processOrder(Order order) {
        log.info("Processing order: {}", order.getId());

        try {
            validateOrder(order);
            log.debug("Order validated successfully");
            return saveOrder(order);
        } catch (Exception e) {
            log.error("Failed to process order {}: {}", order.getId(), e.getMessage(), e);
            throw e;
        }
    }
}

// Equivalent to:
// private static final Logger log = LoggerFactory.getLogger(OrderService.class);
```

## Custom Topic

```java
@Slf4j(topic = "AUDIT")
@Service
public class AuditService {
    public void logAction(String action) {
        log.info("User action: {}", action);  // Goes to AUDIT logger
    }
}
```

## @Log4j2

```java
@Log4j2
public class BatchProcessor {
    public void process(List<Item> items) {
        log.info("Starting batch processing of {} items", items.size());

        items.forEach(item -> {
            log.trace("Processing item: {}", item);
            processItem(item);
        });

        log.info("Batch processing complete");
    }
}
```

## @Log (java.util.logging)

```java
@Log
public class LegacyService {
    public void doWork() {
        log.info("Starting work");
        log.fine("Detailed log message");
        log.warning("Warning message");
    }
}
```

## Best Practices

### Use Parameterized Logging

```java
@Slf4j
public class UserService {
    public void updateUser(User user) {
        // ✅ Good - lazy evaluation
        log.debug("Updating user: {}", user.getId());

        // ❌ Bad - string concatenation always happens
        log.debug("Updating user: " + user.getId());
    }
}
```

### Log at Appropriate Levels

```java
@Slf4j
public class PaymentService {
    public void processPayment(Payment payment) {
        log.trace("Entering processPayment");  // Very detailed
        log.debug("Payment details: {}", payment);  // Debug info
        log.info("Processing payment {}", payment.getId());  // Normal info

        if (payment.getAmount().compareTo(LARGE_AMOUNT) > 0) {
            log.warn("Large payment detected: {}", payment.getAmount());
        }

        try {
            // process
        } catch (Exception e) {
            log.error("Payment failed: {}", payment.getId(), e);  // Include exception
        }
    }
}
```

### Conditional Logging

```java
@Slf4j
public class ReportService {
    public void generateReport(ReportParams params) {
        if (log.isDebugEnabled()) {
            // Expensive operation only if debug is enabled
            log.debug("Report params: {}", params.toDetailedString());
        }
    }
}
```

## Spring Boot Integration

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
@Slf4j
public class OrderController {
    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        log.info("Received order request: {}", request);
        Order order = orderService.create(request);
        log.info("Order created: {}", order.getId());
        return ResponseEntity.ok(order);
    }
}
```

## Pitfalls

| Issue                        | Solution                                 |
| ---------------------------- | ---------------------------------------- |
| Logger not recognized by IDE | Install Lombok plugin                    |
| Wrong framework              | Check dependencies match annotation      |
| Missing topic                | Use `topic` parameter for custom loggers |

## Related Annotations

- **@RequiredArgsConstructor**: Often used together for services
