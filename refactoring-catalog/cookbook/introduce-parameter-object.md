# Introduce Parameter Object

## Intent

Replace a group of parameters that naturally go together with a single object.

## Code Smells That Indicate This Refactoring

- Same group of parameters appears in multiple functions
- Parameters are always passed together (data clump)
- Long parameter lists
- Related parameters that form a concept

## Mechanics

1. If there isn't a suitable structure, create one (class or record)
2. Test
3. Use **Change Function Declaration** to add the new structure as a parameter
4. Test
5. Adjust all callers to pass the new structure, testing after each
6. Remove the original parameters from the signature
7. Consider adding behavior to the new class

## Example

### Before

```java
public class TemperatureReading {
    private double value;
    private LocalDateTime timestamp;

    // Getters...
}

public class TemperatureAnalyzer {
    public List<TemperatureReading> readingsOutsideRange(
            List<TemperatureReading> readings,
            double min,
            double max) {
        return readings.stream()
            .filter(r -> r.getValue() < min || r.getValue() > max)
            .collect(Collectors.toList());
    }

    public boolean hasReadingsOutsideRange(
            List<TemperatureReading> readings,
            double min,
            double max) {
        return readings.stream()
            .anyMatch(r -> r.getValue() < min || r.getValue() > max);
    }

    public TemperatureReading getFirstOutsideRange(
            List<TemperatureReading> readings,
            double min,
            double max) {
        return readings.stream()
            .filter(r -> r.getValue() < min || r.getValue() > max)
            .findFirst()
            .orElse(null);
    }
}
```

### After

```java
public record TemperatureRange(double min, double max) {
    public TemperatureRange {
        if (min > max) {
            throw new IllegalArgumentException("min must be <= max");
        }
    }

    public boolean contains(double value) {
        return value >= min && value <= max;
    }

    public boolean excludes(double value) {
        return !contains(value);
    }
}

public class TemperatureAnalyzer {
    public List<TemperatureReading> readingsOutsideRange(
            List<TemperatureReading> readings,
            TemperatureRange range) {
        return readings.stream()
            .filter(r -> range.excludes(r.getValue()))
            .collect(Collectors.toList());
    }

    public boolean hasReadingsOutsideRange(
            List<TemperatureReading> readings,
            TemperatureRange range) {
        return readings.stream()
            .anyMatch(r -> range.excludes(r.getValue()));
    }

    public TemperatureReading getFirstOutsideRange(
            List<TemperatureReading> readings,
            TemperatureRange range) {
        return readings.stream()
            .filter(r -> range.excludes(r.getValue()))
            .findFirst()
            .orElse(null);
    }
}
```

### Search Criteria Example

```java
// Before - many parameters
public List<Product> search(
        String name,
        String category,
        BigDecimal minPrice,
        BigDecimal maxPrice,
        boolean inStock,
        String sortBy,
        boolean ascending,
        int page,
        int pageSize) {
    // ...
}

// After - parameter object
public record SearchCriteria(
    String name,
    String category,
    BigDecimal minPrice,
    BigDecimal maxPrice,
    boolean inStock
) {
    public static Builder builder() { return new Builder(); }

    public static class Builder {
        // Builder implementation with sensible defaults
    }
}

public record Pagination(String sortBy, boolean ascending, int page, int pageSize) {
    public static Pagination defaultPagination() {
        return new Pagination("name", true, 0, 20);
    }
}

public List<Product> search(SearchCriteria criteria, Pagination pagination) {
    // ...
}

// Usage
List<Product> products = search(
    SearchCriteria.builder()
        .category("electronics")
        .maxPrice(new BigDecimal("500"))
        .build(),
    Pagination.defaultPagination()
);
```

## When to Use

✅ **Use Introduce Parameter Object when:**

- Same parameters appear together in multiple places
- Parameters naturally form a concept
- You want to add behavior to the grouped data
- Long parameter lists hurt readability

❌ **Avoid Introduce Parameter Object when:**

- Parameters appear together only once
- The grouping doesn't represent a meaningful concept
- It would add unnecessary complexity
- The parameters might diverge in the future

## Related Refactorings

- **Preserve Whole Object**: When an existing object has the data
- **Replace Primitive with Object**: For single values needing behavior
- **Extract Class**: When the parameter object gains behavior
