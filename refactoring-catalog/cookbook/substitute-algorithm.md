# Substitute Algorithm

## Intent

Replace the body of a method with a clearer or better algorithm.

## Code Smells That Indicate This Refactoring

- Algorithm is more complex than necessary
- Better algorithm is now available
- Algorithm is hard to understand
- Performance can be significantly improved

## Mechanics

1. Arrange the code to be replaced so it fills a complete function
2. Prepare tests that verify the function's behavior
3. Write the new algorithm
4. Run tests to verify the new algorithm produces the same results
5. If tests fail, use the old algorithm to debug and compare

## Example

### Before

```java
public class PersonFinder {
    public String findPerson(List<String> people) {
        for (int i = 0; i < people.size(); i++) {
            if (people.get(i).equals("Don")) {
                return "Don";
            }
            if (people.get(i).equals("John")) {
                return "John";
            }
            if (people.get(i).equals("Kent")) {
                return "Kent";
            }
        }
        return "";
    }
}
```

### After

```java
public class PersonFinder {
    private static final Set<String> KNOWN_PEOPLE = Set.of("Don", "John", "Kent");

    public String findPerson(List<String> people) {
        return people.stream()
            .filter(KNOWN_PEOPLE::contains)
            .findFirst()
            .orElse("");
    }
}
```

### Sorting Algorithm

```java
// Before - bubble sort
public List<Integer> sort(List<Integer> numbers) {
    List<Integer> result = new ArrayList<>(numbers);
    for (int i = 0; i < result.size() - 1; i++) {
        for (int j = 0; j < result.size() - i - 1; j++) {
            if (result.get(j) > result.get(j + 1)) {
                int temp = result.get(j);
                result.set(j, result.get(j + 1));
                result.set(j + 1, temp);
            }
        }
    }
    return result;
}

// After - use standard library
public List<Integer> sort(List<Integer> numbers) {
    return numbers.stream()
        .sorted()
        .collect(Collectors.toList());
}
```

### Using Standard Library

```java
// Before - manual string joining
public String formatList(List<String> items) {
    StringBuilder result = new StringBuilder();
    for (int i = 0; i < items.size(); i++) {
        if (i > 0) {
            result.append(", ");
        }
        result.append(items.get(i));
    }
    return result.toString();
}

// After - use String.join
public String formatList(List<String> items) {
    return String.join(", ", items);
}
```

### Using Pattern Matching

```java
// Before - manual parsing
public Map<String, String> parseQueryString(String query) {
    Map<String, String> params = new HashMap<>();
    if (query != null && !query.isEmpty()) {
        String[] pairs = query.split("&");
        for (String pair : pairs) {
            int idx = pair.indexOf("=");
            if (idx > 0) {
                String key = pair.substring(0, idx);
                String value = idx < pair.length() - 1 ? pair.substring(idx + 1) : "";
                params.put(key, value);
            }
        }
    }
    return params;
}

// After - using streams
public Map<String, String> parseQueryString(String query) {
    if (query == null || query.isEmpty()) {
        return Map.of();
    }
    return Arrays.stream(query.split("&"))
        .map(s -> s.split("=", 2))
        .filter(arr -> arr.length == 2)
        .collect(Collectors.toMap(arr -> arr[0], arr -> arr[1]));
}
```

## When to Use

✅ **Use Substitute Algorithm when:**

- There's a simpler way to accomplish the same thing
- Standard library provides a better solution
- The algorithm is proven to be incorrect
- Performance needs improvement

❌ **Avoid Substitute Algorithm when:**

- The current algorithm is clear and correct
- The "improvement" is just preference
- The new algorithm has different edge cases
- Extensive testing isn't available

## Related Refactorings

- **Extract Function**: May be needed to isolate the algorithm
- **Replace Loop with Pipeline**: A specific algorithm substitution
- **Decompose Conditional**: To simplify before substituting
