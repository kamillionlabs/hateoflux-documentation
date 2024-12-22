---
title: "Using Reactive Response Types"
layout: default
parent: "Cookbook: Examples &amp; Use Cases"
nav_order: 3
---

# Using Reactive Response Types
{: .no_toc }
<br>
1. TOC
{:toc}
---

{: .important }
<b>Reminder</b>: Every example shown can be viewed and debugged in the [hateoflux-demos repository](https://github.com/kamillionlabs/hateoflux-demos). Clone or fork it and test as you explore options available! Either run the application and curl against the micro service or check the examples directly in the given unit tests.
<br>

## Statuscode-Only Response

Sometimes an API endpoint needs to return just an HTTP status code without any content. While Spring's `ResponseEntity` supports this directly, hateoflux provides a more type-safe approach that maintains consistency with other HAL responses.

### Implementation

```java
@PostMapping("/im-a-teapot")
public HalResourceResponse<BookDTO, AuthorDTO> imATeapot() {
    return HalResourceResponse.of(HttpStatus.I_AM_A_TEAPOT);
}
```

### Key Points
* The `of()` factory method accepts any `HttpStatus` and creates an empty response
* Despite the generic types (`BookDTO`, `AuthorDTO`), no actual content is included
* The response will have:
  * Status code: `HTTP 418  I'm a teapot`
  * Empty body
  * Default headers

### Alternative Approaches
The same pattern works with any response type:
```java
// For single resource responses
HalResourceResponse.of(HttpStatus.NO_CONTENT);

// For list responses
HalListResponse.of(HttpStatus.NOT_FOUND);

// For streaming responses
HalMultiResourceResponse.of(HttpStatus.SERVICE_UNAVAILABLE);
```

## Factory Method Response

hateoflux provides convenient factory methods for common HTTP status codes. While these methods are semantically equivalent to using `of(HttpStatus)`, they provide clearer intent and reduced boilerplate.

### Implementation

```java
@PostMapping("/book")
public HalResourceResponse<BookDTO, AuthorDTO> createBook(@RequestBody BookDTO book) {
    Mono<HalResourceWrapper<BookDTO, AuthorDTO>> createdBook = bookService.addBook(book)
            .map(BookController::wrapBook);
            
    return HalResourceResponse.created(createdBook);
}
```
### Key Points
Factory methods available for common status codes:
* `ok()` - `HTTP 200`
* `created()` - `HTTP 201`
* `accepted()` - `HTTP 202`
* `noContent()` - `HTTP 204`
* `notFound()` - `HTTP 404`

An equivalent manual approach:

```java
return HalResourceResponse.of(createdBook, Mono.just(HttpStatus.CREATED));
```

### Alternative Approaches
The same pattern works for all response types:
```java
// For list responses
HalListResponse.created(listWrapper);

// For streaming responses
HalMultiResourceResponse.created(streamingResources);
```

## Conditional Status Response

Often, the HTTP status of a response needs to be determined based on some condition, such as whether a resource exists. hateoflux allows combining reactive content with reactive status codes.

### Implementation

```java
@PutMapping("/book")
public HalResourceResponse<BookDTO, AuthorDTO> updateBook(@RequestBody BookDTO book) {
    Mono<HalResourceWrapper<BookDTO, AuthorDTO>> updateBook = bookService.updateBook(book)
            .map(BookController::wrapBook);
            
    Mono<HttpStatus> statusCode = updateBook.hasElement()
            .map(exists -> exists ? HttpStatus.OK : HttpStatus.NOT_FOUND);
            
    return HalResourceResponse.of(updateBook, statusCode);
}
```
### Key Points

* Status code is determined reactively based on content existence
* The `of()` method accepts both reactive content and reactive status
* Empty content results in a `HTTP 404`
* Present content results in a `HTTP 200`
* The response wraps both decisions in a single operation

## Collection Response with Status

When returning multiple resources as a single HAL document, `HalListResponse` combines list handling with status determination. This is particularly useful for collections that might be empty or require specific status codes.

### Implementation

```java
@GetMapping("/books")
public HalListResponse<BookDTO, AuthorDTO> getAllBooksOfAuthor(@RequestParam String authorName) {
    Flux<HalResourceWrapper<BookDTO, AuthorDTO>> allBooks = bookService.getAllBooksByAuthorName(authorName)
            .map(BookController::wrapBook);

    // Empty Flux will result in an empty HalListWrapper with a self link, not an empty Mono
    Mono<HalListWrapper<BookDTO, AuthorDTO>> allBooksAsList = wrapBooks(authorName, allBooks);

    Mono<HttpStatus> status = allBooks.hasElements()
            .map(exists -> exists ? HttpStatus.OK : HttpStatus.NOT_FOUND);

    return HalListResponse.of(allBooksAsList, status);
}
```
### Key Points

* Collection of resources wrapped in `HalListWrapper`
* Status determined by collection's emptiness
* Resources collected into a list before wrapping
* Empty collections result in an empty embedded array, not null

## Streaming Resource Response

When you need to stream resources individually rather than collecting them into a list, `HalMultiResourceResponse` enables sending resources as they become available. This is particularly useful for large collections or real-time data.

### Implementation

```java
@GetMapping("/books-as-stream")
public HalMultiResourceResponse<BookDTO, AuthorDTO> getAllBooksOfAuthorAsStream(@RequestParam String authorName) {
    Flux<HalResourceWrapper<BookDTO, AuthorDTO>> allBooks = bookService.getAllBooksByAuthorName(authorName)
            .map(BookController::wrapBook);
            
    return HalMultiResourceResponse.of(allBooks, HttpStatus.OK);
}
```

### Key Points

* Resources are streamed individually as they are produced
* Status code must be determined upfront
* Unlike `HalListResponse`, no collection wrapper is used
* Each resource maintains its HAL structure
* Resources can be consumed by the client as they arrive

## Complex Conditional Response

Sometimes responses need to handle multiple conditions affecting both content and status. This example demonstrates conditional embedding of resources and corresponding status code selection based on resource availability.

### Implementation

```java
 @GetMapping("/book/{id}")
 public HalResourceResponse<BookDTO, AuthorDTO> getBook(
         @PathVariable("id") int bookId,
         @RequestParam(name = "withAuthor", defaultValue = "false") boolean withAuthor) {
     
     Mono<BookDTO> bookMono = bookService.getBookById(bookId);
     
     // Conditionally retrieve the author only if requested
     Mono<AuthorDTO> authorMono = withAuthor
             ? bookMono.flatMap(book -> bookService.getAuthorByName(book.getAuthor()))
             : Mono.empty();
     
     // Determine the status code based on the presence of BookDTO and (optionally) AuthorDTO
     Mono<HttpStatus> statusMono = bookMono.hasElement().flatMap(bookExists -> {
         if (!bookExists) {
             // No book found
             return Mono.just(HttpStatus.NOT_FOUND);
         } else if (!withAuthor) {
             // BookDTO exists and author not requested: always 200
             return Mono.just(HttpStatus.OK);
         } else {
             // BookDTO exists and author was requested: check author presence
             return authorMono.hasElement().map(authorExists ->
                     authorExists ? HttpStatus.OK : HttpStatus.PARTIAL_CONTENT
             );
         }
     });
     
     // Construct the body (HalResourceWrapper)
     Mono<HalResourceWrapper<BookDTO, AuthorDTO>> bodyMono = bookMono.flatMap(book -> {
         Mono<HalResourceWrapper<BookDTO, AuthorDTO>> bookWithoutAuthor = Mono.just(wrapBook(book));
         if (!withAuthor) {
             return bookWithoutAuthor;
         } else {
             // With author requested
             return authorMono
                     .map(author -> wrapBook(book, author))
                     .switchIfEmpty(bookWithoutAuthor);
         }
     });
     
     return HalResourceResponse.of(bodyMono, statusMono)
             .withContentType("application/hal+json");
 }
```
### Key Points

* Multiple conditions affect both content and status:
  * Book existence determines `HTTP 404` vs. other statuses
  * Author request flag affects embedded content
  * Author availability affects status (`HTTP 200` vs. `HTTP 206`)
* Content type header explicitly set
* Response supports partial content scenarios
* Empty embedded resources handled gracefully
* Status determination follows a clear decision tree
