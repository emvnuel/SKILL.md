# Command Pattern - Jakarta EE Cookbook

## Intent

Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

## Jakarta EE Implementation

### 1. Command Interface

```java
public interface Command<T> {
    T execute();
    void undo();
    String getDescription();
}
```

### 2. Concrete Commands

```java
public class CreateUserCommand implements Command<User> {

    private final UserService userService;
    private final CreateUserRequest request;
    private User createdUser;

    public CreateUserCommand(UserService userService, CreateUserRequest request) {
        this.userService = userService;
        this.request = request;
    }

    @Override
    public User execute() {
        createdUser = userService.create(request);
        return createdUser;
    }

    @Override
    public void undo() {
        if (createdUser != null) {
            userService.delete(createdUser.getId());
        }
    }

    @Override
    public String getDescription() {
        return "Create user: " + request.getEmail();
    }
}

public class UpdateUserCommand implements Command<User> {

    private final UserService userService;
    private final Long userId;
    private final UpdateUserRequest request;
    private User previousState;

    public UpdateUserCommand(UserService userService, Long userId, UpdateUserRequest request) {
        this.userService = userService;
        this.userId = userId;
        this.request = request;
    }

    @Override
    public User execute() {
        previousState = userService.findById(userId);
        return userService.update(userId, request);
    }

    @Override
    public void undo() {
        if (previousState != null) {
            userService.update(userId, toRequest(previousState));
        }
    }

    @Override
    public String getDescription() {
        return "Update user: " + userId;
    }
}
```

### 3. Command Executor (Invoker)

```java
@ApplicationScoped
public class CommandExecutor {

    private final Deque<Command<?>> history = new ConcurrentLinkedDeque<>();

    @Inject
    AuditService auditService;

    public <T> T execute(Command<T> command) {
        T result = command.execute();
        history.push(command);
        auditService.log(command.getDescription());
        return result;
    }

    public void undo() {
        if (!history.isEmpty()) {
            Command<?> command = history.pop();
            command.undo();
            auditService.log("UNDO: " + command.getDescription());
        }
    }

    public List<String> getHistory() {
        return history.stream()
            .map(Command::getDescription)
            .toList();
    }
}
```

### 4. Usage

```java
@Path("/users")
@ApplicationScoped
public class UserResource {

    @Inject
    UserService userService;

    @Inject
    CommandExecutor executor;

    @POST
    public Response createUser(CreateUserRequest request) {
        Command<User> command = new CreateUserCommand(userService, request);
        User user = executor.execute(command);
        return Response.created(URI.create("/users/" + user.getId())).entity(user).build();
    }

    @PUT
    @Path("/{id}")
    public User updateUser(@PathParam("id") Long id, UpdateUserRequest request) {
        return executor.execute(new UpdateUserCommand(userService, id, request));
    }

    @POST
    @Path("/undo")
    public Response undo() {
        executor.undo();
        return Response.ok().build();
    }
}
```

### 5. Batch Command (Macro)

```java
public class BatchCommand implements Command<List<Object>> {

    private final List<Command<?>> commands;
    private final List<Command<?>> executed = new ArrayList<>();

    public BatchCommand(List<Command<?>> commands) {
        this.commands = commands;
    }

    @Override
    public List<Object> execute() {
        List<Object> results = new ArrayList<>();
        for (Command<?> command : commands) {
            results.add(command.execute());
            executed.add(command);
        }
        return results;
    }

    @Override
    public void undo() {
        // Undo in reverse order
        for (int i = executed.size() - 1; i >= 0; i--) {
            executed.get(i).undo();
        }
    }

    @Override
    public String getDescription() {
        return "Batch of " + commands.size() + " commands";
    }
}
```

## When to Use

✅ **Use Command when:**

- You need undo/redo functionality
- You want to queue, log, or schedule operations
- You need to support transactional behavior across operations

❌ **Avoid Command when:**

- Simple CRUD without undo requirements
- Operations are trivial and don't need encapsulation
