# CDI/MicroProfile Setup

## Maven Dependencies

```xml
<properties>
    <mapstruct.version>1.6.3</mapstruct.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${mapstruct.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## CDI Component Model

```java
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    OrderResponse toResponse(Order order);
    Order toEntity(CreateOrderRequest request);
}
```

Generated code:

```java
@ApplicationScoped
public class OrderMapperImpl implements OrderMapper {
    // ...
}
```

## Injecting Mappers

```java
@ApplicationScoped
public class OrderService {

    @Inject
    private OrderMapper orderMapper;

    public OrderResponse getOrder(Long id) {
        Order order = repository.findById(id).orElseThrow();
        return orderMapper.toResponse(order);
    }
}
```

## Injecting Other Mappers

```java
@Mapper(componentModel = "cdi", uses = {AddressMapper.class, CustomerMapper.class})
public interface OrderMapper {

    OrderResponse toResponse(Order order);
    // AddressMapper and CustomerMapper automatically used for nested objects
}
```

## Constructor Injection Strategy

```java
@Mapper(
    componentModel = "cdi",
    injectionStrategy = InjectionStrategy.CONSTRUCTOR  // Prefer constructor injection
)
public interface OrderMapper {
    OrderResponse toResponse(Order order);
}
```

Generated:

```java
@ApplicationScoped
public class OrderMapperImpl implements OrderMapper {

    private final AddressMapper addressMapper;

    @Inject
    public OrderMapperImpl(AddressMapper addressMapper) {
        this.addressMapper = addressMapper;
    }
}
```

## Quarkus Configuration

Quarkus automatically detects MapStruct. No additional config needed.

```java
// Works out of the box in Quarkus
@Path("/orders")
@ApplicationScoped
public class OrderResource {

    @Inject
    OrderMapper mapper;

    @GET
    @Path("/{id}")
    public OrderResponse get(@PathParam("id") Long id) {
        return mapper.toResponse(orderService.findById(id));
    }
}
```

## Related Entries

- [constructor-mapping](constructor-mapping.md) - Constructor-based mapping
- [default-annotation](default-annotation.md) - Custom @Default annotation
