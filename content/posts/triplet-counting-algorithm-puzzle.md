---
title: "Triplet Counting Algorithmic Puzzle"
date: 2020-04-15T21:05:24-04:00
draft: false
mathjax: true
tags: ["programming puzzles", "algorithms"]
categories: ["software development"]
keywords: ["python", "python3", "algorithms", "hash table", "hackerrank"]
disqus: "therealheath"
---
We’ve all got a little more time on our hands lately due to social distancing
and COVID-19 (unless you have young children). I’ve been partly entertaining
myself by learning new programming languages and frameworks, and also with some
programming puzzles on sites like [HackerRank](https://www.hackerrank.com/).

I found one problem I ran into recently particularly interesting, and I enjoyed
figuring it out (read: drove me crazy for a bit). This post is a write up of
the problem and the solution that I ended up with.

## Problem Overview
You can read the formal description of the problem
[here](https://www.hackerrank.com/challenges/count-triplets-1/problem), but in
summary: 

*Given an array of integers array, find the number of triplets with indices `i`, `j`,
`k` in the array such that the elements at those positions in the array are in
geometric progression for a given ratio, `r`, and with `i < j < k`.*

For example, given the `[1, 4, 16, 64]`, if `r = 4`, then there are 2
triplets that are in geometric progression, `[1, 4, 16]` and `[4, 16, 64]`.

If r = 2 and we’re given `[1, 2, 2, 4]`, there are two triplets again,
both `[1, 2, 4]` this time (using the first 2 and then the second 2).

The following constraints are given:

- `$ 1 \le n \le 10^{5} $`
- `$ 1 \le r \le 10^{9} $`
- `$ 1 \le arr[i] \le 10^{9} $`

Note `n`, the size of the array, can be quite large here so we could run into some
problems with an inefficient approach.

There are few more examples in the problem description, they are actually the
three sample tests cases that are given to test your code while you figure it
out.

If you’re going to take a crack at this problem on your own, head over there now
and stop reading, because I’m about to start talking about my approach to the
solution.

## My Solution Approach

### Brute force solution
The first attempt I made was to brute force the solution, try all of the triples
in the array that satisfy the constraint `i < j < k` and increment a counter for
each case where there are in geometric progression with respect to a given `r`.

This is always my first approach, even if I expect that the brute force solution
will be tossed away later. For me, it helps me get a solid understanding of the
problem and the constraints. And sometimes, depending on the situation the slow
version really is good enough.

Here is what my version of the brute force solution looks like:

``` python
def countTripletsBruteForce(arr, r):
  count_trips = 0
  for i in range(len(arr)):
    for j in range(i+1, len(arr)):
      for k in range(j+1, len(arr)):
        if arr[j] / arr[i] == r and arr[k] / arr[j] == r:
              count_trips += 1
   return count_trips
```
This has three loops, with the outer loop actually running `N` times, and the
middle and innermost loops running practically `N` times each for large `N`,
which gives a complexity of `~ O(N^3)`. This works fine on the sample test
cases, and it even passes a few of the 15 actual test cases on HackerRank, but
it results in timeout for larger values of `N`. Turns out this is not one of the
cases where the slow version is good enough - and it wasn’t even good enough to
pass half of the test cases.

### Efficient Solution
Of course there is a more efficient algorithm, and I’ll give you a couple hints
in case you’re planning to try to find an efficient algorithm on your own.

I used two hash tables (Python dictionaries in this case)
- It is possible with a singe traversal of the list
- Solve the case of counting pairs in geometric progression, then adapt it for triplets

The third point is the one that I feel really helped me over the hump to finding
the solution. I was totally stumped until I looked at how to count the pairs in
geometric progression in a single pass, so let's look at that first.

### Counting the pairs
So the efficient algorithm for counting pairs using a hash table is to create a
dictionary with each element as the key and the number of times that element
occurs in the list as the value. As we scan the list from left to right, if we
have already found `array[i]/r` in the list, that means that the element `array[i]`
is the second element in one pair for each time we’ve found `array[i]/r`. The
`array[i]/r` check follows because the elements are in geometric progression.

Here is some sample code for counting the pairs that are in geometric
progression (with `i < j` ) using a single traversal of the array and one hash
table.

``` python
def countPairs(arr, r):
  pair_count = 0
  # This is a defaultdict that contains 0 by default,
  # and we use it to count how many times an element
  # shows up in the array.
  element_freq_map = collections.defaultdict(int)
  for i in range(len(arr)):
    if arr[i]/r in element_freq_map:
      # If the arr[i]/r is in our frequency map then arr[i] is 
      # the second value in a pair for all the pairs that start with
      # arr[i]/r - so we increment out counter
      pair_count += element_freq_map[arr[i]/r]
    # Add our current value in the frequency map
    element_freq_map[arr[i]] += 1
  return pair_count
```
So now, using the function above we can count the pairs using a single traversal
of the list and a hash map. I didn't include any error handling, but this
snippet demonstrates the main functionality.

Getting to this point is what made the implementation for triples accessible to
me, so if you haven’t tried it yet, or you took a break, this is a great time to
give it a/another shot.

### Finally, counting the triplets
The final step is to adapt the code above to count triplets instead of pairs. In
place of incrementing a counter each time a pair is found, we’ll store the
number of pairs that end with that value in another dictionary, similar to our
element frequency map. Then we can check whether the current value at `array[i]`
divided by r, array[i]/r, is in:

1. **The pair frequency map** - which means that we’ve already found some pairs that
end with `array[i]/r`, this implies that we now have that many triples that end
with `array[i]`.
2. **The element frequency map** -which means that we’ve found pair(s)
that end with `array[i]` (one for every time we’ve seen `array[i]/r`). So then
we'll save the number of pair(s) in our pair frequency map.

My implementation of that algorithm looks like this:

``` python
def countTriplets(arr, r):
  count_trips = 0
  element_freq_map = collections.defaultdict(int)
  pair_freq_map = collections.defaultdict(int)
  for i in range(len(arr)):
    if arr[i]/r in pair_freq_map: 
      # If arr[i]/r is in the pair frequency map,
      # then it means there is a triple ending with
      # this element, so increment the counter by
      # the number of pairs that end with arr[i]/r
      count_trips += pair_freq_map[arr[i]/r]
    if arr[i]/r in element_freq_map:
      # If arr[i]/r is in the element freq map,
      # then it means there is/are pair(s) ending with
      # this element, arr[i]. Store and increment the
      # pair frequency map
      pair_freq_map[arr[i]] += element_freq_map[arr[i]/r]
    element_freq_map[arr[i]] += 1
  return count_trips
```
This implementation is much more efficient in run time for large values of `N`,
and now all of the test cases pass without timing out!

It took me a while to sort this one out, but I think it’s an interesting
problem and I enjoyed working out the solution. I would be interested to hear
any comments you have or if you have come up with a different solution. The
[Editorial](https://www.hackerrank.com/challenges/count-triplets-1/editorial)
for this problem presents a slightly different solution than this one
also using two hash tables, but looping over the list twice.

All code samples are also available in this [GitHub
Gist](https://gist.github.com/heathhenley/4f08486219ec6f2d6e3142f0ae289911).
