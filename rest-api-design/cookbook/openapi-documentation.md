# MicroProfile OpenAPI Documentation

## Intent

Document REST APIs using MicroProfile OpenAPI annotations. Generate OpenAPI/Swagger specifications automatically.

---

## Basic Annotations

```java
@Path("/users")
@Tag(name = "Users", description = "User management operations")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    @GET
    @Path("/{id}")
    @Operation(summary = "Get user by ID", description = "Returns a single user")
    @APIResponse(responseCode = "200", description = "User found",
        content = @Content(schema = @Schema(implementation = User.class)))
    @APIResponse(responseCode = "404", description = "User not found")
    public User getById(@PathParam("id") Long id) {
        return userRepository.findById(id);
    }
}
```

---

## @Operation

Document endpoint purpose:

```java
@POST
@Operation(
    summary = "Create a new user",
    description = "Creates a user account and returns the created user with generated ID"
)
public Response create(User user) { }
```

---

## @APIResponse

Document possible responses:

```java
@POST
@APIResponse(responseCode = "201", description = "User created",
    content = @Content(schema = @Schema(implementation = User.class)),
    headers = @Header(name = "Location", description = "URL of created user"))
@APIResponse(responseCode = "400", description = "Invalid input",
    content = @Content(schema = @Schema(implementation = ValidationErrorResponse.class)))
@APIResponse(responseCode = "409", description = "User already exists")
public Response create(User user) { }
```

---

## @Schema

Document data models:

```java
@Schema(description = "User account information")
public record User(
    @Schema(description = "Unique identifier", example = "123", readOnly = true)
    Long id,

    @Schema(description = "User's full name", example = "John Doe", required = true)
    String name,

    @Schema(description = "Email address", example = "john@example.com", required = true)
    String email,

    @Schema(description = "Account status", defaultValue = "ACTIVE")
    UserStatus status
) {}

@Schema(description = "User account status")
public enum UserStatus {
    @Schema(description = "Active user")
    ACTIVE,
    @Schema(description = "Suspended user")
    SUSPENDED,
    @Schema(description = "Deleted user")
    DELETED
}
```

---

## @Parameter

Document path and query parameters:

```java
@GET
@Path("/{id}")
public User getById(
    @Parameter(description = "User ID", required = true, example = "123")
    @PathParam("id") Long id
) { }

@GET
public List<User> list(
    @Parameter(description = "Filter by status")
    @QueryParam("status") UserStatus status,

    @Parameter(description = "Page number", example = "0")
    @QueryParam("page") @DefaultValue("0") int page,

    @Parameter(description = "Page size", example = "20")
    @QueryParam("size") @DefaultValue("20") int size
) { }
```

---

## @RequestBody

Document request body:

```java
@POST
@Operation(summary = "Create user")
public Response create(
    @RequestBody(
        description = "User to create",
        required = true,
        content = @Content(schema = @Schema(implementation = CreateUserRequest.class))
    )
    CreateUserRequest request
) { }
```

---

## @Tag

Group related endpoints:

```java
@Tag(name = "Users", description = "User management")
@Tag(name = "Admin", description = "Administrative operations")
@Path("/admin/users")
public class AdminUserResource { }
```

---

## @SecurityScheme and @SecurityRequirement

Document authentication:

```java
@SecurityScheme(
    securitySchemeName = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT"
)
@ApplicationPath("/api")
public class RestApplication extends Application { }

@Path("/users")
@SecurityRequirement(name = "bearerAuth")
public class UserResource { }
```

---

## Complete Example

```java
@Path("/products")
@Tag(name = "Products", description = "Product catalog operations")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@SecurityRequirement(name = "bearerAuth")
public class ProductResource {

    @GET
    @Operation(summary = "List products", description = "Returns paginated product list")
    @APIResponse(responseCode = "200", description = "Products retrieved",
        content = @Content(schema = @Schema(implementation = ProductListResponse.class)))
    public PaginatedResponse<Product> list(
        @Parameter(description = "Filter by category")
        @QueryParam("category") String category,

        @Parameter(description = "Minimum price")
        @QueryParam("minPrice") BigDecimal minPrice,

        @Parameter(description = "Page number")
        @QueryParam("page") @DefaultValue("0") int page,

        @Parameter(description = "Page size")
        @QueryParam("size") @DefaultValue("20") int size
    ) { }

    @GET
    @Path("/{id}")
    @Operation(summary = "Get product by ID")
    @APIResponse(responseCode = "200", description = "Product found")
    @APIResponse(responseCode = "404", description = "Product not found")
    public Product getById(
        @Parameter(description = "Product ID", required = true)
        @PathParam("id") Long id
    ) { }

    @POST
    @Operation(summary = "Create product")
    @APIResponse(responseCode = "201", description = "Product created")
    @APIResponse(responseCode = "400", description = "Invalid input")
    public Response create(
        @RequestBody(required = true)
        CreateProductRequest request
    ) { }
}
```

---

## Accessing OpenAPI Spec

Quarkus exposes the spec at:

- **JSON**: `/q/openapi`
- **YAML**: `/q/openapi?format=yaml`
- **Swagger UI**: `/q/swagger-ui`

---

## Related Recipes

- [Status Codes](./status-codes.md): Which codes to document
- [Filtering & Pagination](./filtering-pagination.md): Query param patterns
