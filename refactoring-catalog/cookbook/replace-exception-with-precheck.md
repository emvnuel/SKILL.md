# Replace Exception with Precheck

_Also known as: Replace Exception with Test_

## Intent

Replace an exception catch with a conditional test when the condition can be checked beforehand.

## Code Smells That Indicate This Refactoring

- Using exceptions for expected conditions
- Try-catch for validating data that could be checked first
- Exceptions as flow control
- Performance issues from exception handling

## Mechanics

1. Add a conditional check before the operation that would throw
2. Move the catch block logic into the else/failure path
3. Remove the try-catch
4. Test

## Example

### Before

```java
public class ResourceManager {
    private Map<String, Resource> resources = new HashMap<>();

    public Resource getResource(String key) {
        try {
            return resources.get(key).clone();
        } catch (NullPointerException e) {
            return Resource.defaultResource();
        }
    }
}
```

### After

```java
public class ResourceManager {
    private Map<String, Resource> resources = new HashMap<>();

    public Resource getResource(String key) {
        Resource resource = resources.get(key);
        if (resource != null) {
            return resource.clone();
        }
        return Resource.defaultResource();
    }
}
```

### Number Parsing

```java
// Before
public int parseNumber(String value) {
    try {
        return Integer.parseInt(value);
    } catch (NumberFormatException e) {
        return 0;
    }
}

// After
public int parseNumber(String value) {
    if (value == null || !value.matches("-?\\d+")) {
        return 0;
    }
    return Integer.parseInt(value);
}

// Or using Optional
public OptionalInt parseNumber(String value) {
    if (value == null || !value.matches("-?\\d+")) {
        return OptionalInt.empty();
    }
    return OptionalInt.of(Integer.parseInt(value));
}
```

### Collection Access

```java
// Before
public Employee getEmployee(int index) {
    try {
        return employees.get(index);
    } catch (IndexOutOfBoundsException e) {
        return null;
    }
}

// After
public Employee getEmployee(int index) {
    if (index < 0 || index >= employees.size()) {
        return null;
    }
    return employees.get(index);
}

// Or with Optional
public Optional<Employee> getEmployee(int index) {
    if (index < 0 || index >= employees.size()) {
        return Optional.empty();
    }
    return Optional.of(employees.get(index));
}
```

### Division

```java
// Before
public double divide(double numerator, double denominator) {
    try {
        return numerator / denominator;
    } catch (ArithmeticException e) {
        return 0;
    }
}

// After
public double divide(double numerator, double denominator) {
    if (denominator == 0) {
        return 0;
    }
    return numerator / denominator;
}
```

## When to Use

✅ **Use Replace Exception with Precheck when:**

- The exception represents an expected condition
- The check is simple and inexpensive
- Exceptions are used for flow control
- Performance is impacted by exception handling
- The condition is common in normal operation

❌ **Avoid Replace Exception with Precheck when:**

- The exception is truly exceptional (rare error)
- The check would be complex or error-prone
- Race conditions exist between check and operation
- The exception provides important diagnostic information
- The operation is atomic and the check can't be

## Race Condition Warning

```java
// DANGEROUS: Time-of-check to time-of-use (TOCTOU) bug
if (file.exists()) {
    file.delete();  // File could be deleted by another process between check and delete
}

// BETTER: Just attempt the operation and handle failure
try {
    Files.delete(file.toPath());
} catch (NoSuchFileException e) {
    // File already deleted, that's fine
} catch (IOException e) {
    throw new RuntimeException("Failed to delete file", e);
}
```

## Related Refactorings

- **Introduce Assertion**: For programmer errors vs expected conditions
- **Replace Error Code with Exception**: The inverse direction
- **Introduce Special Case**: For null/special value handling
