# Replace Control Flag with Break

_Also known as: Remove Control Flag_

## Intent

Replace a boolean control flag used to manage loop exit with break, continue, or return statements.

## Code Smells That Indicate This Refactoring

- Boolean variable controlling loop exit
- Flag set inside loop to signal termination
- Complex flag logic obscuring the loop's purpose
- Flag used as the loop condition with internal modifications

## Mechanics

1. Find the place where the flag is set to a value that breaks out of the loop
2. Replace with a break, continue, or return statement
3. Test after each replacement
4. When all the flag logic is removed, remove the flag declaration

## Example

### Before

```java
public class PersonFinder {
    public Person findPerson(List<Person> people, String targetName) {
        Person found = null;
        boolean done = false;
        int i = 0;

        while (!done) {
            if (i >= people.size()) {
                done = true;
            } else {
                Person person = people.get(i);
                if (person.getName().equals(targetName)) {
                    found = person;
                    done = true;
                }
                i++;
            }
        }

        return found;
    }
}
```

### After

```java
public class PersonFinder {
    public Person findPerson(List<Person> people, String targetName) {
        for (Person person : people) {
            if (person.getName().equals(targetName)) {
                return person;
            }
        }
        return null;
    }
}
```

### Using Break

```java
// Before
public int findFirstNegative(int[] numbers) {
    int result = -1;
    boolean found = false;

    for (int i = 0; i < numbers.length && !found; i++) {
        if (numbers[i] < 0) {
            result = i;
            found = true;
        }
    }

    return result;
}

// After
public int findFirstNegative(int[] numbers) {
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] < 0) {
            return i;
        }
    }
    return -1;
}
```

### Using Continue

```java
// Before
public void processItems(List<Item> items) {
    for (Item item : items) {
        boolean shouldSkip = false;

        if (item.isExpired()) {
            shouldSkip = true;
        }
        if (!shouldSkip && item.isLocked()) {
            shouldSkip = true;
        }

        if (!shouldSkip) {
            process(item);
        }
    }
}

// After
public void processItems(List<Item> items) {
    for (Item item : items) {
        if (item.isExpired()) {
            continue;
        }
        if (item.isLocked()) {
            continue;
        }
        process(item);
    }
}
```

### Using Stream

```java
// Even cleaner with streams
public Person findPerson(List<Person> people, String targetName) {
    return people.stream()
        .filter(p -> p.getName().equals(targetName))
        .findFirst()
        .orElse(null);
}
```

## When to Use

✅ **Use Replace Control Flag with Break when:**

- A boolean flag is used solely for loop control
- The flag makes the loop logic harder to understand
- break/continue/return would be clearer
- You want to simplify loop termination logic

❌ **Avoid Replace Control Flag with Break when:**

- The flag has meaning beyond loop control
- Multiple loops need to be exited (use flag for outer loop)
- Team conventions prefer explicit flags
- The flag enables easier debugging

## Related Refactorings

- **Extract Function**: Often enables using return instead
- **Decompose Conditional**: For complex loop conditions
- **Replace Loop with Pipeline**: May eliminate the loop entirely
