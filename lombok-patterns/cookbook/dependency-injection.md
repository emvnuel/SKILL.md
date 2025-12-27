# Dependency Injection with Lombok

## Intent

Use `@RequiredArgsConstructor` for clean constructor-based dependency injection in Spring applications.

## Basic Pattern

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        paymentService.process(order);
        notificationService.sendConfirmation(order);
        return order;
    }
}

// Lombok generates:
// public OrderService(OrderRepository orderRepository,
//                     PaymentService paymentService,
//                     NotificationService notificationService) {
//     this.orderRepository = orderRepository;
//     this.paymentService = paymentService;
//     this.notificationService = notificationService;
// }
```

## Why Constructor Injection?

| Field Injection     | Constructor Injection |
| ------------------- | --------------------- |
| Hidden dependencies | Explicit dependencies |
| Mutable references  | Immutable (final)     |
| Hard to test        | Easy to mock          |
| NPE possible        | Fail-fast validation  |

```java
// ❌ Field injection - avoid
@Service
public class BadService {
    @Autowired
    private Repository repository;  // Not final, can be null
}

// ✅ Constructor injection with Lombok
@Service
@RequiredArgsConstructor
public class GoodService {
    private final Repository repository;  // Final, guaranteed
}
```

## Complete Spring Service Pattern

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;

    public User createUser(CreateUserRequest request) {
        log.info("Creating user: {}", request.getEmail());

        User user = User.builder()
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .build();

        User saved = userRepository.save(user);
        emailService.sendWelcome(saved);

        return saved;
    }
}
```

## With Optional Dependencies

```java
@Service
@RequiredArgsConstructor
public class NotificationService {
    private final EmailService emailService;
    private final UserRepository userRepository;

    // Optional - not included in constructor
    private SmsService smsService;

    @Autowired(required = false)
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }
}
```

## REST Controller Pattern

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Slf4j
public class OrderController {
    private final OrderService orderService;
    private final OrderMapper orderMapper;

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderRequest request) {
        log.info("Creating order: {}", request);
        Order order = orderService.create(request);
        return ResponseEntity.ok(orderMapper.toResponse(order));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> findById(@PathVariable Long id) {
        return orderService.findById(id)
            .map(orderMapper::toResponse)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

## Testing

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentService paymentService;

    @InjectMocks
    private OrderService orderService;  // Mocks injected via constructor

    @Test
    void shouldCreateOrder() {
        // Arrange
        when(orderRepository.save(any())).thenReturn(new Order());

        // Act
        Order result = orderService.createOrder(new OrderRequest());

        // Assert
        assertThat(result).isNotNull();
        verify(paymentService).process(any());
    }
}
```

## Best Practices

| Do ✅                        | Don't ❌                          |
| ---------------------------- | --------------------------------- |
| Use `final` for dependencies | Field injection with `@Autowired` |
| `@RequiredArgsConstructor`   | Manual constructor writing        |
| Single responsibility        | Too many dependencies (>5)        |
| Interface types              | Concrete implementations          |

## Related Annotations

- **@Slf4j**: Add logging to services
- **@NonNull**: Mark non-optional dependencies
