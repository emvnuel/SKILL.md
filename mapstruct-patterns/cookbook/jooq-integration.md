# jOOQ Integration

## The Challenge

jOOQ generates records with:

- Fluent setters (return `this`)
- Record-style getters (no `get` prefix)
- All-args constructor

## Step 1: Disable Fluent Setters

```java
import org.kohsuke.MetaInfServices;
import org.mapstruct.ap.spi.AccessorNamingStrategy;
import org.mapstruct.ap.spi.DefaultAccessorNamingStrategy;

@MetaInfServices(value = AccessorNamingStrategy.class)
public class JooqAccessorNamingStrategy
    extends DefaultAccessorNamingStrategy {

    @Override
    protected boolean isFluentSetter(ExecutableElement method) {
        return false;  // jOOQ fluent methods are NOT setters
    }
}
```

## Step 2: Add @Default to jOOQ Constructor

jOOQ generates records with multiple constructors. Use a jOOQ generator strategy to add `@Default`:

```java
package com.example.jooq;

import org.jooq.codegen.DefaultGeneratorStrategy;
import org.jooq.codegen.GeneratorStrategy;
import org.jooq.meta.Definition;

public class MapStructGeneratorStrategy extends DefaultGeneratorStrategy {

    @Override
    public List<String> getJavaClassImplements(Definition definition, Mode mode) {
        // Add @Default annotation import
        return super.getJavaClassImplements(definition, mode);
    }
}
```

Or patch via Gradle/Maven plugin to add annotation to all-args constructor.

## Step 3: Mapper Interface

```java
@Mapper(componentModel = "cdi")
public interface OrderRecordMapper {

    // jOOQ Record to Domain
    Order toDomain(OrderRecord record);

    // Domain to jOOQ Record
    OrderRecord toRecord(Order order);
}
```

## Example: Complete jOOQ Mapping

```java
// jOOQ generated record (simplified)
public class OrderRecord extends UpdatableRecordImpl<OrderRecord> {

    private Long id;
    private String customerId;
    private BigDecimal total;

    // jOOQ all-args constructor - add @Default here
    @Default
    public OrderRecord(Long id, String customerId, BigDecimal total) {
        this.id = id;
        this.customerId = customerId;
        this.total = total;
    }

    // Fluent accessors
    public Long id() { return id; }
    public OrderRecord id(Long id) { this.id = id; return this; }
}

// Domain entity
public record Order(Long id, String customerId, BigDecimal total) {}

// Mapper
@Mapper(componentModel = "cdi")
public interface OrderMapper {
    Order toDomain(OrderRecord record);
    OrderRecord toRecord(Order order);
}
```

## Generated Code

```java
public class OrderMapperImpl implements OrderMapper {

    public Order toDomain(OrderRecord record) {
        if (record == null) return null;
        return new Order(
            record.id(),
            record.customerId(),
            record.total()
        );
    }

    public OrderRecord toRecord(Order order) {
        if (order == null) return null;
        return new OrderRecord(
            order.id(),
            order.customerId(),
            order.total()
        );  // Uses constructor = compile-safe
    }
}
```

## Package-Info Hack for @Default

If you can't modify jOOQ generation, use package-info:

```java
// In package-info.java where jOOQ records are generated
@org.mapstruct.extensions.jooq.JooqMappingDefault
package com.example.generated.jooq.tables.records;
```

See: [GitHub Gist for jOOQ @Default](https://gist.github.com/agentgt/0a7484ec8820446c2970d8b2af527bb9)

## Related Entries

- [accessor-strategy](accessor-strategy.md) - Custom accessor naming
- [constructor-mapping](constructor-mapping.md) - Constructor patterns
