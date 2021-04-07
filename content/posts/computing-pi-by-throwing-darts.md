---
title: "Computing Pi by Throwing Darts"
date: 2021-03-14T21:17:10-04:00
draft: true
tags: ["python3", "python", "random sampling", "pi day"]
keywords: ["python3", "python", "random sampling"]
disqus: therealheath
mathjax: true
---
In celebration of pi-day, let's look at a method of computing pi using
random numbers that is often presented in probability, statistics or other classes, as
an elementary example of using random sampling and / or simulation. 

In my case, the first time I remember seeing / hearing about this example was in
a probability class, however we didn't actually write any code to try it, we
just looked at the idea. Later, I was re-introduced to the example when I began to do
undergrad ChE research - I was going to be working on a Monte Carlo molecular
simulation code for small molecules. When I started, I had no idea what that
meant - and one of the first exercises that my adviser gave me, to the idea of
computing some physical quantity using simulation, was by chance, this "estimate
pi by throwing darts" example.

## Estimating Pi by throwing darts

First, this is definitely not an efficient way to approximate pi, but it's a fun
exercise. :) 

Let's assume we have a weird dart board, it's a circle inscribed in a square,
where the height of the square is equal to the diameter of the circle, so it
looks like this:

![square with circle inscribed]()

We know what the area of the square is `$A_s = H^2$` (where `$H$` is the height
and width of the square). As the circle is inscribed in the square, the
diameter, `$D$`, is `$D = H = 2 * radius$`. 

The area of the circle is of course `$ A_c = \pi r^2 $`.

We can assume that if we randomly throw darts at this "dart board" that the
probability the dart hits within the circle is related to the ratio of the area
of the circle to the square. So that,

$$ \frac{P(C)}{P(S)} = \frac{A\_c}{A\_s} = \frac{\pi r^2}{(H^2)} = \frac{\pi r^2}{4r^2} $$

If we rearrange for $\pi$, we can see that: 

$$ \pi = 4 * \frac{P(C)}{P(S)} $$

We can use this result as the basis of our simulation.

Before we start writing any actual code, let's assume:
 - We are so good that we never miss the board completely (or if we do, we don't
   count that throw)
 - Yet we are still bad enough, that hitting any where on the board is equally
   likely. That is to say, despite any attempt aiming at a specific point, we're
   no more likely to hit that point than any other spot on the board.
