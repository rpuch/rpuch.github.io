---
layout: post
title: Unit-testing a Spring WebFlux streaming controller
date: '2019-08-29T21:39:00.000+04:00'
author: rpuchkovskiy
tags:
- spring webflux
- unit testing
- WebTestClient
- java
---

[Spring WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)
makes it easy to write a streaming endpoint for your application. But how do you test it?

## The controller

We'll test the following controller created in
the [previous article]({% post_url 2019-08-28-handling-sse-errors-in-spring-webflux %}):

```java
@RestController
public class StreamingController {
    @Autowired
    private final PersonService personService;

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
}
```

## MockMvc? (nope)

The first thing you could try is using the good old `MockMvc`:

```java
@WebMvcTest
@ContextConfiguration(...)
class StreamingControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private PersonServiceService service;

    @Test
    void streamsPersons() throws Exception {
        when(service.personDtosFlux())
                .thenReturn(Flux.just(new Person("John", "Smith"), new Person("Jane", "Doe")));

        String responseText = mockMvc.perform(get("/persons").accept(MediaType.TEXT_EVENT_STREAM))
                .andExpect(status().is2xxSuccessful())
                .andExpect(content().string(not(isEmptyString())))
                .andReturn()
                .getResponse()
                .getContentAsString();

        assertThatJohnAndJaneAreReturned(responseText);
    }
}
```

With one test method, this seems to work. But if you have another test method, weird things start happening.
More details in the [StackOverflow question](https://stackoverflow.com/questions/57631829/is-mockmvc-eligible-for-webflux-controllers-testing)
Long story short: `MockMvc` should not be used to test WebFlux endpoints using streaming.

## WebTestClient

Ok, there is a reactive analogue: `WebTestClient`.

```java
@WebFluxTest
@ContextConfiguration(classes = TestConfiguration.class)
class StreamingControllerTest {
    private WebTestClient webClient;

    @MockBean
    private PersonService service;


    @BeforeEach
    void initClient() {
        webClient = WebTestClient.bindToController(new StreamingController(personService)).build();
    }

    @Test
    void streamsPersons() {
        when(service.personDtosFlux())
                .thenReturn(Flux.just(new Person("John", "Smith"), new Person("Jane", "Doe")));

        List<PersonDto> dtos = webClient.get().uri("/stream")
                .accept(MediaType.TEXT_EVENT_STREAM)
                .exchange()
                .expectStatus().is2xxSuccessful()
                .expectBodyList(PersonDto.class)
                .returnResult()
                .getResponseBody();

        assertThatFirstAndSecondAreReturned(dtos);
    }

    @Test
    void whenAnExceptionOccursDuringStreaming_thenItShouldBeHandled() {
        when(service.personDtosFlux())
                .thenReturn(Flux.error(new RuntimeException("Oops!")));

        String responseText = webClient.get().uri("/stream")
                .accept(MediaType.TEXT_EVENT_STREAM)
                .exchange()
                .expectStatus().is2xxSuccessful()
                .expectBody(String.class)
                .returnResult()
                .getResponseBody();

        assertThat(responseText, is("event:internal-error\ndata:Oops!\n\n"));
    }
}
```

One note on `WebFluxTest` configuration.
`@WebFluxTest` accepts controller class name. It seems cleaner to specify the controller class
in the annotation and then inject `WebTestClient` automatically with `@Autowired` instead of creating it
by hand in a set-up method. But the controller class specified via `@WebFluxTest` actually works as a filter,
so, for it to work, the controller must already be in the application context configured for the test run.
For me, it seems too overcomplicated to put a controller class in a context just to have a possibility to
inject it back. And if you have autoscan here, you probably put a lot of containers to the test context,
which, again, seems to contradict the simplicity required by unit-testing a single controller.

On the other hand, if we just configure the `WebTestClient` manually in 2 lines of code (could be 1 line if you
like), we get a clear vision about how the controller is connected to the `WebTestClient`.

## Conclusion

A concise way to test WebFlux streaming endpoints in unit tests has been demonstrated.

P.S. Why do we use that `ServerSentEvent` in the controller? Please see the previous article on
[handling errors with SSE in Spring WebFlux]({% post_url 2019-08-28-handling-sse-errors-in-spring-webflux %})