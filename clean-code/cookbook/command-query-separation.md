# Command Query Separation - Clean Code Cookbook

## Intent

Functions should either do something (command) or answer something (query), but not both. A command changes state, a query returns data.

---

## The Principle

| Type    | Returns | Modifies State | Example            |
| ------- | ------- | -------------- | ------------------ |
| Command | void    | Yes            | `saveOrder(order)` |
| Query   | value   | No             | `getOrder(id)`     |

---

## Violation Example

### Before - Mixed Command and Query

```java
public boolean set(String attribute, String value) {
    // Sets AND returns - confusing
    if (attributeExists(attribute)) {
        attributes.put(attribute, value);
        return true;
    }
    return false;
}

// Caller - confusing condition
if (set("username", "john")) {
    // What does true mean? Was it set? Did it exist?
}
```

### After - Separated

```java
// Query - only asks question
public boolean attributeExists(String attribute) {
    return attributes.containsKey(attribute);
}

// Command - only changes state
public void setAttribute(String attribute, String value) {
    if (!attributeExists(attribute)) {
        throw new UnknownAttributeException(attribute);
    }
    attributes.put(attribute, value);
}

// Caller - clear intent
if (attributeExists("username")) {
    setAttribute("username", "john");
}
```

---

## Stack Operations Example

### Before - Pop Returns AND Modifies

```java
public T pop() {
    // Both removes (command) and returns (query)
    if (isEmpty()) {
        throw new EmptyStackException();
    }
    T item = elements.remove(elements.size() - 1);
    return item;
}
```

### After - Separated

```java
// Query - peek at top without removing
public T peek() {
    if (isEmpty()) {
        throw new EmptyStackException();
    }
    return elements.get(elements.size() - 1);
}

// Command - remove top element
public void remove() {
    if (isEmpty()) {
        throw new EmptyStackException();
    }
    elements.remove(elements.size() - 1);
}

// Usage
T item = stack.peek();
stack.remove();

// Or combined when convenient
public T popAndReturn() {
    T item = peek();
    remove();
    return item;
}
```

---

## Database Operations

### Before - Mixed

```java
public Order saveAndReturn(Order order) {
    // Saves to DB and returns the saved entity
    return entityManager.merge(order);
}

public Customer findOrCreate(String email) {
    // Queries AND potentially creates - side effect hidden in query name
    Customer customer = repository.findByEmail(email);
    if (customer == null) {
        customer = new Customer(email);
        repository.save(customer);
    }
    return customer;
}
```

### After - Separated

```java
// Command
public void save(Order order) {
    entityManager.persist(order);
}

// Query
public Order findById(Long id) {
    return entityManager.find(Order.class, id);
}

// When you need the ID after save, use a callback or return type
public Long save(Order order) {
    entityManager.persist(order);
    return order.getId();  // ID is populated by persist - acceptable
}

// Explicit separate methods
public Optional<Customer> findByEmail(String email) {
    // Pure query
    return repository.findByEmail(email);
}

public Customer createCustomer(String email) {
    // Pure command (with return for generated ID)
    Customer customer = new Customer(email);
    repository.save(customer);
    return customer;
}

// Caller decides
Customer customer = findByEmail(email)
    .orElseGet(() -> createCustomer(email));
```

---

## Event Sourcing Approach

Commands generate events, queries read state:

```java
// Command - generates event
public class PlaceOrderCommand {
    private final String customerId;
    private final List<OrderItem> items;

    public OrderPlacedEvent execute(OrderService service) {
        // Validation
        service.validateCustomer(customerId);
        service.validateInventory(items);

        // Generate event (no return of aggregate)
        return new OrderPlacedEvent(
            UUID.randomUUID(),
            customerId,
            items,
            Instant.now()
        );
    }
}

// Query - reads current state
public class GetOrderQuery {
    private final String orderId;

    public Order execute(OrderRepository repository) {
        return repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

---

## REST API Design

Commands use POST/PUT/DELETE, queries use GET:

```java
@Path("/orders")
@ApplicationScoped
public class OrderResource {

    // QUERY - GET returns data, no side effects
    @GET
    @Path("/{id}")
    public OrderDTO getOrder(@PathParam("id") Long id) {
        return orderService.findById(id)
            .map(orderMapper::toDTO)
            .orElseThrow(NotFoundException::new);
    }

    // COMMAND - POST creates, returns location
    @POST
    public Response createOrder(CreateOrderRequest request) {
        Order order = orderService.createOrder(request);
        URI location = uriInfo.getAbsolutePathBuilder()
            .path(order.getId().toString())
            .build();
        return Response.created(location).build();
    }

    // COMMAND - PUT updates, returns 204 No Content
    @PUT
    @Path("/{id}")
    public Response updateOrder(@PathParam("id") Long id, UpdateOrderRequest request) {
        orderService.updateOrder(id, request);
        return Response.noContent().build();
    }

    // COMMAND - DELETE removes, returns 204 No Content
    @DELETE
    @Path("/{id}")
    public Response deleteOrder(@PathParam("id") Long id) {
        orderService.deleteOrder(id);
        return Response.noContent().build();
    }
}
```

---

## Acceptable Exceptions

Some cases where mixing is pragmatic:

```java
// 1. Fluent builders - return this
public OrderBuilder withItem(Item item) {
    this.items.add(item);
    return this;  // Returns for chaining, state mutated
}

// 2. Generated IDs after insert
public Long save(Entity entity) {
    entityManager.persist(entity);
    return entity.getId();  // ID assigned by DB
}

// 3. Iterator pattern
public T next() {
    // Advances AND returns - standard contract
    T current = elements[position];
    position++;
    return current;
}
```

---

## Naming Conventions

| Type    | Verb Patterns                                      |
| ------- | -------------------------------------------------- |
| Command | `create`, `update`, `delete`, `save`, `add`, `set` |
| Query   | `get`, `find`, `is`, `has`, `can`, `calculate`     |

```java
// Commands - verb suggests action
void createUser(UserRequest request);
void updateEmail(Long userId, String email);
void deleteOrder(Long orderId);
void addItem(Cart cart, Item item);

// Queries - verb suggests retrieval
User getUser(Long id);
List<Order> findOrdersByCustomer(Long customerId);
boolean isActive(Account account);
boolean hasPermission(User user, Permission permission);
Money calculateTotal(Order order);
```

---

## Related Recipes

- [Side Effects](./side-effects.md): Commands contain controlled side effects
- [Small Functions](./small-functions.md): One command or one query per function
