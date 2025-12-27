# Replace Loop with Pipeline

## Intent

Replace loop code with a collection pipeline using map, filter, and reduce operations.

## Code Smells That Indicate This Refactoring

- Loop that transforms, filters, or aggregates a collection
- Loop with complex conditional logic
- Loop accumulating into a result variable
- Hard-to-read imperative loop code

## Mechanics

1. Create a variable for the loop's collection
2. Start building the pipeline with the first loop operation
3. Add each operation as a pipeline step
4. Replace the loop result with the pipeline result

## Example

### Before

```java
public List<String> getOfficeNames(List<Employee> employees) {
    List<String> result = new ArrayList<>();
    for (Employee employee : employees) {
        if (employee.getJob().equals("office")) {
            result.add(employee.getName());
        }
    }
    return result;
}
```

### After

```java
public List<String> getOfficeNames(List<Employee> employees) {
    return employees.stream()
        .filter(e -> e.getJob().equals("office"))
        .map(Employee::getName)
        .collect(Collectors.toList());
}
```

### Complex Example

```java
// Before
public List<String> acquireData(String input) {
    String[] lines = input.split("\n");
    boolean firstLine = true;
    List<String> result = new ArrayList<>();

    for (String line : lines) {
        if (firstLine) {
            firstLine = false;
            continue;
        }
        if (line.trim().isEmpty()) {
            continue;
        }
        String[] parts = line.split(",");
        if (parts.length >= 2) {
            result.add(parts[1].trim());
        }
    }
    return result;
}

// After
public List<String> acquireData(String input) {
    return Arrays.stream(input.split("\n"))
        .skip(1)                           // Skip header
        .filter(line -> !line.trim().isEmpty())  // Remove blank lines
        .map(line -> line.split(","))      // Split into parts
        .filter(parts -> parts.length >= 2) // Keep valid rows
        .map(parts -> parts[1].trim())     // Extract second column
        .collect(Collectors.toList());
}
```

### Aggregation

```java
// Before
public int getTotalSalary(List<Employee> employees) {
    int total = 0;
    for (Employee employee : employees) {
        if (employee.getDepartment().equals("engineering")) {
            total += employee.getSalary();
        }
    }
    return total;
}

// After
public int getTotalSalary(List<Employee> employees) {
    return employees.stream()
        .filter(e -> e.getDepartment().equals("engineering"))
        .mapToInt(Employee::getSalary)
        .sum();
}
```

### Grouping

```java
// Before
public Map<String, List<Employee>> groupByDepartment(List<Employee> employees) {
    Map<String, List<Employee>> result = new HashMap<>();
    for (Employee employee : employees) {
        String dept = employee.getDepartment();
        if (!result.containsKey(dept)) {
            result.put(dept, new ArrayList<>());
        }
        result.get(dept).add(employee);
    }
    return result;
}

// After
public Map<String, List<Employee>> groupByDepartment(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
}
```

### Finding Elements

```java
// Before
public Employee findFirst(List<Employee> employees, String name) {
    for (Employee employee : employees) {
        if (employee.getName().equals(name)) {
            return employee;
        }
    }
    return null;
}

// After
public Optional<Employee> findFirst(List<Employee> employees, String name) {
    return employees.stream()
        .filter(e -> e.getName().equals(name))
        .findFirst();
}
```

## When to Use

✅ **Use Replace Loop with Pipeline when:**

- Loop transforms, filters, or aggregates
- Pipeline would be more declarative
- Multiple operations are chained
- The pattern is common (map, filter, reduce)

❌ **Avoid Replace Loop with Pipeline when:**

- Loop has complex control flow (break, labeled continue)
- Side effects are important
- Performance is critical and streams add overhead
- Loop is already simple and clear

## Related Refactorings

- **Replace Control Flag with Break**: May be needed first
- **Extract Function**: For complex pipeline steps
- **Substitute Algorithm**: Related algorithmic change
