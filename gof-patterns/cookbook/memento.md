# Memento Pattern - Jakarta EE Cookbook

## Intent

Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.

## Jakarta EE Implementation

### 1. Memento Class

```java
public class DocumentMemento {

    private final String content;
    private final String title;
    private final Instant timestamp;
    private final String author;

    DocumentMemento(String content, String title, String author) {
        this.content = content;
        this.title = title;
        this.author = author;
        this.timestamp = Instant.now();
    }

    // Package-private getters for Originator access
    String getContent() { return content; }
    String getTitle() { return title; }
    Instant getTimestamp() { return timestamp; }
    String getAuthor() { return author; }
}
```

### 2. Originator

```java
@Entity
public class Document {

    @Id
    @GeneratedValue
    private Long id;

    private String title;

    @Lob
    private String content;

    private String lastAuthor;

    // Create memento (snapshot)
    public DocumentMemento createMemento() {
        return new DocumentMemento(content, title, lastAuthor);
    }

    // Restore from memento
    public void restore(DocumentMemento memento) {
        this.content = memento.getContent();
        this.title = memento.getTitle();
        this.lastAuthor = memento.getAuthor();
    }

    // Getters/setters
}
```

### 3. Caretaker

```java
@ApplicationScoped
public class DocumentHistoryService {

    private final Map<Long, Deque<DocumentMemento>> history = new ConcurrentHashMap<>();

    @Inject
    @ConfigProperty(name = "document.history.maxVersions", defaultValue = "50")
    int maxVersions;

    public void saveVersion(Long documentId, DocumentMemento memento) {
        Deque<DocumentMemento> versions = history.computeIfAbsent(
            documentId, k -> new LinkedList<>()
        );
        versions.push(memento);

        // Limit history size
        while (versions.size() > maxVersions) {
            versions.removeLast();
        }
    }

    public DocumentMemento undo(Long documentId) {
        Deque<DocumentMemento> versions = history.get(documentId);
        if (versions != null && !versions.isEmpty()) {
            versions.pop();  // Remove current
            return versions.peek();  // Return previous
        }
        return null;
    }

    public List<DocumentMemento> getHistory(Long documentId) {
        Deque<DocumentMemento> versions = history.get(documentId);
        return versions != null ? new ArrayList<>(versions) : List.of();
    }
}
```

### 4. Document Service with Undo

```java
@ApplicationScoped
public class DocumentService {

    @Inject
    DocumentRepository repository;

    @Inject
    DocumentHistoryService historyService;

    @Transactional
    public Document updateDocument(Long id, UpdateDocumentRequest request, String author) {
        Document doc = repository.findById(id);

        // Save current state before modification
        historyService.saveVersion(id, doc.createMemento());

        // Apply changes
        doc.setTitle(request.getTitle());
        doc.setContent(request.getContent());
        doc.setLastAuthor(author);

        return repository.save(doc);
    }

    @Transactional
    public Document undoLastChange(Long id) {
        Document doc = repository.findById(id);
        DocumentMemento previousState = historyService.undo(id);

        if (previousState != null) {
            doc.restore(previousState);
            return repository.save(doc);
        }

        throw new IllegalStateException("No previous version available");
    }

    public List<VersionInfo> getVersionHistory(Long id) {
        return historyService.getHistory(id).stream()
            .map(m -> new VersionInfo(m.getTimestamp(), m.getAuthor()))
            .toList();
    }
}
```

### 5. Persistent Memento with JPA

```java
@Entity
@Table(name = "document_versions")
public class PersistentDocumentMemento {

    @Id
    @GeneratedValue
    private Long id;

    private Long documentId;

    @Lob
    private String content;

    private String title;

    private String author;

    private Instant timestamp;

    private int versionNumber;

    // For JPA
    protected PersistentDocumentMemento() {}

    public PersistentDocumentMemento(Document doc, int versionNumber) {
        this.documentId = doc.getId();
        this.content = doc.getContent();
        this.title = doc.getTitle();
        this.author = doc.getLastAuthor();
        this.timestamp = Instant.now();
        this.versionNumber = versionNumber;
    }
}

@ApplicationScoped
public class PersistentHistoryService {

    @Inject
    EntityManager em;

    @Transactional
    public void saveVersion(Document doc) {
        int nextVersion = getLatestVersion(doc.getId()) + 1;
        em.persist(new PersistentDocumentMemento(doc, nextVersion));
    }

    public List<PersistentDocumentMemento> getHistory(Long documentId) {
        return em.createQuery(
            "SELECT m FROM PersistentDocumentMemento m " +
            "WHERE m.documentId = :docId ORDER BY m.versionNumber DESC",
            PersistentDocumentMemento.class
        ).setParameter("docId", documentId).getResultList();
    }

    public PersistentDocumentMemento getVersion(Long documentId, int version) {
        return em.createQuery(
            "SELECT m FROM PersistentDocumentMemento m " +
            "WHERE m.documentId = :docId AND m.versionNumber = :version",
            PersistentDocumentMemento.class
        ).setParameter("docId", documentId)
         .setParameter("version", version)
         .getSingleResult();
    }
}
```

## When to Use

✅ **Use Memento when:**

- You need undo/redo functionality
- You need to save snapshots of object state
- Direct access to state would expose implementation details
- Audit trail or version history is required

❌ **Avoid Memento when:**

- Object state is simple (just copy fields)
- Storage costs for mementos are too high
- Object state changes rarely need to be undone
