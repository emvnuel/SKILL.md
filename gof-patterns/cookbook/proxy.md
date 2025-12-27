# Proxy Pattern - Jakarta EE Cookbook

## Intent

Provide a surrogate or placeholder for another object to control access to it.

## Jakarta EE Implementation

CDI uses proxies extensively. Key proxy types:

### 1. Lazy Loading Proxy (Built-in CDI)

```java
// CDI creates a proxy for @ApplicationScoped beans
@ApplicationScoped
public class ExpensiveService {

    @PostConstruct
    void init() {
        // Heavy initialization - only runs when first used
        loadLargeDataset();
    }

    public Data process(Request request) {
        // ...
    }
}

@ApplicationScoped
public class ClientService {

    @Inject
    ExpensiveService service;  // Proxy injected, not real instance

    public void doWork() {
        // Real instance created only now
        service.process(request);
    }
}
```

### 2. Security Proxy (Jakarta Security)

```java
@ApplicationScoped
public class AdminService {

    @RolesAllowed("admin")
    public void deleteAllUsers() {
        // Only admins can call this
    }

    @RolesAllowed({"admin", "manager"})
    public void modifyUser(Long userId, UserUpdate update) {
        // Admins and managers
    }

    @PermitAll
    public User getUser(Long id) {
        // Anyone authenticated
    }
}
```

### 3. Remote Proxy (MicroProfile REST Client)

```java
// Interface defines remote service contract
@RegisterRestClient(configKey = "user-api")
@Path("/users")
public interface UserApiClient {

    @GET
    @Path("/{id}")
    User getUser(@PathParam("id") Long id);

    @POST
    User createUser(CreateUserRequest request);
}

// MicroProfile creates proxy that handles HTTP calls
@ApplicationScoped
public class UserService {

    @Inject
    @RestClient
    UserApiClient userApi;  // Proxy handles HTTP

    public User findUser(Long id) {
        return userApi.getUser(id);  // Appears local, executed remotely
    }
}
```

### 4. Caching Proxy (Custom)

```java
@ApplicationScoped
public class ProductCacheProxy implements ProductService {

    @Inject
    @Named("real")
    ProductService realService;

    @Inject
    Cache<Long, Product> cache;

    @Override
    public Product findById(Long id) {
        return cache.get(id, () -> realService.findById(id));
    }

    @Override
    public void save(Product product) {
        realService.save(product);
        cache.put(product.getId(), product);
    }
}
```

## When to Use

✅ **Use Proxy when:**

- Lazy initialization of expensive objects (CDI handles this)
- Access control (use @RolesAllowed)
- Remote method invocation (use REST Client)
- Caching, logging, or metrics (use CDI decorators/interceptors)

❌ **Avoid manual Proxy when:**

- CDI or Jakarta EE already provides the mechanism
- Simple delegation without additional behavior
