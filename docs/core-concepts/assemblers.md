---
title: "Assemblers"
layout: default
parent: "Core Concepts"
nav_order: 3
---

# Assemblers
{: .no_toc }
<br>
1. TOC
{:toc}
---

Assemblers in hateoflux are designed to reduce boilerplate code when creating HAL-compliant resource representations. They provide a structured way to wrap resources and enhance them with hypermedia links, following the HATEOAS principles. By implementing an assembler, developers can focus on defining how links are built for resources, while the rest of the wrapping logic is handled by the provided interfaces.

There are four assembler interfaces in hateoflux:

- **`FlatHalWrapperAssembler`**
- **`EmbeddingHalWrapperAssembler`**
- **`ReactiveFlatHalWrapperAssembler`**
- **`ReactiveEmbeddingHalWrapperAssembler`**

## Purpose of Assemblers

Assemblers serve as a bridge between raw resource data and HAL-compliant representations. They encapsulate the logic required to wrap resources, append hypermedia links, and manage embedded resources if necessary. By implementing the skeleton methods for building links, developers can customize how links are generated for their specific resources.

Assemblers help in:

- **Reducing Boilerplate Code**: By handling the repetitive tasks of wrapping and linking, assemblers allow developers to focus on business logic.
- **Enhancing Consistency**: They standardize the way resources are enhanced with links, ensuring that all resources follow the same patterns.
- **Supporting Reactive Programming**: Reactive assemblers provide non-blocking, asynchronous handling of resources.
- **Better Maintainability**: Centralizing link-building logic makes it easier to update or modify how links are generated, improving long-term maintainability.

## Reactive vs. Non-Reactive Assemblers
The distinction between reactive and non-reactive assembler interfaces is not based on certain assemblers being exclusively for reactive or non-reactive programming. Instead, it stems from preferences regarding the level at which wrapping operations are performed—whether within reactive operators such as `map()` and `flatMap()`, or externally.

* **Reactive Assemblers (`ReactiveFlatHalWrapperAssembler` and `ReactiveEmbeddingHalWrapperAssembler`)**:

    * **Integration with Reactive Streams**: These assemblers are designed to operate seamlessly within reactive pipelines. Wrapping logic is incorporated inside reactive operators, promoting a functional and declarative programming style.
    * **Simplified Asynchronous Handling**: Asynchronous data flows are managed efficiently, allowing resource transformations and the addition of hypermedia links without disrupting the reactive chain.
    * **Use Case**: Applications that extensively utilize reactive paradigms benefit from these assemblers, as resource wrapping within the reactive stream enhances both readability and maintainability.
* **Non-Reactive Assemblers (`FlatHalWrapperAssembler` and `EmbeddingHalWrapperAssembler`)**:
    * **Explicit Wrapping Control**: Wrapping logic can be applied outside of reactive operators, providing more explicit control over the timing and manner of resource wrapping, which may align better with certain architectural designs.
    * **Flexibility in Integration**: These assemblers can be integrated into both reactive and non-reactive components of an application, offering versatility across diverse coding environments.
    * **Use Case**: Scenarios where managing wrapping logic separately from the reactive data flow is preferred

The provision of both reactive and non-reactive assembler types within hateoflux ensures that flexibility is maintained, allowing the approach that best aligns with project requirements and development preferences to be chosen. Whether wrapping is handled within the reactive stream for a more streamlined and functional methodology, or outside of it for enhanced control and adaptability, hateoflux supports effective implementation of HAL-compliant resource representations.

## Overview of Assemblers

### Overview of `FlatHalWrapperAssembler` 

The `FlatHalWrapperAssembler` interface is used for assembling wrappers of resources that do not have embedded resources. It handles both single resources and lists of resources, wrapping them in `HalResourceWrapper` and `HalListWrapper` respectively.

#### Key Methods to Implement

- **`buildSelfLinkForResource()`**: Constructs the self-link for the main resource. **(Required)**
- **`buildSelfLinkForResourceList()`**: Constructs the self-link for the resource list. **(Required)**
- **`buildOtherLinksForResource()`**: Adds additional links to the main resource. **(Optional)**
- **`buildOtherLinksForResourceList()`**: Adds additional links to the resource list. **(Optional)**

#### Example Implementation

Implementing a `FlatHalWrapperAssembler` involves defining the required methods and any optional methods as needed.

