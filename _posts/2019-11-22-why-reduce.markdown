---
layout: post
title:  "Why Reduce?"
date:   2019-11-22 11:59:28 -0500
categories: swift
tags: swift programming
---
*This post solves a real-life problem in two ways: using a for...in loop, and using `reduce`.*

An alternative to iterating through a collection with a for...in loop is to use a higher-order function. Three common ones are `map`, `filter`, and `reduce`. These all take a function as a parameter, which is a characteristic of a higher-order function.

There isn't anything wrong with for...in loops, and higher-order functions are not "better". But they are out there, and even if you decide you never want to use them yourself, you may be faced with debugging or enhancing someone else's `reduce`-filled code. So it's good to at least have a rough idea of how these functions work.

If you're like me, though, you'll find yourself reaching for these functions often. They offer much power; and they sidestep some of the possible pitfalls of looping, such as off-by-one errors.

They also lend themselves to the use of immutable variables. As we'll see in the solutions below, the for...in loop relies on modifying a mutable variable. The `reduce` solution has no mutable variables, which makes the logic in the code less complex, and therefore easier to understand.

My First Reduction
------------------
After a lot of practice, I am comfortable with using `map` and `filter` on a collection. But only recently did I grasp the use of `reduce`.

There are many [excellent articles][useyourloaf-map] about these three functions, but I had trouble understanding `reduce` enough to know how to make use of it.

`reduce` is different from `map` and `filter`, which both return collections. `reduce` aggregates the collection's elements, yielding a single value. It also has the trickiest syntax of the three.

I'll share the problem I faced, and solve it two ways: using a good old for...in loop, and then using `reduce`.

The Problem
-----------
I have a map view, `mapView`, and on it I have several polygons, in the array `polygons`. I want to tell the map to adjust its view so that all the polygons are visible.

![How to find the big rectangle?](/assets/2019-11-22-why-reduce-polygons.png "How to find the big rectangle?")

The map view has a method named `setVisibleMapRect`, that accepts a map rectangle object.

Each polygon has a property named `boundingMapRect`, which will yield that polygon's map rectangle. Furthermore, a map rectangle has a `union` method, that accepts a second map rectangle as a parameter, and will return the larger rectangle that bounds the two.

So, how do I determine the rectangle that encompasses all the polygons?

The for...in Solution
---------------------
Given `mapView` and the array `polygons`, I declare a mutable variable, `bigRectangle`, with an initial value of an empty map rectangle.

Then, for each polygon, I obtain its map rectangle via the `boundingMapRect` property, and update `bigRectangle`. If `bigRectangle` is empty, I set it to the polygon's rectangle. Otherwise, I use the rectangle's `union` method to yield the rectangle that encompasses the rectangle and `bigRectangle`'s current value.

{% highlight swift %}
var bigRectangle = MKMapRect()
for polygon in polygons {
  let rectangle = polygon.boundingMapRect
  bigRectangle = bigRectangle.isEmpty ? rectangle : rectangle.union(bigRectangle)
}
mapView.setVisibleMapRect(bigRectangle, animated: true)
{% endhighlight %}

Outside the loop, we have a mutable variable that starts life as an empty rectangle. This `bigRectangle` will be updated from inside the loop, to be used as a running aggregation of the rectangles.

Inside the loop, each polygon is converted to a map rectangle. If `bigRectangle` is empty (i.e. if this is the first iteration), `bigRectangle` is set to the current map rectangle. If it is not empty (i.e. on any subsequent iteration), it is set to the union of it and the current map rectangle.

By the end of the loop, `bigRectangle` will be the rectangle that encompasses all the polygons.

Problem solved! But what if there's another way? (Hint: There is.)

The `reduce` Solution
---------------------
Let's solve the same problem with `reduce`. As stated above, `reduce` will iterate through a collection, and return an aggregate of the elements. Which sounds perfect for the task at hand.
Given `mapView` and the array `polygons`, I declare a variable, `bigRectangle`, and set its value to the result of a `reduce` operation on the array.

