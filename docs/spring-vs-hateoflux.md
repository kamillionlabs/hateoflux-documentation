---
title: "Spring HATEOAS vs. hateoflux"
layout: default
nav_order: 2
---

# Spring HATEOAS vs. hateoflux
{: .no_toc }
This section provides a side-by-side comparison of hateoflux and Spring HATEOAS, showcasing areas where each library excels, aligns, or differs. By exploring key differences, unique advantages, and unsupported features, this guide helps you make informed decisions for building hypermedia APIs in reactive Spring applications.
<br>
1. TOC
{:toc}
---

## Representation Models
In hypermedia-driven APIs, the way resources are represented and structured is fundamental to ensuring seamless data interaction between the server and client. Spring HATEOAS and hateoflux adopt different approaches to handling these representations, with Spring utilizing "Models" and hateoflux employing "Wrappers." This distinction is pivotal in understanding how each framework manages resource enrichment and hypermedia controls.

### Spring HATEOAS
Spring HATEOAS leverages representation models to encapsulate resources along with their hypermedia links. The primary classes used are `RepresentationModel` for individual resources and `CollectionModel` for collections of resources. Developers often extend these classes to include additional attributes and links, facilitating a flexible and extensible way to build resource representations.

For example, an `OrderModel` might be defined as follows:
```java
public class OrderModel extends RepresentationModel<OrderModel> {
    private String orderId;
    private String status;
    
    // Getters and setters
}
```
By extending `RepresentationModel`, the `OrderModel` gains the ability to hold hypermedia links inherently. This inheritance-based approach allows for the easy addition of links and embedded resources, promoting a clear and organized structure for resource representations. However, this method can lead to a more complex class hierarchy, especially when dealing with multiple resource types and varying representations.

### hateoflux
In contrast, hateoflux utilizes wrappers to enhance domain objects with hypermedia links and embedded resources. The key classes provided by hateoflux include `HalResourceWrapper`, `HalListWrapper`. These wrappers are designed to be composed around existing domain models instead of inherited from them, ensuring that the original models remain clean and decoupled from hypermedia concerns.

For instance, an `Order` can be wrapped using `HalResourceWrapper` as shown below:
```java
HalResourceWrapper<Order, Void> wrap = HalResourceWrapper.wrap(order);
```
By using wrappers, hateoflux maintains a clear separation between the domain models and their hypermedia representations. The concept of main and embedded resources is seamlessly integrated into the wrapping process. The `ResourceT` type represents the primary resource, while the `EmbeddedT` denotes any secondary resources that are embedded within the main resource. When no embedded resource is required, `EmbeddedT` is set to `Void`.

For example, when an order includes shipment details, the wrapper would encapsulate both the `Order` and the `Shipment` as an embedded resource:

