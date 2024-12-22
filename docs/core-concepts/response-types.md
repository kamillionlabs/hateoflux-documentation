---
title: "Response Types"
layout: default
parent: "Core Concepts"
nav_order: 4
---

# Response Types
{: .no_toc }
<br>
1. TOC
{:toc}
---

## Challenges with `ResponseEntity` in Reactive Applications

In Spring WebFlux, handling reactive responses with `ResponseEntity` introduces specific challenges, especially when working with `Mono` and `Flux`. These challenges center around when and how to finalize HTTP status codes, headers, and body content.

### Single-Item Responses with `Mono`

For single-item responses, `Mono<ResponseEntity<T>>` allows delaying the creation of the response until the data is available. This enables dynamically setting the HTTP status code and headers based on the body. For instance, you can return a `404 Not Found` if the body is empty or `200 OK` if data is present:

```java
Mono<ResponseEntity<T>> response =  someService.getData()
                                         .map(data -> ResponseEntity.ok(data))
                                         .defaultIfEmpty(ResponseEntity.notFound().build());
```
This flexibility ensures that decisions about the response can be made reactively, just before the response is sent to the client.

### Streaming Responses with `Flux`
Streaming responses using `ResponseEntity<Flux<T>>` require the HTTP status and headers to be finalized before emitting any data. Once the first element of the `Flux` is sent, the response is committed, and the status or headers cannot be modified. This is a fundamental limitation of streaming in HTTP. For example:

```java
ResponseEntity<Flux<T>> response = ResponseEntity.ok(someService.getStreamingData());
```
Here, the `200 OK` status is decided upfront and sent immediately, regardless of the subsequent content of the stream. This means it is not possible to base the HTTP status on the content of the stream after it starts.

## hateoflux Response Types
hateoflux provides a unified API to handle the complexities of constructing `ResponseEntity` objects in reactive Spring WebFlux applications. It manages the underlying response fields and the timing of `ResponseEntity` creation, allowing developers to build single-resource, list-based, or streaming responses consistently without manually handling the intricacies of `Mono` and `Flux` wrappers.

### `HalResourceResponse` Response Type
Used when returning a single HAL resource. This response type wraps a single `HalResourceWrapper` and converts it into a proper reactive response.

```java
@GetMapping("/{id}")
public HalResourceResponse<Product, Void> getProduct(@PathVariable String id, ServerWebExchange exchange) {
    Mono<HalResourceWrapper<Product, Void>> product = 
        productAssembler.wrapInResourceWrapper(productService.findById(id), exchange);
    return HalResourceResponse.ok(product);
}
```

Under the hood, `HalResourceResponse` converts to:

```java
Mono<ResponseEntity<HalResourceWrapper<ResourceT, EmbeddedT>>>
```

### `HalMultiResourceResponse` Response Type
Designed for streaming multiple individual HAL resources. This is particularly useful when you want to stream resources one by one rather than collecting them into a list.
```java
@GetMapping("/")
public HalMultiResourceResponse<Product, Void> streamProducts(ServerWebExchange exchange) {
    Flux<HalResourceWrapper<Product, Void>> products = 
        productService.findAll()
            .flatMap(product -> productAssembler.wrapInResourceWrapper(product, exchange));
    return HalMultiResourceResponse.ok(products);
}
```

Under the hood, `HalMultiResourceResponse` converts to:

```java
Mono<ResponseEntity<Flux<HalResourceWrapper<ResourceT, EmbeddedT>>>>
```

{: .note }
Note that `HalMultiResourceResponse` is the only one that puts the wrapper in a `Flux` (or a publisher in general), whereas the others embed the wrappers directly in the `ResponseEntity`.  

### `HalListResponse` Response Type
Used when returning a collection of resources as a single HAL document, particularly useful when including pagination metadata.

```java
@GetMapping
public HalListResponse<Product, Void> getProducts(Pageable pageable, ServerWebExchange exchange) {
    Mono<HalListWrapper<Product, Void>> products = 
        productAssembler.wrapInListWrapper(productService.findAll(pageable), exchange);
    return HalListResponse.ok(products);
}
```

Under the hood, HalListResponse converts to:
```java
Mono<ResponseEntity<HalListWrapper<ResourceT, EmbeddedT>>>
```

## Common Features
### Fluent Http Header Builder
All response types extend `HttpHeadersModule`, providing a fluent API for header manipulation:

```java
return HalResourceResponse.ok(product)
    .withContentType(MediaType.APPLICATION_JSON)
    .withETag("\"123456\"")
    .withHeader("Custom-Header", "value");
```

### Status Codes

In addition to factory methods that enable the creation of a response with arbitrary status codes:

```java
return HalResourceResponse.of(HttpStatus.I_AM_A_TEAPOT);
// or
return HalResourceResponse.of(someObject, HttpStatus.PARTIAL_CONTENT);
```

Each response type includes also factory methods for common HTTP status codes:

* `ok()` - `HTTP 200`
* `created()` - `HTTP 201`
* `accepted()` - `HTTP 202`
* `noContent()` - `HTTP 204`
* `notFound()` - `HTTP 404`

{: .important }
Status codes in `Hal*Response`s are not designed to include detailed messages, especially error messages. `Hal*Response`s allow for flexibility in API design by enabling expressiveness without relying on exceptions. For example, `HTTP 206 Partial Content` and `304 Not Modified` are used to convey specific states that do not necessarily represent exceptions. However, in cases where an exception does occur, it is advisable to use an `ExceptionHandler` instead.

### Type Safety and Reduced Boilerplate

hateoflux's custom response types not only simplify controller method signatures but also maintain type safety through generics. Instead of explicitly wrapping responses in `Mono` and `ResponseEntity`, the library encapsulates these structures, reducing boilerplate while ensuring type correctness.

For example, instead of writing:

```java
Mono<ResponseEntity<HalListWrapper<ResourceT, EmbeddedT>>> methodName(...) { ... }
```
You can use a concise and type-safe response type:

```java
HalListResponse<ResourceT, EmbeddedT> methodName(...) { ... }
```

## Choosing the Right Response Type
* **`HalResourceResponse`**: Use when your API endpoint returns a **single** `HalResourceWrapper`.
* **`HalMultiResourceResponse`**: Use when your endpoint returns **multiple** `HalResourceWrapper`.
* **`HalListResponse`**: Use when your endpoint returns a single `HalListWrapper`.

## Auto-configuration
hateoflux automatically configures the necessary components for response handling through `ReactiveResponseEntityConfig`. No additional setup is required beyond having the dependency on your classpath.