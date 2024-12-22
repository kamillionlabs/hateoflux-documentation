---
title: "Creating Wrappers Using Assemblers"
layout: default
parent: "Cookbook: Examples &amp; Use Cases"
nav_order: 2
---

# Creating Wrappers Using Assemblers
{: .no_toc }
<br>
1. TOC
{:toc}
---

{: .important }
<b>Reminder</b>: Every example shown can be viewed and debugged in the [hateoflux-demos repository](https://github.com/kamillionlabs/hateoflux-demos). Clone or fork it and test as you explore options available! Either run the application and curl against the micro service or check the examples directly in the given unit tests.
<br>

When building wrappers manually, each field needs to be specified explicitly. This means that a single resource, a list of resources, pagination, and embedded resources all require different setups. However, with assemblers, this process is simplified. We only need to implement stubs that define how links are built, while the assemblers come with default implementations that can create wrappers in a single line, given the appropriate input.

In the following sections, we'll create an assembler and provide multiple examples of the different types of wrappers that can be generated with it.

{: .note }
All combinations shown here can also be created manually by using the public methods of the wrappers themselves. Reviewing the default implementations of the assemblers can help clarify how this is done.

## Creating the Assembler
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

## Using an Assembler to Create a `HalListWrapper` For Resources With an Embedded Resource

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
## Further Examples Using the Same Assembler
In this section, we will reuse the defined assembler to showcase other possibilities for its usage. The following examples do not include detailed explanations, unlike the example above. However, inline comments may be added where some guidance is deemed necessary. These examples exclusively demonstrate how to interact with the assembler using `Mono`s and `Flux`es, even though it is also possible to use non-reactive types.


### Creating an Empty `HalResourceWrapper`

This is generally not possible. There must be at least a resource; otherwise, e.g. when wrapping an empty `Mono`, the assembler simply returns another empty `Mono`. This should not be an issue, as a service would typically respond with an HTTP 404 in such cases.

<br>

### Creating a `HalResourceWrapper` with a Resource and No Embedded
#### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Mono<ShipmentDTO> embedded = Mono.empty();

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embedded, exchange);
```

#### Output
{: .no_toc }

The serialized result of the `HalResourceWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

{: .highlight }
Note that an empty embedded object results in the removal of the `_embedded` node, whereas an [empty list/flux of embedded](./assembler-based-wrapper-creation.html#creating-a-halresourcewrapper-with-a-resource-and-an-empty-list-of-embedded), results in an empty JSON array.

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

### Creating a `HalResourceWrapper` with a Resource and a Single Embedded
#### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Mono<ShipmentDTO> embedded = shipmentService.getShipment(127);

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embedded, exchange);
```

#### Output
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


### Creating a `HalResourceWrapper` with a Resource and a List of Embedded
#### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Flux<ShipmentDTO> embeddedList = shipmentService.getShipments(3287, 4125);

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embeddedList, exchange);
```

#### Output
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

### Creating a `HalResourceWrapper` with a Resource and an Empty List of Embedded
#### Code
{: .no_toc }

```java
// Given input
Mono<OrderDTO> resource = orderService.getOrder(1234);
Flux<ShipmentDTO> embeddedList = Flux.empty();

// Assembler call
Mono<HalResourceWrapper<OrderDTO, ShipmentDTO>> result = orderAssembler.wrapInResourceWrapper(resource, embeddedList, exchange);
```

#### Output
{: .no_toc }

The serialized result of the `HalResourceWrapper` is as follows:
<details closed markdown="block">
<summary style="color: #ffffff; background-color: #989898; padding: 0.6em 1em">
 <b>Click to expand</b>
</summary>

{: .highlight }
Note that an empty list/flux of embedded results in an empty JSON array, whereas an [empty embedded object](./assembler-based-wrapper-creation.html#creating-a-halresourcewrapper-with-a-resource-and-no-embedded), results in the removal of the `_embedded` node altogether.

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

### Creating an Empty `HalListWrapper`
#### Code
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

#### Output
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

### Creating a `HalListWrapper` with Resources Each Having a Single Embedded
#### Code
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

#### Output
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

### Creating a `HalListWrapper` with Resources Each Having a Single Embedded with Paging
#### Code
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

#### Output
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

### Creating a `HalListWrapper` with Resources Each Having a List of Embedded
#### Code
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

#### Output
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

### Creating a `HalListWrapper` with Resources Each Having a List of Embedded with Some Being `null`/Empty
#### Code
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

#### Output
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