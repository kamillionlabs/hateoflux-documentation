---
title: The Library hateoflux
layout: home
nav_order: 1
---

# What is hateoflux?

hateoflux is a reactive Java library specifically created to address the limitations of Spring's HATEOAS implementation concerning WebFlux. While Spring HATEOAS performs very well in Spring MVC, its integration with Spring WebFlux feels like an incomplete "port," deeply entangled with Spring MVC components. This makes it less ideal for modern reactive architectures. In contrast, hateoflux is designed spefically for Spring WebFlux, providing seamless integration with existing Spring projects. It offers a more intuitive and comprehensive approach to building hypermedia APIs.

# What Problem Does hateoflux Solve?
The integration of Spring HATEOAS with WebFlux can feel cumbersome and incomplete, leading to increased complexity and verbosity in code. Key issues include:

* **Verbosity and Boilerplate Code**: Implementing hypermedia controls in reactive applications often requires repetitive and verbose code, especially when assembling resources and adding links.
* **Manual Pagination Handling**: Spring HATEOAS lacks built-in support for pagination in WebFlux, forcing developers to manually implement pagination metadata and navigation links.
* **Limited Reactive Support**: The library's primary focus on synchronous processing makes it less optimized for reactive programming models, potentially impacting performance and developer productivity.
* **Insufficient Documentation**: Documentation and examples for integrating Spring HATEOAS with WebFlux are less extensive, making it challenging for developers to implement advanced hypermedia features in reactive applications.

hateoflux addresses these problems by providing a reactive-first approach to building hypermedia APIs with Spring WebFlux. It simplifies development by:
* **Reducing Boilerplate**: Simplified assemblers and wrappers minimize repetitive code, allowing developers to focus on core business logic.
* **Simplifying Link Building**: Offers concise, manual, templated and type-safe link creation using lambda expressions, improving code readability and maintainability.
* **Enhancing Pagination Support**: Automatically handles pagination details and navigation links, streamlining the implementation of paginated endpoints.
* **Optimizing for Reactive Environments**: Designed specifically for WebFlux and R2DBC in mind, ensuring seamless integration and better performance in reactive architectures.
* **Providing Focused Documentation**: Offers comprehensive guidance and examples tailored for reactive programming, reducing the learning curve.

For an in-depth comparison, including detailed examples and explanations, as well as shortcomings, please refer to [hateoflux vs. Spring HATEOAS](./docs/hateoflux-vs-spring.md)

# Key Components

## 1. Representation Models

In Spring HATEOAS, the `EntityModel`, `CollectionModel`, and `PagedModel` classes are commonly used to represent resources with hypermedia links. hateoflux provides its own alternatives to these models:

- `HalResourceWrapper` replaces `EntityModel` for representing single resources with hypermedia links.
- `HalListWrapper` serves as a unified replacement for both `CollectionModel` and `PagedModel`, allowing collections to   optionally include pagination without needing a distinct class.

Unlike Spring's approach, which uses the notion of 'Model', hateoflux treats these as HAL wrappers, meaning that the underlying resource or DTO remains unchanged but is wrapped to add hypermedia capabilities as needed. As is common with HATEOAS, any resource in hateoflux can have another embedded resource. However, a conscious design choice was made to limit this to a single level, meaning that a main resource may have another embedded one, but the embedded resource cannot have further nested resources. This decision was made primarily to avoid impractical complexity; if a resource needs multiple levels of embedding, it may indicate that the domain model should be refined instead.

## 2. Link-Building Capabilities

Link building is at the heart of HATEOAS, and Spring's HATEOAS library provides the `Link` and `WebMvcLinkBuilder` classes for generating links to controllers and methods. In contrast, hateoflux defines its own `Link` and `LinkRelation` classes, serving the same purpose as Spring's counterparts. While Spring offers separate builders like `WebMvcLinkBuilder` for MVC and `WebFluxLinkBuilder` for WebFlux, hateoflux introduces the `SpringControllerLinkBuilder` as a dedicated replacement for WebFlux scenarios.

The builder usage in hateoflux closely mirrors that of Spring HATEOAS. Both hateoflux and Spring utilize a controller class as a reference and invoke its methods, differing primarily in declaration style. In both frameworks, paths and variables are automatically extracted, and templated URIs are expanded, making link building declarative and straightforward.

Additionally, hateoflux integrates seamlessly with Spring's annotations, such as `@RestController`, `@PathVariable`, and `@RequestMapping`, without requiring any customizations. While Spring HATEOAS includes annotations like `@Relation`, hateoflux provides replacements for the essential ones to ensure compatibility.

## 3. Assemblers

hateoflux also includes its own **assemblers** that serve a similar role to `RepresentationModelAssembler` in Spring's HATEOAS. Assemblers in hateoflux are responsible for converting domain objects into representation models enriched with hypermedia links. Unlike Spring's version, hateoflux assemblers are designed as interfaces that come mostly preprogrammed with default logic, making them easier to use out of the box. They help abstract the usage of all the components of hateoflux, acting as a simplified programming interface. Typically, developers only need to specify which links to create for a given resource type and how, allowing for greater extensibility and customization without the complexity.

# Comparison with Spring's HATEOAS Library

Spring's HATEOAS is a comprehensive package that offers much more functionality than hateoflux. In contrast, hateoflux focuses solely on reactive systems and the core principles of HATEOAS. However, it is precisely this focus that makes hateoflux easy to use and lightweight. All functionalities are carefully curated, allowing for a comfortable programming interface. The following lists the most important features of Spring's HATEOAS and hateoflux, as well as those that are not present in hateoflux:

| Functionalities                                      | Spring HATEOAS                                                                | hateoflux                                          |
|------------------------------------------------------|-------------------------------------------------------------------------------|----------------------------------------------------|
| Representation Model                                 | ✅ `EntityModel`, `CollectionModel`, `PagedModel`                              | ✅ `HalResourceWrapper`, `HalListWrapper`           |
| `linkTo()` on controller method                      | ✅  With `WebMvcLinkBuilder` for MVC and `WebFluxLinkBuilder` for WebFlux | ✅  With `SpringControllerLinkBuilder`              |
| URI templates as links (query and path parameters)   | ✅                                                                             | ✅                                                  |
| Manual expansion of URIs (query and path parameters) | ✅                                                                             | ✅                                                  |
| Assemblers                                           | ✅                                                                             | ✅                                                  |
| Serialization                                        | ✅                                                                             | ✅                                                  |
| Deserialization                                      | ✅                                                                             | ❌ no, only designed for server to client communication (i.e. serialization only) |
| Media Types                                          | ✅ various                                                                     | ⚠️ only `application/hal+json`                      |
| Affordance                                           | ✅                                                                             | ❌                                                  |
| Curie                                                | ✅                                                                             | ❌                                                  |
