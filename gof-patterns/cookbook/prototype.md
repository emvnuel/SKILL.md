# Prototype Pattern - Jakarta EE Cookbook

## Intent

Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype.

## Jakarta EE Implementation

### 1. Cloneable Entity

```java
@Entity
public class DocumentTemplate implements Cloneable {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private String content;

    @ElementCollection
    private Map<String, String> metadata = new HashMap<>();

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Section> sections = new ArrayList<>();

    // Create a copy without ID (for persistence as new entity)
    @Override
    public DocumentTemplate clone() {
        try {
            DocumentTemplate copy = (DocumentTemplate) super.clone();
            copy.id = null;  // New entity
            copy.metadata = new HashMap<>(this.metadata);
            copy.sections = this.sections.stream()
                .map(Section::clone)
                .collect(Collectors.toList());
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 2. Prototype Registry

```java
@ApplicationScoped
public class TemplateRegistry {

    @Inject
    TemplateRepository repository;

    private final Map<String, DocumentTemplate> prototypes = new ConcurrentHashMap<>();

    @PostConstruct
    void loadPrototypes() {
        repository.findAllTemplates().forEach(t ->
            prototypes.put(t.getName(), t)
        );
    }

    public DocumentTemplate create(String templateName) {
        DocumentTemplate prototype = prototypes.get(templateName);
        if (prototype == null) {
            throw new IllegalArgumentException("Unknown template: " + templateName);
        }
        return prototype.clone();
    }

    public void register(String name, DocumentTemplate template) {
        prototypes.put(name, template);
    }
}
```

### 3. Usage

```java
@ApplicationScoped
public class DocumentService {

    @Inject
    TemplateRegistry templates;

    @Inject
    DocumentRepository documents;

    @Transactional
    public Document createFromTemplate(String templateName, String title) {
        DocumentTemplate template = templates.create(templateName);
        template.setName(title);
        return documents.save(template);
    }
}
```

### 4. Deep Copy with Jackson

```java
@ApplicationScoped
public class DeepCopyService {

    @Inject
    ObjectMapper objectMapper;

    public <T> T deepCopy(T object, Class<T> type) {
        try {
            String json = objectMapper.writeValueAsString(object);
            return objectMapper.readValue(json, type);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to deep copy", e);
        }
    }
}
```

## When to Use

✅ **Use Prototype when:**

- Creating objects is expensive and you need copies
- You want to avoid subclasses of factories
- Objects have many shared configurations

❌ **Avoid Prototype when:**

- Objects have circular references (complex to clone)
- Classes have final fields that need different values