```java
@Component
public class ProductAssembler implements FlatHalWrapperAssembler<Product> {

    @Override
    public Link buildSelfLinkForResource(Product product, ServerWebExchange exchange) {
        // Build the self link for a single product
        return Link.of("/products/" + product.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForResourceList(ServerWebExchange exchange) {
        // Build the self link for the product list
        return Link.of("/products")
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public List<Link> buildOtherLinksForResource(Product product, ServerWebExchange exchange) {
        // Optional: Add additional links for the product resource
        Link categoryLink = Link.of("/categories/" + product.getCategoryId())
                .prependBaseUrl(exchange)
                .withRel("category");

        return List.of(categoryLink);
    }

    @Override
    public List<Link> buildOtherLinksForResourceList(ServerWebExchange exchange) {
        // Optional: Add additional links for the product list
        Link searchLink = Link.of("/products/search")
                .prependBaseUrl(exchange)
                .withRel("search");

        return List.of(searchLink);
    }
}
```
#### Usage in a Controller
With the assembler implemented, it can be used in a controller as follows:

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @Autowired
    private final ProductService productService;

    @Autowired
    private final ProductAssembler productAssembler;

    @GetMapping("/{id}")
    public Mono<HalResourceWrapper<Product, Void>> getProduct(@PathVariable String id, ServerWebExchange exchange) {
        Mono<Product> productMono = productService.getProductById(id);
        return productMono
                .map(product -> {
                    // e.g. do something here with the product first
                    return productAssembler.wrapInResourceWrapper(product, exchange);
                });
    }

    @GetMapping
    public Mono<HalListWrapper<Product, Void>> getAllProducts(ServerWebExchange exchange) {
        Flux<Product> productsFlux = productService.getAllProducts();
        return productsFlux.collectList()
                .map(products -> productAssembler.wrapInListWrapper(products, exchange));
    }
}

```
### Overview of `EmbeddingHalWrapperAssembler`
The `EmbeddingHalWrapperAssembler` interface is designed for assembling wrappers that include embedded resources. It extends the capabilities of `FlatHalWrapperAssembler` by handling both the main resource and its associated embedded resources.

#### Key Methods to Implement
* `buildSelfLinkForResource()`: Constructs the self-link for the main resource. **(Required)**
* `buildSelfLinkForEmbedded()`: Constructs the self-link for an embedded resource. **(Required)**
* `buildSelfLinkForResourceList()`: Constructs the self-link for the resource list. **(Required)**
* `buildOtherLinksForResource()`: Adds additional links to the main resource. **(Optional)**
* `buildOtherLinksForEmbedded()`: Adds additional links to the embedded resource. **(Optional)**
* `buildOtherLinksForResourceList()`: Adds additional links to the resource list. **(Optional)**

#### Example Implementation

```java
@Component
public class OrderAssembler implements EmbeddingHalWrapperAssembler<Order, PaymentDetail> {

