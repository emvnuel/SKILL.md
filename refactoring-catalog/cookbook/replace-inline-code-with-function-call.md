# Replace Inline Code with Function Call

## Intent

Replace inline code with a call to an existing function that does the same thing.

## Code Smells That Indicate This Refactoring

- Code duplicates what a function already does
- Code implements common logic that should be reused
- Multiple places have the same inline logic
- Standard library or utility function exists

## Mechanics

1. Replace the inline code with a call to the existing function
2. Test

## Example

### Before

```java
public class StudentService {
    public boolean hasPassingGrade(Student student) {
        boolean found = false;
        for (Grade grade : student.getGrades()) {
            if (grade.getValue() >= 60) {
                found = true;
                break;
            }
        }
        return found;
    }
}
```

### After

```java
public class StudentService {
    public boolean hasPassingGrade(Student student) {
        return student.getGrades().stream()
            .anyMatch(grade -> grade.getValue() >= 60);
    }
}
```

### Using Standard Library

```java
// Before - manual implementation
public String joinNames(List<Person> people) {
    StringBuilder result = new StringBuilder();
    for (int i = 0; i < people.size(); i++) {
        if (i > 0) {
            result.append(", ");
        }
        result.append(people.get(i).getName());
    }
    return result.toString();
}

// After - using standard library
public String joinNames(List<Person> people) {
    return people.stream()
        .map(Person::getName)
        .collect(Collectors.joining(", "));
}
```

### Using Utility Functions

```java
// Before - inline null check and default
public String getDisplayName(User user) {
    String displayName;
    if (user.getNickname() != null && !user.getNickname().isEmpty()) {
        displayName = user.getNickname();
    } else {
        displayName = user.getFullName();
    }
    return displayName;
}

// After - using Optional or utility
public String getDisplayName(User user) {
    return Optional.ofNullable(user.getNickname())
        .filter(s -> !s.isEmpty())
        .orElseGet(user::getFullName);
}
```

## When to Use

✅ **Use Replace Inline Code with Function Call when:**

- A function already exists that does the same thing
- The code is duplicated in multiple places
- Standard library provides a cleaner solution
- Using a function would improve readability

❌ **Avoid Replace Inline Code with Function Call when:**

- The inline code is clearer than the function call
- The function has side effects you don't want
- The function does more than you need
- No suitable function exists (create one first)

## Related Refactorings

- **Extract Function**: When no function exists yet
- **Inline Function**: The inverse operation
- **Substitute Algorithm**: Related simplification
