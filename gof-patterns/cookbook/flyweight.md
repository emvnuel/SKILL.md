# Flyweight Pattern - Jakarta EE Cookbook

## Intent

Use sharing to support large numbers of fine-grained objects efficiently by sharing common state.

## Jakarta EE Implementation

### 1. Flyweight Interface

```java
public interface Icon {
    void render(int x, int y);  // Extrinsic state passed as parameters
    String getName();           // Intrinsic state
}
```

### 2. Concrete Flyweight

```java
public class SharedIcon implements Icon {

    // Intrinsic state - shared across all usages
    private final String name;
    private final byte[] imageData;
    private final String mimeType;

    public SharedIcon(String name, byte[] imageData, String mimeType) {
        this.name = name;
        this.imageData = imageData;
        this.mimeType = mimeType;
    }

    @Override
    public void render(int x, int y) {
        // x, y are extrinsic state - varies per usage
        System.out.println("Rendering " + name + " at (" + x + ", " + y + ")");
    }

    @Override
    public String getName() {
        return name;
    }
}
```

### 3. Flyweight Factory

```java
@ApplicationScoped
public class IconFactory {

    private final Map<String, Icon> iconCache = new ConcurrentHashMap<>();

    @Inject
    IconRepository iconRepository;

    public Icon getIcon(String name) {
        return iconCache.computeIfAbsent(name, this::loadIcon);
    }

    private Icon loadIcon(String name) {
        // Load from database/file system once
        IconData data = iconRepository.findByName(name);
        return new SharedIcon(name, data.getImageData(), data.getMimeType());
    }

    public int getCacheSize() {
        return iconCache.size();
    }

    public void clearCache() {
        iconCache.clear();
    }
}
```

### 4. Usage

```java
@ApplicationScoped
public class DocumentRenderer {

    @Inject
    IconFactory iconFactory;

    public void renderDocument(Document doc) {
        for (IconPlacement placement : doc.getIconPlacements()) {
            // Same icon instance shared, different positions
            Icon icon = iconFactory.getIcon(placement.getIconName());
            icon.render(placement.getX(), placement.getY());
        }
    }
}
```

## Real-World Example: Character Formatting

```java
// Flyweight - shared character style
public class CharacterStyle {
    private final String fontFamily;
    private final int fontSize;
    private final String color;
    private final boolean bold;
    private final boolean italic;

    // Constructor, getters...
}

@ApplicationScoped
public class StyleFactory {

    private final Map<String, CharacterStyle> styles = new ConcurrentHashMap<>();

    public CharacterStyle getStyle(String fontFamily, int fontSize,
            String color, boolean bold, boolean italic) {
        String key = fontFamily + "-" + fontSize + "-" + color + "-" + bold + "-" + italic;
        return styles.computeIfAbsent(key,
            k -> new CharacterStyle(fontFamily, fontSize, color, bold, italic));
    }
}

// Client uses flyweights
public class TextCharacter {
    private final char character;           // Extrinsic
    private final int position;             // Extrinsic
    private final CharacterStyle style;     // Flyweight (shared)

    public TextCharacter(char character, int position, CharacterStyle style) {
        this.character = character;
        this.position = position;
        this.style = style;
    }
}
```

## Connection Pool as Flyweight

```java
@ApplicationScoped
public class ConnectionPool {

    private final Queue<Connection> available = new ConcurrentLinkedQueue<>();
    private final Set<Connection> inUse = ConcurrentHashMap.newKeySet();

    @Inject
    @ConfigProperty(name = "db.pool.size", defaultValue = "10")
    int poolSize;

    @PostConstruct
    void init() {
        for (int i = 0; i < poolSize; i++) {
            available.add(createConnection());
        }
    }

    public Connection acquire() {
        Connection conn = available.poll();
        if (conn == null) {
            throw new RuntimeException("No connections available");
        }
        inUse.add(conn);
        return conn;
    }

    public void release(Connection conn) {
        inUse.remove(conn);
        available.offer(conn);
    }

    private Connection createConnection() {
        // Create database connection
    }
}
```

## When to Use

✅ **Use Flyweight when:**

- Application uses a large number of similar objects
- Storage costs are high due to object quantity
- Most object state can be made extrinsic
- Many groups of objects can be replaced by few shared objects

❌ **Avoid Flyweight when:**

- Object count is small
- Objects have mostly unique state
- The complexity of separating intrinsic/extrinsic state outweighs savings
