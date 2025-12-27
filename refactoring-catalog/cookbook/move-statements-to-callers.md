# Move Statements to Callers

## Intent

Move code from a function to its callers when callers need different behavior for that code.

## Code Smells That Indicate This Refactoring

- A function previously shared is now needed differently by different callers
- Adding a parameter or flag would make the function messy
- Part of a function needs to vary between callers
- Function is doing too much and should be split

## Mechanics

1. In simple cases, cut the code from the source function and paste into all callers
2. For complex cases, use **Extract Function** on the part you wish to keep
3. Apply **Inline Function** on the remaining source function

## Example

### Before

```java
public class PersonRenderer {
    public String renderPerson(Person person) {
        StringBuilder result = new StringBuilder();
        result.append("<p>").append(person.getName()).append("</p>");
        result.append(emitPhotoData(person.getPhoto()));
        return result.toString();
    }

    public String listRecentPhotos(List<Photo> photos) {
        return photos.stream()
            .filter(p -> p.getDate().isAfter(recentDateCutoff()))
            .map(this::emitPhotoData)
            .collect(Collectors.joining("\n"));
    }

    private String emitPhotoData(Photo photo) {
        StringBuilder result = new StringBuilder();
        result.append("<p>title: ").append(photo.getTitle()).append("</p>");
        result.append("<p>date: ").append(photo.getDate()).append("</p>");
        result.append("<p>location: ").append(photo.getLocation()).append("</p>");
        return result.toString();
    }
}

// Now listRecentPhotos shouldn't show location...
```

### After

```java
public class PersonRenderer {
    public String renderPerson(Person person) {
        StringBuilder result = new StringBuilder();
        result.append("<p>").append(person.getName()).append("</p>");
        result.append(emitPhotoData(person.getPhoto()));
        result.append("<p>location: ").append(person.getPhoto().getLocation()).append("</p>");
        return result.toString();
    }

    public String listRecentPhotos(List<Photo> photos) {
        return photos.stream()
            .filter(p -> p.getDate().isAfter(recentDateCutoff()))
            .map(this::emitPhotoData)
            .collect(Collectors.joining("\n"));
    }

    private String emitPhotoData(Photo photo) {
        StringBuilder result = new StringBuilder();
        result.append("<p>title: ").append(photo.getTitle()).append("</p>");
        result.append("<p>date: ").append(photo.getDate()).append("</p>");
        return result.toString();
    }
}
```

### Splitting Shared Behavior

```java
// Before - all callers do the same thing
public class ReportGenerator {
    private void generateReport(Report report) {
        writeHeader(report);
        writeBody(report);
        writeFooter(report);
        saveToFile(report);  // Some callers want to email instead
    }
}

// After - output method moved to callers
public class ReportGenerator {
    private Report generateReport(Report report) {
        writeHeader(report);
        writeBody(report);
        writeFooter(report);
        return report;
    }

    public void generateAndSave(Report report) {
        generateReport(report);
        saveToFile(report);
    }

    public void generateAndEmail(Report report, String recipient) {
        generateReport(report);
        emailReport(report, recipient);
    }
}
```

## When to Use

✅ **Use Move Statements to Callers when:**

- Different callers need slightly different behavior
- A function's responsibilities are diverging
- A flag parameter is threatening to complicate the function
- Some callers want to omit part of the function

❌ **Avoid Move Statements to Callers when:**

- All callers still need the same behavior
- It would create significant duplication
- The variation can be handled with parameters
- The caller shouldn't know these details

## Related Refactorings

- **Move Statements into Function**: The inverse operation
- **Extract Function**: Often used in mechanics
- **Inline Function**: Used to complete the migration
