---
title: "Core Concepts"
layout: default
nav_order: 3
has_toc: false
---

# Core Concepts
{: .no_toc }

Understanding the core concepts of hateoflux is key to building hypermedia-driven APIs that follow HATEOAS principles.

## Representation Model
hateoflux wraps resources to enhance domain objects with hypermedia links and embedded resources, following the HAL (Hypertext Application Language) standard. These capabilities include:

* **Wrapping Resources**: Encapsulate single resources and collections with hypermedia links.
* **Reactive Systems**: Handle multiple results effectively in reactive APIs.
* **Main and Embedded Resources**: Distinguish between primary and embedded resources.
* **Resource Naming**: Customize serialization names using `@Relation`.

[Learn more about the representation model](./representation-model.html)

## Link Building
hateoflux provides tools for link creation, including manual methods, URI templates, and integration with Spring WebFlux via `SpringControllerLinkBuilder`. These tools feature:

* **Manual Link Building**: Use the `Link` class to create links.
* **URI Templates**: Define dynamic URLs with placeholders.
* **SpringControllerLinkBuilder**: Generate links from controller methods for consistency.

[Learn how to build links with hateoflux](./linkbuilding.html)

## Assemblers
Assemblers simplify creating HAL-compliant resource representations by wrapping resources and adding hypermedia links.

* **Purpose**: Reduce boilerplate and standardize resource wrapping.
* **Types of Assemblers**:
  * **FlatHalWrapperAssembler**: For resources without embedded content.
  * **EmbeddingHalWrapperAssembler**: For resources with embedded content.

[Learn how to use assemblers in hateoflux](./assemblers.html)

## Response Handling
hateoflux provides specialized response types for different resource scenarios while maintaining full integration with Spring WebFlux. These response types comprise:

* **Type-Safe Responses**: Three response types for different use cases - single resources, streamed resources, and lists.
* **Header Management**: Fluent API for HTTP header manipulation.
* **Status Code Support**: Built-in support for HTTP status codes.
* **Auto-configuration**: Zero setup required for response handling.

[Learn about response handling in hateoflux](./response-handling.html)