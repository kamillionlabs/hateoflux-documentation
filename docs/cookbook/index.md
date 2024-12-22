---
title: "Cookbook: Examples &amp; Use Cases"
layout: default
nav_order: 4
has_toc: false
---

# Cookbook: Examples &amp; Use Cases
{: .no_toc }

The cookbook provides practical examples and patterns for working with hateoflux in real-world scenarios. Each section focuses on specific aspects of the library, offering concrete code examples and best practices.

{: .important }
Every example shown can be viewed and debugged in the [hateoflux-demos repository](https://github.com/kamillionlabs/hateoflux-demos). Clone or fork it and test as you explore options available! Either run the application and curl against the micro service or check the examples directly in the given unit tests.
<br>

## Available Recipes
### Manual Wrapper Creation
Manual creation and configuration of HAL wrappers for resources:

* **Single Resources**: Create wrappers for individual domain objects.
* **Embedded Resources**: Add secondary resources to your main resource.
* **Collections**: Handle lists of resources and pagination.
* **Link Building**: Add and customize hypermedia links.

[Learn about manual wrapper creation](./manual-wrapper-creation.html)

### Assembler-Based Wrapper Creation
Use of assemblers to streamline wrapper creation and reduce boilerplate code:

* **Basic Assemblers**: Implement assemblers for simple resources.
* **Embedded Support**: Work with primary and secondary resources.
* **Collection Handling**: Manage lists and pagination efficiently.
* **Link Management**: Generate consistent links across resources.

[Learn about assembler-based wrapper creation](./assembler-based-wrapper-creation.html)


### Reactive Response Types
Use of specialized response types for different resource scenarios:

* **Status Code Handling**: Return appropriate HTTP status codes for different scenarios.
* **Header Management**: Handle HTTP headers with a fluent API.
* **Single Resources**: Return individual HAL resources with proper reactive wrapping.
* **Streaming Resources**: Stream multiple resources efficiently.
* **Collection Responses**: Handle lists and pagination with type safety.

[Learn about reactive response handling](./response-type-creation.html)

Each recipe section contains complete, working examples that you can use as templates for your own implementations.
