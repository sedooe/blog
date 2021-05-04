+++
date = "2017-04-17"
title = "Intercepting and modifying logs in Logback"
type = "post"
ogtype = "article"
+++

Actually, Logback does not offer a direct way to modify logs but there is a workaround we can achieve our goal by using filter even though it looks like a bit hacky.

Let's say that you logged some id no of some user hundreds of times, it scattered through all over the application and now you have a new requirement that says you have to encrypt this id number. Of course you're smart enough to write an interceptor for this task instead of go find and change necessary logs manually. And also with this way, we can be sure that we'll never log that id number accidentally.

For this case, we're gonna extend [`TurboFilter`](https://logback.qos.ch/apidocs/ch/qos/logback/classic/turbo/TurboFilter.html) and override its `decide` method.

{{< gist sedooe a1658c2fb7ae89084b2f21457321f0e2 >}}

If the log matches our criteria and we want to change it, we deny it by returning `FilterReply.DENY` and log again with the changed object.

Here, I used `info` as log level for the sake of simplicity but if you don't want to change log level, you can easily check which `level` coming log has and use that level.

It's that easy once you deal with recursion and not forget to declare your custom filter in configuration file properly.

{{< gist sedooe 97f93f124dae080c9d958d1689c1697e >}}
