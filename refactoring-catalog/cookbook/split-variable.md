# Split Variable

_Also known as: Remove Assignments to Parameters, Split Temp_

## Intent

Split a variable that is assigned more than once into separate variables for each assignment.

## Code Smells That Indicate This Refactoring

- Variable assigned multiple times for different purposes
- Loop variable reused after the loop
- Input parameter modified within the method
- Variable name that no longer reflects its current value

## Mechanics

1. Change the name of the variable at its first declaration and assignment
2. If possible, declare the new variable as final
3. Change all references up to the second assignment
4. Test
5. Repeat for each subsequent assignment

## Example

### Before

```java
public double distanceTraveled(double time) {
    double result;
    double acc = primaryForce / mass;
    double primaryTime = Math.min(time, delay);
    result = 0.5 * acc * primaryTime * primaryTime;

    double secondaryTime = time - delay;
    if (secondaryTime > 0) {
        double primaryVelocity = acc * delay;
        acc = (primaryForce + secondaryForce) / mass;  // acc reused!
        result += primaryVelocity * secondaryTime +
                  0.5 * acc * secondaryTime * secondaryTime;
    }
    return result;
}
```

### After

```java
public double distanceTraveled(double time) {
    final double primaryAcceleration = primaryForce / mass;
    final double primaryTime = Math.min(time, delay);
    double result = 0.5 * primaryAcceleration * primaryTime * primaryTime;

    final double secondaryTime = time - delay;
    if (secondaryTime > 0) {
        final double primaryVelocity = primaryAcceleration * delay;
        final double secondaryAcceleration = (primaryForce + secondaryForce) / mass;
        result += primaryVelocity * secondaryTime +
                  0.5 * secondaryAcceleration * secondaryTime * secondaryTime;
    }
    return result;
}
```

### Loop Variables

```java
// Before - temp reused for different purposes
public int calculate(int[] values) {
    int temp = 0;
    for (int value : values) {
        temp += value;
    }
    temp = temp / values.length;  // Now it's average, not sum!
    return temp;
}

// After - clear separation
public int calculate(int[] values) {
    int sum = 0;
    for (int value : values) {
        sum += value;
    }
    int average = sum / values.length;
    return average;
}
```

### Input Parameters

```java
// Before - modifying parameter
public int discount(int inputValue, int quantity) {
    if (inputValue > 50) {
        inputValue -= 2;
    }
    if (quantity > 100) {
        inputValue -= 1;
    }
    return inputValue;
}

// After - using result variable
public int discount(int inputValue, int quantity) {
    int result = inputValue;
    if (inputValue > 50) {
        result -= 2;
    }
    if (quantity > 100) {
        result -= 1;
    }
    return result;
}
```

## When to Use

✅ **Use Split Variable when:**

- A variable is assigned different values for different purposes
- A loop variable is reused after the loop
- An input parameter is modified
- The variable name stops making sense after reassignment

❌ **Avoid Split Variable when:**

- The variable is a collecting variable (sum, concatenation)
- The variable is intentionally an accumulator
- Splitting would make the code harder to read
- The variable genuinely represents the same concept

## Related Refactorings

- **Extract Variable**: May follow splitting
- **Replace Temp with Query**: Alternative for calculated values
- **Rename Variable**: Often done along with splitting