```javascript
{
  "id": 12345,
  "status": "Shipped",
  "_embedded": {
    "shipment": {
      "id": 98765,
      "carrier": "UPS",
      "trackingNumber": "1Z999AA10123456784",
      "status": "In Transit"
    }
  },
  "_links": {
    "self": { "href": "http://localhost/orders/12345" }
  }
}
```
_**See also:**_
* [Fundamentals about resource wrappers](./core-concepts/representation-model.html#resource-wrappers)
* [Explained full example on how to wrap a resource](./cookbook.html#creating-a-halresourcewrapper-without-an-embedded-resource)
* [Explained full example on how to wrap a resource with another embedded resource](./cookbook.html#creating-a-halresourcewrapper-with-an-embedded-resource)

## Assemblers and Boilerplate Code
### Spring HATEOAS
Spring HATEOAS provides reactive assemblers like `ReactiveRepresentationModelAssembler` to help assemble resources in reactive applications. However, these assemblers are more skeletons than actual helpful Assemblers as opposed to their non-reactive counterparts. Implementing them is therefore verbose and may require additional boilerplate code. Developers need to manually create `EntityModel` instances and add links for each resource, which can clutter the code.

An implementation of an assembler might look like this:
```java
@Component
public class ProductAssembler implements ReactiveRepresentationModelAssembler<Product, EntityModel<Product>> {

    @Override
    public Mono<EntityModel<Product>> toModel(Product product, ServerWebExchange exchange) {
        return WebFluxLinkBuilder.linkTo(
                        WebFluxLinkBuilder.methodOn(ProductController.class).getProduct(product.getId()), exchange)
                .withSelfRel()
                .toMono()
                .map(link -> EntityModel.of(product).add(link));
    }
}
```
This looks acceptable until a more complex structure is needed, where other resources might be required. Given the example above, we might want to embed a `ShipmentDetail`. Spring unfortunately does not help you here, and the embedding needs to be done manually in the `Product` class. In general, we see:
* **Verbosity**: Requires manual resource wrapping and link addition in assemblers.
* **Boilerplate Code**: Repetitive patterns for essentially the same logic that need to be implemented in every assembler. This increases maintenance overhead.

### hateoflux
In contrast, hateoflux's assemblers provide built-in methods to wrap a single or a list of resources. The only thing to implement is the part that is different from one resource to another: the links. The following is an implementation of `ReactiveFlatHalWrapperAssembler`. The end result is the same as above:

```java
import de.kamillionlabs.hateoflux.linkbuilder.SpringControllerLinkBuilder;

@Component
public class ProductAssembler implements ReactiveFlatHalWrapperAssembler<Product> {

    @Override
    public Link buildSelfLinkForResource(Product product, ServerWebExchange exchange) {
        return SpringControllerLinkBuilder
                .linkTo(ProductController.class, controller -> controller.getProduct(product.getId()))
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    Link buildSelfLinkForResourceList(ServerWebExchange exchange) {
        //Also needs to be implemented
    }
}
```
This appears similar to Spring; however, the important difference is that no "Models" or "Wrappers" are created. Instead, the self link for any `Product` or a list of them is defined. The wrapping itself is managed by the assembler. Usage in a controller is then as simple as:

```java
@Autowired
ProductAssembler productAssembler;

@GetMapping("/{id}")
public Mono<HalResourceWrapper<Product, Void>> getProduct(@PathVariable String id, ServerWebExchange exchange) {
    Mono<Product> product = productService.findById(id);
    return productAssembler.wrapInResourceWrapper(product, exchange);
}
```
If a situation arises where an embedded resource (e.g., `ShipmentDetail` as above) is required, the `ReactiveEmbeddingHalWrapperAssembler` can be implemented. Here as well, only the logic for creating the self links needs to be provided:

```java
import de.kamillionlabs.hateoflux.linkbuilder.SpringControllerLinkBuilder;

@Component
public class ProductAssembler implements ReactiveEmbeddingHalWrapperAssembler<Product, ShipmentDetail> {

    @Override
    public Link buildSelfLinkForResource(Product product, ServerWebExchange exchange) {
        return SpringControllerLinkBuilder
                .linkTo(ProductController.class, controller -> controller.getProduct(product.getId()))
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    Link buildSelfLinkForEmbedded(ShipmentDetail shipmentDetail, ServerWebExchange exchange) {
        return Link.of("/shipment").slash(shipmentDetail.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }
}
```
The usage remains straightforward:

```java
@Autowired
ProductAssembler productAssembler;

@GetMapping("/{id}")
public Mono<HalResourceWrapper<Product, ShipmentDetail>> getProduct(@PathVariable String id,
                                                                    ServerWebExchange exchange) {
    Mono<Product> product = productService.findById(id);
    Mono<ShipmentDetail> shipment = shipmentService.findByProductId(id);
    return productAssembler.wrapInResourceWrapper(product, shipment, exchange);
}
```
As demonstrated, adding an embedded resource (or a list of them if required) is accomplished with a simple one-liner.

_**See also:**_
* [Fundamentals about assembler](./core-concepts/assemblers.html)
* [Explained full example implementation of an assembler](./cookbook.html#using-an-assembler-to-create-a-hallistwrapper-for-resources-with-an-embedded-resource)

## Pagination Handling
### Spring HATEOAS
In Spring MVC, the `PagedResourcesAssembler` helps create `PagedModel` instances with pagination metadata and navigation links (`prev`, `next`). However, in reactive environments using WebFlux and R2DBC, paging support is limited:

* **Repositories Return `Flux<T>`**: With R2DBC, repositories return `Flux<T>` instead of `Page<T>`.
* **No Native Paging Support**: The lack of `Page<T>` means tools like `PagedResourcesAssembler` are not available in WebFlux.
* **Manual Implementation Required**: Developers must manually assemble pagination metadata and navigation links.

### hateoflux
Hateoflux simplifies pagination in reactive environments by providing `HalListWrapper`, which encapsulate pagination metadata and handle navigation links automatically.

Given we implemented the `ProductAssembler` i.e. simply overriding how and what links to create for a single and a list of `Product`s, adding pagination can be added as simple as below:

```java
import static de.kamillionlabs.hateoflux.utility.SortDirection.ASCENDING;
import static de.kamillionlabs.hateoflux.utility.SortDirection.DESCENDING;
import org.springframework.data.domain.Pageable;

@GetMapping
public Mono<HalListWrapper<Product, Void>> getProducts(Pageable pageable,
                                                       ServerWebExchange exchange) {
    //Note: Pageable is not required!
    int pageSize = pageable.getPageSize();
    long offset = pageable.getOffset();
    List<SortCriteria> sortCriteria = pageable.getSort().get()
            .map(o -> SortCriteria.by(o.getProperty(), o.getDirection().isAscending() ? ASCENDING : DESCENDING))
            .toList();

    Flux<Product> productFlux = productService.findAll(pageSize, pageNumber);
    Mono<Long> totalElements = productService.countAll();

    return productAssembler.wrapInListWrapper(productFlux, totalElements, pageSize, pageNumber, sortCriteria, exchange);
}
```
The `wrapInListWrapper` will automatically append configured links in the assembler (like self link and others if specified), add paging information, and add the navigation links `next`, `prev`, `first`, and `last`.

_**See also:**_
* [Fundamentals about Pagination](./core-concepts/representation-model.html#pagination)
* [Explained full example on how to wrap a list of resources with pagination](./cookbook.html#creating-a-hallistwrapper-with-pagination)

## Representation Model Processors
Spring HATEOAS provides Representation Model Processors allowing developers to adjust hypermedia responses globally or conditionally. In contrast, hateoflux incorporates this functionality within its assemblers, combining the responsibilities of both assemblers and processors from Spring HATEOAS into a single, cohesive unit.

### Spring HATEOAS
With Spring HATEOAS, `RepresentationModelProcessor` allows modifications to be applied to a resource representation post-assembly. This approach ensures that the logic for adding additional links or modifying the representation can be handled without modifying the core controller or assembler.

Suppose an `Order` resource could have a link added to initiate payment without embedding the logic in the `OrderController` or `OrderAssembler`. The implementation could look like so:

```java
@Component
public class PaymentProcessor implements RepresentationModelProcessor<EntityModel<Order>> {

    @Override
    public EntityModel<Order> process(EntityModel<Order> model) {
        Order order = model.getContent();
        if (order != null && "AWAITING_PAYMENT".equals(order.getState())) {
            model.add(Link.of("/payments/{orderId}")
                    .withRel(LinkRelation.of("payments")) 
                            .expand(model.getContent().getOrderId()));
        }
        return model;
    }
}
```
The processor is executed automatically by spring effectively postprocessing every `EntitiyModel<Order>`. It is therefore not required to change the controller or the assembler class.

### hateoflux
Assemblers in hateoflux are used to encapsulate the creation of resource representations and the addition of hypermedia links, combining the functionality of both Spring HATEOAS assemblers and processors. This consolidated approach allows link-building logic and resource modifications to be centralized within the assembler, simplifying the development process.

To mimic the functionality of Spring described above, an `OrderAssembler` would look like the following:

```java
@Component
public class OrderAssembler implements ReactiveFlatHalWrapperAssembler<Order> {

    @Override
    public Link buildSelfLinkForResource(Order order, ServerWebExchange exchange) {
        return Link.of("/orders/" + order.getOrderId())
                   .withSelfRel()
                   .prependBaseUrl(exchange);
    }

    @Override
    public List<Link> buildOtherLinksForResource(Order order, ServerWebExchange exchange) {
        List<Link> links = new ArrayList<>();

        // A payment link is added if the order is awaiting payment
        if ("AWAITING_PAYMENT".equals(order.getState())) {
            links.add(Link.of("/payments/" + order.getOrderId())
                          .withRel("payment")
                          .prependBaseUrl(exchange));
        }

        return links;
    }

    @Override
    public Link buildSelfLinkForResourceList(ServerWebExchange exchange) {
        return Link.of("/orders")
                   .withSelfRel()
                   .prependBaseUrl(exchange);
    }
}
```
The assembler handles both the wrapping of resources and the addition of other links. The wrapping is done in the `ReactiveFlatHalWrapperAssembler` and does not need to be reimplemented.

Similar to Spring, the controller stays unchanged by the types of links added.

## Resource Naming
Both Spring HATEOAS and hateoflux use the `@Relation` annotation with the same semantics, allowing developers to customize serialization names of resources.

This annotation controls how resources and collections are named in the serialized output, ensuring consistency with domain terminology.

**Example:**
```java
@Relation(itemRelation = "book", collectionRelation = "books")
public class BookDTO {
    private String title;
    private String author;
    // Getters and setters
}
```

**Serialized Output:**
```json
{
  "_embedded": {
    "books": [
      {
        "title": "Effective Java",
        "author": "Joshua Bloch"
      }
    ]
  }
}
```
In conclusion, there is no difference.

## Handling Collections
When assembling collections of resources, Spring HATEOAS requires collecting a `Flux` into a List using `collectList()`. This is necessary for creating `CollectionModel` instances but may have performance implications with large datasets due to loading all elements into memory. See e.g. the default implementation in the `ReactiveRepresentationModelAssembler`:

```java
 default Mono<CollectionModel<D>> toCollectionModel(Flux<? extends T> entities, ServerWebExchange exchange) {
    return entities.flatMap((entity) -> {
        return this.toModel(entity, exchange);
    }).collectList() // <---- here
            .map(CollectionModel::of);
}
```
This is a necessary evil, or rather a trade-off, for being able to gather multiple resources together and assign information to them. In hateoflux the pagination or linking for a given list of resources, relies also on `collectList()`.

{: .important }
It is important to be mindful of the performance implications that `collectList()` may have. Returning lists or pages should be done with caution.

## Type Safety and Link Building
Spring HATEOAS provides type-safe link building using `WebFluxLinkBuilder`s `methodOn()` and `linkTo()`. It effectively handles path variables and query parameters. hateoflux provides a similar option and is also able to distinguish between query and path variables.

**Example usage in Spring HATEOAS:**
```java
Mono<Link> selfLink = WebFluxLinkBuilder.linkTo(
        WebFluxLinkBuilder.methodOn(UserController.class).getUser(id))
        .withSelfRel()
        .toMono();
```
**Example usage in hateoflux:**
```java
import de.kamillionlabs.hateoflux.linkbuilder.SpringControllerLinkBuilder;

Link selfLink = SpringControllerLinkBuilder.linkTo(
        UserController.class, controller -> controller.getUser(id))
        .withSelfRel();
```

{: .note }
Note that hateoflux provides a `Link` immediately whereas Spring's `WebFluxLinkBuilder` is only able to return a `Mono<Link>`. If `toMono()` is not called, the process remains in the builder.


_**See also:**_
* [Fundamentals about linkbuilding](./core-concepts/linkbuilding.html)
* [How the `SpringControllerLinkBuilder` is used](./core-concepts/linkbuilding.html#using-springcontrollerlinkbuilder)

## Documentation and WebFlux Support
Spring HATEOAS was primarily developed for Spring MVC applications. Although it does provide support for Spring WebFlux, the documentation and examples are more comprehensive for MVC, which can pose challenges for developers building hypermedia-driven APIs in reactive environments.

In contrast, hateoflux is specifically designed for reactive applications using Spring WebFlux. It offers targeted documentation and examples tailored to reactive programming, making the development process more straightforward. Additionally, hateoflux does not carry the extensive baggage that Spring HATEOAS has, making it smaller and lighter both in size and in terms of ease of use, overview, and comprehension.

**Advantages:**
* **Reactive-First Design**: Optimized for reactive applications.
* **Focused Documentation**: Comprehensive guidance for WebFlux reduces the learning curve.
* **Enhanced Developer Experience**: Simplifies implementation in reactive environments.
* **Lightweight Framework**: Smaller in size and easier to understand compared to Spring HATEOAS.

## Media Types

{: .highlight }
TL;DR hateoflux only supports HAL+JSON

A significant distinction between Spring HATEOAS and hateoflux is their support for various media types. **Spring HATEOAS offers extensive support for multiple media types**, including HAL, Collection+JSON, and others. This flexibility allows developers to cater to diverse client requirements and standards, enabling richer interactions and broader compatibility across different API consumers. However, supporting multiple media types introduces additional complexity and abstraction layers, which can make the implementation more cumbersome and harder to maintain.

In contrast, **hateoflux is intentionally designed to support only HAL+JSON**. This focused approach aligns with hateoflux’s goal of being a small, concise library that covers the most essential features needed for building hypermedia-driven APIs in reactive environments. By limiting support to HAL+JSON, hateoflux simplifies the development process, offering a more straightforward and direct integration with Spring WebFlux and existing Spring projects. This design choice reduces overhead and makes the library easier to use for developers who primarily require HAL+JSON for their APIs.

While hateoflux excels in providing a streamlined and efficient solution for HAL+JSON-based hypermedia APIs, Spring HATEOAS retains an edge in scenarios where support for multiple media types is essential. Developers needing advanced hypermedia features and broader media type compatibility might prefer Spring HATEOAS despite its increased complexity. Conversely, for projects that prioritize simplicity, performance, and are content with HAL+JSON, hateoflux offers a more intuitive and maintainable alternative.

## Affordance

{: .highlight }
TL;DR hateoflux does not supports affordance

**Spring HATEOAS supports affordances**, a powerful feature that allows developers to define actions that clients can perform on resources. By using affordances, developers can enrich API responses with metadata describing potential actions, such as form fields for data input or additional details on how a client might interact with a resource. This makes APIs more self-descriptive, enabling clients to discover possible operations dynamically without requiring hardcoded knowledge of the server’s capabilities. For example, affordances can define input parameters or constraints directly within the response, making it clear what actions a client is permitted to take.

In contrast, **hateoflux does not support affordances**, aligning with its goal of being a lightweight, simplified library that focuses on core hypermedia functionality. By omitting affordances, hateoflux remains compact and direct, prioritizing essential HATEOAS elements such as resource wrapping, link building, and pagination for HAL+JSON representations. This approach benefits developers who require a straightforward, low-overhead solution and can handle any additional action-specific metadata outside the API response itself, such as through separate documentation or client logic.

## CURIE Support

{: .highlight }
TL;DR hateoflux does not supports CURIE

Spring HATEOAS includes support for CURIEs (Compact URIs), which enable the use of namespaced relation types in hypermedia representations. CURIEs simplify the representation of custom relation types, making APIs more readable and organized. This feature is valuable in complex APIs where consistent referencing of custom link relations is necessary, allowing for clearer client interpretation and easier navigation.

At present, **hateoflux does not support CURIEs**. While this is a beneficial feature for managing custom relation types, it has not been implemented in hateoflux as it has not been prioritized for current releases. Despite its absence, CURIE support is recognized as a useful enhancement, and it remains a potential addition for future versions of the library.

## Summary Table of Features
The following table summarizes the main comparisons between Spring HATEOAS and hateoflux to highlight their strengths, limitations, and areas of alignment when building hypermedia-driven APIs in reactive Spring applications:

| **Aspect**                          |                                                             **Spring HATEOAS (in WebFlux)**                                                             |                                           **hateoflux**                                           |
|-------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------:|:-------------------------------------------------------------------------------------------------:|
| **Representation Models**           |                                     ⚠️<br/>Uses inheritance-based models, can lead to complex objects/hierarchies.                                      |                  ✅<br/>Uses wrappers, keeping domain models clean and decoupled.                  |
| **Assemblers and Boilerplate Code** |                              ❌<br/>Verbose with manual resource wrapping and link addition; more boilerplate code needed.                               |      ✅<br/>Simplified with built-in methods; only links need to be specified in assemblers.       |
| **Pagination Handling**             |                                 ❌<br/>Limited or non-existing in reactive environments; requires manual implementation.                                 | ✅<br/>Easy pagination with `HalListWrapper`; handles metadata and navigation links automatically. |
| **Representation Model Processors** |                  ✅<br/>Supports processors (`RepresentationModelProcessor`) to adjust hypermedia responses globally or conditionally.                   |       ⚠️<br/>Functionality is incorporated within assemblers; processors are not separate.        |
| **Resource Naming**                 |                                               ✅<br/>Uses `@Relation` for customizing serialization names.                                               |             ✅<br/>Also uses `@Relation` with identical functionality; no difference.              |
| **Handling Collections**            |                                   ⚠️<br/>Requires `collectList()`, which can impact performance with large datasets.                                    |           ⚠️<br/>Also requires `collectList()`, with similar performance implications.            |
| **Type Safety & Link Building**     |                                ✅<br/>Type-safe link building using `WebFluxLinkBuilder`; links returned as `Mono<Link>`.                                |   ✅<br/>Type-safe link building; provides `Link` immediately without wrapping in `Mono<Link>`.    |
| **Documentation Support**           |                           ❌<br/>Better for Spring MVC; less comprehensive for WebFlux, challenging for reactive development.                            |        ✅<br/>Tailored for reactive Spring WebFlux with focused documentation and examples.        |
| **Media Types**                     |                                 ✅<br/>Supports multiple media types (HAL, Collection+JSON, etc.), offering flexibility.                                 |  ⚠️<br/>Only supports HAL+JSON; simpler but less flexible for clients needing other media types.  |
| **Affordance**                      |                             ✅<br/>Supports affordances, enabling clients to discover actions they can perform on resources.                             |                                ❌<br/>Does not support affordances                                 |
| **CURIE Support**                   |                                           ✅<br/>Supports CURIEs (Compact URIs) for namespaced relation types.                                           |                              ❌<br/>Does not support CURIEs currently                              |
| **Framework Weight**                |                                      ⚠️<br/>Heavier with more extensive features; may add complexity and overhead.                                      |   ✅<br/>Lightweight and easier to use in reactive applications; focuses on essential features.    |
