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

There are two assembler interfaces in hateoflux:

- **`FlatHalWrapperAssembler`**
- **`EmbeddingHalWrapperAssembler`**

## Purpose of Assemblers

Assemblers serve as a bridge between raw resource data and HAL-compliant representations. They encapsulate the logic required to wrap resources, append hypermedia links, and manage embedded resources if necessary. By implementing the skeleton methods for building links, developers can customize how links are generated for their specific resources.

Assemblers help in:

- **Reducing Boilerplate Code**: By handling the repetitive tasks of wrapping and linking, assemblers allow developers to focus on business logic.
- **Enhancing Consistency**: They standardize the way resources are enhanced with links, ensuring that all resources follow the same patterns.
- **Supporting Reactive Programming**: Reactive assemblers provide non-blocking, asynchronous handling of resources.
- **Better Maintainability**: Centralizing link-building logic makes it easier to update or modify how links are generated, improving long-term maintainability.

## Reactive vs. Non-Reactive Methods
The distinction between reactive and non-reactive assembler wrapping methods, such as `wrapInResourceWrapper()`, stems from preferences regarding the level at which wrapping operations are performed. Whether within reactive operators such as `map()` and `flatMap()`, or externally. 

### Reactive Wrapping
Wrapping reactively is a seamless operation and takes `Publisher`s in and provides `Publisher`s out. If no further processing is needed, it is often the most straight forward approach.

**Example**
```java 
@GetMapping
public Mono<HalListWrapper<Product, Void>> getAllProducts(ServerWebExchange exchange) {
    Flux<Product> productFlux = productService.getAllProducts();
    Mono<Long> totalElements = productService.countProducts();

    return productAssembler.wrapInListWrapper(productFlux, totalElements, 20, 0L, null, exchange);
}
```
### Non-Reactive Wrapping
Wrapping non-reactively is useful when modifications are necessary. However, this is certainly not required, but ultimately is a question of preference.

**Example**
```java 
@GetMapping("/{id}")
public Mono<HalResourceWrapper<Product, Void>> getProduct(@PathVariable String id, ServerWebExchange exchange) {
    Mono<Product> productMono = productService.getProductById(id);
    return productMono
            .map(product -> {
                // e.g. do something here with the product first
                return productAssembler.wrapInResourceWrapper(product, exchange);
            });
}
```

## Specifying the `ResourceT` explicitly 
Java implements generics through a mechanism called **type erasure**, which means that generic type information (e.g. `ResourceT` in the case of assemblers) is not available at runtime. This poses a problem when the retrieval of the class type of an object is needed, but no object is available.

In HATEOAS, objects withing the `_embedded` node are all named. Assemblers use the class type information to infer the name that they need to use for such cases:
```javascript
{
  "id": 12345,
  "_embedded": {
    "shipment": {  // <-- This name e.g. is based on the Class type of the embedded resource 
      "id": 98765,
      // other fields etc.
    }
  }
}
```
While a missing embedded resource would simply result in the removal of the `_embedded` node altogether. However, if the node is expected to contain a list, it is common to send an empty list instead (as is the case with hateoflux). An empty list of orders in HAL JSON would look like this:
```javascript
{
  "_embedded": {
    "orders": []
  },
  "_links": {
    //...
  }
}
```
In order for assemblers to still be able to infer the class type from an empty `List<Order>` or `Flux<Order>`, the method `getResourceTClass()` is used.

## Overview of Assemblers

### Overview of `FlatHalWrapperAssembler`
The `FlatHalWrapperAssembler` interface is used for assembling wrappers of resources that do not have embedded resources. It handles both single resources and lists of resources, wrapping them in `HalResourceWrapper` and `HalListWrapper` respectively.

#### Key Methods to Implement
- **`getResourceTClass()`**: Specifies the class type of `ResourceT` that the assembler builds. **(Required)**
- **`buildSelfLinkForResource()`**: Constructs the self-link for the main resource. **(Required)**
- **`buildSelfLinkForResourceList()`**: Constructs the self-link for the resource list. **(Required)**
- **`buildOtherLinksForResource()`**: Adds additional links, other than self link, to the resource. **(Optional)**
- **`buildOtherLinksForResourceList()`**: Adds additional links, other than self link, to the resource list. **(Optional)**

#### Example Implementation

```java
@Component
public class ProductAssembler implements FlatHalWrapperAssembler<Product> {

    @Override
    public Class<Product> getResourceTClass() {
        return Product.class;
    }

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

    @Override
    public List<Link> buildOtherLinksForResource(Product product, ServerWebExchange exchange) {
        // Optional: Add additional links for the product resource
        Link categoryLink = Link.of("/category/" + product.getCategoryId())
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
### Overview of `EmbeddingHalWrapperAssembler`
The `EmbeddingHalWrapperAssembler` interface is designed for assembling wrappers that include embedded resources. It extends the capabilities of `FlatHalWrapperAssembler` by handling both the main resource and its associated embedded resources.

#### Key Methods to Implement
**`getResourceTClass()`**: Specifies the class type of `ResourceT` that the assembler builds. **(Required)**
* `buildSelfLinkForResource()`: Constructs the self-link for the main resource. **(Required)**
* `buildSelfLinkForEmbedded()`: Constructs the self-link for the embedded resource. **(Required)**
* `buildSelfLinkForResourceList()`: Constructs the self-link for the resource list. **(Required)**
* `buildOtherLinksForResource()`: Adds additional links to the main resource. **(Optional)**
* `buildOtherLinksForEmbedded()`: Adds additional links to the embedded resource. **(Optional)**
* `buildOtherLinksForResourceList()`: Adds additional links to the resource list. **(Optional)**

#### Example Implementation

```java

@Component
public class OrderAssembler implements EmbeddingHalWrapperAssembler<Order, PaymentDetail> {

    @Override
    public Class<Order> getResourceTClass() {
        return Order.class;
    }

    @Override
    public Link buildSelfLinkForResource(Order order, ServerWebExchange exchange) {
        return Link.of("/order/" + order.getId())
                .prependBaseUrl(exchange)
                .withSelfRel();
    }

    @Override
    public Link buildSelfLinkForEmbedded(PaymentDetail detail, ServerWebExchange exchange) {
        return Link.of("/order-payment/" + detail.getId())
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

        Flux<Pair<OrderDTO,PaymentDetailDTO>> ordersWithPaymentDetails = ordersOfUser.flatMap(order -> {
            int id = order.getId();
            return paymentService.getPaymentDetailByOrderId(id)
                    .map(paymentDetail -> Pair.of(order, paymentDetail));
        });
        
        return orderAssembler.wrapInListWrapper(ordersWithPaymentDetails, exchange);
    }
}
```