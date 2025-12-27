# JPA Entity Patterns

## Intent

Use Lombok effectively with JPA entities while avoiding common pitfalls.

## Recommended Pattern

```java
@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@ToString(exclude = {"category", "orderItems"})
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @EqualsAndHashCode.Include
    private Long id;

    @Column(nullable = false)
    private String name;

    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;

    @OneToMany(mappedBy = "product")
    private List<OrderItem> orderItems = new ArrayList<>();
}
```

## Why Not @Data?

`@Data` generates methods that cause problems with JPA:

| @Data Feature                         | JPA Issue                   |
| ------------------------------------- | --------------------------- |
| Setters on all fields                 | ID shouldn't be settable    |
| `equals()`/`hashCode()` on all fields | Breaks with lazy loading    |
| `toString()` on all fields            | LazyInitializationException |

## Constructor Requirements

```java
@Entity
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA requirement
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    private String customerName;

    // Business constructor
    public Order(String customerName) {
        this.customerName = customerName;
    }
}
```

## Relationship Handling

### Exclude Lazy Relationships from ToString

```java
@Entity
@Getter
@Setter
@ToString(exclude = "orders")  // Prevent LazyInitializationException
public class Customer {
    @Id
    private Long id;

    private String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

### Bidirectional Relationships

```java
@Entity
@Getter
@Setter
@ToString(exclude = "parent")
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Category {
    @Id
    @EqualsAndHashCode.Include
    private Long id;

    private String name;

    @ManyToOne
    private Category parent;

    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();
}
```

## Entity with Builder

```java
@Entity
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Builder
public class Article {
    @Id
    @GeneratedValue
    private Long id;

    private String title;
    private String content;

    @Builder.Default
    private LocalDateTime createdAt = LocalDateTime.now();
}

// Usage
Article article = Article.builder()
    .title("Lombok with JPA")
    .content("...")
    .build();
```

## Best Practices Summary

| Do ✅                                               | Don't ❌                  |
| --------------------------------------------------- | ------------------------- |
| `@Getter` `@Setter` separately                      | `@Data` on entities       |
| `@NoArgsConstructor(access = PROTECTED)`            | Public no-arg constructor |
| `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` | Default equals/hashCode   |
| `@ToString(exclude = "lazyFields")`                 | Include lazy associations |
| ID-based equality                                   | All-fields equality       |

## Related Entries

- [equals-hashcode-jpa](equals-hashcode-jpa.md) - Deep dive into entity equality
- [data-annotation](data-annotation.md) - When to use (not for entities)
