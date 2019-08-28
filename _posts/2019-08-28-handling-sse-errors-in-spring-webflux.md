---
layout: post
title: Handling SSE errors in Spring WebFlux
date: '2019-08-28T23:01:00.000+04:00'
author: rpuchkovskiy
tags:
- spring webflux
- server-sent-events
- error handling
- java
---

[Spring WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)
makes it easy to write a streaming endpoint for your application. One of the options is
[Server-Sent-Events](https://en.wikipedia.org/wiki/Server-sent_events). Most of the required work is done behind
the scenes by Spring Webflux magic, you just need to return a `Flux` producing the required objects and
specify the correct content type. It's as easy as...

## Take one, naive

```java
@RestController
public class StreamingController {
    @Autowired
    private final PersonService personService;

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<PersonDto> stream() {
        return personSevice.personDtosFlux();
    }
}
```

We just return a `Flux` and specify `text/event-stream` content type as the produced one. Spring Webflux will
automatically convert each `PersonDto` instance to text (by default, it's JSON).

There you go, you can subscribe to `/stream` endpoint in your browser, and it will get a stream of `PersonDto`,
each of them serialized as JSON.

The output could look like the following:

```
data: {"name":"John"}

data: {"name":"Maria"}
```

You write a 'sunny day scenario' test, and everything works as expected.

```java
    @Test
    void streamsCollection() {
        when(personService.personDtosFlux()
                .thenReturn(Flux.just(new PersonDto("John"), new PersonDto("Maria")));
    
        List<PersonDto> dtos = webClient.get().uri("/stream")
                .accept(MediaType.TEXT_EVENT_STREAM)
                .exchange()
                .expectStatus().is2xxSuccessful()
                .expectBodyList(PersonDto.class)
                .returnResult()
                .getResponseBody();
    
        assertThatJohnAndMariaAreReturned(dtos);
    }
```

But what if an exception occur somewhere at the database level?

You model this situation with `.thenReturn(Flux.error(new RuntimeException("Oops)))` and... you don't see anything
'erroneous' in your test output! How come?

Well, when a stream channel between a user agent and a server is already established, the response is already
committed, so you just cannot change the response code when an exception is encountered *inside* the reactive
channel (represented with a `Flux`), so you just don't get any data after the error.
More than that, even if Spring Webflux is potentially able to detect
just before establishing the channel that the `Flux` is erroneous, it refuses to do so and just ignores the failure.
I don't know whether it is a bug or it is an intended behavior.
 
The bottomline is that if we want to somehow signal to the client that some error have happened, we have to use
the established channel for this. To do it, we can just return an error message instead of a DTO. To allow the
client to distinguish the erroneous case from the normal one, we can use 'event' mechanism of the
`Server-Sent-Events`.
 
To do it, we could employ `ServerSentEvent` class.
 
## Take two, error handling
 
```java
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Object>> stream() {
        return personSevice.personDtosFlux()
                .map(this::dtoToSse)
                .onErrorResume(e -> Mono.just(throwableToSse(e)));
    }

    private ServerSentEvent<Object> dtoToSse(PersonDto dto) {
        return ServerSentEvent.builder()
                .data(dto)
                .build();
    }

    private ServerSentEvent<Object> throwableToSse(Throwable e) {
        return ServerSentEvent.builder().event("internal-error")
                .data(e.getMessage())
                .build();
    }
```

Here, we explicitly specify what to return in the case of an error. If it happans, we produce an event called
'internal-error'.

Here is how the output could look like:

```
event: internal-error
data: Oops
```