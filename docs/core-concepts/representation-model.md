---
title: "Representation Model"
layout: default
parent: "Core Concepts"
nav_order: 1
---
# Representation Model
{: .no_toc }
<br>
1. TOC
{:toc}
---

## Resource Wrappers

In hypermedia-driven APIs, wrapping resources enhances domain objects by adding hypermedia links and embedded resources. This section covers using `HalResourceWrapper`, `HalListWrapper`, and `HalEmbeddedWrapper` to wrap resources according to HAL standards.

### Wrapper Classes

{: .important }
As opposed to Spring's Models, wrapper classes are final and not intended for inheritance.

This library provides the following wrapper classes:

- `HalResourceWrapper`: Encapsulates a single resource, enriching it with hypermedia links and optionally embedding related resources.
- `HalListWrapper`: Manages a collection of `HalResourceWrapper` instances, facilitating the representation of lists of resources along with pagination support.
- `HalEmbeddedWrapper`: Encapsulates embedded resources that provide additional context or related data to the main resource.

The class `HalPageInfo` is similar to Spring's `Pageable` and can be optionally used with `HalListWrapper`. It represents pagination details, enabling scalable data interaction through paged responses.

Using these classes ensures JSON representations conform to HAL standards, including navigational links, related resources, and pagination information.

### Sending Multiple Objects over a Reactive System

When designing a reactive API, a key design choice involves handling multiple results. Should an endpoint send a stream of objects (`Flux<Object>`) or bundle multiple items in a single list (`Mono<Collection>`)? Both approaches have their merits.

**Sending data as a stream of objects** aligns seamlessly with the reactive principles of responsiveness and elasticity. This stream-based approach allows systems to handle data incrementally, reducing memory overhead and enabling efficient backpressure management. It is ideal for applications requiring continuous updates, such as live feeds, real-time analytics, or monitoring systems.

**Sending lists/pages of objects** segments data into discrete, manageable chunks. This pagination strategy organizes data into fixed-size sets, which are transmitted in separate requests or responses. Pagination is advantageous for scenarios involving large datasets, such as e-commerce catalogs, search results, or any application where users need to navigate through extensive lists of items.

Both approaches are valid and supported by hateoflux. When sending a stream of objects, you can use `Flux<HalResourceWrapper>`. Each item is a valid HAL document with links and potential embedded resources. When segmentation is desired, a `Mono<HalListWrapper>` can be used. The `HalListWrapper` bundles all items into a single document and provides links for the group as a whole. Pagination is typically of interest in this case, which the `HalListWrapper` also supports by including `HalPageInfo`. `HalPageInfo` contains the size of the page, the current page number, the total number of items, and the total number of pages. Each item remains a valid HAL document on its own, with links and optional embedded resources.


### Main and Embedded Resources

Generally speaking, a resource is an object in a HAL document and can be seen as the payload. Throughout all wrappers (and assemblers), 2 generic types are reoccurring that represent resources:
- `ResourceT`: Represents the "main" resource, meaning the object type that the HAL document focuses on.
- `EmbeddedT`: Also a resource, but of secondary importance. An embedded resource is not sent on its own but always attached to a "main" resource `ResourceT`.

All wrapper classes define an `EmbeddedT`. When there is no need to embed another resource, the type is set to `Void`, indicating that no embedded resource exists. So e.g. `HalResourceWrapper<OrderDTO,PaymentDTO>` has a main resource `OrderDTO` and an embedded resource `PaymentDTO`, whereas `HalResourceWrapper<OrderDTO,Void>` contains only a main resource.

It is important to note however, that the `EmbeddedT` is still not mandatory even though it is e.g. specified as `PaymentDTO`. In general, if a wrapper has a concrete `EmbeddedT`, it typically signifies that the object is expected to have one. However, it is acceptable for the embedded resource to be represented as a single empty `HalEmbeddedWrapper`, an empty list, or simply `null`. These configurations are interpreted differently. The serialized JSON will either omit the `_embedded` node entirely or include it as an empty array (`[]`).

