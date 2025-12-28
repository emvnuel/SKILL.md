# H2 In-Memory Database Setup

## Intent

Use H2 for fast integration tests when container startup overhead is unacceptable or full database compatibility isn't required.

---

## Basic Configuration

### application.properties

```properties
# Test profile uses H2 instead of PostgreSQL
%test.quarkus.datasource.db-kind=h2
%test.quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
%test.quarkus.datasource.username=sa
%test.quarkus.datasource.password=

# Auto-generate schema from entities
%test.quarkus.hibernate-orm.database.generation=drop-and-create
```

---

## With Flyway Migrations

If using Flyway with PostgreSQL-specific SQL:

```properties
%test.quarkus.datasource.db-kind=h2
%test.quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%test.quarkus.flyway.locations=classpath:db/migration,classpath:db/migration-h2
```

Create H2-compatible versions in `src/test/resources/db/migration-h2/` when needed.

---

## H2 Compatibility Modes

H2 can emulate other databases:

```properties
# PostgreSQL mode
jdbc:h2:mem:testdb;MODE=PostgreSQL

# MySQL mode
jdbc:h2:mem:testdb;MODE=MySQL

# Oracle mode
jdbc:h2:mem:testdb;MODE=Oracle
```

---

## When to Use H2

| Use H2                      | Use Testcontainers                |
| --------------------------- | --------------------------------- |
| Simple CRUD operations      | Database-specific features        |
| Unit tests for repositories | Complex queries, window functions |
| CI with limited resources   | Production parity critical        |
| Fast feedback loop          | Testing migrations                |

---

## H2 Limitations

- No full-text search (PostgreSQL `tsvector`)
- Limited JSON operators
- Different locking behavior
- No native JSONB, ARRAY types

---

## Test Example

```java
@QuarkusTest
class CustomerRepositoryTest {

    @Inject
    CustomerRepository repository;

    @Test
    @Transactional
    void shouldFindByEmail() {
        Customer customer = new Customer("test@example.com", "Test User");
        repository.persist(customer);

        Optional<Customer> found = repository.findByEmail("test@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Test User");
    }
}
```

---

## Related Recipes

- [Testcontainers Setup](./testcontainers-setup.md): Real database testing
- [REST Integration Tests](./rest-integration-tests.md): Full endpoint testing
