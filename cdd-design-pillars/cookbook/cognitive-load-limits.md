# Cognitive Load Limits

## Principle

> Measure and evaluate code readability tangibly. No room for feelings. Use Cognitive Load Theory to distribute complexity.

## The Theory

**Working memory** holds ~7 items simultaneously (research by Miller, refined by Cowan). Code that exceeds this limit becomes hard to understand.

### Intrinsic Cognitive Load

The inherent complexity of the code itself. Measured by counting **cognitive load points**.

## Counting Rules

| Element                         | Points        | Example                          |
| ------------------------------- | ------------- | -------------------------------- |
| Custom class (from your system) | +1            | `OrderService`, `PaymentGateway` |
| Branch (if/else)                | +1            | `if (condition)`                 |
| Loop                            | +1            | `for`, `while`, `forEach`        |
| Try block                       | +1            | `try {`                          |
| Catch block                     | +1 each       | `catch (Exception e)`            |
| Lambda/function                 | +1            | `.map(x -> ...)`                 |
| Stream operation                | +1            | `.filter()`, `.collect()`        |
| Nested branch                   | +1 additional | if inside if                     |

### What Doesn't Count

- Standard library classes (String, List, Optional)
- Framework classes (you should know them)
- Method calls on known types
- Simple variable assignments

## Limits by Component

| Component Type          | Max Points | Rationale                        |
| ----------------------- | ---------- | -------------------------------- |
| **Controller/Resource** | 7          | Entry points must be simple      |
| **Domain Service**      | 7          | Business flow must be clear      |
| **Application Service** | 7          | Orchestration should be readable |
| **Entity**              | 9          | Can hold more complex logic      |
| **Value Object**        | 9          | Encapsulated behavior allowed    |
| **Repository**          | 5          | Should be straightforward        |

## Example: Counting Load

```java
@ApplicationScoped
public class PaymentService {

    @Inject
    private PaymentGateway gateway;  // +1

    @Inject
    private TransactionRepository transactions;  // +1

    @Inject
    private AuditLogger auditLogger;  // +1

    public PaymentResult process(PaymentRequest request) {
        try {  // +1
            if (!request.isValid()) {  // +1
                return PaymentResult.invalid(request.getErrors());
            }

            Payment payment = gateway.charge(request);  // +1

            if (payment.isSuccessful()) {  // +1
                Transaction tx = Transaction.successful(payment);  // +1
                transactions.save(tx);
                auditLogger.logSuccess(tx);
                return PaymentResult.success(tx);
            } else {
                auditLogger.logFailure(payment);
                return PaymentResult.failed(payment.getErrorCode());
            }
        } catch (GatewayException e) {  // +1
            auditLogger.logError(e);
            return PaymentResult.error(e);
        }
    }
}
// Total: 10 points ❌ Exceeds limit!
```

## Solution: Reduce Load

```java
@ApplicationScoped
public class PaymentService {

    @Inject
    private PaymentGateway gateway;  // +1

    @Inject
    private PaymentResultHandler resultHandler;  // +1 (extracted)

    public PaymentResult process(PaymentRequest request) {
        request.validate();  // Throws if invalid, no branch here

        Payment payment = gateway.charge(request);  // +1
        return resultHandler.handle(payment);  // +1
    }
}
// Total: 4 points ✓

@ApplicationScoped
class PaymentResultHandler {

    @Inject
    private TransactionRepository transactions;  // +1

    @Inject
    private AuditLogger auditLogger;  // +1

    public PaymentResult handle(Payment payment) {
        if (payment.isSuccessful()) {  // +1
            return handleSuccess(payment);
        }
        return handleFailure(payment);
    }

    private PaymentResult handleSuccess(Payment payment) {
        Transaction tx = Transaction.successful(payment);  // +1
        transactions.save(tx);
        auditLogger.logSuccess(tx);
        return PaymentResult.success(tx);
    }

    private PaymentResult handleFailure(Payment payment) {
        auditLogger.logFailure(payment);
        return PaymentResult.failed(payment.getErrorCode());
    }
}
// Total: 4 points ✓
```

## Balancing Act

### Don't Over-Extract

```java
// ❌ Too fragmented - 2 points per class is wasteful
class StepOne { void execute() { ... } }
class StepTwo { void execute() { ... } }
class StepThree { void execute() { ... } }

// ✓ Aim for 5-9 points
// This respects cognitive capacity without over-fragmenting
```

### Target Range

| Points | Assessment                          |
| ------ | ----------------------------------- |
| 1-3    | Too simple, may be over-extracted   |
| 4-7    | Ideal sweet spot                    |
| 8-9    | Complex but acceptable for entities |
| 10+    | Needs refactoring                   |

## Related Entries

- [extract-abstractions](extract-abstractions.md) - When to create new classes
- [concentrate-operations](concentrate-operations.md) - Distribute load to domain