    @Override
    public Link buildSelfLinkForResource(Order order, ServerWebExchange exchange) {
        // Build the self link for a single order
        return Link.of("/orders/" + order.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForEmbedded(PaymentDetail detail, ServerWebExchange exchange) {
        // Build the self link for a payment
        return Link.of("/order-payments/" + detail.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForResourceList(ServerWebExchange exchange) {
        // Build the self link for the order list
        return Link.of("/orders")
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public List<Link> buildOtherLinksForResource(Order order, ServerWebExchange exchange) {
        // Optional: Add additional links for the order resource
        Link customerLink = Link.of("/customers/" + order.getCustomerId())
                .prependBaseUrl(exchange)
                .withRel("customer");

        return List.of(customerLink);
    }

    @Override
    public List<Link> buildOtherLinksForEmbedded(PaymentDetail detail, ServerWebExchange exchange) {
        // Optional: Add additional links for the embedded payment detail
        Link productLink = Link.of("/products/" + detail.getProductId())
                .prependBaseUrl(exchange)
                .withRel("product");

        return List.of(productLink);
    }

    @Override
    public List<Link> buildOtherLinksForResourceList(ServerWebExchange exchange) {
        // Optional: Add additional links for the order list
        Link searchLink = Link.of("/orders/search")
                .prependBaseUrl(exchange)
                .withRel("search");

        return List.of(searchLink);
    }
}
```
#### Usage in a Controller
````java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private final OrderService orderService;

    @Autowired
    private final PaymentService paymentService;

    @Autowired
    private final OrderAssembler orderAssembler;


    @GetMapping("/{id}")
    public Mono<HalResourceWrapper<Order, PaymentDetail>> getOrder(@PathVariable String id, ServerWebExchange exchange) {
        Mono<Order> order = orderService.getOrderById(id);
        
        return order.flatMap(order ->
                        paymentService.getPaymentDetailByOrderId(id)
                                .map(paymentDetail -> orderAssembler.wrapInResourceWrapper(order, paymentDetail, exchange))
                );
    }

    @GetMapping
    public Mono<HalListWrapper<Order, PaymentDetail>> getAllOrders(@RequestParam String userId, ServerWebExchange exchange) {
        Flux<Order> ordersOfUser = orderService.getOrdersByUserId(userId);
        
        return ordersOfUser.flatMap(order ->
                        paymentService.getPaymentDetailByOrderId(order.getId())
                                .map(paymentDetail -> Pair.of(order, paymentDetail))
                )
                .collect(PairListCollector.toPairList()) // note this utility collector
                .map(orderPaymentPairs -> orderAssembler.wrapInListWrapper(orderPaymentPairs, exchange));
    }
}
````
### Overview of `ReactiveFlatHalWrapperAssembler`
The `ReactiveFlatHalWrapperAssembler` interface inherits from `FlatHalWrapperAssembler` and adds reactive counterparts to handle resources in a reactive programming model.

#### Key Methods to Implement
As `ReactiveFlatHalWrapperAssembler` extends `FlatHalWrapperAssembler`, it inherits all its methods. **There is no need to implement other methods beside those in `FlatHalWrapperAssembler`**.

- **`buildSelfLinkForResource()`**: Constructs the self-link for the main resource. **(Required)**
- **`buildSelfLinkForResourceList()`**: Constructs the self-link for the resource list. **(Required)**
- **`buildOtherLinksForResource()`**: Adds additional links to the main resource. **(Optional)**
- **`buildOtherLinksForResourceList()`**: Adds additional links to the resource list. **(Optional)**

#### Example Implementation

```java
@Component
public class ProductAssembler implements ReactiveFlatHalWrapperAssembler<Product> {

    @Override
    public Link buildSelfLinkForResource(Product product, ServerWebExchange exchange) {
        return Link.of("/products/" + product.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForResourceList(ServerWebExchange exchange) {
        return Link.of("/products")
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    // Other methods can be implemented as needed
}
```

#### Usage in a Controller

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @Autowired
    private final ProductService productService;

    @Autowired
    private final ProductAssembler productAssembler;

    @GetMapping("/{id}")
    public Mono<HalResourceWrapper<Product, Void>> getProduct(@PathVariable String id, ServerWebExchange exchange) {
        return productAssembler.wrapInResourceWrapper(productService.getProductById(id), exchange);
    }

    @GetMapping
    public Mono<HalListWrapper<Product, Void>> getAllProducts(ServerWebExchange exchange) {
        Flux<Product> productFlux = productService.getAllProducts();
        Mono<Long> totalElements = productService.countProducts();

        return productAssembler.wrapInListWrapper(productFlux, totalElements, 20, 0L, null, exchange);
    }
}
```
### Overview of `ReactiveEmbeddingHalWrapperAssembler`
The `ReactiveEmbeddingHalWrapperAssembler` interface extends `EmbeddingHalWrapperAssembler` to handle resources and their embedded resources reactively.

#### Key Methods to Implement
As `ReactiveEmbeddingHalWrapperAssembler` extends `EmbeddingHalWrapperAssembler`, it inherits all its methods.**There is no need to implement other methods beside those in `EmbeddingHalWrapperAssembler`**.

* `buildSelfLinkForResource()`: Constructs the self-link for the main resource. **(Required)**
* `buildSelfLinkForEmbedded()`: Constructs the self-link for an embedded resource. **(Required)**
* `buildSelfLinkForResourceList()`: Constructs the self-link for the resource list. **(Required)**
* `buildOtherLinksForResource()`: Adds additional links to the main resource. **(Optional)**
* `buildOtherLinksForEmbedded()`: Adds additional links to the embedded resource. **(Optional)**
* `buildOtherLinksForResourceList()`: Adds additional links to the resource list. **(Optional)**

#### Example Implementation

```java

@Component
public class OrderAssembler implements ReactiveEmbeddingHalWrapperAssembler<Order, PaymentDetail> {

    @Override
    public Link buildSelfLinkForResource(Order order, ServerWebExchange exchange) {
        return Link.of("/orders/" + order.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForEmbedded(PaymentDetail detail, ServerWebExchange exchange) {
        return Link.of("/order-payments/" + detail.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForResourceList(ServerWebExchange exchange) {
        return Link.of("/orders")
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    // Other methods can be implemented as needed
}
```
#### Usage in a Controller

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private final OrderService orderService;

    @Autowired
    private final PaymentService paymentService;

    @Autowired
    private final OrderAssembler orderAssembler;

    @GetMapping("/{id}")
    public Mono<HalResourceWrapper<Order, PaymentDetail>> getOrder(@PathVariable String id, ServerWebExchange exchange) {
        Mono<Order> orderMono = orderService.getOrderById(id);
        Mono<PaymentDetail> paymentMono = paymentService.getPaymentDetailByOrderId(id);

        return orderAssembler.wrapInResourceWrapper(orderMono, paymentMono, exchange);
    }

    @GetMapping
    public Mono<HalListWrapper<Order, PaymentDetail>> getAllOrders(String userId, ServerWebExchange exchange) {
        Flux<Order> ordersOfUser = orderService.getOrdersByUserId(userId);

        Flux<Pair<OrderDTO,PaymentDetailDTO>> ordersWithPaymentDetails = ordersOfUser.flatMap(order ->{
            int id = order.getId();
            return paymentService.getPaymentDetailByOrderId(id)
                    .map(paymentDetail -> Pair.of(order, paymentDetail));
        });
        
        return orderAssembler.wrapInListWrapper(ordersWithPaymentDetails, exchange);
    }
}
```