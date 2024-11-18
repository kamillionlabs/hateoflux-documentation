---
title: "Link Building"
layout: default
parent: "Core Concepts"
nav_order: 2
---
# Link Building
{: .no_toc }
<br>
1. TOC
{:toc}
---

Building links is a fundamental aspect of creating hypermedia-driven APIs with hateoflux. This section covers how to build links manually, use URI templates, and leverage the `SpringControllerLinkBuilder` for seamless integration with Spring WebFlux.

## Manually Building Links

The `Link` class in hateoflux allows the creation and customization of links for API resources. Links can be built manually by specifying the `href` and adding additional attributes as needed.

### Creating a Simple Link

To create a simple link, the static `Link.of()` method is used:

```java
Link link = Link.of("/orders/12345");
```

### Adding Relations
Specifying the relationship between the current resource and the linked resource is done by using the `withRel()` method. This sets the `rel` attribute of the link, which describes how the linked resource is related to the current one.

```java
Link link = Link.of("/orders/12345")
                .withRel("self");
```

Alternatively, a predefined IANA relations from the `IanaRelation` enum can be used:

```java
Link link = Link.of("/orders/12345")
                .withRel(IanaRelation.SELF);
```

### Using the `slash()` Method
The `slash()` method appends URI parts to the existing href, ensuring proper formatting with slashes. Additionally, this method is handy because it automatically handles double slashes. Whether the base link already ends with a slash or not, it makes no difference. The same applies to the URI part provided as an argument to `slash()`—any extra slashes are managed by the method, ensuring that the resulting URL is correctly formatted without redundant slashes. This robustness is especially useful for building links dynamically.

```java
Link baseLink = Link.of("/orders");           //alternatively "/orders/"
Link detailedLink = baseLink.slash("12345");  //alternatively "/12345"

System.out.println(detailedLink.getHref()); 

// Will always output: /orders/12345
```

### Chaining Methods
The `Link` class methods support chaining, allowing links to be built and customized in a fluent manner.

```java
Link link = Link.of("/orders")
                .slash("12345")
                .withRel(IanaRelation.SELF)
                .withTitle("Order Details")
                .withType("application/json");
```

## Using URI Templates
hateoflux supports URI templates, enabling the definition of dynamic URLs with placeholders that can be expanded with actual values. URI templates are particularly useful for links that require query parameters or optional path variables.

### Defining a URI Template
A URI template can be defined as follows:

```java 
Link templatedLink = Link.of("/orders/{userId}/shipments{?status,page,size}");
```

hateoflux supports URI templates following [RFC6570](https://www.rfc-editor.org/rfc/rfc6570) where it makes sense. The `UriExpander` or even easier, the `Link.expand()` method allows you to expand URI templates with provided parameters. The following are the supported formats:

 * **Mandatory Variables**: Placeholders like `{var}` represent mandatory variables and generally used for path variables.
 * **Optional Query Parameters**: Placeholders like `{?var}` represent optional query parameters.
 * **Multiple Query Parameters**: `{?var1,var2}` allows for multiple optional query parameters.
 * **Exploded Query Parameters**: `{?var*}` can represent a list of values for a query parameter.

The `expand()` method handles the expansion of these templates, replacing placeholders with actual values or ignoring them if no value is provided (for optional parameters). Collections can be expanded as exploded query parameters, either in composite (`?var=1,2`) or non-composite (`?var=1&var=2`) form, depending on your configuration.

### Expanding URI Templates
The `expand()` method can expand the URI template using an array of values (varargs) or a map of parameters. Reusing the `templatedLink` from above, an example expansion could look like this:

```java
Map<String, Object> parameters = Map.of(
    "userId", "123456789"
    "status", "shipped",
    "page", 0,
    "size", 10
);

Link expandedLink = templatedLink.expand(parameters);

System.out.println(expandedLink.getHref()); 

// Outputs: /orders/123456789/shipments?status=shipped&page=0&size=10
```

### Handling Optional and Exploded Parameters
* **Optional Parameters**: Denoted by `{?param}`, optional parameters are omitted if no value is provided.
* **Exploded Parameters**: Denoted by `{?param*}`, these are used for lists or arrays of values. For expansion, they always need to be in a `List`.

The following is another example using exploded variables:

```java
Map<String, Object> params = Map.of(
    "tags", List.of("new", "sale"),
    "category", "electronics"
);

Link link = Link.of("/products{?tags*,category}")
    .expand(params);

System.out.println(link.getHref()); 

// Outputs: /products?tags=new,sale&category=electronics
```

## Using `SpringControllerLinkBuilder`

### Introduction
hateoflux is designed to be used with Spring WebFlux, and the `SpringControllerLinkBuilder` class provides a straightforward way to create links pointing to controller methods in a type-safe and annotation-aware manner. As a counterpart to Spring HATEOAS's `MvcLinkBuilder` and `WebFluxLinkBuilder`, the `SpringControllerLinkBuilder` works on the same principles but differs slightly in usage.

The `SpringControllerLinkBuilder` integrates seamlessly with Spring annotations, such as `@Controller`, `@RestController`, `@GetMapping`, `@PathVariable`, and `@RequestParam`.

### Example Usage
Consider the following controller:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        // ...
    }
}
```
A link to the `getUser()` method can be created using `SpringControllerLinkBuilder`:

```java
import static de.kamillionlabs.hateoflux.linkbuilder.SpringControllerLinkBuilder.linkTo;

