# Move Statements into Function

## Intent

Move statements that always accompany a function call into the function itself.

## Code Smells That Indicate This Refactoring

- Same code appears before or after every call to a function
- The statements are logically part of the function's operation
- Callers duplicate setup or cleanup code
- Changing the function requires updating all call sites

## Mechanics

1. If the statements are not adjacent to the function call, use **Slide Statements** first
2. Target function is called from only one place: move the statements into the function's body and test
3. If there are more callers, apply **Extract Function** to the call plus statements, then **Inline Function** on the original

## Example

### Before

```java
public class PhotoRenderer {
    public String renderPerson(Person person) {
        StringBuilder result = new StringBuilder();
        result.append("<p>").append(person.getName()).append("</p>");
        result.append(renderPhoto(person.getPhoto()));
        result.append("<p>title: ").append(person.getTitle()).append("</p>");
        result.append(emitPhotoData(person.getPhoto()));
        return result.toString();
    }

    public String photoDiv(Photo photo) {
        StringBuilder result = new StringBuilder();
        result.append("<div>");
        result.append(renderPhoto(photo));
        result.append(emitPhotoData(photo));
        result.append("</div>");
        return result.toString();
    }

    private String renderPhoto(Photo photo) {
        return "<img src=\"" + photo.getUrl() + "\" />";
    }

    private String emitPhotoData(Photo photo) {
        return "<p>title: " + photo.getTitle() + "</p>" +
               "<p>location: " + photo.getLocation() + "</p>";
    }
}
```

### After

```java
public class PhotoRenderer {
    public String renderPerson(Person person) {
        StringBuilder result = new StringBuilder();
        result.append("<p>").append(person.getName()).append("</p>");
        result.append(emitPhotoData(person.getPhoto()));
        result.append("<p>title: ").append(person.getTitle()).append("</p>");
        return result.toString();
    }

    public String photoDiv(Photo photo) {
        StringBuilder result = new StringBuilder();
        result.append("<div>");
        result.append(emitPhotoData(photo));
        result.append("</div>");
        return result.toString();
    }

    private String emitPhotoData(Photo photo) {
        StringBuilder result = new StringBuilder();
        result.append("<img src=\"").append(photo.getUrl()).append("\" />");
        result.append("<p>title: ").append(photo.getTitle()).append("</p>");
        result.append("<p>location: ").append(photo.getLocation()).append("</p>");
        return result.toString();
    }
}
```

### Service Method Example

```java
// Before - logging done at each call site
public class OrderService {
    public void processOrder(Order order) {
        logger.info("Processing order: " + order.getId());
        orderProcessor.process(order);
        logger.info("Finished processing order: " + order.getId());
    }
}

// After - logging moved into processor
public class OrderProcessor {
    public void process(Order order) {
        logger.info("Processing order: " + order.getId());
        doProcess(order);
        logger.info("Finished processing order: " + order.getId());
    }

    private void doProcess(Order order) {
        // actual processing logic
    }
}
```

## When to Use

✅ **Use Move Statements into Function when:**

- Same code always appears with the function call
- The statements are conceptually part of the function
- You're seeing duplication around function calls
- Callers shouldn't need to worry about the details

❌ **Avoid Move Statements into Function when:**

- Some callers don't need the statements
- The statements might need to vary per caller
- It would make the function do too much
- The statements are caller-specific setup

## Related Refactorings

- **Move Statements to Callers**: The inverse operation
- **Extract Function**: Used in the migration mechanics
- **Slide Statements**: May be needed first
