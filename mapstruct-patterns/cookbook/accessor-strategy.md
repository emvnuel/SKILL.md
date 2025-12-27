# Custom Accessor Naming Strategy

## The Problem

Some libraries (like jOOQ) use fluent accessors that MapStruct incorrectly identifies as setters.

```java
// jOOQ Record - fluent methods return 'this'
public class OrderRecord {
    public OrderRecord id(Long id) {
        this.id = id;
        return this;  // Fluent - returns self
    }

    public Long id() {
        return this.id;  // No 'get' prefix
    }
}
```

## Solution: Custom AccessorNamingStrategy

```java
package com.example.mapstruct.spi;

import javax.lang.model.element.ExecutableElement;
import org.mapstruct.ap.spi.DefaultAccessorNamingStrategy;

/**
 * Ignores fluent setters that return 'this'.
 */
public class IgnoreFluentAccessorNamingStrategy
    extends DefaultAccessorNamingStrategy {

    @Override
    protected boolean isFluentSetter(ExecutableElement method) {
        return false;  // Never treat methods as fluent setters
    }
}
```

## Register with SPI

Create `META-INF/services/org.mapstruct.ap.spi.AccessorNamingStrategy`:

```
com.example.mapstruct.spi.IgnoreFluentAccessorNamingStrategy
```

## Using MetaInfServices

With `org.kohsuke:metainf-services`:

```java
import org.kohsuke.MetaInfServices;
import org.mapstruct.ap.spi.AccessorNamingStrategy;

@MetaInfServices(value = AccessorNamingStrategy.class)
public class IgnoreFluentAccessorNamingStrategy
    extends DefaultAccessorNamingStrategy {

    @Override
    protected boolean isFluentSetter(ExecutableElement method) {
        return false;
    }
}
```

## Custom Getter Recognition

For non-standard getter patterns:

```java
public class RecordStyleAccessorNamingStrategy
    extends DefaultAccessorNamingStrategy {

    @Override
    public boolean isGetterMethod(ExecutableElement method) {
        String methodName = method.getSimpleName().toString();

        // Recognize record-style accessors: name() instead of getName()
        if (method.getParameters().isEmpty()
            && !methodName.startsWith("get")
            && !methodName.equals("hashCode")
            && !methodName.equals("toString")) {
            return true;
        }

        return super.isGetterMethod(method);
    }

    @Override
    public String getPropertyName(ExecutableElement method) {
        String methodName = method.getSimpleName().toString();

        // For record-style: name() -> "name"
        if (!methodName.startsWith("get") && !methodName.startsWith("is")) {
            return methodName;
        }

        return super.getPropertyName(method);
    }
}
```

## Builder Recognition

```java
public class CustomBuilderProvider implements BuilderProvider {

    @Override
    public BuilderInfo findBuilderInfo(TypeElement typeElement) {
        // Custom builder detection logic
        // Return null if no builder, or BuilderInfo with builder class
        return null;
    }
}
```

## Related Entries

- [jooq-integration](jooq-integration.md) - jOOQ specific patterns
- [cdi-setup](cdi-setup.md) - Basic setup
