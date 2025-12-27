# Extract Function

_Also known as: Extract Method_

## Intent

Turn a code fragment that can be grouped together into a function whose name explains its purpose.

## Code Smells That Indicate This Refactoring

- Long method that does too many things
- Code fragment that requires a comment to explain what it does
- Duplicate code that appears in multiple places
- Code at a different level of abstraction than surrounding code
- Complex conditional logic or loop bodies

## Mechanics

1. Create a new function with a name describing what the function does
2. Copy the extracted code from the source function into the new target function
3. Scan the extracted code for references to variables local to the source function (local variables and parameters)
4. Check if any temporary variables are used only within this extracted code
5. Check whether any variables are modified by the extracted code
6. Pass any variables that are referenced but not modified as parameters
7. Test

## Example

### Before

```java
public class OrderPrinter {
    public void printOwing(Invoice invoice) {
        int outstanding = 0;

        // print banner
        System.out.println("***********************");
        System.out.println("**** Customer Owes ****");
        System.out.println("***********************");

        // calculate outstanding
        for (Order order : invoice.getOrders()) {
            outstanding += order.getAmount();
        }

        // print details
        System.out.println("name: " + invoice.getCustomer());
        System.out.println("amount: " + outstanding);
    }
}
```

### After

```java
public class OrderPrinter {
    public void printOwing(Invoice invoice) {
        printBanner();
        int outstanding = calculateOutstanding(invoice);
        printDetails(invoice, outstanding);
    }

    private void printBanner() {
        System.out.println("***********************");
        System.out.println("**** Customer Owes ****");
        System.out.println("***********************");
    }

    private int calculateOutstanding(Invoice invoice) {
        return invoice.getOrders().stream()
            .mapToInt(Order::getAmount)
            .sum();
    }

    private void printDetails(Invoice invoice, int outstanding) {
        System.out.println("name: " + invoice.getCustomer());
        System.out.println("amount: " + outstanding);
    }
}
```

### Extracting with Local Variables

```java
// Before
public void renderPerson(Person person) {
    StringBuilder result = new StringBuilder();
    result.append("<p>").append(person.getName()).append("</p>");
    result.append("<p>Title: ").append(person.getTitle()).append("</p>");
    result.append("<ul>");
    for (String hobby : person.getHobbies()) {
        result.append("<li>").append(hobby).append("</li>");
    }
    result.append("</ul>");
    display(result.toString());
}

// After
public void renderPerson(Person person) {
    display(renderPersonHtml(person));
}

private String renderPersonHtml(Person person) {
    StringBuilder result = new StringBuilder();
    result.append(renderHeader(person));
    result.append(renderHobbies(person));
    return result.toString();
}

private String renderHeader(Person person) {
    return "<p>" + person.getName() + "</p>" +
           "<p>Title: " + person.getTitle() + "</p>";
}

private String renderHobbies(Person person) {
    StringBuilder result = new StringBuilder("<ul>");
    for (String hobby : person.getHobbies()) {
        result.append("<li>").append(hobby).append("</li>");
    }
    result.append("</ul>");
    return result.toString();
}
```

## When to Use

✅ **Use Extract Function when:**

- A code fragment needs a comment to explain what it does
- The function is longer than a few lines
- The same code appears in multiple places
- You need to reuse a piece of logic
- The code is at a different level of abstraction

❌ **Avoid Extract Function when:**

- The code is already simple and clear
- Extracting would add more complexity
- The function would have too many parameters
- The code is tightly coupled to local state

## Related Refactorings

- **Inline Function**: The inverse operation
- **Replace Temp with Query**: Often done before extraction
- **Extract Variable**: Alternative for complex expressions
- **Move Function**: Often follows extraction
