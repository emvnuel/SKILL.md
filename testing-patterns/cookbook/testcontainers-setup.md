# Testcontainers Setup

## Intent

Test against real database engines (PostgreSQL, MySQL) in isolated containers. Ensures tests behave identically to production without manual database setup.

---

## Quarkus DevServices (Recommended)

Quarkus automatically starts containers for supported services. Zero configuration required:

### application.properties

```properties
# No datasource URL needed - DevServices provides it
quarkus.datasource.db-kind=postgresql
quarkus.datasource.devservices.enabled=true

# Optional: specific image version
quarkus.datasource.devservices.image-name=postgres:15-alpine
```

DevServices starts PostgreSQL container automatically when running tests.

---

## Manual Testcontainers Setup

For more control or non-Quarkus projects:

```java
@QuarkusTest
@Testcontainers
class ProductRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @BeforeAll
    static void setup() {
        System.setProperty("quarkus.datasource.jdbc.url", postgres.getJdbcUrl());
        System.setProperty("quarkus.datasource.username", postgres.getUsername());
        System.setProperty("quarkus.datasource.password", postgres.getPassword());
    }

    @Test
    void shouldPersistProduct() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"name": "Test Product", "price": 99.99}
                """)
        .when()
            .post("/products")
        .then()
            .statusCode(201);

        given()
        .when()
            .get("/products")
        .then()
            .body("size()", equalTo(1));
    }
}
```

---

## Quarkus Test Resource (Reusable)

For sharing container across test classes:

```java
public class PostgresTestResource implements QuarkusTestResourceLifecycleManager {

    private static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:15-alpine");

    @Override
    public Map<String, String> start() {
        POSTGRES.start();
        return Map.of(
            "quarkus.datasource.jdbc.url", POSTGRES.getJdbcUrl(),
            "quarkus.datasource.username", POSTGRES.getUsername(),
            "quarkus.datasource.password", POSTGRES.getPassword()
        );
    }

    @Override
    public void stop() {
        POSTGRES.stop();
    }
}
```

```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
class OrderRepositoryTest {
    // Container shared across all tests using this resource
}
```

---

## Database Cleanup Between Tests

### Option 1: Transaction Rollback

```java
@QuarkusTest
@TestTransaction
class ProductTest {

    @Test
    void shouldCreateProduct() {
        // Changes rolled back after test
    }
}
```

### Option 2: Cleanup Query

```java
@Inject
EntityManager em;

@BeforeEach
@Transactional
void cleanup() {
    em.createQuery("DELETE FROM Product").executeUpdate();
}
```

### Option 3: Flyway Clean

```properties
# application.properties (test profile only)
%test.quarkus.flyway.clean-at-start=true
```

---

## Multiple Services

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
    .withExposedPorts(6379);

@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
);
```

---

## DevServices vs Manual Testcontainers

| Feature           | DevServices            | Manual Testcontainers |
| ----------------- | ---------------------- | --------------------- |
| Configuration     | Zero-config            | Explicit setup        |
| Container sharing | Automatic              | Via TestResource      |
| Image version     | Defaults (overridable) | Explicit              |
| Non-Quarkus       | No                     | Yes                   |

**Prefer DevServices** for Quarkus projects. Use manual setup for complex scenarios or non-Quarkus.

---

## Related Recipes

- [H2 Setup](./h2-setup.md): Faster alternative without containers
- [REST Integration Tests](./rest-integration-tests.md): Testing with real database
