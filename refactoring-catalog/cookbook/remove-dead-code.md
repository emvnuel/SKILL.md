# Remove Dead Code

## Intent

Delete code that is no longer executed or used.

## Code Smells That Indicate This Refactoring

- Code that is never called
- Variables that are written but never read
- Unreachable code after return/throw
- Commented-out code
- Deprecated code with no usages

## Mechanics

1. Use IDE/tools to find unused code
2. Verify the code is truly unused (search for reflection, configuration)
3. Delete the unused code
4. Test to confirm nothing breaks

## Example

### Before

```java
public class OrderProcessor {
    private boolean debug = false;  // Never read

    public void processOrder(Order order) {
        validateOrder(order);
        calculateTotal(order);
        saveOrder(order);
    }

    // Never called
    private void logOrder(Order order) {
        System.out.println("Order: " + order);
    }

    // Old implementation, replaced
    private void calculateTotalOld(Order order) {
        double total = 0;
        for (LineItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        order.setTotal(total);
    }

    private void calculateTotal(Order order) {
        order.setTotal(
            order.getItems().stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum()
        );
    }

    // Commented-out code - delete it
    // private void sendConfirmation(Order order) {
    //     EmailService.send(order.getCustomerEmail(), "Order confirmed");
    // }

    private void validateOrder(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Order cannot be null");
        }
        return;  // Unreachable code after return isn't possible,
                 // but sometimes there's code after throw
    }

    private void saveOrder(Order order) {
        // ... save logic
    }
}
```

### After

```java
public class OrderProcessor {
    public void processOrder(Order order) {
        validateOrder(order);
        calculateTotal(order);
        saveOrder(order);
    }

    private void calculateTotal(Order order) {
        order.setTotal(
            order.getItems().stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum()
        );
    }

    private void validateOrder(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Order cannot be null");
        }
    }

    private void saveOrder(Order order) {
        // ... save logic
    }
}
```

### Finding Dead Code

Modern IDEs highlight:

- Unused local variables
- Private methods with no references
- Parameters that are never read
- Unreachable statements

Static analysis tools find:

- Public methods never called
- Classes never instantiated
- Imports not used
- Dead conditional branches

```java
// IDE will warn about these:
public class Example {
    private int unusedField;  // Never read

    public void method(String unusedParam) {  // Parameter never used
        int unusedVar = 42;  // Variable assigned but never read

        if (false) {  // Dead branch
            doSomething();
        }
    }

    private void neverCalled() {  // No references
        // Dead code
    }
}
```

## When to Use

✅ **Use Remove Dead Code when:**

- Code is verifiably never executed
- Features have been removed
- Old implementations are superseded
- Commented-out code has been in VCS long enough

❌ **Avoid Remove Dead Code when:**

- Code is called via reflection
- Code is configuration-driven
- It's test/debugging code that's occasionally useful
- Removal would break public API

## Version Control

Dead code deletion is safe because:

- Version control preserves history
- You can always recover deleted code
- Comments explaining "why" can be in the commit message

## Related Refactorings

- **Inline Function**: May reveal dead code
- **Collapse Hierarchy**: May create dead code to remove
- **Remove Setting Method**: Specific case of dead setter
