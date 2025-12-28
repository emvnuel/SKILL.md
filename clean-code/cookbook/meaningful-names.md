# Meaningful Names - Clean Code Cookbook

## Intent

Names should reveal intent, be pronounceable, and be searchable. A good name eliminates the need for comments and makes code self-documenting.

---

## Intention-Revealing Names

### Before

```java
public List<int[]> getThem() {
    List<int[]> list1 = new ArrayList<>();
    for (int[] x : theList) {
        if (x[0] == 4) {
            list1.add(x);
        }
    }
    return list1;
}
```

### After

```java
public List<Cell> getFlaggedCells() {
    List<Cell> flaggedCells = new ArrayList<>();
    for (Cell cell : gameBoard) {
        if (cell.isFlagged()) {
            flaggedCells.add(cell);
        }
    }
    return flaggedCells;
}
```

---

## Avoid Disinformation

### Before

```java
// Misleading - not actually a List
Map<String, Account> accountList = new HashMap<>();

// Inconsistent naming
String hp; // horsepower? hit points? health points?
```

### After

```java
Map<String, Account> accountsByName = new HashMap<>();

String horsepower;
// or
String hitPoints;
```

---

## Pronounceable Names

### Before

```java
class DtaRcrd102 {
    private Date genymdhms;    // generation date, year, month, day, hour, minute, second
    private Date modymdhms;    // modification date...
    private final String pszqint = "102";
}
```

### After

```java
class Customer {
    private Date generationTimestamp;
    private Date modificationTimestamp;
    private final String recordId = "102";
}
```

---

## Searchable Names

### Before

```java
for (int j = 0; j < 34; j++) {
    s += (t[j] * 4) / 5;
}
```

### After

```java
private static final int WORK_DAYS_PER_WEEK = 5;
private static final int NUMBER_OF_TASKS = 34;

int realDaysPerIdealDay = 4;
int sum = 0;

for (int taskIndex = 0; taskIndex < NUMBER_OF_TASKS; taskIndex++) {
    int realTaskDays = taskEstimate[taskIndex] * realDaysPerIdealDay;
    int realTaskWeeks = realTaskDays / WORK_DAYS_PER_WEEK;
    sum += realTaskWeeks;
}
```

---

## Class Names

Use **noun or noun phrases** for class names. Avoid verbs.

### Good Examples

```java
public class Customer { }
public class Account { }
public class AddressParser { }
public class WikiPage { }
public class PaymentProcessor { }
```

### Avoid

```java
public class Manager { }      // Too vague
public class Processor { }    // Too vague
public class Data { }         // Meaningless
public class Info { }         // Meaningless
```

---

## Method Names

Use **verb or verb phrases** for method names.

### Good Examples

```java
public void postPayment();
public void deletePage();
public Money calculateTotal();
public boolean isPosted();
public boolean hasExpired();
public String getName();
public void setName(String name);
```

### Accessors, Mutators, and Predicates

```java
// Accessors - get prefix
String getName();
Account getAccount();

// Mutators - set prefix
void setName(String name);
void setBalance(Money balance);

// Predicates - is/has/can prefix
boolean isActive();
boolean hasPermission();
boolean canWithdraw();
```

---

## Pick One Word Per Concept

Use the same word for the same abstract concept throughout your codebase.

### Inconsistent (avoid)

```java
// Mixing synonyms creates confusion
public class CustomerFetcher { }
public class OrderRetriever { }
public class ProductGetter { }

public void fetchUser();
public void getOrder();
public void retrieveProduct();
```

### Consistent (good)

```java
public class CustomerRepository { }
public class OrderRepository { }
public class ProductRepository { }

public Customer findById(Long id);
public Order findById(Long id);
public Product findById(Long id);
```

---

## Domain Names

Use solution domain names for technical concepts and problem domain names for business concepts.

### Technical Patterns

```java
AccountVisitor        // Visitor pattern
JobQueue              // Queue data structure
PolicyFactory         // Factory pattern
AddressRepository     // Repository pattern
```

### Business Domain

```java
Invoice               // Business entity
ShippingAddress       // Business concept
CreditLimit           // Business rule
```

---

## Add Meaningful Context

### Before - Ambiguous

```java
private void printGuessStatistics(char candidate, int count) {
    String number;
    String verb;
    String pluralModifier;

    if (count == 0) {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    } else if (count == 1) {
        number = "1";
        verb = "is";
        pluralModifier = "";
    } else {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }

    String guessMessage = String.format(
        "There %s %s %s%s", verb, number, candidate, pluralModifier
    );
    print(guessMessage);
}
```

### After - Context via Class

```java
public class GuessStatisticsMessage {
    private String number;
    private String verb;
    private String pluralModifier;

    public String make(char candidate, int count) {
        createPluralDependentMessageParts(count);
        return String.format(
            "There %s %s %s%s", verb, number, candidate, pluralModifier
        );
    }

    private void createPluralDependentMessageParts(int count) {
        if (count == 0) {
            thereAreNoLetters();
        } else if (count == 1) {
            thereIsOneLetter();
        } else {
            thereAreManyLetters(count);
        }
    }

    private void thereAreNoLetters() {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    }

    private void thereIsOneLetter() {
        number = "1";
        verb = "is";
        pluralModifier = "";
    }

    private void thereAreManyLetters(int count) {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
}
```

---

## Naming Conventions Summary

| Element   | Convention                   | Examples                        |
| --------- | ---------------------------- | ------------------------------- |
| Class     | Noun/noun phrase, PascalCase | `Customer`, `PaymentProcessor`  |
| Interface | Adjective or noun            | `Comparable`, `Repository`      |
| Method    | Verb/verb phrase, camelCase  | `calculateTotal()`, `isValid()` |
| Variable  | Noun, camelCase              | `customerName`, `totalAmount`   |
| Constant  | UPPER_SNAKE_CASE             | `MAX_SIZE`, `DEFAULT_TIMEOUT`   |
| Package   | lowercase, singular          | `com.company.customer`          |

---

## Common Anti-Patterns

| Anti-Pattern  | Example              | Better Alternative           |
| ------------- | -------------------- | ---------------------------- |
| Single letter | `d`, `e`, `s`        | `daysSinceCreation`          |
| Noise words   | `theData`, `aString` | `data`, `name`               |
| Type encoding | `strName`, `intAge`  | `name`, `age`                |
| Abbreviations | `cstmr`, `calc`      | `customer`, `calculate`      |
| Numbers       | `data1`, `data2`     | `inputData`, `processedData` |

---

## Related Recipes

- [Small Functions](./small-functions.md): Names are easier when functions are small
- [Single Responsibility](./single-responsibility.md): One responsibility = one clear name
