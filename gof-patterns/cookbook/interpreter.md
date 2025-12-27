# Interpreter Pattern - Jakarta EE Cookbook

## Intent

Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

## Jakarta EE Implementation

### 1. Expression Interface

```java
public interface Expression {
    boolean interpret(Map<String, Object> context);
}
```

### 2. Terminal Expressions

```java
public class VariableExpression implements Expression {
    private final String name;

    public VariableExpression(String name) {
        this.name = name;
    }

    @Override
    public boolean interpret(Map<String, Object> context) {
        Object value = context.get(name);
        return value != null && Boolean.TRUE.equals(value);
    }
}

public class ComparisonExpression implements Expression {
    private final String variable;
    private final String operator;
    private final Object value;

    public ComparisonExpression(String variable, String operator, Object value) {
        this.variable = variable;
        this.operator = operator;
        this.value = value;
    }

    @Override
    public boolean interpret(Map<String, Object> context) {
        Object actual = context.get(variable);
        if (actual == null) return false;

        return switch (operator) {
            case "==" -> actual.equals(value);
            case "!=" -> !actual.equals(value);
            case ">" -> compare(actual, value) > 0;
            case "<" -> compare(actual, value) < 0;
            case ">=" -> compare(actual, value) >= 0;
            case "<=" -> compare(actual, value) <= 0;
            default -> false;
        };
    }

    @SuppressWarnings("unchecked")
    private int compare(Object a, Object b) {
        return ((Comparable<Object>) a).compareTo(b);
    }
}
```

### 3. Non-Terminal Expressions

```java
public class AndExpression implements Expression {
    private final Expression left;
    private final Expression right;

    public AndExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }

    @Override
    public boolean interpret(Map<String, Object> context) {
        return left.interpret(context) && right.interpret(context);
    }
}

public class OrExpression implements Expression {
    private final Expression left;
    private final Expression right;

    public OrExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }

    @Override
    public boolean interpret(Map<String, Object> context) {
        return left.interpret(context) || right.interpret(context);
    }
}

public class NotExpression implements Expression {
    private final Expression expression;

    public NotExpression(Expression expression) {
        this.expression = expression;
    }

    @Override
    public boolean interpret(Map<String, Object> context) {
        return !expression.interpret(context);
    }
}
```

### 4. Parser

```java
@ApplicationScoped
public class RuleParser {

    // Simple parser for: "age > 18 AND status == 'active'"
    public Expression parse(String rule) {
        String[] orParts = rule.split("\\s+OR\\s+");
        if (orParts.length > 1) {
            return Arrays.stream(orParts)
                .map(this::parse)
                .reduce(OrExpression::new)
                .orElseThrow();
        }

        String[] andParts = rule.split("\\s+AND\\s+");
        if (andParts.length > 1) {
            return Arrays.stream(andParts)
                .map(this::parseComparison)
                .reduce(AndExpression::new)
                .orElseThrow();
        }

        return parseComparison(rule.trim());
    }

    private Expression parseComparison(String expr) {
        // Parse "variable operator value"
        Pattern pattern = Pattern.compile("(\\w+)\\s*(==|!=|>=|<=|>|<)\\s*(.+)");
        Matcher matcher = pattern.matcher(expr.trim());

        if (matcher.matches()) {
            String variable = matcher.group(1);
            String operator = matcher.group(2);
            String valueStr = matcher.group(3).trim();
            Object value = parseValue(valueStr);
            return new ComparisonExpression(variable, operator, value);
        }

        return new VariableExpression(expr.trim());
    }

    private Object parseValue(String value) {
        if (value.startsWith("'") && value.endsWith("'")) {
            return value.substring(1, value.length() - 1);
        }
        try {
            return Integer.parseInt(value);
        } catch (NumberFormatException e) {
            return value;
        }
    }
}
```

### 5. Usage - Business Rules Engine

```java
@ApplicationScoped
public class RuleEngine {

    @Inject
    RuleParser parser;

    @Inject
    RuleRepository ruleRepository;

    public boolean evaluate(String ruleName, Map<String, Object> context) {
        Rule rule = ruleRepository.findByName(ruleName);
        Expression expression = parser.parse(rule.getExpression());
        return expression.interpret(context);
    }

    public List<String> findMatchingRules(Map<String, Object> context) {
        return ruleRepository.findAll().stream()
            .filter(rule -> {
                Expression expr = parser.parse(rule.getExpression());
                return expr.interpret(context);
            })
            .map(Rule::getName)
            .toList();
    }
}

// Usage
@Path("/discounts")
public class DiscountResource {

    @Inject
    RuleEngine ruleEngine;

    @POST
    @Path("/check")
    public DiscountResult checkDiscount(Customer customer) {
        Map<String, Object> context = Map.of(
            "age", customer.getAge(),
            "status", customer.getStatus(),
            "totalPurchases", customer.getTotalPurchases()
        );

        // Rule: "age >= 65 OR totalPurchases > 1000"
        if (ruleEngine.evaluate("senior-discount", context)) {
            return new DiscountResult(0.15);
        }
        return new DiscountResult(0);
    }
}
```

## When to Use

✅ **Use Interpreter when:**

- You have a simple grammar to interpret
- Efficiency is not critical
- You need user-configurable business rules

❌ **Avoid Interpreter when:**

- Grammar is complex (use parser generators like ANTLR)
- Performance is critical
- Standard expression libraries suffice (SpEL, JEXL)
