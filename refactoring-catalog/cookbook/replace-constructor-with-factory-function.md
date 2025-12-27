# Replace Constructor with Factory Function

_Also known as: Replace Constructor with Factory Method_

## Intent

Replace a constructor with a factory function that can have a more descriptive name and more flexible behavior.

## Code Smells That Indicate This Refactoring

- Constructor name doesn't describe the object being created
- Need to return different subclasses based on input
- Want to return cached instances instead of new ones
- Constructor has limitations (must return exact type, same name as class)
- Complex creation logic that doesn't belong in constructor

## Mechanics

1. Create a factory function that calls the constructor
2. Replace each call to the constructor with a call to the factory function
3. Test after each replacement
4. Consider making the constructor private

## Example

### Before

```java
public class Employee {
    private String name;
    private String type;

    public Employee(String name, String type) {
        this.name = name;
        this.type = type;
    }
}

// Usage - type string is error-prone
Employee engineer = new Employee("John", "engineer");
Employee manager = new Employee("Jane", "manager");
```

### After

```java
public class Employee {
    private String name;
    private String type;

    private Employee(String name, String type) {
        this.name = name;
        this.type = type;
    }

    public static Employee createEngineer(String name) {
        return new Employee(name, "engineer");
    }

    public static Employee createManager(String name) {
        return new Employee(name, "manager");
    }
}

// Usage - clear intent, type-safe
Employee engineer = Employee.createEngineer("John");
Employee manager = Employee.createManager("Jane");
```

### Returning Subclasses

```java
public abstract class Document {
    protected String content;

    public static Document create(String type, String content) {
        switch (type) {
            case "pdf": return new PdfDocument(content);
            case "word": return new WordDocument(content);
            case "text": return new TextDocument(content);
            default: throw new IllegalArgumentException("Unknown type: " + type);
        }
    }

    public abstract void render();
}

class PdfDocument extends Document {
    PdfDocument(String content) { this.content = content; }
    @Override public void render() { /* PDF rendering */ }
}

// Usage
Document doc = Document.create("pdf", "Hello world");
```

### With Caching

```java
public class Color {
    private static final Map<String, Color> cache = new HashMap<>();

    private final int red, green, blue;

    private Color(int red, int green, int blue) {
        this.red = red;
        this.green = green;
        this.blue = blue;
    }

    public static Color of(int red, int green, int blue) {
        String key = red + "," + green + "," + blue;
        return cache.computeIfAbsent(key, k -> new Color(red, green, blue));
    }

    public static Color red() { return of(255, 0, 0); }
    public static Color green() { return of(0, 255, 0); }
    public static Color blue() { return of(0, 0, 255); }
}
```

### CDI Factory

```java
@ApplicationScoped
public class PaymentProcessorFactory {
    @Inject @Any
    Instance<PaymentProcessor> processors;

    public PaymentProcessor create(PaymentMethod method) {
        return processors.select(new PaymentMethodLiteral(method)).get();
    }
}
```

## When to Use

✅ **Use Replace Constructor with Factory Function when:**

- You want descriptive names for creation scenarios
- You need to return different subtypes
- You want to cache and reuse instances
- Complex logic determines what to create
- You want to hide implementation details

❌ **Avoid Replace Constructor with Factory Function when:**

- Constructor is simple and clear
- No variation in creation logic
- Framework requires public constructor
- Over-engineering for simple cases

## Related Refactorings

- **Replace Function with Command**: When factory logic is complex
- **Replace Conditional with Polymorphism**: For type-based creation
- **Extract Function**: To simplify factory logic
