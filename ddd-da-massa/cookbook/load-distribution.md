# Load Distribution

## Principle

> If calling code has HIGH load but entity has LOW load, you have bad distribution. Move logic to where it belongs.

## Detecting Bad Distribution

### Signal: High Load + Low Entity Load

```java
// ❌ Controller has 8+ points, Entity has 2 points
@POST
public Response accept(@Valid AcceptRequest request) {
    Bolao bolao = bolaoRepo.findById(request.bolaoId()).get();  // +1
    User user = userRepo.findByEmail(request.email()).get();     // +1

    // All this should be in Bolao
    if (bolao.getExpiresAt().isBefore(Instant.now())) {          // +1
        return Response.status(422).entity("Expired").build();
    }
    if (!bolao.getInvitedEmails().contains(request.email())) {   // +1
        return Response.status(422).entity("Not invited").build();
    }
    if (bolao.getParticipations().stream()                       // +1
            .anyMatch(p -> p.getUser().equals(user))) {          // +1
        return Response.status(422).entity("Already joined").build();
    }

    Participation p = new Participation(bolao, user);            // +1
    participationRepo.save(p);
    return Response.ok().build();
}
// Total: 8+ points in controller, ~2 in Entity = BAD
```

### Solution: Move Logic to Entity

```java
// ✓ Controller has 4 points, Entity has 4 points
@POST
public Response accept(@Valid AcceptRequest request) {
    Bolao bolao = bolaoRepo.findById(request.bolaoId())
        .orElseThrow(NotFoundException::new);  // +1
    User user = userRepo.findByEmail(request.email())
        .orElseThrow(UserNotFoundException::new);  // +1

    Participation p = bolao.accept(user);  // +1 - Entity has the logic
    participationRepo.save(p);

    return Response.ok().build();
}
// Total: 4 points in controller ✓

// Entity
public Participation accept(User user) {
    if (expiresAt.isBefore(Instant.now())) {  // +1
        throw new InvitationExpiredException();
    }
    if (!invitedEmails.contains(user.getEmail())) {  // +1
        throw new NotInvitedException(user);
    }
    if (hasParticipation(user)) {  // +1
        throw new AlreadyParticipatingException(user);
    }
    return new Participation(this, user);  // +1
}
// Total: 4 points in Entity ✓
```

## Target Distribution

| Component  | Target Load | Current | Status |
| ---------- | ----------- | ------- | ------ |
| Controller | 4-7         | 4       | ✓      |
| Form/DTO   | 4-9         | 3       | ✓      |
| Entity     | 4-9         | 4       | ✓      |
| Service    | 4-7         | 5       | ✓      |

**Balanced** = Each component near middle of its range.

## Red Flags

| Signal                         | Problem             | Solution                      |
| ------------------------------ | ------------------- | ----------------------------- |
| Entity 2 pts, Controller 8 pts | Anemic entity       | Move logic to entity          |
| All entities 9 pts             | Overloaded entities | Extract value objects         |
| Service 2 pts                  | Unnecessary layer   | Use domain-service-controller |
| DTO 9 pts, Entity 2 pts        | DTO doing too much  | Move to entity                |

## Visualization

```
Balanced Distribution:

Controller [====    ] 4/7
DTO        [=====   ] 5/9
Entity     [======  ] 6/9
Service    [=====   ] 5/7
Repository [==      ] 2/3

Unbalanced Distribution:

Controller [========X] 8/7  ← Over limit!
DTO        [===      ] 3/9
Entity     [==       ] 2/9  ← Anemic!
Service    [=        ] 1/7  ← Unnecessary?
Repository [==       ] 2/3
```

## Action Checklist

When analyzing a method:

1. Count cognitive load points
2. Check entity load used by this method
3. If caller HIGH + entity LOW → move logic
4. If caller over limit → extract abstraction
5. If entity over limit → extract value object

## Related Entries

- [rich-entities](rich-entities.md) - Entities with behavior
- [splitting-load](splitting-load.md) - When to extract
