# Decorator Pattern - Jakarta EE Cookbook

## Intent

Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

## Jakarta EE Implementation

CDI provides two powerful mechanisms: **Decorators** and **Interceptors**.

### 1. CDI Decorator

```java
// Service interface
public interface UserRepository {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
}

// Base implementation
@ApplicationScoped
public class JpaUserRepository implements UserRepository {
    @Inject
    EntityManager em;

    public User findById(Long id) {
        return em.find(User.class, id);
    }

    public List<User> findAll() {
        return em.createQuery("FROM User", User.class).getResultList();
    }

    public void save(User user) {
        em.persist(user);
    }
}

// Caching decorator
@Decorator
@Priority(Interceptor.Priority.APPLICATION)
public abstract class CachingUserRepository implements UserRepository {

    @Inject
    @Delegate
    UserRepository delegate;

    @Inject
    CacheManager cache;

    @Override
    public User findById(Long id) {
        String key = "user:" + id;
        return cache.get(key, () -> delegate.findById(id));
    }

    @Override
    public void save(User user) {
        delegate.save(user);
        cache.evict("user:" + user.getId());
    }

    // findAll() uses default delegation
}
```

### 2. CDI Interceptor

```java
// Interceptor binding
@InterceptorBinding
@Retention(RUNTIME)
@Target({TYPE, METHOD})
public @interface Logged {}

// Interceptor implementation
@Logged
@Interceptor
@Priority(Interceptor.Priority.APPLICATION)
public class LoggingInterceptor {

    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);

    @AroundInvoke
    public Object logMethod(InvocationContext ctx) throws Exception {
        String method = ctx.getMethod().getName();
        log.info("Entering: {}", method);
        long start = System.currentTimeMillis();
        try {
            return ctx.proceed();
        } finally {
            log.info("Exiting: {} ({}ms)", method, System.currentTimeMillis() - start);
        }
    }
}

// Usage
@ApplicationScoped
@Logged  // All methods logged
public class OrderService {

    @Logged  // Or per method
    public Order createOrder(OrderRequest request) {
        // ...
    }
}
```

### 3. Timing Interceptor

```java
@InterceptorBinding
@Retention(RUNTIME)
@Target({TYPE, METHOD})
public @interface Timed {}

@Timed
@Interceptor
@Priority(Interceptor.Priority.APPLICATION)
public class TimingInterceptor {

    @Inject
    MetricsRegistry metrics;

    @AroundInvoke
    public Object time(InvocationContext ctx) throws Exception {
        String name = ctx.getTarget().getClass().getSimpleName()
                    + "." + ctx.getMethod().getName();
        Timer timer = metrics.timer(name);
        Timer.Sample sample = Timer.start();
        try {
            return ctx.proceed();
        } finally {
            sample.stop(timer);
        }
    }
}
```

### 4. Retry Decorator (Using MicroProfile Fault Tolerance)

```java
@ApplicationScoped
public class ExternalApiClient {

    @Retry(maxRetries = 3, delay = 100)
    @Timeout(value = 5, unit = ChronoUnit.SECONDS)
    @Fallback(fallbackMethod = "fallbackGetData")
    public Data getData(String id) {
        return externalApi.fetchData(id);
    }

    public Data fallbackGetData(String id) {
        return Data.empty();
    }
}
```

## When to Use

✅ **Use Decorator when:**

- Adding responsibilities to objects dynamically
- Cross-cutting concerns (logging, caching, security)
- Extending behavior without modifying original class

❌ **Avoid Decorator when:**

- Single fixed enhancement (use inheritance)
- Simple cases where AOP/interceptors add complexity
