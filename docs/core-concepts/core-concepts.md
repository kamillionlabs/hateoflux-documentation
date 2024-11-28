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
Learn how hateoflux wraps resources to enhance domain objects with hypermedia links and embedded resources, following the HAL (Hypertext Application Language) standard.

* **Wrapping Resources**: Encapsulate single resources and collections with hypermedia links.
* **Reactive Systems**: Handle multiple results effectively in reactive APIs.
* **Main and Embedded Resources**: Distinguish between primary and embedded resources.
* **Resource Naming**: Customize serialization names using `@Relation`.

[Learn more about the representation model](./representation-model.html)

## Link Building
Discover how to build navigable APIs using hateoflux's tools for link creation, including manual methods, URI templates, and integration with Spring WebFlux via `SpringControllerLinkBuilder`.

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
