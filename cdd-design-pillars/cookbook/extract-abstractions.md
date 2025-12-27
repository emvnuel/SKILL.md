# Extract Abstractions Logically

## Principle

> Every indirection (new file/class) must earn its existence. Distributing cognitive load is a valid reason.

## Problem

A method exceeds cognitive load limits even when following encapsulation rules.

```java
@ApplicationScoped
public class PaymentOrchestrator {

    @Inject
    private List<PaymentProcessor> processors;  // +1

    public Transaction pay(PaymentAttempt attempt) {
        // Stream operations add up quickly
        TreeSet<Payer> payers = this.processors.stream()  // +1
            .map(p -> p.accepts(attempt))  // +1
            .filter(Optional::isPresent)  // +2 (filter + Optional branch)
            .map(Optional::get)  // +1
            .collect(Collectors.toCollection(TreeSet::new));  // +1

        Assert.isTrue(!payers.isEmpty(), "No payer available");

        for (Payer payer : payers) {  // +1
            try {  // +1
                Transaction tx = payer.pay();
                log.info("Payment successful: {}", tx);
                return tx;
            } catch (FeignException e) {  // +1
                log.error("Network error: {}", e);
            } catch (Exception e) {  // +1
                log.error("Payment failed: {}", e);
            }
        }

        return Transaction.failed();
    }
}
// Total: 11 points ❌ For a Domain Service (max 7)
```

## Solution

Extract a new abstraction to distribute the load.

```java
// New class handles payer selection - earns its place
@ApplicationScoped
class SortedPayerSelector {

    @Inject
    private List<PaymentProcessor> processors;  // +1

    public SortedSet<Payer> selectFor(PaymentAttempt attempt) {
        return this.processors.stream()  // +1
            .map(p -> p.accepts(attempt))  // +1
            .filter(Optional::isPresent)  // +2
            .map(Optional::get)  // +1
            .collect(Collectors.toCollection(TreeSet::new));  // +1
    }
}
// Total: 7 points ✓

@ApplicationScoped
public class PaymentOrchestrator {

    @Inject
    private SortedPayerSelector payerSelector;  // +1

    public Transaction pay(PaymentAttempt attempt) {
        SortedSet<Payer> payers = payerSelector.selectFor(attempt);  // +1

        Assert.isTrue(!payers.isEmpty(), "No payer available");

        for (Payer payer : payers) {  // +1
            try {  // +1
                Transaction tx = payer.pay();
                log.info("Payment successful: {}", tx);
                return tx;
            } catch (FeignException e) {  // +1
                log.error("Network error: {}", e);
            } catch (Exception e) {  // +1
                log.error("Payment failed: {}", e);
            }
        }

        return Transaction.failed();
    }
}
// Total: 6 points ✓ (+1 new dependency = 7 total)
```

## When to Extract

| Signal                  | Action              |
| ----------------------- | ------------------- |
| Exceeds load limit      | Must extract        |
| Long stream chain       | Consider extraction |
| Distinct responsibility | Name it and extract |
| Reused elsewhere        | Extract and share   |

## When NOT to Extract

| Signal                       | Action         |
| ---------------------------- | -------------- |
| Would create 2-3 point class | Too fragmented |
| No clear responsibility name | Keep inline    |
| Only used once, simple       | Inline is fine |

## Naming the Extraction

Good names describe what the class does:

```java
// ✓ Names describe responsibility
SortedPayerSelector        // Selects and sorts payers
PaymentRetryHandler        // Handles retry logic
TransactionAuditor         // Audits transactions
OrderEligibilityChecker    // Checks order eligibility

// ❌ Vague names
PaymentHelper
OrderUtils
ProcessorManager
```

## Package Visibility

Extracted classes often don't need public visibility:

```java
// Package-private - internal implementation detail
@ApplicationScoped
class SortedPayerSelector {
    // ...
}

// Public - part of the API
@ApplicationScoped
public class PaymentOrchestrator {
    // ...
}
```

## The Balance

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   1-3 points: Too simple, may be over-extracted        │
│                                                         │
│   4-7 points: Sweet spot ✓                             │
│                                                         │
│   8-9 points: OK for entities/complex domain           │
│                                                         │
│   10+ points: Needs extraction                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Related Entries

- [cognitive-load-limits](cognitive-load-limits.md) - Measuring load
- [cdi-bean-design](cdi-bean-design.md) - CDI bean patterns
