---
title: "Cookbook: Examples &amp; Use Cases"
layout: default
nav_order: 4
---

# Cookbook: Examples &amp; Use Cases
{: .no_toc }
<br>
1. TOC
{:toc}
---

{: .important }
Every example shown can be viewed and debugged in the [hateoflux-demos repository](https://github.com/kamillionlabs/hateoflux-demos). Fork it and test as you explore options available! Either run the application and curl against the micro service or check the examples directly in the given unit tests.
<br>

## Setup
Let's first define the following two DTO classes that are going to be used in further examples for creating `HalResourceWrapper`, `HalEmbeddedWrapper` and `HalListWrapper`:

```java
import lombok.Data;

@Data
public class OrderDTO {
    int id;
    long userId;
    double total;
    String status;
}
```
and

```java
import de.kamillionlabs.hateoflux.model.hal.Relation;
import lombok.Data;

//defines the name of the class when serialized
@Relation(itemRelation = "shipment", collectionRelation = "shipments")  
@Data 
public class ShipmentDTO {
    int id;
    String carrier;
    String trackingNumber;
    String status;
}
```
## Manually Creating Wrappers

### Creating a `HalResourceWrapper` without an Embedded Resource

To create a `HalResourceWrapper` for a simple order without shipment details, the wrapper is implemented as shown below. This implementation typically resides within a controller method (or better yet, in an [assembler](./core-concepts/assemblers.html)):

```java
@GetMapping("/order-no-embedded/{orderId}")
public Mono<HalResourceWrapper<OrderDTO, Void>> getOrder(@PathVariable int orderId) {    // 1

    Mono<OrderDTO> orderMono = orderService.getOrder(orderId);                           // 2

    return orderMono.map(order -> HalResourceWrapper.wrap(order)                         // 3
            .withLinks(                                                                  // 4
                    Link.of("orders/{orderId}/shipment")                                 // 5
                            .expand(orderId)                                             // 6
                            .withRel("shipment"),                                        // 7
                    Link.linkAsSelfOf("orders/" + orderId)                               // 8
            ));
}
```
The numbered comments in the code correspond to the following explanations:

1. **Endpoint Mapping:** The `@GetMapping("/{id}")` annotation maps HTTP GET requests to the `getOrder()` method. Note the generic types for `HalResourceWrapper`. The first is the main resource, which is an `OrderDTO`. Since we don't have another embedded, i.e., secondary resource, the second generic is set to `Void`.
2. **Fetching the Order:** `orderService.getOrder()` is an arbitrary service call that could be reading a database or calling another service.
3. **Wrapping the Order:** The `Mono` is accessed, and the order is now wrapped. `HalResourceWrapper.wrap()` creates already a `HalResourceWrapper`. However, as it is now, no links or embedded resources are available.
4. **Adding Links:** This method adds an arbitrary number of links. Technically, there is no limit to how many links a resource can have. However, each `Link` needs a relation that is also present exactly once. So, even if multiple `Links` are added, each must be of a different `LinkRelation`.
5. **Creating a New Link:** With `Link.of()` a new `Link` can be created _manually_. The provided string is interpreted as being the `href` of the `Link`. In this case, it is a templated URI with `id` as a variable.
6. **Expanding the Template:** `expand(id)` replaces the `{id}` placeholder in the shipment link with the actual order ID, finalizing the URL.
7. **Defining the Relation:** `withRel()` assigns a relation name to the link, indicating its purpose in the context of the order resource.
8. **Self Link:** `withLinks()` in line 4 accepts an array of links (varargs). `Link.linkAsSelfOf("orders/" + id)` generates a `Link` with the relation `SELF`, i.e., a self-referential link. This method is unique of its kind as it sets the relation and an `href` simultaneously.

```javascript
{
   "id": 1234,
   "userId": 37,
   "total": 99.99,
   "status": "Processing",
   "_links": {
      "shipment": {
         "href": "orders/1234/shipment"
      },
      "self": {
         "href": "orders/1234"
      }
   }
}
```

The fields `id`, `total`, and `status` are part of the `OrderDTO` and were fetched by the `OrderService`. The links were built in the code example above. Note that each relation is the key to the link's attributes. In this case, we have two links with the relations "shipment" and "self", while both provide only an `href`.

### Creating a `HalResourceWrapper` with an Embedded Resource

{: .note }
The code might seem a bit lengthy; however, if you choose to use assemblers, they will handle most of it automatically for you (e.g. [see here](#creating-a-halresourcewrapper-with-a-resource-and-a-single-embedded))!

Now we want to create a `HalResourceWrapper` that doesn't just reference a shipment via a link, but also includes the whole object instead:

```java
@GetMapping("/order-with-embedded/{orderId}")
public Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> getOrderWithShipment(@PathVariable int orderId) {             //  1
    Mono<OrderDTO> orderMono = orderService.getOrder(orderId);                                                       //  2
    Mono<ShipmentDTO> shipmentMono = shipmentService.getShipmentByOrderId(orderId);                                  //  3
    return orderMono.zipWith(shipmentMono, (order, shipment) ->
            HalResourceWrapper.wrap(order)                                                                           //  4
                    .withLinks(                                                                                      //  5
                            Link.linkAsSelfOf("orders/" + orderId))                                                  //  6
                    .withEmbeddedResource(                                                                           //  7
                            HalEmbeddedWrapper.wrap(shipment)                                                        //  8
                                    .withLinks(                                                                      //  9
                                            linkTo(ShipmentController.class, c -> c.getShipment(shipment.getId()))   // 10
                                                    .withRel(IanaRelation.SELF)                                      // 11
                                                    .withHreflang("en-US")                                           // 12
                                    )
                    )
    );
}
```

The numbered comments in the code correspond to the following explanations:

1. **Endpoint Definition**: The `@GetMapping("/{id}")` annotation maps HTTP GET requests containing an order ID to the `getOrderWithShipment()` method. Note the generic types for `HalResourceWrapper`. The first is the main resource, which is an `OrderDTO` as before. However, since we now have an embedded resource, the second generic is set to `ShipmentDTO`.
2. **Order Retrieval**: `orderService.getOrder()` is an arbitrary service call that could be reading a database or calling another service.
3. **Shipment Retrieval**: `shipmentService.getShipmentByOrderId()` is an arbitrary service call that could be reading a database or calling another service.
4. **Combining Order and Shipment Data**: `orderMono.zipWith(shipmentMono, (order, shipment) -> ...)` merges the results of both `orderMono` and `shipmentMono`. This combination allows the creation of a `HalResourceWrapper` that encapsulates both the order and its associated shipment details.
5. **Adding Links to the Order Resource**: The `withLinks()` method attaches hypermedia links to the order resource.
6. **Self Link for the Order**: `Link.linkAsSelfOf("orders/" + orderId)` generates a self-referential link pointing to the current order resource. This time we didn't use a template but simply concatenated the `href`.
7. **Embedding the Shipment Resource**: The `withEmbeddedResource()` adds an embedded resource to the main resource.
8. **Wrapping Shipment Data for Embedding**: `HalEmbeddedWrapper.wrap(shipment)` transforms the `ShipmentDTO` into a `HalEmbeddedWrapper`. This wrapper prepares the shipment data for embedding within the order resource, ensuring it conforms to HAL standards.
9. **Adding Links to the Shipment Resource**: The `withLinks()` method is now part of the `HalEmbeddedWrapper` and adds links to the embedded resource, not to the main one.
10. **Creating a Self Link for the Shipment**: `linkTo(ShipmentController.class, c -> c.getShipment(shipment.getId()))` is part of the `SpringControllerLinkBuilder`. It constructs a link to the shipment resource by referencing the `getShipment()` method of the `ShipmentController`. This approach automatically builds the `href` based on the controller's and the method's endpoint by reading values from Spring's annotations such as `@RequestMapping`. It also automatically expands the URI templates and adds query parameters if any arguments are marked with `@RequestParam`.
11. **Defining the Relation Type for the Shipment Link**: `withRel(IanaRelation.SELF)` assigns the `self` relation type to the shipment link. This time we're using the `IanaRelation` enum, which defines a set of standardized relation types defined by [IANA](https://www.iana.org/assignments/link-relations/link-relations.xhtml).
12. **Specifying Additional Attributes of the Link**: `withHreflang("en-US")` sets the `hreflang` attribute of the shipment link to "en-US". This attribute informs clients about the language of the linked resource, which can be useful for content negotiation and accessibility.

The serialized result of this `HalResourceWrapper` is as follows:
```javascript
{
  "id": 1234,
  "userId": 37,
  "total": 99.99,
  "status": "Processing",
  "_embedded": {
    "shipment": {    // notice the name is not "shipmentDTO"
      "id": 127,
      "carrier": "UPS",
      "trackingNumber": "154-ASD-1238724",
      "status": "Completed",
      "_links": {
        "self": {
          "href": "/shipment/127",
          "hreflang": "en-US"
        }
      }
    }
  },
  "_links": {
    "self": {
      "href": "orders/1234"
    }
  }
}
```

The root fields are part of the main resource, `OrderDTO`. The node `_embedded` includes the embedded resource, `ShipmentDTO`. Notice how the name of the object is not "shipmentDTO" but "shipment". This is because the `ShipmentDTO` class has an `@Relation` annotation that defines how the class name should be written when it is serialized. Under `_links`, we can also see the two attributes that we added to the self link. One is the expanded `href` that the `SpringControllerLinkBuilder` extracted from the controller class and method, and the other is the `hreflang` we added.

### Creating a `HalListWrapper` with Pagination

{: .note }
The code might seem a bit lengthy; however, if you choose to use assemblers, they will handle most of it automatically for you (e.g. [see here](#creating-a-hallistwrapper-with-resources-each-having-a-single-embedded-with-paging))!

To not deviate too much from the previous examples, lets consider the use case, where the user wants to list all his orders. It is quite possible to implement this as `Flux`of `HalResourceWrapper<OrderDTO>`. However, we decide to create a `Mono` of `HalListWrapper<OrderDTO>`:

```java
import static de.kamillionlabs.hateoflux.utility.SortDirection.ASCENDING;
import static de.kamillionlabs.hateoflux.utility.SortDirection.DESCENDING;

@GetMapping("/orders-with-pagination")
public Mono<HalListWrapper<OrderDTO, Void>> getOrdersManualBuilt(@RequestParam Long userId,                       //  1
                                                                 Pageable pageable,                               //  2
                                                                 ServerWebExchange exchange) {                    //  3
    Flux<OrderDTO> ordersFlux = orderService.getOrdersByUserId(userId, pageable);                                 //  4
    Mono<Long> totalElementsMono = orderService.countAllOrdersByUserId(userId);                                   //  5

    int pageSize = pageable.getPageSize();
    long offset = pageable.getOffset();
    List<SortCriteria> sortCriteria = pageable.getSort().get()                                                    //  6
            .map(o -> SortCriteria.by(o.getProperty(), o.getDirection().isAscending() ? ASCENDING : DESCENDING))  //  7
            .toList();

    return ordersFlux.map(
                    order -> HalResourceWrapper.wrap(order)                                                       //  8
                            .withLinks(
                                    Link.linkAsSelfOf("orders/" + order.getId())
                                            .prependBaseUrl(exchange)))
            .collectList()                                                                                        //  9
            .zipWith(totalElementsMono,(ordersList, totalElements) -> {                                           // 10
                        HalPageInfo pageInfo = HalPageInfo.assembleWithOffset(pageSize, totalElements, offset);   // 11
                        return HalListWrapper.wrap(ordersList)                                                    // 12
                                .withLinks(Link.of("orders{?userId,someDifferentFilter}")                         // 13
                                        .expand(userId)                                                           // 14
                                        .prependBaseUrl(exchange)                                                 // 15
                                        .deriveNavigationLinks(pageInfo, sortCriteria))                           // 16
                                .withPageInfo(pageInfo);                                                          // 17
                    }
            );
}
```
The numbered comments in the code correspond to the following explanations:

1. **Endpoint Definition**: The `@GetMapping` annotation maps HTTP GET requests to the `getOrders()` method. Note that the method returns a `Mono` of list instead of a `Flux` of values.

2. **Accepting Pagination Parameters**: In cases where Spring Data is used, `Pageable` can encapsulate pagination information such as page size, page number, and sorting criteria. These details can be used to create hateoflux-specific objects that will be utilized, among other things, for link building.

3. **Accessing Server Exchange Context**: `ServerWebExchange` is automatically injected by Spring to provide access to the current HTTP request and response. It's used later to prepend the base URL when constructing hypermedia links, ensuring they are absolute and correctly reference the server.

4. **Retrieving Paginated Orders**: `orderService.getOrdersByUserId()` is a service call that retrieves a sorted list of `OrderDTO` objects for the specified `userId`. In contrast to Spring Data, hateoflux transforms an arbitrary `Flux` and does not rely on a `Page` or page-like structure.

5. **Counting Total Number of Orders**: `orderService.countAllOrdersByUserId()` returns a `Mono<Long>` representing the total number of orders for the given `userId`. This total count is essential for constructing pagination metadata like total pages and total elements in the response.

6. **Extracting Sort Information**: The `pageable.getSort().get()` method retrieves the sorting directives from the `Pageable` object. It returns a stream of `Sort.Order` objects, each representing a sorting instruction on a particular field, such as sorting by date or price.

7. **Mapping to Internal Sort Criteria**: The sorting stream is mapped to a list of `SortCriteria` objects.

8. **Wrapping Orders**: The `ordersFlux.map()` operation transforms each individual `OrderDTO` into a `HalResourceWrapper<OrderDTO>` by wrapping it and adding hypermedia links. This is important because the end result, i.e., `HalListWrapper`, only accepts resources that are wrapped in `HalResourceWrapper` and not raw resources on their own.

9. **Collecting Orders into a List**: The `collectList()` method aggregates all the `HalResourceWrapper<OrderDTO>` items into a `List<HalResourceWrapper<OrderDTO>>`.

10. **Combining Orders with Total Elements**: The `zipWith(totalElementsMono, (ordersList, totalElements) -> { ... })` subscribes to both `Publisher`s and makes them together available for further processing.

11. **Creating Pagination Metadata**: `HalPageInfo.assembleWithOffset()` creates a `HalPageInfo` object that contains pagination details like page size, total elements.

12. **Wrapping the Orders List in a HAL List Wrapper**: `HalListWrapper.wrap(ordersList)` creates a wrapper around the list of order resources. Similar to `HalResourceWrapper.wrap()`, the wrapper at this point only contains the resource and nothing else.

13. **Defining the Base Link for the Orders List**: `Link.of("orders{?userId,someDifferentFilter}")` creates a templated link for the `HalListWrapper` as a whole. `userId` and `someDifferentFilter` are query parameters that follow the structure defined in RFC6570, which hateoflux uses.

14. **Expanding Link Templates with Parameters**: `expand(userId)` replaces the `{userId}` placeholder in the link template with the actual `userId` value from the request parameters. Note that `someDifferentFilter` is missing and will therefore be simply ignored by the expansion, as query parameters are optional by nature.

15. **Appending Base URL to Links**: `prependBaseUrl()` extracts the base URL from the `ServerWebExchange` and prefixes the defined href with it. This ensures that the link is absolute and correctly references the server's address, which is crucial when the service is behind a proxy or load balancer.

16. **Adding Pagination Navigation Links**: `deriveNavigationLinks(pageInfo, sortCriteria)` automatically generates standard pagination links (i.e., `first`, `prev`, `next`, `last` as well as `self`) based on the specified href in line 13, the current page information, and sorting criteria. Note that the `href` generally corresponds to the **self link** of the resource list. Navigation links are always relative to the href initially provided. This line also concludes the creation of links.

17. **Including Pagination Metadata in the Response**: `withPageInfo()` attaches the `HalPageInfo` object to the `HalListWrapper`, providing clients with detailed pagination metadata. This includes information such as page size, total elements, total pages, and current page number.

The serialized result with example payload data of this `HalListWrapper` is as follows:


```javascript
{
  "page": {                                                    //1
    "size": 2,
    "totalElements": 6,
    "totalPages": 3,
    "number": 0
  },
  "_embedded": {                                               //2
    "orderDTOs": [                                             //3
      {
        "id": 1234,
        "userId": 37,
        "total": 99.99,
        "status": "Processing",
        "_links": {
          "self": {
            "href": "http://myservice:8080/orders/1234"
          }
        }
      },
      {
        "id": 1057,
        "userId": 37,
        "total": 72.48,
        "status": "Delivered",
        "_links": {
          "self": {
            "href": "http://myservice:8080/orders/1057"
          }
        }
      }
    ]
  },
  "_links": {                                                 //4
    "next": {
      "href": "http://myservice:8080/orders?userId=37?page=1&size=2&sort=id,desc"
    },
    "self": {
      "href": "http://myservice:8080/orders?userId=37?page=0&size=2&sort=id,desc"
    },
    "last": {
      "href": "http://myservice:8080/orders?userId=37?page=2&size=2&sort=id,desc"
    }
  }
}
```
The JSON has a few interesting points worth highlighting. The numbered comments are explained as follows:

1. **Pagination Metadata**: The `"page"` block contains pagination details provided by `withPageInfo()`. It includes fields such as:
   - `size`: Number of items per page (here, 2).
   - `totalElements`: Total number of items available (15).
   - `totalPages`: Total number of pages (8).
   - `number`: Current page index (starting from 0).

   These fields mirror those in a HAL JSON response from a service using Spring HATEOAS, ensuring consistency in pagination representation.

2. **Embedding Resources**: In HAL documents, lists of resources are included under the `_embedded` key. While single resources add their fields at the top level, collections are nested within `_embedded`. This structure allows clients to easily locate and parse embedded resources.

3. **Naming of Embedded Resources**: The list of resources within `_embedded` must have a key that names the collection. In this example, the resources are under `"orderDTOs"`, which is derived from the class name `OrderDTO` by converting it to lowercase and adding an "s" to pluralize. Since we did not use the `@Relation` annotation to specify a custom relation name, the default naming convention applies.

4. **Hypermedia Links and Navigation**: The `_links` section includes hypermedia links generated by the `deriveNavigationLinks()` method. It automatically created the `"next"`, `"self"`, and `"last"` links. The `"prev"` and `"first"` links were omitted because they are not applicable:
   - The `number` field indicates we are on page `0`, the first page, so there is no previous page.
   - The `"first"` link is redundant since `"self"` already points to the first page.
   - Each link's `href` includes query parameters such as `userId`, `page`, `size`, and `sort`, reflecting the current request parameters and pagination state.

   The inclusion of sorting parameters (e.g., `sort=id,desc`) is optional but provides clients with complete context for the data they are viewing.

## Create Wrappers Using Assemblers
When building wrappers manually, each field needs to be specified explicitly. This means that a single resource, a list of resources, pagination, and embedded resources all require different setups. However, with assemblers, this process is simplified. We only need to implement stubs that define how links are built, while the assemblers come with default implementations that can create wrappers in a single line, given the appropriate input.

In the following sections, we'll create an assembler and provide multiple examples of the different types of wrappers that can be generated with it.

{: .note }
All combinations shown here can also be created manually by using the public methods of the wrappers themselves. Reviewing the default implementations of the assemblers can help clarify how this is done.

### Creating the Assembler
The following is an assembler for wrappers that primarily wrap an `OrderDTO` and embed a `ShipmentDTO`. The assembler implements only the required methods, which mainly focus on creating self-links. Optional methods are available for adding additional links that may be desired.

```java
@Component
public class OrderAssembler implements EmbeddingHalWrapperAssembler<OrderDTO, ShipmentDTO> {                //1

    @Override
    public Class<OrderDTO> getResourceTClass() {                                                            //2
        return OrderDTO.class;                                                                              //3
    }

    @Override
    public Class<ShipmentDTO> getEmbeddedTClass() {                                                         //4
        return ShipmentDTO.class;
    }

    @Override
    public Link buildSelfLinkForResource(OrderDTO resourceToWrap, ServerWebExchange exchange) {             //5
        return Link.of("order/" + resourceToWrap.getId())                                                   //6
                .prependBaseUrl(exchange);                                                                  //7
    }

    @Override
    public Link buildSelfLinkForEmbedded(ShipmentDTO embedded, ServerWebExchange exchange) {                //8
        return Link.of("shipment/" + embedded.getId())
                .prependBaseUrl(exchange)
                .withHreflang("en-US");
    }

    @Override
    public Link buildSelfLinkForResourceList(ServerWebExchange exchange) {                                  //9
        MultiValueMap<String, String> queryParams = exchange.getRequest().getQueryParams();                 //10
        return Link.of("order{?userId,someDifferentFilter}")                                                //11
                .expand(queryParams)
                .prependBaseUrl(exchange);
    }
}
```
The numbered comments in the code correspond to the following explanations:

1. **Implementing the Interface**: The `OrderAssembler` implements the `EmbeddingHalWrapperAssembler` with the generic types `OrderDTO` and `ShipmentDTO`. Similarly to how the generics describe the main and embedded resource, the generics in assemblers follow the same logic.

2. **The Method `getResourceTClass()`**: It is a technical necessity. It is required so empty lists can be named correctly, as the name for lists is always derived from the class type (see [here for more details](/core-concepts/assemblers.html#additionally-specifying-the-resource-types-resourcet-and-embeddedt-explicitly)). The `@Relation` annotation is still honored (e.g., `OrderDTO` becomes `orders`).

3. **Implementation of `getResourceTClass()`**: The implementation is also trivial as it simply returns a `ResourceT` which the assembler already specifies.

4. **Implementation of `getEmbeddedTClass()`**: Similarly we implement the same for the embedded resource type.

5. **The Method `buildSelfLinkForResource()`**: Defines how the self link for the (main) resource should look. In this case, the self link is for any `OrderDTO` that is wrapped by the assembler. This applies only for a **single** resource.

6. **Implementation of `buildSelfLinkForResource()`**: The link is manually built here (as opposed to building it with `SpringControllerLinkBuilder`). The `resourceToWrap` is a given resource that the assembler wraps when prompted to. Note that the relation is automatically set, i.e., overwritten to "self". This means that setting the relation here has no effect.

7. **Prepending the base URL**: The `ServerWebExchange` is injected automatically into the controller by Spring, if specified, and holds various information about the HTTP request that a controller received. `Link.prependBaseUrl()` extracts the base URL (i.e., protocol, host, and port) from the `ServerWebExchange` and prepends it to the specified `href`.

8. **Implementation of `buildSelfLinkForEmbedded()`**: The method defines how the self link for the embedded resource should look. Technically, the embedded resource is also wrapped. In this case, the self link is for any `ShipmentDTO` that is embedded in an `OrderDTO` by the assembler. The base URL and an additional attribute `hreflang` are also added to the link.

9. **The Method `buildSelfLinkForResourceList`**: Defines the self link for a **list** of resources, i.e., a list of `OrderDTO`s. In contrast to `buildSelfLinkForResource`, which builds the self link for a single resource, this method does not provide the elements. Generally, this shouldn't be required in the first place, as the self link shouldn't contain information about each and every list.

10. **Accessing Query Parameters**: Among the other things that the `ServerWebExchange` provides are the query parameters used. By accessing them, we can construct the URL that was called to trigger the controller.

11. **Defining the Link**: Since we know that the controller makes use of query parameters, we need to specify them in the URL (it depends on the controller implementation, of course). The URL should correspond to whatever was called to trigger the controller. Note that we didn't use the `SpringControllerLinkBuilder` because `linkTo` is type-safe and expects exact types, whereas the `ServerWebExchange` only provides a `MultiValueMap<String, String>` that bundles together all variables. The link is then expanded and prepended with the base URL.

### Using an Assembler to Create a `HalListWrapper` For Resources With an Embedded Resource

Lets start with a more complicated setup to showcase what assemblers are capable of. In this example we'll create wrapper with the following characteristics:

* Contains a list of resources
* All resources have an embedded resource
* The list is paginated

Using the assembler we just created, the `OrderController` could have the following method:

```java
import static de.kamillionlabs.hateoflux.utility.SortDirection.ASCENDING;
import static de.kamillionlabs.hateoflux.utility.SortDirection.DESCENDING;

    @GetMapping("/orders-with-single-embedded-and-pagination")
    public Mono<HalListWrapper<OrderDTO, ShipmentDTO>> getOrdersWithShipmentAndPagination(
                                                                               @RequestParam(required = false) Long userId,     // 1
                                                                               Pageable pageable,                               // 2
                                                                               ServerWebExchange exchange) {                    // 3

        Flux<OrderDTO> orders = orderService.getOrders(userId, pageable);
        PairFlux<OrderDTO, ShipmentDTO> ordersWithShipment = 
                PairFlux.zipWith(orders, (order -> shipmentService.getLastShipmentByOrderId(order.getId())));                   // 4
                
        Mono<Long> totalElements = orderService.countAllOrders(userId);                                                         // 5

        int pageSize = pageable.getPageSize();                                                                                  // 6

        long offset = pageable.getOffset();
        List<SortCriteria> sortCriteria = pageable.getSort().get()
                .map(o -> SortCriteria.by(o.getProperty(), o.getDirection().isAscending() ? ASCENDING : DESCENDING))
                .toList();
        return orderAssembler.wrapInListWrapper(ordersWithShipment, totalElements, pageSize, offset, sortCriteria, exchange);   // 7
    }
}
```

The numbered comments in the code correspond to the following explanations:

1. **Endpoint Definition**: The `@GetMapping` annotation maps HTTP GET requests to the `getOrders()` method. Note that the method returns a `Mono` of a `HalListWrapper` with the same generics the assembler was configured with.

2. **Using `Pageable`**: Spring Data's `Pageable` can be very useful. hateoflux itself does not use Spring Data and hence has no access to it, and therefore there is no automatic conversion provided. However, below an example is given.

3. **Injecting a `ServerWebExchange`**: Spring automatically injects a `ServerWebExchange` if provided in the method signature of a REST controller. This can be useful to the assembler in order to build links.

4. **Getting Data**: The services `orderService` and `shipmentService` are arbitrary services that could be reading from a database or calling another service. The `PairFlux` is created by combing each order with its corresponding shipment.

5. **Get the Number of Total Elements**: Since generally in WebFlux we work with `Flux` and not `Page` instances, another query is required to get the total number of elements.

6. **Converting `Pageable` to a List of `SortCriteria`**: We read the data provided by Spring Data's `Pageable` and convert it to a list of hateoflux's `SortCriteria`.

7. **Create the Wrapper**: Finally, wrap all individual main and embedded resources and put them in a `HalListWrapper`. Pagination is added simply by using `wrapInListWrapper` that also accepts paging information.

The serialized result with example payload data of this `HalListWrapper` is as follows:

```javascript
{
   "page": {
      "size": 2,
      "totalElements": 6,
      "totalPages": 3,
      "number": 0
   },
   "_embedded": {
      "orderDTOs": [
         {
            "id": 1234,
            "userId": 37,
            "total": 99.99,
            "status": "Processing",
            "_embedded": {
               "shipment": {
                  "id": 127,
                  "carrier": "UPS",
                  "trackingNumber": "154-ASD-1238724",
                  "status": "Completed",
                  "_links": {
                     "self": {
                        "href": "http://myservice:8080/shipment/127",
                        "hreflang": "en-US"
                     }
                  }
               }
            },
            "_links": {
               "self": {
                  "href": "http://myservice:8080/order/1234"
               }
            }
         },
         {
            "id": 1057,
            "userId": 37,
            "total": 72.48,
            "status": "Delivered",
            "_embedded": {
               "shipment": {
                  "id": 105,
                  "carrier": "UPS",
                  "trackingNumber": "154-ASD-1284724",
                  "status": "Completed",
                  "_links": {
                     "self": {
                        "href": "http://myservice:8080/shipment/105",
                        "hreflang": "en-US"
                     }
                  }
               }
            },
            "_links": {
               "self": {
                  "href": "http://myservice:8080/order/1057"
               }
            }
         }
      ]
   },
   "_links": {
      "next": {
         "href": "http://myservice:8080/order?userId=37?page=1&size=2&sort=id,asc"
      },
      "self": {
         "href": "http://myservice:8080/order?userId=37?page=0&size=2&sort=id,asc"
      },
      "last": {
         "href": "http://myservice:8080/order?userId=37?page=2&size=2&sort=id,asc"
      }
   }
}
```
### Further Examples Using the Same Assembler

{: .important }
Reminder: Every example shown can be viewed and debugged in the [hateoflux-demos repository](https://github.com/kamillionlabs/hateoflux-demos). Fork it and test as you explore options available! Either run the application and curl against the micro service or check the examples directly in the given unit tests.
<br>

In this section, we will reuse the defined assembler to showcase other possibilities for its usage. The following examples do not include detailed explanations, unlike the example above. However, inline comments may be added where some guidance is deemed necessary. These examples exclusively demonstrate how to interact with the assembler using `Mono`s and `Flux`es, even though it is also possible to use non-reactive types.

<br>

#### Creating an Empty `HalResourceWrapper`

This is generally not possible. There must be at least a resource; otherwise, e.g. when wrapping an empty `Mono`, the assembler simply returns another empty `Mono`. This should not be an issue, as a service would typically respond with an HTTP 404 in such cases.

<br>

#### Creating a `HalResourceWrapper` with a Resource and No Embedded
##### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Mono<ShipmentDTO> embedded = Mono.empty();

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embedded, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalResourceWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

{: .highlight }
Note that an empty embedded object results in the removal of the `_embedded` node, whereas an [empty list/flux of embedded](./cookbook.html#creating-a-halresourcewrapper-with-a-resource-and-an-empty-list-of-embedded), results in an empty JSON array.

```json
{
   "id": 1234,
   "userId": 37,
   "total": 99.99,
   "status": "Processing",
   "_links": {
      "self": {
         "href": "https://www.example.com/order/1234"
      }
   }
}
```
</details>

<br>

#### Creating a `HalResourceWrapper` with a Resource and a Single Embedded
##### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Mono<ShipmentDTO> embedded = shipmentService.getShipment(127);

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embedded, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalResourceWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
   "id": 1234,
   "userId": 37,
   "total": 99.99,
   "status": "Processing",
   "_embedded": {
     "shipment": {
       "id": 127,
       "carrier": "UPS",
       "trackingNumber": "154-ASD-1238724",
       "status": "Completed",
       "_links": {
         "self": {
           "href": "https://www.example.com/shipment/127",
           "hreflang": "en-US"
         }
       }
     }
   },
   "_links": {
     "self": {
       "href": "https://www.example.com/order/1234"
     }
   }
 }
```
</details>

<br>


#### Creating a `HalResourceWrapper` with a Resource and a List of Embedded
##### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Flux<ShipmentDTO> embeddedList = shipmentService.getShipments(3287, 4125);

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embeddedList, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalResourceWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
   "id": 1234,
   "userId": 37,
   "total": 99.99,
   "status": "Processing",
   "_embedded": {
      "shipments": [
         {
            "id": 3287,
            "carrier": "DHL",
            "trackingNumber": "562-DHL-9182736",
            "status": "Pending",
            "_links": {
               "self": {
                  "href": "https://www.example.com/shipment/3287",
                  "hreflang": "en-US"
               }
            }
         },
         {
            "id": 4125,
            "carrier": "USPS",
            "trackingNumber": "739-USP-1827364",
            "status": "Completed",
            "_links": {
               "self": {
                  "href": "https://www.example.com/shipment/4125",
                  "hreflang": "en-US"
               }
            }
         }
      ]
   },
   "_links": {
      "self": {
         "href": "https://www.example.com/order/1234"
      }
   }
}
```
</details>

<br>

#### Creating a `HalResourceWrapper` with a Resource and an Empty List of Embedded
##### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Flux<ShipmentDTO> embeddedList = Flux.empty();

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embeddedList, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalResourceWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

{: .highlight }
Note that an empty list/flux of embedded results in an empty JSON array, whereas an [empty embedded object](./cookbook.html#creating-a-halresourcewrapper-with-a-resource-and-no-embedded), results in the removal of the `_embedded` node altogether.

```json
{
   "id": 1234,
   "userId": 37,
   "total": 99.99,
   "status": "Processing",
   "_embedded": {
     "shipments": []
   },
   "_links": {
     "self": {
       "href": "https://www.example.com/order/1234"
     }
   }
 }
```
</details>

<br>

#### Creating an Empty `HalListWrapper`
##### Code
{: .no_toc }

```java
//Given input
PairFlux<OrderDTO,ShipmentDTO> emptyPairFlux = PairFlux.empty();
MultiRightPairFlux<OrderDTO,ShipmentDTO> emptyMultiRightPairFlux = MultiRightPairFlux.empty();

//Assembler call
// Option 1
HalListWrapper<OrderDTO,ShipmentDTO> resultOp1 = orderAssembler.createEmptyListWrapper(OrderDTO.class, exchange);
// Option 2
Mono<HalListWrapper<OrderDTO,ShipmentDTO>> resultOp2 = orderAssembler.wrapInListWrapper(emptyPairFlux, exchange);
// Option 3
Mono<HalListWrapper<OrderDTO,ShipmentDTO>> resultOp3 = orderAssembler.wrapInListWrapper(emptyMultiRightPairFlux,exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalListWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
  "_embedded": {
    "orderDTOs": []
  },
  "_links": {
    "self": {
      "href": "https://www.example.com/order"
    }
  }
}
```
</details>

<br>

#### Creating a `HalListWrapper` with Resources Each Having a Single Embedded
##### Code
{: .no_toc }

```java
//Given input
Flux<OrderDTO> orders = orderService.getOrdersByUserId(38L);
PairFlux<OrderDTO, ShipmentDTO> resourcesWithEmbedded;

resourcesWithEmbedded = PairFlux.from(orders)
                                .with(order -> shipmentService.getLastShipmentByOrderId(order.getId()));

//Assembler call
Mono<HalListWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInListWrapper(resourcesWithEmbedded, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalListWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
    "_embedded": {
      "orderDTOs": [
        {
          "id": 9550,
          "userId": 38,
          "total": 149.99,
          "status": "Created",
          "_embedded": {
            "shipment": {
              "id": 3105,
              "carrier": "FedEx",
              "trackingNumber": "759-FDX-1029384",
              "status": "Out for Delivery",
              "_links": {
                "self": {
                  "href": "https://www.example.com/shipment/3105",
                  "hreflang": "en-US"
                }
              }
            }
          },
          "_links": {
            "self": {
              "href": "https://www.example.com/order/9550"
            }
          }
        },
        {
          "id": 5058,
          "userId": 38,
          "total": 149.99,
          "status": "Delivered",
          "_embedded": {
            "shipment": {
              "id": 5032,
              "carrier": "FedEx",
              "trackingNumber": "357-FDX-2938475",
              "status": "In Transit",
              "_links": {
                "self": {
                  "href": "https://www.example.com/shipment/5032",
                  "hreflang": "en-US"
                }
              }
            }
          },
          "_links": {
            "self": {
              "href": "https://www.example.com/order/5058"
            }
          }
        }
      ]
    },
    "_links": {
      "self": {
        "href": "https://www.example.com/order"
      }
    }
  }
```
</details>

<br>

#### Creating a `HalListWrapper` with Resources Each Having a Single Embedded with Paging
##### Code
{: .no_toc }

```java
//Given input
int pageNumber = 0;
int pageSize = 2;
Pageable pageable = PageRequest.of(pageNumber, pageSize); // This would usually be provided by Spring automatically
Flux<OrderDTO> orders = orderService.getOrdersByUserId(37L, pageable);

PairFlux<OrderDTO, ShipmentDTO> resourcesWithEmbedded;
resourcesWithEmbedded= PairFlux.from(orders)
                                .with(order -> shipmentService.getLastShipmentByOrderId(order.getId()));

Mono<Long> totalNumberOfElements = orderService.countAllOrdersByUserId(37L);

//Assembler call
Mono<HalListWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInListWrapper(resourcesWithEmbedded,
                                                                                              totalNumberOfElements,
                                                                                              pageSize,
                                                                                              pageable.getOffset(),
                                                                                              null, //for simplicity, we do not sort
                                                                                              exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalListWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
  "page": {
    "size": 2,
    "totalElements": 6,
    "totalPages": 3,
    "number": 0
  },
  "_embedded": {
    "orderDTOs": [
      {
        "id": 1234,
        "userId": 37,
        "total": 99.99,
        "status": "Processing",
        "_embedded": {
          "shipment": {
            "id": 127,
            "carrier": "UPS",
            "trackingNumber": "154-ASD-1238724",
            "status": "Completed",
            "_links": {
              "self": {
                "href": "https://www.example.com/shipment/127",
                "hreflang": "en-US"
              }
            }
          }
        },
        "_links": {
          "self": {
            "href": "https://www.example.com/order/1234"
          }
        }
      },
      {
        "id": 1057,
        "userId": 37,
        "total": 72.48,
        "status": "Delivered",
        "_embedded": {
          "shipment": {
            "id": 105,
            "carrier": "UPS",
            "trackingNumber": "154-ASD-1284724",
            "status": "Completed",
            "_links": {
              "self": {
                "href": "https://www.example.com/shipment/105",
                "hreflang": "en-US"
              }
            }
          }
        },
        "_links": {
          "self": {
            "href": "https://www.example.com/order/1057"
          }
        }
      }
    ]
  },
  "_links": {
    "next": {
      "href": "https://www.example.com/order?page=1&size=2"
    },
    "self": {
      "href": "https://www.example.com/order?page=0&size=2"
    },
    "last": {
      "href": "https://www.example.com/order?page=2&size=2"
    }
  }
}
```
</details>

<br>

#### Creating a `HalListWrapper` with Resources Each Having a List of Embedded
##### Code
{: .no_toc }

```java
//Given input
Flux<OrderDTO> ordersWithReturns = orderService.getOrdersByUserId(17L);
MultiRightPairFlux<OrderDTO, ShipmentDTO> resourcesWithEmbedded;
resourcesWithEmbedded = MultiRightPairFlux.from(ordersWithReturns)
                                          .with(order -> shipmentService.getShipmentsByOrderId(order.getId()));
//Assembler call
Mono<HalListWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInListWrapper(resourcesWithEmbedded, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalListWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
  "_embedded": {
    "orderDTOs": [
      {
        "id": 1070,
        "userId": 17,
        "total": 199.99,
        "status": "Returned",
        "_embedded": {
          "shipments": [
            {
              "id": 2551,
              "carrier": "UPS",
              "trackingNumber": "610-UPS-3748291",
              "status": "Completed",
              "_links": {
                "self": {
                  "href": "https://www.example.com/shipment/2551",
                  "hreflang": "en-US"
                }
              }
            },
            {
              "id": 3904,
              "carrier": "DHL",
              "trackingNumber": "680-DHL-9182736",
              "status": "Completed",
              "_links": {
                "self": {
                  "href": "https://www.example.com/shipment/3904",
                  "hreflang": "en-US"
                }
              }
            }
          ]
        },
        "_links": {
          "self": {
            "href": "https://www.example.com/order/1070"
          }
        }
      },
      {
        "id": 5078,
        "userId": 17,
        "total": 34.0,
        "status": "Returned",
        "_embedded": {
          "shipments": [
            {
              "id": 3750,
              "carrier": "USPS",
              "trackingNumber": "755-USP-8374652",
              "status": "Completed",
              "_links": {
                "self": {
                  "href": "https://www.example.com/shipment/3750",
                  "hreflang": "en-US"
                }
              }
            },
            {
              "id": 4203,
              "carrier": "FedEx",
              "trackingNumber": "920-FDX-5647382",
              "status": "Completed",
              "_links": {
                "self": {
                  "href": "https://www.example.com/shipment/4203",
                  "hreflang": "en-US"
                }
              }
            }
          ]
        },
        "_links": {
          "self": {
            "href": "https://www.example.com/order/5078"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "https://www.example.com/order"
    }
  }
}
```
</details>

<br> 

#### Creating a `HalListWrapper` with Resources Each Having a List of Embedded with Some Being `null`/Empty
##### Code
{: .no_toc }

```java
//Given input
Flux<OrderDTO> ordersWithAndWithoutShipments = orderService.getOrdersByUserId(39L);
MultiRightPairFlux<OrderDTO, ShipmentDTO> resourcesWithEmbedded;
resourcesWithEmbedded = MultiRightPairFlux.from(ordersWithAndWithoutShipments)
                                          .with(order -> shipmentService.getShipmentsByOrderId(order.getId()));
        
//Assembler call
Mono<HalListWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInListWrapper(resourcesWithEmbedded, exchange);
```

##### Output
{: .no_toc }

The serialized result of the `HalListWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

```json
{
   "_embedded": {
      "orderDTOs": [
         {
            "id": 7250,
            "userId": 39,
            "total": 34.0,
            "status": "Created",
            "_embedded": {
               "shipments": []
            },
            "_links": {
               "self": {
                  "href": "https://www.example.com/order/7250"
               }
            }
         },
         {
            "id": 1230,
            "userId": 39,
            "total": 99.99,
            "status": "Delivered",
            "_embedded": {
               "shipments": [
                  {
                     "id": 4005,
                     "carrier": "FedEx",
                     "trackingNumber": "634-FDX-8473621",
                     "status": "Delivered",
                     "_links": {
                        "self": {
                           "href": "https://www.example.com/shipment/4005",
                           "hreflang": "en-US"
                        }
                     }
                  }
               ]
            },
            "_links": {
               "self": {
                  "href": "https://www.example.com/order/1230"
               }
            }
         }
      ]
   },
   "_links": {
      "self": {
         "href": "https://www.example.com/order"
      }
   }
}
```
</details>

<br>
