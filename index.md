---
title: Home
layout: home
nav_order: 1
---

# What is hateoflux?

HATEOAS (Hypermedia as the Engine of Application State) is a critical concept in RESTful API design, allowing clients to navigate APIs dynamically using hyperlinks provided by the server. While Spring's HATEOAS library has been a standard choice for many Spring developers, it falls short in certain areas, especially when it comes to full HATEOAS support in Spring WebFlux. Although it works very well in Spring MVC, the integration with Spring WebFlux feels like an incomplete "port", with deep-rooted entanglements in Spring MVC that make it less ideal for modern reactive architectures. This is where hateoflux (HATEOas + webFLUX) steps in as a viable alternative.

hateoflux is a Java library created specifically to address the limitations of Spring's HATEOAS implementation. It simplifies the representation, assembly, and linking of hypermedia-driven resources in Spring-based applications. Additionally, hateoflux is lightweight and doesn't require any setup or changes in code, as it works seamlessly with Spring, including common annotations such as `@RestController`, `@PathVariable`, and `@RequestMapping`. The library has been designed with the core Spring WebFlux and JSON components in mind, providing seamless integration with existing Spring projects while offering a more intuitive and comprehensive approach to building hypermedia APIs.

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
