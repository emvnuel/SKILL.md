# Split Loop

## Intent

Split a loop that does multiple things into separate loops, each doing one thing.

## Code Smells That Indicate This Refactoring

- Loop accumulates values for multiple purposes
- Loop body has multiple unrelated sections
- Different parts of the loop could be parallelized independently
- Preparing to extract a function from loop logic

## Mechanics

1. Copy the loop
2. Remove the code for one purpose from each loop
3. Test
4. Consider extracting functions from the resulting loops

## Example

### Before

```java
public class EmployeeReport {
    public void printReport(List<Employee> employees) {
        int youngest = Integer.MAX_VALUE;
        int totalSalary = 0;

        for (Employee employee : employees) {
            if (employee.getAge() < youngest) {
                youngest = employee.getAge();
            }
            totalSalary += employee.getSalary();
        }

        System.out.println("youngest: " + youngest);
        System.out.println("totalSalary: " + totalSalary);
    }
}
```

### After

```java
public class EmployeeReport {
    public void printReport(List<Employee> employees) {
        int youngest = findYoungestAge(employees);
        int totalSalary = calculateTotalSalary(employees);

        System.out.println("youngest: " + youngest);
        System.out.println("totalSalary: " + totalSalary);
    }

    private int findYoungestAge(List<Employee> employees) {
        return employees.stream()
            .mapToInt(Employee::getAge)
            .min()
            .orElse(Integer.MAX_VALUE);
    }

    private int calculateTotalSalary(List<Employee> employees) {
        return employees.stream()
            .mapToInt(Employee::getSalary)
            .sum();
    }
}
```

### Intermediate Stage

```java
// After split, before extraction
public void printReport(List<Employee> employees) {
    // First loop - find youngest
    int youngest = Integer.MAX_VALUE;
    for (Employee employee : employees) {
        if (employee.getAge() < youngest) {
            youngest = employee.getAge();
        }
    }

    // Second loop - calculate total
    int totalSalary = 0;
    for (Employee employee : employees) {
        totalSalary += employee.getSalary();
    }

    System.out.println("youngest: " + youngest);
    System.out.println("totalSalary: " + totalSalary);
}
```

## Performance Considerations

Splitting means iterating twice:

```java
// But this enables cleaner extraction:
int youngest = findYoungestAge(employees);
int totalSalary = calculateTotalSalary(employees);

// And potential parallel execution:
int youngest = employees.parallelStream()
    .mapToInt(Employee::getAge)
    .min()
    .orElse(Integer.MAX_VALUE);
```

In most cases, the performance impact is negligible compared to database queries or I/O. Profile before optimizing!

## When to Use

✅ **Use Split Loop when:**

- Loop does multiple unrelated calculations
- You want to extract functions from loop logic
- Clarity is more important than micro-optimization
- Different parts could benefit from different implementations

❌ **Avoid Split Loop when:**

- Performance is critical and profiling shows the loop is a bottleneck
- The loop operations are genuinely interdependent
- The data source can only be iterated once (streams)
- The loop is already simple and clear

## Related Refactorings

- **Extract Function**: Often follows splitting
- **Replace Loop with Pipeline**: Alternative approach
- **Slide Statements**: May precede splitting
