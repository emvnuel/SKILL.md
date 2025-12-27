# @Synchronized Annotation

## Intent

Provide a safer alternative to Java's `synchronized` keyword by locking on private fields instead of `this` or `class`.

## Basic Usage

```java
public class Counter {
    private int count = 0;

    @Synchronized
    public void increment() {
        count++;
    }

    @Synchronized
    public int getCount() {
        return count;
    }
}

// Equivalent to:
public class Counter {
    private final Object $lock = new Object();
    private int count = 0;

    public void increment() {
        synchronized ($lock) {
            count++;
        }
    }

    public int getCount() {
        synchronized ($lock) {
            return count;
        }
    }
}
```

## Why Not `synchronized` Keyword?

```java
// ❌ Dangerous - external code can lock on same object
public class UnsafeCounter {
    public synchronized void increment() { }
}

// External code can cause deadlock:
UnsafeCounter counter = new UnsafeCounter();
synchronized (counter) {  // Locks same monitor as increment()
    // Can cause issues
}

// ✅ Safe - private lock object
@Synchronized
public void increment() { }
// External code cannot access the $lock field
```

## Static Methods

```java
public class Registry {
    private static final Map<String, Object> cache = new HashMap<>();

    @Synchronized
    public static void register(String key, Object value) {
        cache.put(key, value);
    }

    @Synchronized
    public static Object get(String key) {
        return cache.get(key);
    }
}

// Generates static lock: private static final Object $LOCK = new Object();
```

## Custom Lock Object

```java
public class BankAccount {
    private final Object readLock = new Object();
    private final Object writeLock = new Object();

    private BigDecimal balance = BigDecimal.ZERO;

    @Synchronized("writeLock")
    public void deposit(BigDecimal amount) {
        balance = balance.add(amount);
    }

    @Synchronized("readLock")
    public BigDecimal getBalance() {
        return balance;
    }
}
```

## When to Use

✅ **Use `@Synchronized` for:**

- Simple thread-safe operations
- Protecting shared mutable state
- When you don't need fine-grained locking

❌ **Avoid `@Synchronized` for:**

- Complex concurrent scenarios (use java.util.concurrent)
- When you need ReadWriteLock
- High-contention scenarios
- When Lock features (tryLock, timeout) are needed

## Comparison

| Feature         | `synchronized` | `@Synchronized` | `Lock`       |
| --------------- | -------------- | --------------- | ------------ |
| Lock visibility | Public         | Private         | Configurable |
| Simplicity      | Simple         | Simpler         | Complex      |
| Features        | Basic          | Basic           | Full         |
| Interruptible   | No             | No              | Yes          |

## Related Annotations

None directly, but consider:

- `java.util.concurrent.locks.Lock` for advanced scenarios
- `@Atomic` operations for simple counters
