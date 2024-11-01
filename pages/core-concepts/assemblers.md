---
title: "Assemblers"
layout: default
nav_order: 4
---
# Assemblers
{: .no_toc }
<br>
1. TOC
{:toc}
---

Assemblers in hateoflux are designed to reduce boilerplate code when creating HAL-compliant resource representations. They provide a structured way to wrap resources and enhance them with hypermedia links, following the HATEOAS principles. By implementing an assembler, developers can focus on defining how links are built for resources, while the rest of the wrapping logic is handled by the provided interfaces.

There are four main assembler interfaces in hateoflux:

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

## `FlatHalWrapperAssembler`

The `FlatHalWrapperAssembler` interface is used for assembling wrappers of resources that do not have embedded resources. It handles both single resources and lists of resources, wrapping them in `HalResourceWrapper` and `HalListWrapper` respectively.

### Key Methods to Implement

- **`buildSelfLinkForResource()`**: Constructs the self-link for the main resource. **(Required)**
- **`buildSelfLinkForResourceList()`**: Constructs the self-link for the resource list. **(Required)**
- **`buildOtherLinksForResource()`**: Adds additional links to the main resource. **(Optional)**
- **`buildOtherLinksForResourceList()`**: Adds additional links to the resource list. **(Optional)**

### Example Implementation

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
### Usage in a Controller
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
        return productService.getProductById(id)
                .map(product -> productAssembler.wrapInResourceWrapper(product, exchange));
    }

    @GetMapping
    public Mono<HalListWrapper<Product, Void>> getAllProducts(ServerWebExchange exchange) {
        List<Product> products = productService.getAllProducts();
        return Mono.just(productAssembler.wrapInListWrapper(products, exchange));
    }
}

```
## `EmbeddingHalWrapperAssembler`
The `EmbeddingHalWrapperAssembler` interface is designed for assembling wrappers that include embedded resources. It extends the capabilities of `FlatHalWrapperAssembler` by handling both the main resource and its associated embedded resources.

### Key Methods to Implement
* `buildSelfLinkForResource()`: Constructs the self-link for the main resource. **(Required)**
* `buildSelfLinkForEmbedded()`: Constructs the self-link for an embedded resource. **(Required)**
* `buildSelfLinkForResourceList()`: Constructs the self-link for the resource list. **(Required)**
* `buildOtherLinksForResource()`: Adds additional links to the main resource. **(Optional)**
* `buildOtherLinksForEmbedded()`: Adds additional links to the embedded resource. **(Optional)**
* `buildOtherLinksForResourceList()`: Adds additional links to the resource list. **(Optional)**

### Example Implementation

```java
@Component
public class OrderAssembler implements EmbeddingHalWrapperAssembler<Order, OrderItem> {

