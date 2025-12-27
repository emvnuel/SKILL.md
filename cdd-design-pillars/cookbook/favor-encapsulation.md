# Favor Encapsulation for Cohesion

## Principle

> Favor cohesion through encapsulation. Only modify references you created. Don't touch other people's references unless that reference was created for modification.

## Problem

Exposing internal state leads to logic scattered across the codebase and fragile dependencies.

```java
// ❌ BAD: Internal state exposed and manipulated externally
public class ShoppingCart {
    private List<CartItem> items = new ArrayList<>();

    public List<CartItem> getItems() {
        return items;  // Exposes mutable internal state
    }
}

// External manipulation - logic scattered
cart.getItems().add(new CartItem(product, 2));
cart.getItems().removeIf(item -> item.getQuantity() == 0);
BigDecimal total = cart.getItems().stream()
    .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

## Solution

Keep state manipulation inside the owning class. Expose behavior, not data.

```java
@Entity
public class ShoppingCart {

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<CartItem> items = new ArrayList<>();

    // ✓ Behavior encapsulated
    public void addItem(Product product, int quantity) {
        findItem(product)
            .ifPresentOrElse(
                item -> item.increaseQuantity(quantity),
                () -> items.add(new CartItem(product, quantity))
            );
    }

    public void removeItem(Product product) {
        items.removeIf(item -> item.getProduct().equals(product));
    }

    public void updateQuantity(Product product, int newQuantity) {
        if (newQuantity <= 0) {
            removeItem(product);
            return;
        }
        findItem(product)
            .ifPresent(item -> item.setQuantity(newQuantity));
    }

    // ✓ Calculation encapsulated
    public BigDecimal getTotal() {
        return items.stream()
            .map(CartItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // ✓ Read-only view if needed
    public List<CartItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    public int getItemCount() {
        return items.size();
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    private Optional<CartItem> findItem(Product product) {
        return items.stream()
            .filter(item -> item.getProduct().equals(product))
            .findFirst();
    }
}
```

### CartItem with Encapsulated Logic

```java
@Entity
public class CartItem {

    @ManyToOne
    private Product product;

    private int quantity;

    protected CartItem() {}

    public CartItem(Product product, int quantity) {
        this.product = Objects.requireNonNull(product);
        setQuantity(quantity);
    }

    public void increaseQuantity(int amount) {
        setQuantity(this.quantity + amount);
    }

    public void setQuantity(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        this.quantity = quantity;
    }

    // ✓ Calculation in the owning class
    public BigDecimal getSubtotal() {
        return product.getPrice().multiply(BigDecimal.valueOf(quantity));
    }

    public Product getProduct() {
        return product;
    }

    public int getQuantity() {
        return quantity;
    }
}
```

### Service Using Encapsulated Domain

```java
@ApplicationScoped
public class CartService {

    @Inject
    private CartRepository carts;

    @Inject
    private ProductRepository products;

    @Transactional
    public CartResponse addItem(Long cartId, AddItemRequest request) {
        ShoppingCart cart = carts.findById(cartId)
            .orElseThrow(CartNotFoundException::new);

        Product product = products.findById(request.productId())
            .orElseThrow(ProductNotFoundException::new);

        // ✓ Delegate to domain object
        cart.addItem(product, request.quantity());

        return CartResponse.from(cart);
    }
}
```

## Cognitive Load Analysis

**Service method:**
| Element | Points |
|---------|--------|
| CartRepository | +1 |
| ProductRepository | +1 |
| orElseThrow (branch) | +1 |
| cart.addItem() | +1 |
| CartResponse.from() | +1 |
| **Total** | **5 points ✓** |

Logic is distributed to domain objects, keeping service simple.

## Benefits

- **Single source of truth**: Logic lives with data
- **Reduced coupling**: Clients depend on behavior, not structure
- **Easier testing**: Domain objects testable in isolation
- **Maintainability**: Changes localized

## Related Entries

- [concentrate-operations](concentrate-operations.md) - Operations on owned data
- [jpa-entity-design](jpa-entity-design.md) - JPA with encapsulation
