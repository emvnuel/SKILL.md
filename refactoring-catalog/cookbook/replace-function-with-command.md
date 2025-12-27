# Replace Function with Command

_Also known as: Replace Method with Method Object_

## Intent

Encapsulate a function into its own command object, enabling more complex operations like undo, queuing, and parameterization.

## Code Smells That Indicate This Refactoring

- Function needs undo/redo capability
- Function has complex local state
- Need to queue or schedule operations
- Want to log or audit function calls
- Function needs to be serialized

## Mechanics

1. Create an empty class for the function, naming it based on the function
2. Use **Move Function** to move the function to the command class (keep the original as a forwarding function)
3. Convert each argument to a field with assignment in constructor
4. Change all local variables that are arguments to use fields instead
5. Consider extracting complex logic into helper methods in the command class

## Example

### Before

```java
public class Scorer {
    public int score(String candidate, String medicalExam, String scoringGuide) {
        int result = 0;
        int healthLevel = 0;
        boolean highMedicalRiskFlag = false;

        if (medicalExam.contains("smoker")) {
            healthLevel += 10;
            highMedicalRiskFlag = true;
        }

        // ... lots more complex scoring logic

        String certificationGrade = "regular";
        if (scoringGuide.contains("stateTest") && result > 85) {
            certificationGrade = "high";
        }

        result -= Math.max(healthLevel - 5, 0);
        return result;
    }
}
```

### After

```java
public class ScoringCommand {
    private final String candidate;
    private final String medicalExam;
    private final String scoringGuide;

    private int result;
    private int healthLevel;
    private boolean highMedicalRiskFlag;
    private String certificationGrade;

    public ScoringCommand(String candidate, String medicalExam, String scoringGuide) {
        this.candidate = candidate;
        this.medicalExam = medicalExam;
        this.scoringGuide = scoringGuide;
    }

    public int execute() {
        result = 0;
        healthLevel = 0;
        highMedicalRiskFlag = false;

        scoreMedicalExam();
        determineCertification();
        applyHealthPenalty();

        return result;
    }

    private void scoreMedicalExam() {
        if (medicalExam.contains("smoker")) {
            healthLevel += 10;
            highMedicalRiskFlag = true;
        }
        // ... more scoring logic
    }

    private void determineCertification() {
        certificationGrade = "regular";
        if (scoringGuide.contains("stateTest") && result > 85) {
            certificationGrade = "high";
        }
    }

    private void applyHealthPenalty() {
        result -= Math.max(healthLevel - 5, 0);
    }
}

// Usage
int score = new ScoringCommand(candidate, exam, guide).execute();
```

### With Undo Support

```java
public interface Command {
    void execute();
    void undo();
}

public class AddItemCommand implements Command {
    private final ShoppingCart cart;
    private final Item item;

    public AddItemCommand(ShoppingCart cart, Item item) {
        this.cart = cart;
        this.item = item;
    }

    @Override
    public void execute() {
        cart.addItem(item);
    }

    @Override
    public void undo() {
        cart.removeItem(item);
    }
}

public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();

    public void execute(Command command) {
        command.execute();
        history.push(command);
    }

    public void undo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}
```

## When to Use

✅ **Use Replace Function with Command when:**

- You need undo/redo functionality
- The function has complex local state
- You want to queue, delay, or schedule execution
- You need to serialize the operation
- You want to break up complex logic into methods

❌ **Avoid Replace Function with Command when:**

- The function is simple
- You don't need command features (undo, queue, etc.)
- It would over-engineer the solution
- Simple function call is clearer

## Related Refactorings

- **Replace Command with Function**: The inverse
- **Extract Function**: Used to break down the command
- **Move Function**: Used in the mechanics
