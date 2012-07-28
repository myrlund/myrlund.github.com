---
title: 64-bit integers in Javascript
layout: post
categories: Javascript
author: myrlund
---

A couple of weeks ago, upon implementing a web socket protocol using JSON-encoded messages, we ran into quite the problem.

Server side, various ids and generation numbers were stored as 64-bit integers. But alas, as it turns out, Javascript doesn't support 64-bit integers... At all. It doesn't even whine. Any number exceeding 9007199254740992 will silently be padded with zeros... No wonder we didn't get the expected results in our interactions with the server.

_In case you're one of those people: Javascript's numbers are represented using 64-bit floating points with double precision, and hence the integers can't exceed 2<sup>53</sup>. Consult the [ECMAscript language specification](http://ecma262-5.com/ELS5_HTML.htm#Section_8.5) for more information._

Anyways, back to the story. With the protocol well in production on other platforms, it wasn't likely to change, so _a hairy fix was needed_.

Luckily, the encoded JSON strings arrive over web sockets as just that: strings. Problem is, the moment one tries to `JSON.parse(jsonString)` it's already too late -- information is lost.

### Regex to the rescue

Our <del>temporary</del> solution consisted of three parts:

1. running a simple regex over the incoming strings before parsing them
2. running a similar regex over the outgoing strings after serializing them
3. using string representations of the int64 fields internally

The incoming messages we handled like so:

{% highlight javascript %}
var jsonString = '{"foo": 123456789123456789, "bar": 987654321987654321}'
var numbers = /("[^"]*":\s*)(\d{15,})(\[,}])/g
jsonString = jsonString.replace(numbers, "$1\"$2\"$3")
jsonString // => '{"foo": "123456789123456789", "bar": "987654321987654321"}'
{% endhighlight %}

...and the outgoing messages we handled like so:

{% highlight javascript %}
var jsonString = '{"foo": "123456789123456789", "bar": "987654321987654321"}'
var stringifiedNumbers = /("[^"]*":\s*)"(\d{15,})"(\[,}])/g
jsonString = jsonString.replace(stringifiedNumbers, "$1$2$3")
jsonString // => '{"foo": 123456789123456789, "bar": 987654321987654321}'
{% endhighlight %}

### The moral of the story...

1. Unless absolutely necessary, avoid using numeric ids if they can be expected to become very large.
2. Regexes are awesome, although completely unreadable.
