---
name: quarkus-panache-smells
description: Detects and refactors ORM code smells in Quarkus Panache applications. Use when reviewing PanacheEntity/Repository code, diagnosing N+1 queries, data overfetching, or pagination issues.
---

# Quarkus Panache ORM Code Smells Detection

Identify and fix common ORM anti-patterns in Quarkus Panache applications (Active Record or Repository patterns).

## Code Smell Categories

### 1. Eager Fetching at Class Level

**Problem**: `FetchType.EAGER` on relationships always loads related entities.

**Detection**:

```java
// BAD: Always loads Person even when not needed
@ManyToOne(fetch = FetchType.EAGER)
public Person owner;

// BAD: @ManyToOne/@OneToOne default to EAGER
@ManyToOne  // implicitly EAGER
public Person owner;
```

**Refactoring**:

```java
// GOOD: Explicitly LAZY
@ManyToOne(fetch = FetchType.LAZY)
public Person owner;
```

---

### 2. Using listAll() Without Pagination

**Problem**: `listAll()` or `streamAll()` on large tables causes memory issues.

**Detection**:

```java
// BAD: Loads entire table into memory
List<Person> all = Person.listAll();

// BAD: Even with stream, still fetches all
Stream<Person> stream = Person.streamAll();
```

**Refactoring**:

```java
// GOOD: Use PanacheQuery with pagination
PanacheQuery<Person> query = Person.findAll();
query.page(Page.ofSize(25));
List<Person> page = query.list();

// Or with range
List<Person> range = Person.findAll().range(0, 24).list();
```

---

### 3. Missing Projections for Read-Only Queries

**Problem**: Fetching entire entities when only specific fields are needed.

**Detection**:

```java
// BAD: Loads all columns including BLOBs
List<Person> persons = Person.list("status", Status.Alive);
// Then only uses person.name
```

**Refactoring**:

```java
// GOOD: Use DTO projection
@RegisterForReflection
public class PersonName {
    public final String name;
    public PersonName(String name) {
        this.name = name;
    }
}

// Only 'name' column loaded from database
List<PersonName> names = Person.find("status", Status.Alive)
    .project(PersonName.class)
    .list();
```

**With related entity fields**:

```java
@RegisterForReflection
public class DogDto {
    public String name;
    public String ownerName;

    public DogDto(String name,
                  @ProjectedFieldName("owner.name") String ownerName) {
        this.name = name;
        this.ownerName = ownerName;
    }
}

List<DogDto> dogs = Dog.findAll().project(DogDto.class).list();
```

---

### 4. N+1 Problem: Lazy Access in Loop

**Problem**: Accessing LAZY relationships inside loops triggers N queries.

**Detection**:

```java
// BAD: Each iteration triggers a query
List<Person> persons = Person.listAll();
for (Person p : persons) {
    p.address.city;  // N+1 queries!
}
```

**Refactoring**:

```java
// Option 1: Use @BatchSize on relationship
@OneToMany(fetch = FetchType.LAZY)
@BatchSize(size = 25)
public List<Order> orders;

// Option 2: JOIN FETCH in query
List<Person> persons = Person.find(
    "FROM Person p LEFT JOIN FETCH p.address"
).list();

// Option 3: Use EntityGraph
@EntityGraph(attributePaths = {"address"})
List<Person> findAllWithAddress();
```

---

### 5. N+1 Problem: Missing JOIN FETCH for Eager Relations

**Problem**: Queries on entities with EAGER relationships generate N additional queries.

**Detection**:

```java
// Entity has @ManyToOne(fetch = FetchType.EAGER)
// Query doesn't use JOIN FETCH
List<Dog> dogs = Dog.list("breed", "Labrador");  // N+1!
```

**Refactoring**:

```java
// GOOD: Explicit JOIN FETCH
List<Dog> dogs = Dog.find(
    "FROM Dog d JOIN FETCH d.owner WHERE d.breed = ?1",
    "Labrador"
).list();
```

---

### 6. Unidirectional @OneToMany with List

**Problem**: Adding/removing elements causes DELETE ALL + N INSERTs.

**Detection**:

```java
// BAD: Unidirectional with List
@Entity
public class Person extends PanacheEntity {
    @OneToMany(cascade = CascadeType.ALL)
    public List<Order> orders = new ArrayList<>();
}
```

**Refactoring**:

```java
// Option 1: Make bidirectional
@Entity
public class Person extends PanacheEntity {
    @OneToMany(mappedBy = "person", cascade = CascadeType.ALL)
    public List<Order> orders = new ArrayList<>();
}

@Entity
public class Order extends PanacheEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    public Person person;
}

// Option 2: Use Set (single INSERT per operation)
@OneToMany(cascade = CascadeType.ALL)
public Set<Order> orders = new HashSet<>();
```

---

### 7. Not Closing Streams

**Problem**: Unclosed streams hold database connections.

**Detection**:

```java
// BAD: Stream not closed, leaks connection
Stream<Person> persons = Person.streamAll();
persons.map(p -> p.name).collect(toList());
```

**Refactoring**:

```java
// GOOD: Use try-with-resources
try (Stream<Person> persons = Person.streamAll()) {
    return persons.map(p -> p.name).collect(toList());
}
```

---

### 8. Ignoring Panache Query Methods

**Problem**: Writing verbose HQL when Panache shortcuts exist.

**Detection**:

```java
// BAD: Verbose HQL for simple queries
Person.find("SELECT p FROM Person p WHERE p.status = ?1", Status.Alive);
```

**Refactoring**:

```java
// GOOD: Panache simplified query
Person.find("status", Status.Alive);

// GOOD: Multiple parameters
Person.find("status = ?1 and name = ?2", Status.Alive, "John");

// GOOD: Named parameters
Person.find("status = :status",
    Parameters.with("status", Status.Alive));
```

## Quick Reference Table

| Smell               | Detection                                      | Fix                          |
| ------------------- | ---------------------------------------------- | ---------------------------- |
| Eager class-level   | `FetchType.EAGER` or missing LAZY on `@*ToOne` | Add `FetchType.LAZY`         |
| No pagination       | `listAll()` / `streamAll()` on large tables    | Use `find().page()`          |
| No projection       | Full entity for read-only                      | Use `.project(Dto.class)`    |
| N+1 in loop         | LAZY access in `for` loop                      | `@BatchSize` or `JOIN FETCH` |
| N+1 Eager           | Query without FETCH for EAGER relations        | Add `JOIN FETCH`             |
| Unidirectional List | `@OneToMany` without `mappedBy` + List         | Use Set or bidirectional     |
| Unclosed stream     | `stream()` without try-with-resources          | Wrap in `try()`              |
| Verbose HQL         | Full `SELECT` for simple queries               | Use Panache shortcuts        |
| Missing reflection  | DTO without `@RegisterForReflection`           | Add annotation               |

## Panache-Specific Best Practices

1. **Prefer Active Record** for simple CRUD, **Repository** for complex queries
2. **Use `find()` over `list()`** when you need pagination/projection
3. **Configure fetch batch size** globally: `quarkus.hibernate-orm.fetch.batch-size=25`
4. **Use `@Transactional` on write operations** in REST resources
5. **Consider reactive Panache** (`PanacheEntityReactive`) for non-blocking I/O