As an example, an API of an online store might expose "Order" objects, i.e.,`OrderDTO`s. In a scenario where a customer may be checking the status of an order in an app to see if it's still processing or completed, the HAL document might look like this:

```javascript
{
  "id": 12345,
  "userId": 37,
  "total": 99.99,
  "status": "Processing",
  "_links": {
    // not the focus for now
  }
}
```

Suppose now the order was shipped. The customer might want to check both the order’s status and the shipment’s current tracking information in one consolidated view. The HAL document might now look like this:


```javascript
{
  "id": 12345,
  "userId": 37,
  "total": 99.99,
  "status": "Shipped",
  "_embedded": {
    "shipment": {
        "id": 98765,
        "carrier": "UPS",
        "trackingNumber": "1Z999AA10123456784",
        "status": "In Transit",
        "_links": {
          // not the focus for now
        }
      }
  },
  "_links": {
    // not the focus for now
  }
}
```
In the first scenario, `ResourceT` would be an `OrderDTO`. Depending on the specific use case and design choices, `EmbeddedT` is set to `Void`, for example, if the given controller exclusively provides `OrderDTO`s. However, if `ShipmentDTO`s are typically the norm and this particular order is an exception, then `EmbeddedT` could also be a `ShipmentDTO`—in this case, it would simply be empty or `null`.

In the second scenario, the "main" resource remains an `OrderDTO`. However, an additional resource, `ShipmentDTO`, has been included as a secondary resource to provide supplementary information. In this case, `ShipmentDTO` would naturally serve as the `EmbeddedT`.


### Name of a Resource

HATEOAS is ultimately used to send documents. hateoflux relies on Jackson for serialization, meaning that fields are named the same way they are in Java or as they are annotated with `@JsonProperty`. While main resources, when serialized, only include the member variables of the resource, embedded resources also specify the resource's name.

In Spring's HATEOAS, by default, the name of an embedded resource, or a list of them, is simply the name of the class for a single resource or the name with an additional "s" appended at the end. While this works fine for a resource called `Book`, which would become `book` and `books` for a list, it is less ideal when we use e.g. suffixes like "DTO", as it would serialize into `bookDTO` and `bookDTOs`. Spring's HATEOAS defines the annotation `@Relation` for this purpose, which specifies the name of a class during serialization.

hateoflux adopts the same annotation with the same name and meaning. Therefore, it is sufficient to annotate a resource class with `@Relation` while specifying the name of the class during serialization.


## Links
Links are essential in HATEOAS-driven APIs, providing navigational controls for clients to interact with resources dynamically. They allow clients to discover actions and related resources through embedded links instead of hardcoding API endpoints, promoting a flexible client-server architecture.

In HAL, links are included in a `_links` section of the JSON representation. Each link consists of a relation  that describes the nature of the link and a URL (`href`) that points to the related resource or action.

### Overview of Links in HATEOAS

Links in HATEOAS enable clients to:

- **Navigate**: Move between resources without prior knowledge of the API's structure.
- **Discover Actions**: Identify available actions on a resource (e.g., update, delete).
- **Access Related Resources**: Retrieve related data efficiently.

### Similarities and Differences Between Links and Embedded Resources

Both links and embedded resources enhance the richness of a HAL document, but they serve different purposes:

- **Similarities**:
    - Both are used to represent relationships between resources.
    - They are included within the HAL JSON structure (`_links` and `_embedded` respectively).
    - Both improve the navigability and discoverability of the API.

- **Differences**:
    - **Links** (`_links`):
        - Provide URLs to related resources or actions.
        - Do not include the actual data of the related resources.
        - Allow clients to fetch related data on-demand.
    - **Embedded Resources** (`_embedded`):
        - Contain the full representation of related resources within the main resource.
        - Provide immediate access to related data without additional API calls.
        - Can increase the payload size if many embedded resources are included.

Choosing between links and embedded resources depends on the use case. Use links when you want to keep the payload lightweight and allow clients to fetch related data as needed. Use embedded resources when related data is frequently accessed together with the main resource.

