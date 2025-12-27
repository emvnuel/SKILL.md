# Split Phase

## Intent

Split code that handles multiple separate concerns into distinct phases, each with a clear focus.

## Code Smells That Indicate This Refactoring

- Code that both parses input and processes it
- Method that validates and transforms in one pass
- Logic mixing data preparation with calculations
- Code that could be pipelined has phases tangled

## Mechanics

1. Extract the second phase into its own function
2. Introduce an intermediate data structure to communicate between phases
3. Examine each parameter to the second phase; if used by first phase, add to the intermediate structure
4. Apply **Extract Function** to the first phase
5. Test

## Example

### Before

```java
public class OrderParser {
    public double priceOrder(String orderData) {
        String[] lines = orderData.split("\n");
        String firstLine = lines[0].split("\\s+")[0];
        String productName = firstLine;
        int orderQuantity = Integer.parseInt(lines[0].split("\\s+")[1]);

        double price = 0;
        for (int i = 1; i < lines.length; i++) {
            String[] parts = lines[i].split("\\s+");
            if (parts[0].equals(productName)) {
                price = Double.parseDouble(parts[1]);
                break;
            }
        }

        return orderQuantity * price;
    }
}
```

### After

```java
public class OrderParser {
    public double priceOrder(String orderData) {
        OrderDetails order = parseOrder(orderData);
        return calculatePrice(order);
    }

    private OrderDetails parseOrder(String orderData) {
        String[] lines = orderData.split("\n");
        String[] firstLineParts = lines[0].split("\\s+");
        String productName = firstLineParts[0];
        int quantity = Integer.parseInt(firstLineParts[1]);

        double unitPrice = 0;
        for (int i = 1; i < lines.length; i++) {
            String[] parts = lines[i].split("\\s+");
            if (parts[0].equals(productName)) {
                unitPrice = Double.parseDouble(parts[1]);
                break;
            }
        }

        return new OrderDetails(productName, quantity, unitPrice);
    }

    private double calculatePrice(OrderDetails order) {
        return order.quantity() * order.unitPrice();
    }

    private record OrderDetails(String productName, int quantity, double unitPrice) {}
}
```

### Compiler-Style Phases

```java
public class ExpressionEvaluator {
    // Phase 1: Tokenize
    public List<Token> tokenize(String expression) {
        List<Token> tokens = new ArrayList<>();
        // ... tokenization logic
        return tokens;
    }

    // Phase 2: Parse
    public AstNode parse(List<Token> tokens) {
        // ... parsing logic
        return rootNode;
    }

    // Phase 3: Evaluate
    public double evaluate(AstNode ast) {
        // ... evaluation logic
        return result;
    }

    // Combined
    public double process(String expression) {
        List<Token> tokens = tokenize(expression);
        AstNode ast = parse(tokens);
        return evaluate(ast);
    }
}
```

## Common Phases

- **Parse** → **Process**: Separate reading/parsing from business logic
- **Validate** → **Transform**: Separate validation from data transformation
- **Collect** → **Analyze**: Separate data gathering from analysis
- **Prepare** → **Execute**: Separate setup from execution

## When to Use

✅ **Use Split Phase when:**

- Code mixes parsing with processing
- You want to test phases independently
- Different parts could be reused separately
- The code would be clearer as a pipeline

❌ **Avoid Split Phase when:**

- The phases are genuinely interleaved
- Splitting would require complex data structures
- The code is already simple
- Performance requires single-pass processing

## Related Refactorings

- **Extract Function**: Used to create the phases
- **Introduce Parameter Object**: For the intermediate data structure
- **Split Loop**: Related separation technique
