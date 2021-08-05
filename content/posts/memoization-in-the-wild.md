---
title: "Memoization in the Wild"
date: 2021-08-02T17:32:43-04:00
draft: false
tags: ["algorithms", "computer science", "python3", "python", "optimization"]
keywords: ["memoization", "caching", "optimization", "algorithms"]
---

## Overview
Memoization or memoisation is a method used to optimize programs. Usually, at
least in my experience, it’s one of the first topics introduced when dynamic
programming algorithms are being discussed. With a quick google search you can
find the [Wiki](https://en.wikipedia.org/wiki/Memoization) or a trillion other
blogs about it - most will show the canonical example - the “hello world” of the
topic - that is, using memoization to optimize a recursive implementation of a
function that generates the n-th Fibonacci number (or sometimes a function
computing factorials).

In brief, the idea is that there are some cases where the same function is going
to be called many times, and some of those inputs will be repeated. So the trick
is to cache the results of the function calls for each input - and then on each
call of the function, check whether the result is already cached. This allows
the function to only be called with a given set of input parameters once. 

As mentioned, the internet is full of detailed examples of this using the
recursive Fibonacci function. In Python3, that looks like going from:
``` python
def fib(n):
  if n < 2:
    return 1
  return fib(n-1) + fib(n-2)
```

To something like:
``` python
cache = {}
def fib(n):
  if n in cache:
    return cache[n]
  if n < 2:
    cache[n] = 1
  else:
    cache[n] = fib(n-1) + fib(n-2)
  return cache[n] 
```

Of course for a recursive function like this, the speedup is great - even for
relatively small input like n=20, the "memoized" version runs orders of magnitude
faster (about 2000x faster for n=20). More generally, if you’re a Python
developer, you can use Python standard library tools to do this, for example you
can use [functools.cache](
https://docs.python.org/3/library/functools.html#functools.cache) introduced in
3.9 (for older versions, take a look at [func.tools.lru_cache](
https://docs.python.org/3/library/functools.html#functools.lru_cache) instead).

## Application to Drawing Module
That’s the toy example, however the inspiration for this post was that I finally
had the opportunity to apply this optimization technique to a real world problem
in a very concrete and self contained way at [FarSounder](www.farsounder.com)!

In this case, the speed up was a bit more modest, and the function was not a
recursive one, but the idea was the same.

In one of the viewers of FarSounder’s main data display, sonar data is overlaid
with the vessel outline and some other things in real time. This data includes
"in-water detections", things like pilings, containers, rocks, etc, things in
the water that produce a loud return and typically correspond to objects that a
user doesn’t want to hit. The seafloor depths ahead of the vessel are also
displayed as a surface, typically with the color mapped to depth. The location
of each point on the bottom surface is known, but in order to draw it, its real
world location must be converted into screen coordinates so that it can be
placed on the screen in the correct location.

When the display code was profiled for this viewer, it showed that this
transformation was taking an unexpectedly large chunk of the processing time in
cases with a large bottom surface. Digging into this further, it was clear that
there was an opportunity here to use "memoization", or simply cache the results
of the transformation for a given input, so that if the screen coordinates for
any vertex had already been computed they would not be recomputed (provided the
chart has not moved / scaled). 

The real case was in C++, but here is some pseudocode for this approach:

#### Result
Transform geographic coordinates (latitude, longitude) into a position on the screen.

#### Initialization
Set up a cache for function results. A hash table, or map (or python dictionary)
should be used so that value lookups can be done in constant time.

#### **Algorithm:**

```
for each geopoint in points to transform:
  if geopoint is in cache:
    return cached screen point result for this geopoint
  else:
    compute screen point for this geopoint
    update cache with screen point
    finally, return the screen point 
```

This approach was applied, and in the "worst" cases, the ones
with a large bottom surface being displayed, the processing time was reduced by
about 30%!

Of course the cost of this method is that now more memory is used (to store
the cache), but in this case, it was not a problem. In general, the speed up was
modest - in cases with little bottom to display, other steps in the processing
chain are certainly the bottleneck. That being said, a 30% reduction in
processing time for in the slowest cases was a big win as this is code that is
run many times in normal program execution, basically any time new sonar data is
pushed or the chart is moved or scaled.

I think this is a neat example and straightforward application of a concept
typically introduced early in the Computer Science Algorithms curriculum. This
was pretty easy to implement for a 30% reduction in the processing time of our
worst cases, so it was totally worth it. I recommend keeping your eyes peeled
for places where you might be able to apply this simple concept (but profile
first of course)!
