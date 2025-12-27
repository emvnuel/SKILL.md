# Parameterize Function

_Also known as: Parameterize Method_

## Intent

Combine similar functions that differ only in literal values into a single function with parameters.

## Code Smells That Indicate This Refactoring

- Multiple functions doing essentially the same thing with different values
- Copy-paste code with only constants changed
- Adding a new variation requires creating another function
- Similar functions with same structure, different literals

## Mechanics

1. Select one of the similar methods
2. Use **Change Function Declaration** to add parameters for the varying literals
3. Adjust the body to use the new parameters
4. Update callers to pass appropriate values
5. Test
6. Replace other similar functions with calls to the parameterized function
7. Test after each replacement

## Example

### Before

```java
public class Employee {
    private double salary;

    public void tenPercentRaise() {
        salary *= 1.10;
    }

    public void fivePercentRaise() {
        salary *= 1.05;
    }

    public void threePercentRaise() {
        salary *= 1.03;
    }
}
```

### After

```java
public class Employee {
    private double salary;

    public void raise(double percentage) {
        salary *= (1 + percentage / 100);
    }
}

// Usage
employee.raise(10);  // 10% raise
employee.raise(5);   // 5% raise
employee.raise(3);   // 3% raise
```

### URL Builder Example

```java
// Before
public class UrlBuilder {
    public String buildProductUrl(String productId) {
        return baseUrl + "/products/" + productId;
    }

    public String buildCategoryUrl(String categoryId) {
        return baseUrl + "/categories/" + categoryId;
    }

    public String buildUserUrl(String userId) {
        return baseUrl + "/users/" + userId;
    }
}

// After
public class UrlBuilder {
    public String buildResourceUrl(String resourceType, String resourceId) {
        return baseUrl + "/" + resourceType + "/" + resourceId;
    }
}

// Or with enum for type safety
public enum ResourceType {
    PRODUCTS("products"),
    CATEGORIES("categories"),
    USERS("users");

    private final String path;
    ResourceType(String path) { this.path = path; }
    public String getPath() { return path; }
}

public String buildResourceUrl(ResourceType type, String resourceId) {
    return baseUrl + "/" + type.getPath() + "/" + resourceId;
}
```

### Range Checking Example

```java
// Before
public boolean isLowTemperature(int temperature) {
    return temperature < 0;
}

public boolean isHighTemperature(int temperature) {
    return temperature > 30;
}

// After
public boolean isInRange(int temperature, int low, int high) {
    return temperature >= low && temperature <= high;
}

public boolean isOutOfRange(int temperature, int low, int high) {
    return temperature < low || temperature > high;
}

// Usage
boolean isCold = isOutOfRange(temp, 0, Integer.MAX_VALUE);  // temp < 0
boolean isHot = isOutOfRange(temp, Integer.MIN_VALUE, 30);  // temp > 30
```

## When to Use

✅ **Use Parameterize Function when:**

- Multiple functions differ only in literal values
- You anticipate needing more variations
- The pattern of duplication is clear
- Adding parameters improves flexibility

❌ **Avoid Parameterize Function when:**

- Functions have different behavior beyond the literal
- The parameterization would be confusing
- The functions are part of a public API with specific names
- Adding parameters would make the function harder to use

## Related Refactorings

- **Change Function Declaration**: Used to add the parameter
- **Remove Flag Argument**: Can be the inverse
- **Replace Parameter with Query**: When parameter can be derived
- **Introduce Parameter Object**: When too many parameters result
