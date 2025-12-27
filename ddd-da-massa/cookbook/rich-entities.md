ww# Rich Entities

## Principle

> Entities should contain business logic that operates on their state, not just be data containers.

## Problem: Anemic Entities

```java
// ❌ Anemic - just data
@Entity
public class Order {
    @Id private Long id;
    private Instant expiresAt;
    @Enumerated private OrderStatus status;

    // Just getters and setters
    public Instant getExpiresAt() { return expiresAt; }
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus s) { this.status = s; }
}

// Logic scattered in services
@ApplicationScoped
public class OrderService {
    public void cancel(Order order) {
        if (order.getExpiresAt().isBefore(Instant.now())) {
            throw new OrderExpiredException();
        }
        if (order.getStatus() == OrderStatus.SHIPPED) {
            throw new CannotCancelShippedException();
        }
        order.setStatus(OrderStatus.CANCELLED);
    }
}
```

## Solution: Rich Entities

```java
@Entity
public class Order {

    @Id @GeneratedValue
    private Long id;

    private Instant expiresAt;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    protected Order() {}

    public Order(Customer customer, List<OrderItem> items) {
        // Validation in constructor
        this.status = OrderStatus.PENDING;
        this.expiresAt = Instant.now().plus(Duration.ofHours(24));
    }

    // ✓ Logic on owned state
    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }

    public boolean canBeCancelled() {
        return !isExpired() && status != OrderStatus.SHIPPED;
    }

    // ✓ State transition with rules
    public void cancel() {
        if (!canBeCancelled()) {
            throw new OrderCannotBeCancelledException(this);
        }
        this.status = OrderStatus.CANCELLED;
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new InvalidStatusTransitionException(status, OrderStatus.CONFIRMED);
        }
        this.status = OrderStatus.CONFIRMED;
    }

    public void ship() {
        if (status != OrderStatus.CONFIRMED) {
            throw new InvalidStatusTransitionException(status, OrderStatus.SHIPPED);
        }
        this.status = OrderStatus.SHIPPED;
    }
}
```

## Example: Bolao (Raffle Pool)

```java
@Entity
public class Bolao {

    @Id @GeneratedValue
    private Long id;

    private Instant expiresAt;

    @ElementCollection
    private Set<String> invitedEmails = new HashSet<>();

    @OneToMany(mappedBy = "bolao")
    private List<Participation> participations = new ArrayList<>();

    protected Bolao() {}

    public Bolao(String name, Duration duration) {
        this.expiresAt = Instant.now().plus(duration);
    }

    public void invite(String email) {
        invitedEmails.add(email.toLowerCase());
    }

    // ✓ Complex business logic in entity
    public Participation accept(User user) {
        if (expiresAt.isBefore(Instant.now())) {
            throw new InvitationExpiredException();
        }

        if (!invitedEmails.contains(user.getEmail().toLowerCase())) {
            throw new NotInvitedException(user);
        }

        boolean alreadyParticipating = participations.stream()
            .anyMatch(p -> p.getUser().equals(user));
        if (alreadyParticipating) {
            throw new AlreadyParticipatingException(user);
        }

        Participation participation = new Participation(this, user);
        participations.add(participation);
        return participation;
    }
}
```

## Service Becomes Simple

```java
@Path("/bolao/{bolaoId}/accept")
@RequestScoped
public class BolaoAcceptResource {

    @Inject private BolaoRepository bolaos;
    @Inject private UserRepository users;
    @Inject private ParticipationRepository participations;

    @POST
    @Transactional
    public Response accept(
            @PathParam("bolaoId") Long bolaoId,
            @Valid AcceptRequest request) {

        Bolao bolao = bolaos.findById(bolaoId)
            .orElseThrow(NotFoundException::new);

        User user = users.findByEmail(request.email())
            .orElseThrow(UserNotFoundException::new);

        // ✓ Entity has all the logic
        Participation participation = bolao.accept(user);
        participations.save(participation);

        return Response.ok().build();
    }
}
// Service is just orchestration, entity has rules
```

## Load Distribution Check

| Component | Points | Limit | Status |
| --------- | ------ | ----- | ------ |
| Resource  | 5      | 7     | ✓      |
| Entity    | 4      | 9     | ✓      |

Logic distributed properly across layers.

## Related Entries

- [load-distribution](load-distribution.md) - Balancing load
- [form-value-objects](form-value-objects.md) - Conversion to entities