### Links and Link Relations in hateoflux

In hateoflux, links are represented by the `Link` class, which encapsulates both the URL (`href`) and the relationship (`rel`) to the current resource. Link relations describe the nature of the relationship between the current resource and the linked resource, following standards like [IANA Link Relations](https://www.iana.org/assignments/link-relations/link-relations.xhtml).

#### Conceptual Overview

- **Link (`Link` Class)**:
    - **`href`**: The URL pointing to the linked resource.
    - **`rel`**: The relationship type, indicating how the linked resource is related to the current one.

- **Link Relations**:
    - **Standard Relations**: Predefined relations like `self`, `next`, `prev`, which have well-understood semantics.
    - **Custom Relations**: Application-specific relations that provide additional semantics relevant to the domain.

#### Links in hateoflux

- **Creation**: Links can be created manually or generated using utilities provided by hateoflux.
- **Representation**: Links are included in the `_links` section of the resource representation i.e. are part of any type of `Wrapper`, making the resources adhere to the HAL standard.
- **Navigation**: Clients use the links provided to navigate between resources, perform actions, or discover available operations. While a link consist mainly of an `href` and a `rel`, other attributes are also available such as `title` or `hreflang` and flags such as `templated` or `deprecated`. These additional attributes help chose the right link for that right purpose. 

## Pagination
Pagination is a crucial aspect of API development, especially when dealing with large datasets. It allows clients to request data in manageable chunks, reducing load times and improving the overall user experience. hateoflux provides robust support for pagination, enabling developers to easily implement it in their APIs while adhering to HATEOAS principles.

### Implementing Pagination with hateoflux

To implement pagination in hateoflux, you'll typically work with the following components:

- **`HalListWrapper`**: A wrapper for collections of resources.
- **`HalPageInfo`**: Contains pagination metadata.
- **`Link`**: Used for building hypermedia links, including navigation links.
- **`SortCriteria`**: Encapsulates sorting information for link generation.

hateoflux does not require you to use any specific pagination framework like Spring Data's `Pageable`. You can use your own query parameters, such as `pageSize` and `pageNumber`, to control pagination. This flexibility ensures that hateoflux can be integrated into projects without adding unnecessary dependencies.

### Key Considerations When Using Pagination in hateoflux

- **Flexibility in Pagination Parameters**: You are not limited to using `Pageable` from Spring Data. hateoflux allows you to define your own pagination parameters (e.g., `pageSize`, `pageNumber`) that suit your application's needs.

- **Non-Reliance on Specific Data Structures**: hateoflux doesn't require you to use specific data structures like `Page` from Spring Data. It operates on any `Flux`, `Mono`, or collection, providing flexibility in how you source and structure your data.

- **Sorting and Filtering**: While `SortCriteria` can be used to encapsulate sorting information, you're free to define your own mechanisms for sorting and filtering data. Ensure that your sorting parameters are correctly mapped to `SortCriteria` for accurate navigation link generation.

- **Link Template Compliance**: hateoflux supports URI templates in link definitions (see [here](./linkbuilding.html#defining-a-uri-template))

- **Handling Optional Query Parameters**: If certain query parameters are optional and may not have values, they can be safely included in your link templates. hateoflux will ignore them during expansion if no value is provided, as query parameters are optional by nature.

- **Absolute URLs**: Use `Link.prependBaseUrl(exchange)` or provide the base URL by other means to ensure that your links include the correct base URL. This is especially important when your service operates behind proxies or load balancers.

- **Pagination Metadata Consistency**: Providing consistent pagination metadata via `HalPageInfo` helps clients navigate your API more effectively. It aligns with common practices in RESTful API design and improves client-side pagination controls.

- **Customization of Embedded Resource Names**: If you wish to customize the key under which your resources appear in `_embedded`, you can use annotations like `@Relation`. This allows you to define a specific name rather than relying on the default naming convention, enhancing clarity in your API responses.
 
