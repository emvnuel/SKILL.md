# Encapsulate Collection

## Intent

Ensure a collection is properly encapsulated by returning a read-only view and providing methods to add/remove elements.

## Code Smells That Indicate This Refactoring

- Collection returned directly, allowing external modification
- No control over what gets added to the collection
- Collection can be replaced entirely from outside
- Inconsistent state due to uncontrolled modifications

## Mechanics

1. Apply **Encapsulate Variable** if the collection isn't already encapsulated
2. Add methods to add and remove elements from the collection
3. Run static checks
4. Find all references to the setter. Replace with add/remove calls or a copy-based assignment
5. Modify the getter to return a read-only view of the collection
6. Test

## Example

### Before

```java
public class Course {
    private String name;

    public Course(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

public class Person {
    private List<Course> courses = new ArrayList<>();

    public List<Course> getCourses() {
        return courses;
    }

    public void setCourses(List<Course> courses) {
        this.courses = courses;
    }
}

// Problematic usage
person.getCourses().add(new Course("Math"));  // Uncontrolled modification
person.setCourses(new ArrayList<>());         // Complete replacement
```

### After

```java
public class Person {
    private final List<Course> courses = new ArrayList<>();

    public List<Course> getCourses() {
        return Collections.unmodifiableList(courses);
    }

    public void addCourse(Course course) {
        courses.add(course);
    }

    public void removeCourse(Course course) {
        courses.remove(course);
    }

    public int numberOfCourses() {
        return courses.size();
    }
}

// Controlled usage
person.addCourse(new Course("Math"));
person.removeCourse(mathCourse);
```

### With Validation

```java
public class Team {
    private static final int MAX_TEAM_SIZE = 10;
    private final Set<Player> players = new HashSet<>();

    public Set<Player> getPlayers() {
        return Collections.unmodifiableSet(players);
    }

    public void addPlayer(Player player) {
        if (players.size() >= MAX_TEAM_SIZE) {
            throw new IllegalStateException("Team is full");
        }
        if (player == null) {
            throw new IllegalArgumentException("Player cannot be null");
        }
        players.add(player);
    }

    public void removePlayer(Player player) {
        if (!players.contains(player)) {
            throw new IllegalArgumentException("Player not on team");
        }
        players.remove(player);
    }

    public boolean hasPlayer(Player player) {
        return players.contains(player);
    }
}
```

### Alternative: Defensive Copy

```java
public class Department {
    private List<Employee> employees;

    public List<Employee> getEmployees() {
        return new ArrayList<>(employees);  // Returns a copy
    }

    public void setEmployees(List<Employee> employees) {
        this.employees = new ArrayList<>(employees);  // Stores a copy
    }
}
```

## When to Use

✅ **Use Encapsulate Collection when:**

- Collection can be modified from outside the class
- You need validation when adding/removing elements
- You want to enforce invariants on the collection
- Collection modifications should trigger side effects

❌ **Avoid Encapsulate Collection when:**

- The class is a simple data transfer object
- Performance overhead of copying is unacceptable
- The collection is intentionally shared for modification

## Related Refactorings

- **Encapsulate Variable**: The general case
- **Replace Primitive with Object**: May lead to collection encapsulation
- **Extract Class**: When collection operations become complex
