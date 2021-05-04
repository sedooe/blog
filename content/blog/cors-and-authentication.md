+++
date = "2016-08-28"
title = "CORS and Authentication"
type = "post"
ogtype = "article"
+++

Let's say that you have an API which stands on **xdomain.com** and you have a Javascript application that consuming this API from **ydomain.com**. For ydomain.com to consume the API, xdomain.com has to send [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) header  `Access-Control-Allow-Origin` in its response.

The simplest way to do this in Spring is annotate whole controller or just handler method with the annotation [`CrossOrigin`](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/CrossOrigin.html).

It's all what you need when your API does not require authentication.

On the other hand, when your API requires authentication, things get a bit complicated. For example, let's say that you authenticate your API with HTTP Basic Authentication. So, consumers of this API must send an **Authorization** header which contains user credentials in their request. Since this is a [non-simple request](http://stackoverflow.com/a/10636765/3099704), **OPTIONS** request will be send firstly. The thing is that, even if you send Authorization header with correct credentials in your request, you will face something like below:

![response](https://sedooe.github.io/img/options_response.png)

The reason of this is, according to CORS spec, it excludes your Authorization header. Hence, you should permit OPTIONS requests in your security configuration class explicitly.

{{< gist sedooe 5c5735ca41a96a6d7be7a73d783334ba >}}

Last of all, I suggest you to read these two posts:

[http://stackoverflow.com/a/15734032/3099704](http://stackoverflow.com/a/15734032/3099704)
[https://code.google.com/archive/p/twitter-api/issues/2273](https://code.google.com/archive/p/twitter-api/issues/2273)
