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

...which is just fine, obviously, but that first one feels so much cleaner. And, not having to initialize `config` feels like a win.

My next thought was: Can we do this in Swift?

Yes, We Can Do Something Like This In Swift
-------------------------------------------

So, how do we mimic this behavior in Swift?

First, let's create the `config` construct:

{% highlight swift %}
struct HttpClientConfig {
    var expectSuccess: Bool = true
}
{% endhighlight %}

Nothing fancy here: Just a struct with a single property. Note that the property is mutable: This is required, since we're planning on allowing changes to this property after instantiation.

Next, we need an object to configure:

{% highlight swift %}
final class HttpClient {
    let config: HttpClientConfig
}
{% endhighlight %}

Again, not much going on here. This is a class with a single property to hold an instance of `config`.

All that's left is to create the initializer:

{% highlight swift %}
init(_ configurationClosure: (inout HttpClientConfig) -> Void) {
    var config = HttpClientConfig()
    configurationClosure(&config)
    self.config = config
}
{% endhighlight %}

Let's go through this, line by line. Unfortunately, the first line is the most complicated, but the others are much simpler, so don't get discouraged!

1. `init(_ configurationClosure: (inout HttpClientConfig) -> Void)`  
    This is the method signature for the initializer. It accepts a single parameter.
      - The `_` as the parameter name indicates that it can be called without a parameter name, like `init(value)`.
      - `configurationClosure` is the parameter name that can be used inside the method.
      - The parameter data type is a closure. The closure accepts a single parameter, of the type `HttpClientConfig`.
      - Note that the closure's parameter is marked as `inout`. This means that changes to it within the closure will be passed back out to the method. Which is key! We definitely want the person who initializes `HttpClient` to be able to modify `config`.  


2. `var config = HttpClientConfig()`  
    Here is where we instantiate `config`. Note that we use `var` here, since we want to allow changes.  

3. `configurationClosure(&config)`  
    This line calls the closure that was passed in to the initializer. Note that we pass the newly instantiated `config` into the closure. Here is where the code that the person who instantiated `HttpClient` will run: Or, in other words, here is where any tweaks to `config` are applied.  

4. `self.config = config`  
    Finally, we stash `config` into our private property, for future reference.  

But Does It Work?
-----------------

Here is some test code, to see if it actually works:

{% highlight swift %}
let client = HttpClient {
    config in
    config.expectSuccess = false
}
print(client.config.expectSuccess) // will return `false`
{% endhighlight %}

Line by line:

1. First, we instantiate `HttpClient` using our fancy new initializer.
2. Line 2 is where we name the incoming parameter. Remember how we create a `config` object in the `init` for `HttpClient`? Well, here it is.
3. Let's tweak the config, by setting `expectSuccess` to `false`. Note that its default value is `true`.
4. Outside the closure, let's examine the value of `expectSuccess` in the client's `config`. (Spoiler alert: It will be `false`).

Since the property, which has a default value of `true`, is now `false`, we know that our initializer worked as expected!

The Solution Code
-----------------

Here is the complete solution:

{% highlight swift %}
struct HttpClientConfig {
    var expectSuccess: Bool = true
}

final class HttpClient {
    let config: HttpClientConfig

    init(_ configurationClosure: (inout HttpClientConfig) -> Void) {
        var config = HttpClientConfig()
        configurationClosure(&config)
        self.config = config
    }
}

let client = HttpClient {
    config in
    config.expectSuccess = false
}
print(client.config.expectSuccess) // will return `false`
{% endhighlight %}

Conclusion
----------
I think this could be a useful solution to a complicated initializer. Assuming we have a set of reasonable defaults, it seems useful to provide a closure with a pre-built configuration object that can be tweaked as needed.

[kmp]: https://kotlinlang.org/docs/reference/multiplatform.html
[ktor]: https://ktor.io/
