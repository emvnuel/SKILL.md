# @EqualsAndHashCode with JPA

## Intent

Implement correct entity identity using Lombok while avoiding common JPA pitfalls.

## The Problem

Default `@EqualsAndHashCode` includes all fields, which breaks JPA entities:

```java
// ❌ WRONG - Don't do this
@Data  // Includes @EqualsAndHashCode on all fields
@Entity
public class Product {
    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;  // Triggers lazy loading in equals()!
}
```

### Issues:

1. **ID is null before persist**: hashCode changes after save
2. **Lazy loading**: Accessing relationships triggers database queries
3. **Collections break**: HashSet/HashMap fail when hashCode changes

## Recommended Solution

### Business Key (Immutable Natural Key)

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @Id
    @GeneratedValue
    private Long id;

    @EqualsAndHashCode.Include  // Use immutable business key
    @Column(unique = true, updatable = false)
    private String email;

    private String name;
}
```

### ID-Based (After Persist Only)

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Order {
    @Id
    @GeneratedValue
    @EqualsAndHashCode.Include
    private Long id;

    private String description;
}
```

⚠️ **Warning**: ID is null before `persist()`, so new entities are all "equal".

### UUID (Best for Distributed Systems)

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Document {
    @Id
    @GeneratedValue
    private Long id;

    @EqualsAndHashCode.Include
    @Column(unique = true, updatable = false)
    private UUID uuid = UUID.randomUUID();  // Assigned immediately

    private String title;
}
```

## Excluding Fields

```java
@Entity
@Getter
@Setter
@EqualsAndHashCode(exclude = {"version", "audit", "orders"})
public class Customer {
    @Id
    private Long id;

    @Version
    private Long version;  // Should not affect equality

    @Embedded
    private AuditInfo audit;  // Metadata, not identity

    @OneToMany(mappedBy = "customer")
    private List<Order> orders;  // Lazy, would trigger loading
}
```

## Inheritance

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@Getter
@Setter
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public abstract class Vehicle {
    @Id
    @GeneratedValue
    @EqualsAndHashCode.Include
    private Long id;
}

@Entity
@Getter
@Setter
@EqualsAndHashCode(callSuper = true, onlyExplicitlyIncluded = true)
public class Car extends Vehicle {
    @EqualsAndHashCode.Include
    private String licensePlate;
}
```

## Decision Matrix

| Strategy     | Pros               | Cons                       |
| ------------ | ------------------ | -------------------------- |
| Business Key | Stable, meaningful | Need immutable natural key |
| ID Only      | Simple             | Null before persist        |
| UUID         | Always stable      | Extra column, storage      |

## Best Practices

1. **Always use `onlyExplicitlyIncluded = true`** for entities
2. **Never include lazy relationships** in equals/hashCode
3. **Choose a stable identifier** (business key or UUID)
4. **Use `callSuper = true`** for inheritance hierarchies
5. **Test equality behavior** before and after persist

## Related Entries

- [entity-patterns](entity-patterns.md) - Complete JPA entity setup
