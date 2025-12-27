# JPA Entity Mapping

## Entity to DTO

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;

    private BigDecimal total;
    private OrderStatus status;
    private Instant createdAt;

    // Getters...
}

// Response DTO - Record for compile safety
public record OrderResponse(
    String orderId,
    String customerName,
    List<OrderItemResponse> items,
    String total,
    String status,
    String createdAt
) {}

@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "orderId", source = "id")
    @Mapping(target = "customerName", source = "customer.name")
    @Mapping(target = "total", expression = "java(order.getTotal().toString())")
    @Mapping(target = "status", expression = "java(order.getStatus().name())")
    @Mapping(target = "createdAt", expression = "java(order.getCreatedAt().toString())")
    OrderResponse toResponse(Order order);

    OrderItemResponse toResponse(OrderItem item);
}
```

## DTO to Entity (with @Default)

```java
// Request DTO
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<OrderItemRequest> items
) {}

// Entity with @Default
@Entity
public class Order {

    @Id @GeneratedValue
    private Long id;

    private String customerId;
    private BigDecimal total;
    private OrderStatus status;
    private Instant createdAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();

    protected Order() {}  // JPA

    @Default  // MapStruct uses this
    public Order(String customerId, List<OrderItem> items) {
        this.customerId = customerId;
        this.items = items;
        this.total = calculateTotal(items);
        this.status = OrderStatus.PENDING;
        this.createdAt = Instant.now();
    }
}

@Mapper(componentModel = "cdi", uses = OrderItemMapper.class)
public interface OrderMapper {

    @Mapping(target = "customerId", source = "customerId")
    @Mapping(target = "items", source = "items")
    Order toEntity(CreateOrderRequest request);
}
```

## Ignoring Generated Fields

```java
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    // ID, createdAt, updatedAt are set by JPA/entity
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Order toEntity(CreateOrderRequest request);
}
```

## Updating Existing Entity

```java
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    // Updates existing entity, ignores null values
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    void updateEntity(UpdateOrderRequest request, @MappingTarget Order order);
}

// Usage
public Order update(Long id, UpdateOrderRequest request) {
    Order order = repository.findById(id).orElseThrow();
    orderMapper.updateEntity(request, order);
    return repository.save(order);
}
```

## Lazy Loading Considerations

```java
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    // Only map if relationship is already loaded
    @Mapping(target = "customerName",
             expression = "java(Hibernate.isInitialized(order.getCustomer()) ? order.getCustomer().getName() : null)")
    OrderResponse toResponse(Order order);
}

// Or use qualifiers
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "customer", qualifiedByName = "safeLazy")
    OrderResponse toResponse(Order order);

    @Named("safeLazy")
    default CustomerResponse mapCustomer(Customer customer) {
        if (customer == null || !Hibernate.isInitialized(customer)) {
            return null;
        }
        return new CustomerResponse(customer.getId(), customer.getName());
    }
}
```

## Related Entries

- [default-annotation](default-annotation.md) - @Default for entities
- [constructor-mapping](constructor-mapping.md) - Constructor patterns
