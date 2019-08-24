---
layout: post
title: Testing an Aspect in a Spring application
date: '2019-06-09T20:46:00.001+03:00'
author: rpuchkovskiy
tags:
- testing
- spring
- aspect
- java
modified_time: '2019-06-09T20:46:14.024+03:00'
blogger_id: tag:blogger.com,1999:blog-4160252863216482620.post-2574041500572598954
blogger_orig_url: https://rpuchkovskiy.blogspot.com/2019/06/testing-aspect-in-spring-application.html
---

Sometimes, an Aspect is the best tool to solve a task at hand. But how can we unit-test it? An obvious solution
is to start up the whole application context, including the Aspect, and then test how it behaves, but this will
be actually an integration test; such tests are heavy, and you have less control over the situations you can
model in such a test (for example, it may be pretty difficult to simulate an exceptional situation).

Actually, there are two things that are interesting from the testing perspective in an aspect:

1. The business logic that the aspect executes when triggered
1. The pointcut expression(s) which trigger aspect execution

Even the first of them is not so easy to test because you need an instance of
`ProceedingJoinPoint` which is cumbersome to implement or mock (and it is not recommended to mock external
interfaces, as it is explained in
<a target="_blank" href="http://www.informit.com/store/growing-object-oriented-software-guided-by-tests-9780321503626">Growing Object-Oriented Software, Guided by Tests)</a>,
for example).

### The solution

Let's imagine that we have an aspect that must throw an exception if a method's first argument is
`null`, otherwise allow the method invocation proceed.

It should only be applied to controllers annotated with our custom
`@ThrowOnNullFirstArg` annotation.

```java
@Aspect
public class ThrowOnNullFirstArgAspect {
    @Pointcut("" +
            "within(@org.springframework.stereotype.Controller *) || " +
            "within(@(@org.springframework.stereotype.Controller *) *)")
    private void isController() {}

    @Around("isController()")
    public Object executeAroundController(ProceedingJoinPoint point) throws Throwable {
        throwIfNullFirstArgIsPassed(point);
        return point.proceed();
    }

    private void throwIfNullFirstArgIsPassed(ProceedingJoinPoint point) {
        if (!(point.getSignature() instanceof MethodSignature)) {
            return;
        }

        if (point.getArgs().length > 0 && point.getArgs()[0] == null) {
            throw new IllegalStateException("The first argument is not allowed to be null");
        }
    }
}
```

We could test it like this:

```java
public class ThrowOnNullFirstArgAspectTest {
    private final ThrowOnNullFirstArgAspect aspect = new ThrowOnNullFirstArgAspect();
    private TestController controllerProxy;

    @Before
    public void setUp() {
        AspectJProxyFactory aspectJProxyFactory = new AspectJProxyFactory(new TestController());
        aspectJProxyFactory.addAspect(aspect);

        DefaultAopProxyFactory proxyFactory = new DefaultAopProxyFactory();
        AopProxy aopProxy = proxyFactory.createAopProxy(aspectJProxyFactory);

        controllerProxy = (TestController) aopProxy.getProxy();
    }

    @Test
    public void whenInvokingWithNullFirstArg_thenExceptionShouldBeThrown() {
        try {
            controllerProxy.someMethod(null);
            fail("An exception should be thrown");
        } catch (IllegalStateException e) {
            assertThat(e.getMessage(), is("The first argument is not allowed to be null"));
        }
    }

    @Test
    public void whenInvokingWithNonNullFirstArg_thenNothingShouldBeThrown() {
        String result = controllerProxy.someMethod(Descriptor.builder().externalId("id").build());

        assertThat(result, is("ok"));
    }

    @Controller
    @ThrowOnNullFirstArg
    private static class TestController {
        @SuppressWarnings("unused")
        String someMethod(Descriptor descriptor) {
            return "ok";
        }
    }
}
```

The key part is inside the `setUp()` method. Please note that it also allows to verify the correctness of
your pointcut expression, so it solves both problems.

Of course, in a real project it is better to extract the 'proxy contstruction' code to some helper class
to avoid code duplication and make the intentions clearer.
