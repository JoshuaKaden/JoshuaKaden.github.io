---
layout: post
title:  "Shorthand Initializers For Complex Objects"
date:   2020-03-03 13:45:28 -0500
categories: ios
tags: swift programming closure initializer
---
*Try a new way of instantiating your complicated objects with this closure-based initializer.*

While looking at colleague's code in a [Kotlin Multiplatform][kmp] project, I ran across this interesting instantiation of an object in the networking framework [ktor][ktor]:

{% highlight kotlin %}
val client = HttpClient {
    expectSuccess = false
}
{% endhighlight %}

What is `expectSuccess`? Where did that come from? I had questions.

I cmd-clicked through to the definition, and found:

{% highlight kotlin %}
/**
 * Constructs an asynchronous [HttpClient] using optional [block] for configuring this client.
 *
 * The [HttpClientEngine] is selected from the dependencies.
 * https://ktor.io/clients/http-client/engines.html
 */
@HttpClientDsl
expect fun HttpClient(
    block: HttpClientConfig<*>.() -> Unit = {}
): HttpClient
{% endhighlight %}

I don't claim to understand every aspect of this, but I can see that the object is initialized via a closure. The closure has a configuration object (`HttpClientConfig`) passed into it as a parameter. This configuration object can then be tweaked as needed, before it is used to configure the client object.

I had not run across this technique before.

Here's the original code again:

{% highlight kotlin %}
val client = HttpClient {
    expectSuccess = false
}
{% endhighlight %}

This is the equivalent of:

{% highlight kotlin %}
var config = HttpClientConfig()
config.expectSuccess = true
val client = HttpClient(config: config)
{% endhighlight %}

...which is just fine, obviously, but that first one feels so much cleaner. Or, at least, it feels new, which is kind of exciting.

My next thought was: Can I do this in Swift?

Yes, I Can Do This In Swift
---------------------------

Here is what I came up with in Swift, to mimic this behavior.

{% highlight swift %}
struct HttpClientConfig {
    var expectSuccess: Bool = true
}

final class HttpClient {
    let config: HttpClientConfig

    convenience init(_ configurationClosure: (inout HttpClientConfig) -> Void) {
        var config = HttpClientConfig()
        configurationClosure(&config)
        self.init(config: config)
    }

    init(config: HttpClientConfig) {
        self.config = config
    }
}

let client = HttpClient {
    config in
    config.expectSuccess = false
}
print(client.config.expectSuccess) // returns `false`
{% endhighlight %}

Conclusion
----------
I think this could be a useful solution to a complicated initializer. Assuming we have a set of reasonable defaults, it seems useful to provide a closure with a pre-built configuration object that can be tweaked as needed.

[kmp]: https://kotlinlang.org/docs/reference/multiplatform.html
[ktor]: https://ktor.io/
