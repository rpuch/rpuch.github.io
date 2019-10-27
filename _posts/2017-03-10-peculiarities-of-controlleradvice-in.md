---
layout: post
title: Peculiarities of @ControllerAdvice in Spring MVC
date: '2017-03-10T23:07:00.000+03:00'
author: rpuchkovskiy
tags:
- "@ControllerAdvice"
- spring-mvc
- peculiarities
- "@ExceptionHandler"
modified_time: '2017-03-10T23:07:19.687+03:00'
blogger_id: tag:blogger.com,1999:blog-4160252863216482620.post-6742828658983487732
blogger_orig_url: https://rpuchkovskiy.blogspot.com/2017/03/peculiarities-of-controlleradvice-in.html
excerpt_separator: <!--more-->
---

### What is @ControllerAdvice?

`@ControllerAdvice` annotation is used to define some attributes for many
[Spring](https://github.com/spring-projects/spring-framework) controllers
(`@Controller`) at a time.

For example, it can be used for a centralized exception control (with `@ExceptionHandler` annotation).
This post will concentrate on this use.

<!--more-->

### Global and specific advices

If a `@ControllerAdvice` does not have any selectors specified via annotation attritubes, it defines a global
advice which affects all the controllers, and its `@ExceptionHandler` will catch all the exceptions thrown by
handle methods (and not just these exceptions, see below).

In [Spring 4.0](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/new-in-4.0.html), the ability
to define specific advices was added. `@ControllerAdvice` now has some
[attributes](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/bind/annotation/ControllerAdvice.java)
with which we can define advice selectors. These selectors define the scope of the advice, i.e. the exact set of
controllers which will be adviced by it.

### Global @ExceptionHandler catches 'no man's' exceptions

'No man's' exceptions are exceptions which occur before the handler to process the request is obtained.

So, if we don't specify any attributes at `@ControllerAdvice` annotation, its `@ExceptionHandler` method will catch even
`HttpRequestMethodNotSupportedException` when someone tries to issue a GET request to our POST-only controller.

But if we define a class, annotation or `basePackage` *for an existing package*, then the advice will not be global
anymore and will not catch 'no man's' exceptions.

### basePackages does not work for controllers proxied with Proxy

To have the ability to intercept controller invocations (for example, to handle exceptions), our advice has to wrap
controller instance in a proxy. Proxy creation options:

1. If the controller class has at least one implemented interface, an interface-based proxy is created using `Proxy` class.
2. If the controller class does not implement any inferfaces, then [CGLIB](https://github.com/cglib/cglib) is used;
proxy class is created at runtime; this class extends our initial controller class.
3. For CGLIB option, Spring's [`ControllerAdviceBean`](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/method/ControllerAdviceBean.java)
determines correctly the controller class package: it just takes the superclass of CGLIB-generated class (this will be
the real controller class as written by us) and then takes its package. In this case, `basePackages` of
`@ControllerAdvice` works correctly.

But for the interface-proxy based option Spring has no ability to determine the real class of an instance wrapped with
`Proxy`, so it tries to take the package of the `Proxy` instance. But `proxy.getClass().getPackage()` returns `null`!
In Spring 4.0.5 this even causes a `NullPointerException`. In 4.0.9 the NPE was fixed, but the package will not be determined
correctly, so `basePackages` will not work.

To sum up:

1. If the controller class has at least one implemented interface, an interface-based proxy is created using `Proxy` class,
and `basePackages` of `@ControllerAdvice` **DOES NOT** work.
2. If the controller class does not implement any inferfaces, then CGLIB is used and `basePackages` attribute works.

### What can we do?

Use `annotations` attribute. First, let's create an annotation like

```java
@ControllerAdvicedByMyAdvice
```

Then we annotate with this annotation all the controllers to which we want to apply the advice. And then we annotate
the advice:

```java
@ControllerAdvice(annotations = {ControllerAdvicedByMyAdvice.class})
```

This approach seems more reliable than the utilization of the `basePackages` attribute.