Link userLink = linkTo(UserController.class, controller -> controller.getUser("12345"));

System.out.println(userLink.getHref()); 

// Outputs: /users/12345
```
### How it Works
* **Type-Safe Method References**: By passing a method reference to `linkTo()`, links are built corresponding to an actual controller method, which helps catch errors at compile time.
* **Annotation Awareness**: The builder reads Spring annotations to construct the correct path and parameters.
  * `@RequestMapping`: Extracts base paths from the controller.
  * `@GetMapping`, `@PostMapping`, etc.: Determines the method-specific paths.
  * `@PathVariable`: Populates path variables in the URL.
  * `@RequestParam`: Adds query parameters to the URL.
* **Parameter Handling**: Supports both single values and collections for parameters, with optional composite expansion.
 
### Adding Relations and Other Attributes
Relations and other attributes can be added after creating the link:
```java
Link userLink = linkTo(UserController.class, controller -> controller.getUser("12345"))
    .withRel(IanaRelation.SELF)
    .withTitle("User Details");

System.out.println(userLink.getHref()); 
// Outputs: /users/12345
```

### Benefits
* **Maintainability**: Links automatically reflect changes in controller paths or method signatures.
* **Consistency**: Reduces the risk of errors in manual URL construction.
* **Type Safety**: Ensures that links correspond to existing methods, preventing broken links.
* **Flexibility**: Supports optional parameters and complex path structures.

### Additional Examples
#### Generating Links with Query Parameters
Given a controller method:
```java
@GetMapping("/{id}/orders")
public Mono<Order> getUserOrders(@PathVariable String id, @RequestParam String status) {
    // ...
}
```

A link with query parameters can be created as follows:

```java
Link ordersLink = linkTo(UserController.class, controller -> controller.getUserOrders("12345", "shipped"));

System.out.println(ordersLink.getHref()); 
// Outputs: /users/12345/orders?status=shipped
```
#### Handling Collections in Parameters

If a controller method accepts collections:
```java
@GetMapping("/search")
public Mono<List<User>> searchUsers(@RequestParam List<String> roles) {
// ...
}
```

A link with collection parameters can be created as follows:

```java
Link searchLink = linkTo(UserController.class, controller -> controller.searchUsers(List.of("admin", "user")));

System.out.println(searchLink.getHref()); 
// Outputs: /users/search?roles=admin,user
```

### Note on Composite Parameters
In contrast to Spring HATEOAS, collections are expanded in a non-composite way (e.g., `roles=admin,user`). If annotated with `@Composite`, the builder will render them in a composite way (e.g., `roles=admin&roles=user`). This is the negated behaviour of Spring's `@NonComposite`. 