`reduce` takes a single parameter, which is the initial value of the aggregation. In this case, the initial value is an empty map rectangle.

Its closure takes two parameters: the running result of the aggregation, and the current element. So, inside the closure, I can return either the rectangle itself, or a union of it and the running result.

{% highlight swift %}
let bigRectangle = polygons.reduce(MKMapRect()) {
  aggregatedRectangle, polygon in
  let rectangle = polygon.boundingMapRect
  return aggregatedRectangle.isEmpty ? rectangle : rectangle.union(aggregatedRectangle)
}
mapView.setVisibleMapRect(bigRectangle, animated: true)
{% endhighlight %}

Let's dig into this code and see what's going on.

The first line declares `bigRectangle`, and sets it to the result of a `reduce` operation on `polygons`.

Notice that `reduce` requires a special parameter, which is used as the starting value for the aggregation. In this case, the starting value is an empty map rectangle. This is exactly analogous to the starting value of the mutable `bigRectangle` variable in the for...in solution above.

The `reduce` closure has two parameters. The first parameter is the running aggregated result (`aggregatedRectangle`), and the second parameter is the current element (`polygon`).

Note that the closure's parameters have different types. The first parameter is of the same type as `reduce`'s starting value parameter: in this case, `MKMapRect`. The second parameter is of the same type as the array's elements: in this case, `MKPolygon`.

So, on the first iteration, `aggregatedRectangle` will be an empty map rectangle, and `polygon` will be the first polygon in the array. On the second iteration, `aggregatedRectangle` will be the first map rectangle, and `polygon` will be the second polygon. On the third iteration, `aggregatedRectangle` will be the union of the first map rectangle and the second map rectangle, and `polygon` will be the third polygon. ...and so on, until we have a rectangle that is the union of all the rectangles.

What's the Difference?
----------------------
Here are the two solutions again:

{% highlight swift %}
// The for...in solution
var bigRectangle = MKMapRect()
for polygon in polygons {
  let rectangle = polygon.boundingMapRect
  bigRectangle = bigRectangle.isEmpty ? rectangle : rectangle.union(bigRectangle)
}
mapView.setVisibleMapRect(bigRectangle, animated: true)
{% endhighlight %}

{% highlight swift %}
// The reduce solution
let bigRectangle = polygons.reduce(MKMapRect()) {
  aggregatedRectangle, polygon in
  let rectangle = polygon.boundingMapRect
  return aggregatedRectangle.isEmpty ? rectangle : rectangle.union(aggregatedRectangle)
}
mapView.setVisibleMapRect(bigRectangle, animated: true)
{% endhighlight %}

There's not much to compare them in terms of length. We can make the `reduce` solution shorter, if we want, by using shorthand:

{% highlight swift %}
// A shorter version of the reduce solution
let bigRectangle = polygons.reduce(MKMapRect()) { $0.isEmpty ? $1.boundingMapRect : $1.boundingMapRect.union($0) }
mapView.setVisibleMapRect(bigRectangle, animated: true)
{% endhighlight %}

This uses $0 and $1 in place of the named parameters, and works well for closures that contain only a single line of code.

In terms of clarity, the for...in code feels more readable, but remember how subjective "readability" can be. I think a lot of that feeling about the for...in code being readable comes from being so familiar with that kind of code. Given familiarity with higher-order functions and Swift's closure syntax, the `reduce` solution is quite readable as well.

One point in `reduce`'s favor is the lack of any mutable variables. The for...in solution uses a mutable variable, which is incremented from inside the loop. The `reduce` solution has no mutable variables, since the aggregation is handled by the `reduce` function itself. The fewer mutable variables in a given scope, the easier it is to comprehend the logic. So that's a big win.

Conclusion
----------
Going forward with our projects, there will be some cases where a for...in loop is the best approach, and there will be others that might be best solved by a higher-order function.

To loop, or to reduce? It's up to you to decide.

But the next time you're reaching for a loop: Pause, and consider a higher-order function instead. Like me, you may find yourself using them more and more.

[useyourloaf-map]: https://useyourloaf.com/blog/swift-guide-to-map-filter-reduce/
