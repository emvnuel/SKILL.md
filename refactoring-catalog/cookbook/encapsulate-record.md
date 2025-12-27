# Encapsulate Record

_Also known as: Replace Record with Data Class_

## Intent

Replace a record structure (map/hash/plain object) with a proper class that encapsulates the data and can evolve with behavior.

## Code Smells That Indicate This Refactoring

- Using Map<String, Object> for structured data
- Dynamic key access to data structures
- No type safety for record fields
- Data structure needs validation or derived fields
- Record structure becoming complex

## Mechanics

1. Create a class for the record
2. Wrap the underlying record in the class
3. For each element being accessed, create an accessor
4. Replace accesses to the raw record with accessor calls
5. Test after each change
6. Remove the raw data access methods
7. If the record has nested records, consider recursively encapsulating them

## Example

### Before

```java
public class Organization {
    private Map<String, Object> data;

    public Organization(Map<String, Object> data) {
        this.data = data;
    }

    public Map<String, Object> getData() {
        return data;
    }
}

// Usage
Map<String, Object> orgData = new HashMap<>();
orgData.put("name", "Acme Corp");
orgData.put("country", "USA");
Organization org = new Organization(orgData);
String name = (String) org.getData().get("name");
```

### After

```java
public class Organization {
    private String name;
    private String country;

    public Organization(String name, String country) {
        this.name = name;
        this.country = country;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getCountry() {
        return country;
    }

    public void setCountry(String country) {
        this.country = country;
    }
}

// Usage
Organization org = new Organization("Acme Corp", "USA");
String name = org.getName();
```

### With Nested Records

```java
// Before: nested maps
Map<String, Object> customer = new HashMap<>();
customer.put("name", "John");
customer.put("address", Map.of("city", "NYC", "zip", "10001"));

// After: proper classes
public class Address {
    private final String city;
    private final String zipCode;

    public Address(String city, String zipCode) {
        this.city = city;
        this.zipCode = zipCode;
    }

    public String getCity() { return city; }
    public String getZipCode() { return zipCode; }
}

public class Customer {
    private final String name;
    private final Address address;

    public Customer(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public String getName() { return name; }
    public Address getAddress() { return address; }
}
```

### Using Java Record

```java
public record Organization(String name, String country) {
    public Organization {
        Objects.requireNonNull(name, "Name is required");
        Objects.requireNonNull(country, "Country is required");
    }
}
```

## When to Use

✅ **Use Encapsulate Record when:**

- Data structure has well-known fields
- You need type safety
- You want to add validation
- Structure needs derived/computed fields
- You're moving from dynamic to static typing

❌ **Avoid Encapsulate Record when:**

- Data structure is truly dynamic
- Working with external data that varies (JSON APIs)
- Simple key-value storage is appropriate
- Record is used only briefly

## Related Refactorings

- **Encapsulate Variable**: For individual fields
- **Encapsulate Collection**: When record contains collections
- **Replace Primitive with Object**: Often done for record fields
