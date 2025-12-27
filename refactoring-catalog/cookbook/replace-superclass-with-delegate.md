# Replace Superclass with Delegate

_Also known as: Replace Inheritance with Delegation_

## Intent

Replace inheritance with composition when a class doesn't really need a full is-a relationship with its superclass.

## Code Smells That Indicate This Refactoring

- Subclass doesn't use most of the superclass interface
- Subclass has to override methods to disable behavior
- Inheritance was used for code reuse, not substitutability
- LSP (Liskov Substitution Principle) is violated

## Mechanics

1. Create a field in the subclass to hold an instance of the superclass
2. Initialize the delegate in the subclass constructor
3. Create forwarding methods for each element of the superclass used
4. Remove the inheritance relationship
5. Test

## Example

### Before

```java
public class Stack<E> extends ArrayList<E> {
    public E push(E item) {
        add(item);
        return item;
    }

    public E pop() {
        return remove(size() - 1);
    }

    public E peek() {
        return get(size() - 1);
    }

    public boolean empty() {
        return isEmpty();
    }
}

// Problem: Stack inherits many methods it shouldn't have
Stack<String> stack = new Stack<>();
stack.push("first");
stack.add(0, "illegal");  // Shouldn't be allowed! Violates stack contract
```

### After

```java
public class Stack<E> {
    private final List<E> storage = new ArrayList<>();

    public E push(E item) {
        storage.add(item);
        return item;
    }

    public E pop() {
        if (storage.isEmpty()) {
            throw new EmptyStackException();
        }
        return storage.remove(storage.size() - 1);
    }

    public E peek() {
        if (storage.isEmpty()) {
            throw new EmptyStackException();
        }
        return storage.get(storage.size() - 1);
    }

    public boolean empty() {
        return storage.isEmpty();
    }

    public int size() {
        return storage.size();
    }
}

// Now only proper stack operations are available
Stack<String> stack = new Stack<>();
stack.push("first");
// stack.add(0, "illegal");  // Compile error - method doesn't exist
```

### Selective Forwarding

```java
// Before - inheriting too much
public class CustomList<E> extends ArrayList<E> {
    // Only wanted a few methods...
}

// After - expose only what's needed
public class CustomList<E> implements Iterable<E> {
    private final List<E> delegate = new ArrayList<>();

    public void add(E item) {
        delegate.add(item);
    }

    public E get(int index) {
        return delegate.get(index);
    }

    public int size() {
        return delegate.size();
    }

    @Override
    public Iterator<E> iterator() {
        return delegate.iterator();
    }

    // Only these operations are exposed
}
```

### Using the Superclass Interface

```java
// When you still want to be interchangeable
public class CatalogItem implements Comparable<CatalogItem> {
    private final String name;
    private final Scroll delegate;

    public CatalogItem(String name, Scroll scroll) {
        this.name = name;
        this.delegate = scroll;
    }

    public String getName() {
        return name;
    }

    public String getContents() {
        return delegate.getContents();  // Forward to delegate
    }

    @Override
    public int compareTo(CatalogItem other) {
        return this.name.compareTo(other.name);
    }
}
```

## When to Use

✅ **Use Replace Superclass with Delegate when:**

- Subclass only uses part of superclass interface
- Inheritance breaks LSP
- You want to hide superclass methods
- Composition is more appropriate than inheritance

❌ **Avoid Replace Superclass with Delegate when:**

- True is-a relationship exists
- All inherited interface is appropriate
- Many methods would need forwarding
- Polymorphism with superclass is needed

## Related Refactorings

- **Replace Subclass with Delegate**: Similar for subclasses
- **Extract Superclass**: Related hierarchy operation
- **Extract Class**: For creating the delegate
