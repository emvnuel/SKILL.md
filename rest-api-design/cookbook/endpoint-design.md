# Endpoint Design

## Intent

Design intuitive, consistent REST endpoints using nouns, plural names, and appropriate nesting.

---

## Use Nouns, Not Verbs

HTTP methods already describe the action. Endpoints should name the resource.

```java
// ❌ Bad - verbs in path
@GET @Path("/getUsers")
@POST @Path("/createUser")
@PUT @Path("/updateUser/{id}")
@DELETE @Path("/deleteUser/{id}")

// ✓ Good - nouns only
@GET @Path("/users")
@POST @Path("/users")
@PUT @Path("/users/{id}")
@DELETE @Path("/users/{id}")
```

---

## Plural Nouns for Collections

Always use plural, even for single resource access.

```java
@Path("/users")
public class UserResource {

    @GET
    public List<User> list() { }      // GET /users

    @GET
    @Path("/{id}")
    public User getById(
        @PathParam("id") Long id
    ) { }                              // GET /users/123

    @POST
    public Response create(User user) { }  // POST /users

    @PUT
    @Path("/{id}")
    public User update(
        @PathParam("id") Long id,
        User user
    ) { }                              // PUT /users/123

    @DELETE
    @Path("/{id}")
    public void delete(
        @PathParam("id") Long id
    ) { }                              // DELETE /users/123
}
```

---

## Resource Nesting

Nest related resources to show relationships. Limit to 3 levels max.

```java
// User's orders
@Path("/users/{userId}/orders")
public class UserOrderResource {

    @GET
    public List<Order> list(
        @PathParam("userId") Long userId
    ) { }                              // GET /users/123/orders

    @POST
    public Response create(
        @PathParam("userId") Long userId,
        Order order
    ) { }                              // POST /users/123/orders
}

// Post's comments
@Path("/posts/{postId}/comments")
public class PostCommentResource {

    @GET
    public List<Comment> list(
        @PathParam("postId") Long postId
    ) { }                              // GET /posts/456/comments
}
```

### Avoid Deep Nesting

```java
// ❌ Too deep (4 levels)
/users/{userId}/orders/{orderId}/items/{itemId}/reviews

// ✓ Flatten when possible
/order-items/{itemId}/reviews
```

---

## Path Parameter Naming

Use camelCase or kebab-case consistently.

```java
// camelCase (common in Java)
@Path("/users/{userId}/orders/{orderId}")

// kebab-case (common in URLs)
@Path("/order-items/{itemId}")
```

---

## Action Endpoints (Exception)

For non-CRUD operations, actions can use verbs as sub-resources.

```java
// Actions on a resource
@POST @Path("/orders/{id}/cancel")     // Cancel order
@POST @Path("/orders/{id}/ship")       // Ship order
@POST @Path("/users/{id}/activate")    // Activate user
```

---

## Related Recipes

- [HTTP Methods](./http-methods.md): When to use GET, POST, PUT, DELETE
- [Status Codes](./status-codes.md): Response codes for each operation
