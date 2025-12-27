# Chain of Responsibility Pattern - Jakarta EE Cookbook

## Intent

Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

## Jakarta EE Implementation

### 1. Handler Interface

```java
public interface RequestHandler {
    boolean handle(HttpRequest request, HttpResponse response);
    int getOrder();  // Lower number = higher priority
}
```

### 2. Concrete Handlers

```java
@ApplicationScoped
public class AuthenticationHandler implements RequestHandler {

    @Inject
    AuthService authService;

    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        String token = request.getHeader("Authorization");
        if (token == null || !authService.isValid(token)) {
            response.setStatus(401);
            response.setBody("Unauthorized");
            return false;  // Stop chain
        }
        return true;  // Continue chain
    }

    @Override
    public int getOrder() {
        return 10;  // First in chain
    }
}

@ApplicationScoped
public class RateLimitHandler implements RequestHandler {

    @Inject
    RateLimiter rateLimiter;

    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        String clientId = request.getHeader("X-Client-Id");
        if (!rateLimiter.allowRequest(clientId)) {
            response.setStatus(429);
            response.setBody("Too Many Requests");
            return false;
        }
        return true;
    }

    @Override
    public int getOrder() {
        return 20;
    }
}

@ApplicationScoped
public class ValidationHandler implements RequestHandler {

    @Inject
    Validator validator;

    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        Set<ConstraintViolation<?>> violations = validator.validate(request.getBody());
        if (!violations.isEmpty()) {
            response.setStatus(400);
            response.setBody(formatViolations(violations));
            return false;
        }
        return true;
    }

    @Override
    public int getOrder() {
        return 30;
    }
}

@ApplicationScoped
public class LoggingHandler implements RequestHandler {

    private static final Logger log = LoggerFactory.getLogger(LoggingHandler.class);

    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        log.info("Request: {} {}", request.getMethod(), request.getPath());
        return true;  // Always continue
    }

    @Override
    public int getOrder() {
        return 5;  // Very first (logging)
    }
}
```

### 3. Chain Executor

```java
@ApplicationScoped
public class RequestPipeline {

    @Inject
    @Any
    Instance<RequestHandler> handlers;

    public void process(HttpRequest request, HttpResponse response) {
        handlers.stream()
            .sorted(Comparator.comparingInt(RequestHandler::getOrder))
            .allMatch(handler -> handler.handle(request, response));
    }
}
```

### 4. Alternative: Filter Chain (JAX-RS Filters)

```java
// JAX-RS provides built-in chain of responsibility
@Provider
@Priority(Priorities.AUTHENTICATION)
public class AuthFilter implements ContainerRequestFilter {

    @Override
    public void filter(ContainerRequestContext context) throws IOException {
        String auth = context.getHeaderString("Authorization");
        if (auth == null) {
            context.abortWith(Response.status(401).build());
        }
    }
}

@Provider
@Priority(Priorities.AUTHORIZATION)
public class RoleFilter implements ContainerRequestFilter {

    @Inject
    SecurityContext securityContext;

    @Override
    public void filter(ContainerRequestContext context) throws IOException {
        // Role checking logic
    }
}
```

### 5. Approval Chain Example

```java
public interface Approver {
    ApprovalResult approve(ExpenseRequest request);
    BigDecimal getApprovalLimit();
}

@ApplicationScoped
public class ManagerApprover implements Approver {
    public ApprovalResult approve(ExpenseRequest request) {
        if (request.getAmount().compareTo(getApprovalLimit()) <= 0) {
            return ApprovalResult.approved("Manager");
        }
        return ApprovalResult.escalate();
    }
    public BigDecimal getApprovalLimit() { return new BigDecimal("5000"); }
}

@ApplicationScoped
public class DirectorApprover implements Approver {
    public ApprovalResult approve(ExpenseRequest request) {
        if (request.getAmount().compareTo(getApprovalLimit()) <= 0) {
            return ApprovalResult.approved("Director");
        }
        return ApprovalResult.escalate();
    }
    public BigDecimal getApprovalLimit() { return new BigDecimal("25000"); }
}

@ApplicationScoped
public class ApprovalService {

    @Inject
    @Any
    Instance<Approver> approvers;

    public ApprovalResult requestApproval(ExpenseRequest request) {
        return approvers.stream()
            .sorted(Comparator.comparing(Approver::getApprovalLimit))
            .map(a -> a.approve(request))
            .filter(r -> r.isTerminal())
            .findFirst()
            .orElse(ApprovalResult.rejected("Exceeds all limits"));
    }
}
```

## When to Use

✅ **Use Chain of Responsibility when:**

- Multiple objects may handle a request
- Handler should be determined dynamically
- You want to decouple sender from receivers

❌ **Avoid when:**

- Only one handler is ever needed
- Processing order is fixed and simple
