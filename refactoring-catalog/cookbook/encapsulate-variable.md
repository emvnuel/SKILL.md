# Encapsulate Variable

_Also known as: Encapsulate Field, Self-Encapsulate Field_

## Intent

Encapsulate a public field with accessor methods to control access and enable future modifications.

## Code Smells That Indicate This Refactoring

- Public fields accessed directly
- Data changed from multiple places
- Need to add validation or transformation on access
- Want to log or track field changes
- Preparing for **Replace Primitive with Object**

## Mechanics

1. Create getter and setter methods for the field
2. Find all references to the field and replace reads with getter calls, writes with setter calls
3. Make the field private
4. Test
5. Consider restricting visibility of the setter if the field should be immutable after construction

## Example

### Before

```java
public class Person {
    public String name;
    public int age;
}

// Usage
Person person = new Person();
person.name = "John";
person.age = 30;
String greeting = "Hello, " + person.name;
```

### After

```java
public class Person {
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
        this.age = age;
    }
}

// Usage
Person person = new Person();
person.setName("John");
person.setAge(30);
String greeting = "Hello, " + person.getName();
```

### With Immutability

```java
public class Person {
    private final String name;
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public Person withName(String name) {
        return new Person(name, this.age);
    }

    public Person withAge(int age) {
        return new Person(this.name, age);
    }
}
```

### For Mutable Data Structures

```java
public class Team {
    private final List<String> members = new ArrayList<>();

    // Return defensive copy to prevent modification
    public List<String> getMembers() {
        return Collections.unmodifiableList(members);
    }

    // Controlled modification methods
    public void addMember(String member) {
        members.add(member);
    }

    public void removeMember(String member) {
        members.remove(member);
    }
}
```

## When to Use

✅ **Use Encapsulate Variable when:**

- Field is accessed from outside the class
- You want to add validation logic
- You need to track changes to the field
- You're preparing for further refactoring
- You want to enforce immutability

❌ **Avoid Encapsulate Variable when:**

- The class is a simple data holder (consider records instead)
- The field is truly internal and simple
- Encapsulation adds no value (e.g., pure DTOs)

## Related Refactorings

- **Replace Primitive with Object**: Often follows encapsulation
- **Encapsulate Collection**: Specific to collection fields
- **Rename Field**: Often done during encapsulation
- **Remove Setting Method**: To make field immutable
