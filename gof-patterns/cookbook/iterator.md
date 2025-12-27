# Iterator Pattern - Jakarta EE Cookbook

## Intent

Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.

## Jakarta EE Implementation

Java provides built-in `Iterator` and `Iterable` interfaces, but custom iterators are useful for:

- Lazy loading from database
- Streaming large datasets
- Custom traversal logic

### 1. Custom Iterator for Pagination

```java
public class PaginatedIterator<T> implements Iterator<T> {

    private final Function<Integer, List<T>> fetcher;
    private final int pageSize;
    private int currentPage = 0;
    private List<T> currentBatch;
    private int indexInBatch = 0;
    private boolean exhausted = false;

    public PaginatedIterator(Function<Integer, List<T>> fetcher, int pageSize) {
        this.fetcher = fetcher;
        this.pageSize = pageSize;
        fetchNextBatch();
    }

    @Override
    public boolean hasNext() {
        if (exhausted) return false;
        if (indexInBatch < currentBatch.size()) return true;
        fetchNextBatch();
        return !exhausted;
    }

    @Override
    public T next() {
        if (!hasNext()) throw new NoSuchElementException();
        return currentBatch.get(indexInBatch++);
    }

    private void fetchNextBatch() {
        currentBatch = fetcher.apply(currentPage++);
        indexInBatch = 0;
        if (currentBatch.isEmpty() || currentBatch.size() < pageSize) {
            exhausted = currentBatch.isEmpty();
        }
    }
}
```

### 2. Iterable Collection Wrapper

```java
public class PaginatedCollection<T> implements Iterable<T> {

    private final Function<Integer, List<T>> fetcher;
    private final int pageSize;

    public PaginatedCollection(Function<Integer, List<T>> fetcher, int pageSize) {
        this.fetcher = fetcher;
        this.pageSize = pageSize;
    }

    @Override
    public Iterator<T> iterator() {
        return new PaginatedIterator<>(fetcher, pageSize);
    }

    public Stream<T> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
}
```

### 3. Usage with Repository

```java
@ApplicationScoped
public class UserRepository {

    @Inject
    EntityManager em;

    public Iterable<User> findAllPaginated(int pageSize) {
        return new PaginatedCollection<>(
            page -> em.createQuery("SELECT u FROM User u ORDER BY u.id", User.class)
                .setFirstResult(page * pageSize)
                .setMaxResults(pageSize)
                .getResultList(),
            pageSize
        );
    }
}

@ApplicationScoped
public class UserExportService {

    @Inject
    UserRepository userRepository;

    public void exportAllUsers(Writer writer) {
        // Memory-efficient iteration over millions of users
        for (User user : userRepository.findAllPaginated(100)) {
            writer.write(formatUser(user));
        }
    }
}
```

### 4. Composite Iterator

```java
public class CompositeIterator<T> implements Iterator<T> {

    private final Queue<Iterator<T>> iterators;

    public CompositeIterator(List<Iterator<T>> iterators) {
        this.iterators = new LinkedList<>(iterators);
    }

    @Override
    public boolean hasNext() {
        while (!iterators.isEmpty()) {
            if (iterators.peek().hasNext()) return true;
            iterators.poll();
        }
        return false;
    }

    @Override
    public T next() {
        if (!hasNext()) throw new NoSuchElementException();
        return iterators.peek().next();
    }
}
```

### 5. Filtered Iterator

```java
public class FilteredIterator<T> implements Iterator<T> {

    private final Iterator<T> delegate;
    private final Predicate<T> filter;
    private T next;
    private boolean hasNext;

    public FilteredIterator(Iterator<T> delegate, Predicate<T> filter) {
        this.delegate = delegate;
        this.filter = filter;
        advance();
    }

    @Override
    public boolean hasNext() {
        return hasNext;
    }

    @Override
    public T next() {
        if (!hasNext) throw new NoSuchElementException();
        T result = next;
        advance();
        return result;
    }

    private void advance() {
        hasNext = false;
        while (delegate.hasNext()) {
            T candidate = delegate.next();
            if (filter.test(candidate)) {
                next = candidate;
                hasNext = true;
                break;
            }
        }
    }
}
```

## When to Use

✅ **Use Custom Iterator when:**

- Lazy loading from database or external source
- Processing large datasets without loading all in memory
- Custom traversal order needed (e.g., tree traversal)
- Filtering or transforming during iteration

❌ **Avoid Custom Iterator when:**

- Standard Java Stream/Iterator suffices
- Collection fits in memory
- Simple sequential access needed
