---
title: 64-bit integers in Javascript
layout: post
categories: Javascript
---

A couple of weeks ago, upon implementing a web socket protocol using JSON-encoded messages, we ran into quite the problem.

Server side, various ids and generation numbers were stored as 64-bit integers. But alas, as it turns out, Javascript doesn't support 64-bit integers... At all. It doesn't even whine. Any number exceeding 9007199254740992 will silently be padded with zeros... No wonder we didn't get the expected results in our interactions with the server.

With the protocol well in production on other platforms, it wasn't likely to change, so _a hairy fix was needed_.

Luckily, the encoded JSON strings arrive over web sockets as just that: strings. Problem is, the moment one tries to `JSON.parse(jsonString)` it's already too late -- information is lost.

### Regex to the rescue

Our <del>temporary</del> solution became running a simple regex over the incoming strings before parsing them, the outgoing strings after serializing them, and using string representations of the ids internally in our app.

The incoming messages, we fixed by:

{% highlight javascript %}
var numbers = /("[^"]*":\s*)(\d{15,})(\[,}])/g
jsonString = jsonString.replace(numbers, "$1\"$2\"$3")
{% endhighlight %}

...and the outgoing messages by:

{% highlight javascript %}
var stringifiedNumbers = /("[^"]*":\s*)"(\d{15,})"(\[,}])/g
jsonString = jsonString.replace(stringifiedNumbers, "$1$2$3")
{% endhighlight %}

### The moral of the story...

1. Unless absolutely necessary, avoid using numeric ids if they can be expected to become very large.
2. Regexes are awesome, although completely unreadable.
