# API Versioning

## Intent

Version APIs to evolve without breaking existing clients. Allow gradual migration to new versions.

---

## URL Path Versioning (Recommended)

Most common and explicit approach:

```java
// Version 1
@Path("/v1/users")
public class UserResourceV1 {

    @GET
    @Path("/{id}")
    public UserV1 get(@PathParam("id") Long id) {
        return new UserV1(id, user.getName());
    }
}

// Version 2 - added email field
@Path("/v2/users")
public class UserResourceV2 {

    @GET
    @Path("/{id}")
    public UserV2 get(@PathParam("id") Long id) {
        return new UserV2(id, user.getName(), user.getEmail());
    }
}
```

**URLs**:

- `/v1/users/123`
- `/v2/users/123`

---

## Header Versioning

Version in request header:

```java
@Path("/users")
public class UserResource {

    @GET
    @Path("/{id}")
    public Response get(
        @PathParam("id") Long id,
        @HeaderParam("API-Version") @DefaultValue("1") int version
    ) {
        User user = userRepository.findById(id);

        if (version >= 2) {
            return Response.ok(new UserV2(user)).build();
        }
        return Response.ok(new UserV1(user)).build();
    }
}
```

**Request**: `GET /users/123` with `API-Version: 2`

---

## Accept Header Versioning

Using content negotiation:

```java
@GET
@Path("/{id}")
@Produces({"application/vnd.myapi.v2+json", "application/vnd.myapi.v1+json"})
public Response get(
    @PathParam("id") Long id,
    @HeaderParam("Accept") String accept
) {
    if (accept.contains("v2")) {
        return Response.ok(new UserV2(user)).build();
    }
    return Response.ok(new UserV1(user)).build();
}
```

**Request**: `Accept: application/vnd.myapi.v2+json`

---

## Version Comparison

| Strategy | Pros                        | Cons                         |
| -------- | --------------------------- | ---------------------------- |
| URL path | Explicit, cacheable, simple | URL changes between versions |
| Header   | Clean URLs                  | Hidden, harder to test       |
| Accept   | RESTful                     | Complex, less discoverable   |

**Recommendation**: Use URL path versioning for simplicity and discoverability.

---

## When to Version

Create a new version when:

- Removing or renaming fields
- Changing field types
- Changing response structure
- Removing endpoints

Don't version for:

- Adding optional fields
- Adding new endpoints
- Bug fixes

---

## Deprecation Strategy

```java
@Path("/v1/users")
@Deprecated
@Tag(name = "Users (Deprecated)", description = "Use /v2/users instead")
public class UserResourceV1 {

    @GET
    @Path("/{id}")
    @Operation(
        summary = "Get user",
        deprecated = true,
        description = "Deprecated. Use GET /v2/users/{id}"
    )
    public Response get(@PathParam("id") Long id) {
        return Response.ok(user)
            .header("Deprecation", "true")
            .header("Sunset", "2025-06-01")
            .header("Link", "</v2/users/" + id + ">; rel=\"successor-version\"")
            .build();
    }
}
```

### Deprecation Headers

| Header        | Purpose                       |
| ------------- | ----------------------------- |
| `Deprecation` | Signals API is deprecated     |
| `Sunset`      | Date when API will be removed |
| `Link`        | Points to new version         |

---

## Shared Logic Between Versions

```java
@ApplicationScoped
public class UserService {

    public User findById(Long id) {
        return userRepository.findById(id);
    }
}

// Both versions use same service
@Path("/v1/users")
public class UserResourceV1 {
    @Inject UserService userService;
}

@Path("/v2/users")
public class UserResourceV2 {
    @Inject UserService userService;
}
```

---

## Related Recipes

- [Endpoint Design](./endpoint-design.md): URL structure
- [OpenAPI Documentation](./openapi-documentation.md): Document versions
