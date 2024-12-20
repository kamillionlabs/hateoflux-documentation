---
title: The Library hateoflux
layout: home
nav_order: 1
---

# What is hateoflux?

hateoflux is a reactive Java library specifically created to address the limitations of Spring's HATEOAS implementation concerning WebFlux. While Spring HATEOAS performs very well in Spring MVC, its integration with Spring WebFlux feels like an incomplete "port," deeply entangled with Spring MVC components. This makes it less ideal for modern reactive architectures. In contrast, hateoflux is designed specifically for Spring WebFlux, providing seamless integration with existing Spring projects. It offers a more intuitive and comprehensive approach to building hypermedia APIs.

# What Problem Does hateoflux Solve?

Developing hypermedia-driven APIs in reactive Spring applications using WebFlux presents unique challenges. Traditional libraries like Spring HATEOAS are primarily designed for Spring MVC and can lead to verbose code, extensive boilerplate, and complex class hierarchies when adapted to reactive environments. This complexity arises from:

* **Verbose Assemblers and Boilerplate Code**: Spring HATEOAS requires manual resource wrapping and link addition, increasing maintenance overhead.
* **Inefficient Representation Models**: Inheritance-based models can clutter domain objects and create complex hierarchies.
* **Limited Reactive Support and Documentation**: Sparse guidance for WebFlux makes it difficult to implement hypermedia APIs effectively in reactive applications.

hateoflux addresses these issues by providing a lightweight, reactive-first library tailored for Spring WebFlux. It simplifies hypermedia API development by:

* **Using Resource Wrappers**: Keeps domain models clean and decoupled from hypermedia concerns by composing wrappers around them.
* **Simplifying Assemblers**: Reduces boilerplate by focusing on link creation, automating resource wrapping and embedding.
* **Enhancing Pagination Handling**: Offers built-in support with `HalListWrapper`, automatically managing metadata and navigation links.
* **Providing Focused Documentation**: Offers comprehensive guidance and examples specifically for reactive environments, improving the developer experience.

By concentrating on essential features and supporting HAL+JSON, hateoflux streamlines the creation of hypermedia APIs in reactive applications, eliminating unnecessary complexity and fostering more maintainable and intuitive codebases.

For an in-depth comparison, including detailed examples and explanations, as well as shortcomings, please refer to [Spring HATEOAS vs. hateoflux](./docs/spring-vs-hateoflux.html)

# How is hateoflux Configured in My Spring Webflux Project?

Configuring hateoflux in your Spring WebFlux project is straightforward and requires no setup. hateoflux is designed to integrate seamlessly with the Spring framework, leveraging its existing configurations and annotations without necessitating additional adjustments.

## Adding hateoflux to Your Project

To include hateoflux in your Spring WebFlux project, simply add it as a dependency using your build tool of choice.

{: .highlight }
See latest available version in [Maven Central](https://central.sonatype.com/artifact/de.kamillionlabs/hateoflux)

### Gradle
```groovy
dependencies {
    implementation 'de.kamillionlabs:hateoflux:latest-version'
}
```

### Maven
```xml
<dependency>
    <groupId>de.kamillionlabs</groupId>
    <artifactId>hateoflux</artifactId>
    <version>latest-version</version>
</dependency>
```
## Seamless Integration with Spring
* **Honors Spring Annotations**: hateoflux fully supports all standard Spring Controller annotations such as `@RestController`, `@RequestParam`, and others. This means you can continue to use your existing annotations without any modifications.

* **Plug and Play with Jackson**: hateoflux relies solely on Jackson's default mechanisms for JSON processing. There is no need to register additional modules or define specific beans, making the setup process hassle-free.

* **No Additional Configuration Required**: Once the dependency is added, hateoflux automatically integrates with your Spring WebFlux application. You don't need to adapt your existing codebase or configurations to accommodate hateoflux.

# How Do I Get Started with hateoflux?

Getting started with hateoflux is straightforward, allowing you to seamlessly integrate hypermedia-driven features into your Spring WebFlux project without extensive configuration or boilerplate code.

## Core Components

hateoflux provides several components to manage your resources effectively:

### Resource Wrappers
Resource wrappers enhance resources with HAL elements to ensure they adhere to HAL standards:
* **`HalResourceWrapper`**: Wraps individual resources, adding essential hypermedia links and optional embedded secondary resources.
* **`HalListWrapper`**: Wraps collections of resources, handling pagination metadata and navigation links automatically.

### Response Types
hateoflux provides a standardized, reactive, and convenient way for controller methods to return HAL responses. They include an HTTP status, optional HTTP headers, and implement the `ReactiveResponseEntity` interface. They wrap specific resource types as follows:
* **`HalResourceResponse`**: Contains a `Mono` of  `HalResourceWrapper` for a single resources.
* **`HalMultiResourceResponse`**: Contains a `Flux` of `HalResourceWrapper`s for a stream of multiple resources.
* **`HalListResponse`**: Contains a `HalListWrapper` for returning collections of resources with pagination support.


## Assemblers

To simplify the creation of these wrappers, hateoflux offers assembler interfaces with default implementations. These assemblers handle the repetitive tasks of wrapping resources and adding links, allowing you to concentrate on defining the specific links and embedded resources relevant to your application.

## Additional Resources

For practical guidance and detailed examples, visit the [cookbook](./docs/cookbook.html), which showcases multiple use cases and scenarios. To gain a deeper understanding of hateoflux's foundational concepts, including representation models, link building, and assemblers, explore the [core concepts](./docs/core-concepts/core-concepts.html) section.

By leveraging these components and resources, you can efficiently build robust, hypermedia-driven APIs with hateoflux in your reactive Spring applications.