    @Override
    public Link buildSelfLinkForResource(Order order, ServerWebExchange exchange) {
        // Build the self link for a single order
        return Link.of("/orders/" + order.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForEmbedded(OrderItem item, ServerWebExchange exchange) {
        // Build the self link for an order item
        return Link.of("/order-items/" + item.getId())
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
    public List<Link> buildOtherLinksForEmbedded(OrderItem item, ServerWebExchange exchange) {
        // Optional: Add additional links for the embedded order item
        Link productLink = Link.of("/products/" + item.getProductId())
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
### Usage in a Controller
````java
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @Autowired
    private final OrderService orderService;
    
    @Autowired
    private final OrderAssembler orderAssembler;

    @GetMapping("/{id}")
    public Mono<HalResourceWrapper<Order, OrderItem>> getOrder(@PathVariable String id, ServerWebExchange exchange) {
        Mono<Order> orderMono = orderService.getOrderById(id);
        List<OrderItem> items = orderService.getOrderItemsByOrderId(id);

        return orderMono.map(order -> orderAssembler.wrapInResourceWrapper(order, items, exchange));
    }

    @GetMapping
    public Mono<HalListWrapper<Order, OrderItem>> getAllOrders(ServerWebExchange exchange) {
        List<Order> orders = orderService.getAllOrders();
        PairList<Order, OrderItem> pairList = new PairList<>();

        for (Order order : orders) {
            List<OrderItem> items = orderService.getOrderItemsByOrderId(order.getId());
            pairList.add(order, items);
        }

        return Mono.just(orderAssembler.wrapInListWrapper(pairList, exchange));
    }
}
````
## `ReactiveFlatHalWrapperAssembler`
The `ReactiveFlatHalWrapperAssembler` interface inherits from `FlatHalWrapperAssembler` and adds reactive counterparts to handle resources in a reactive programming model.

### Key Methods to Implement
As `ReactiveFlatHalWrapperAssembler` extends `FlatHalWrapperAssembler`, it inherits all its methods. **There is no need to implement other methods beside those in `FlatHalWrapperAssembler`**.

- **`buildSelfLinkForResource()`**: Constructs the self-link for the main resource. **(Required)**
- **`buildSelfLinkForResourceList()`**: Constructs the self-link for the resource list. **(Required)**
- **`buildOtherLinksForResource()`**: Adds additional links to the main resource. **(Optional)**
- **`buildOtherLinksForResourceList()`**: Adds additional links to the resource list. **(Optional)**

### Example Implementation

```java
@Component
public class ReactiveProductAssembler implements ReactiveFlatHalWrapperAssembler<Product> {

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

### Usage in a Reactive Controller

```java
@RestController
@RequestMapping("/products")
public class ReactiveProductController {

    @Autowired
    private final ProductService productService;

    @Autowired
    private final ReactiveProductAssembler productAssembler;

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
## `ReactiveEmbeddingHalWrapperAssembler`
The `ReactiveEmbeddingHalWrapperAssembler` interface extends `EmbeddingHalWrapperAssembler` to handle resources and their embedded resources reactively.

### Key Methods to Implement
As `ReactiveEmbeddingHalWrapperAssembler` extends `EmbeddingHalWrapperAssembler`, it inherits all its methods.**There is no need to implement other methods beside those in `EmbeddingHalWrapperAssembler`**.

* `buildSelfLinkForResource()`: Constructs the self-link for the main resource. **(Required)**
* `buildSelfLinkForEmbedded()`: Constructs the self-link for an embedded resource. **(Required)**
* `buildSelfLinkForResourceList()`: Constructs the self-link for the resource list. **(Required)**
* `buildOtherLinksForResource()`: Adds additional links to the main resource. **(Optional)**
* `buildOtherLinksForEmbedded()`: Adds additional links to the embedded resource. **(Optional)**
* `buildOtherLinksForResourceList()`: Adds additional links to the resource list. **(Optional)**

### Example Implementation

```java
@Component
public class ReactiveOrderAssembler implements ReactiveEmbeddingHalWrapperAssembler<Order, OrderItem> {

    @Override
    public Link buildSelfLinkForResource(Order order, ServerWebExchange exchange) {
        return Link.of("/orders/" + order.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForEmbedded(OrderItem item, ServerWebExchange exchange) {
        return Link.of("/order-items/" + item.getId())
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
### Usage in a Reactive Controller

```java
@RestController
@RequestMapping("/orders")
public class ReactiveOrderController {

    @Autowired
    private final OrderService orderService;
   
    @Autowired
    private final ReactiveOrderAssembler orderAssembler;

    @GetMapping("/{id}")
    public Mono<HalResourceWrapper<Order, OrderItem>> getOrder(@PathVariable String id, ServerWebExchange exchange) {
        Mono<Order> orderMono = orderService.getOrderById(id);
        Flux<OrderItem> itemsFlux = orderService.getOrderItemsByOrderId(id);

        return orderAssembler.wrapInResourceWrapper(orderMono, itemsFlux, exchange);
    }

    @GetMapping
    public Mono<HalListWrapper<Order, OrderItem>> getAllOrders(ServerWebExchange exchange) {
        Flux<Order> ordersFlux = orderService.getAllOrders();

        Flux<Pair<Order, OrderItem>> orderPairs = ordersFlux.flatMap(order ->
                orderService.getOrderItemsByOrderId(order.getId())
                        .collectList()
                        .map(items -> Pair.of(order, items))
        );

        return orderAssembler.wrapInListWrapper(orderPairs, exchange);
    }
}
```