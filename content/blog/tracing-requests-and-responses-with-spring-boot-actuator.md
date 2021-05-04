+++
date = "2017-08-19"
title = "Tracing Requests & Responses with Spring Boot Actuator"
type = "post"
ogtype = "article"
+++

Spring Boot Actuator is a sub-project that provides endpoints allow you to monitor and interact with your application. You can take a look at [complete list of endpoints](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html#production-ready-endpoints) but we will focus on `trace` and its customization in this post.

By default, `trace` endpoint is enabled for all HTTP requests and shows last 100 of them with 5 default properties:

- Request Headers
- Response Headers
- Cookies
- Errors
- Time Taken

So, when you hit `/trace`, the typical response will be like:

{{< gist sedooe 6ed35760c8a1388a39bc95ac1cbc2f88 >}}

The question is: how can we customize these properties? Is there any way to exclude some of them or include even more properties? The answer is YES.

[Here](https://github.com/spring-projects/spring-boot/blob/dd08c31b9a5c21da31ecc4e9408e33445dd2ea73/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/trace/TraceProperties.java#L67), you can see all available properties. Sometimes, the best documentation is the source code :).

In your configuration file you can specify the properties you want to trace:

`management.trace.include = remote_address, parameters`

> Note that this will override default properties. If you want to keep them, you have to specify them along with additional properties you want.

In the introduction part we said that it's enabled for all HTTP requests. It really is. If you played with it, probably you saw that requests made to `/trace` also traced. Let's say you don't want some endpoints to be traced such as `/admin/**`, `/trace` and endpoints provide static files like `/css/**` and `/js/**`.

To be able to prevent those endpoints to be traced, we need to extend [`WebRequestTraceFilter`](http://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/actuate/trace/WebRequestTraceFilter.html) and override its [`shouldNotFilter`](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/OncePerRequestFilter.html#shouldNotFilter-javax.servlet.http.HttpServletRequest-) method which inherited from [`OncePerRequestFilter`](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/OncePerRequestFilter.html).

{{< gist sedooe f3e4ec1dd49bb316a5a3f94425d267f7 >}}

`shouldNotFilter`'s default implementation was always returning false. Now with this implementation, it returns **true** for the endpoints we don't want to trace.
By annotating this class with `@Component`, we tell Spring to register it as a bean instead inherited `WebRequestTraceFilter`. No more configuration needed for this to work.

It's not over yet, we need more customization!

In the introduction part we also said that, `trace` endpoint shows last 100 requests by default. Unfortunately, there is no way to change it from configuration file directly but still it's a piece of cake.

{{< gist sedooe 0310f74811c6d811382c27d4d76fc2b1 >}}

By default, [`InMemoryTraceRepository`](http://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/actuate/trace/InMemoryTraceRepository.html) is used as an implementation of [`TraceRepository`](http://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/actuate/trace/TraceRepository.html). By extending it, we can expand the capacity, log requests explicitly or even persist them. It's up to you.

[Here](https://github.com/sedooe/blog-projects/tree/master/spring-boot-actuator-customization) is the source code of sample application.
