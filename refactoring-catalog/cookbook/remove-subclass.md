# Remove Subclass

_Also known as: Replace Subclass with Fields_

## Intent

Replace subclasses with fields when they don't justify their complexity.

## Code Smells That Indicate This Refactoring

- Subclasses only return different constant values
- No significant behavioral difference between subclasses
- Subclass hierarchy adds complexity without value
- Most subclass methods just return static data

## Mechanics

1. Use **Replace Constructor with Factory Function** on the subclasses
2. Add a type code field to the superclass
3. Modify subclass constructors to set the type code
4. Create factory methods to set the type code and return superclass type
5. Remove subclass references in type declarations
6. Move type-specific code to superclass using conditionals
7. Delete subclasses
8. Test after each step

## Example

### Before

```java
public abstract class Person {
    protected String name;

    public abstract String getGenderCode();
}

public class Male extends Person {
    @Override
    public String getGenderCode() {
        return "M";
    }
}

public class Female extends Person {
    @Override
    public String getGenderCode() {
        return "F";
    }
}

// Usage
Person john = new Male();
Person jane = new Female();
```

### After

```java
public class Person {
    private String name;
    private final String genderCode;

    private Person(String name, String genderCode) {
        this.name = name;
        this.genderCode = genderCode;
    }

    public static Person createMale(String name) {
        return new Person(name, "M");
    }

    public static Person createFemale(String name) {
        return new Person(name, "F");
    }

    public String getGenderCode() {
        return genderCode;
    }
}

// Usage
Person john = Person.createMale("John");
Person jane = Person.createFemale("Jane");
```

### Using Enum

```java
public enum Gender {
    MALE("M", "Mr."),
    FEMALE("F", "Ms.");

    private final String code;
    private final String title;

    Gender(String code, String title) {
        this.code = code;
        this.title = title;
    }

    public String getCode() { return code; }
    public String getTitle() { return title; }
}

public class Person {
    private String name;
    private final Gender gender;

    public Person(String name, Gender gender) {
        this.name = name;
        this.gender = gender;
    }

    public String getGenderCode() {
        return gender.getCode();
    }

    public String getFormalName() {
        return gender.getTitle() + " " + name;
    }
}
```

## When to Use

✅ **Use Remove Subclass when:**

- Subclasses only differ in constant return values
- The subclass hierarchy is over-engineered
- Subclasses have no behavior, just data
- Maintaining subclasses costs more than they're worth

❌ **Avoid Remove Subclass when:**

- Subclasses have significant behavioral differences
- New types are expected to be added
- Polymorphism provides cleaner code
- The subclass structure is well-used and understood

## Related Refactorings

- **Replace Type Code with Subclasses**: The inverse
- **Replace Subclass with Delegate**: Alternative for more behavior
- **Collapse Hierarchy**: Similar simplification
