---
title: "Advent of Code 2024 Notes"
date: 2024-12-25T10:50:52-05:00
draft: false
description: "Notes from the 2024 Advent of Code"
tags: ["python", "ocaml", "advent of code", "notes"]
categories: ["python", "ocaml", "advent of code", "notes"]
keywords: ["python", "ocaml", "advent of code", "notes"]
---

**TL;DR** - Misc notes from the 2024 Advent of Code in Python and Ocaml and a little bit of Golang.

The repo with my solutions to the [Advent of Code 2024](https://adventofcode.com/2024) problems can be found [here](https://github.com/heathhenley/AOC). Here's a bunch of random notes from
this year

## Languages
Learned a bunch of Ocaml (even though I didn’t use it everyday as intended)
- Working out to do things without using mutable state
- Doing things that would be loops with recursion / thinking about things as recursive vs iterative
- Data structures in the std lib
	- Sets are not constant time look up (use Hashtbl)
	- Lists seem really popular but arrays exist and are nice
- More comfortable using Utop here and there instead running the whole script all the time
- Scanf with individual parser functions set up for tricky input files - [example](https://github.com/heathhenley/AOC/blob/main/2024/day13/day13.ml#L5)
- Match statements everywhere - I even started using them in Python 😂
	- I understand it doesn't have the same benefits, but it's a style thing that's starting to rub off on me
- Wish that you didn’t have to debug print using format - but I get it
- Also wish you could keep unreferenced functions around while debugging - but I get it 

I plan on completing the ones that I have not yet done in Ocaml here and there as the year goes on. I did a few in Go too but it’s honestly not as interesting given I already know C++ and Python. I decided not continue with it because it wasn’t as fun / challenging, it was mostly just looking up "how do I do x in go" and doing it. Where learning OCaml was a big switch because I don't have much FP experience. However, that's a big RIP 🪦 to the original goal of an animal mascot themed AOC (python, gopher, camel).

### Favorite Day(s)
Finding the Christmas tree in the robot grid on day 14. Day 24 pt 2 was cool also - I still haven’t solved programmatically (I worked out semi-manually) - it's a bunch of chained [full adders](https://www.geeksforgeeks.org/full-adder-in-digital-logic/) to add two numbers bit by bit, but a couple of the wires have been swapped. Box pushing on day 15 was a close second - I was really happy with coming up with a “nice” recursive solution without too much trouble (still was tricky to debug when working on samples but not real input). I also enjoyed the reverse engineering one on day 17, it felt a little like a CTF or something. 

### Hardest Day(s)
Definitely day 21 by far - I didn’t really get the second part of this one on my own. Spent hours on it but could not figure out how to formulate in a way that would be cacheable. Got it with help though and next time I’ll be in better shape for this type of problem. It really helped to look at it as going depth first vs breadth first. It was actually similar to the much simpler stones problem on an earlier day. Day 12 part 2 was also pretty difficult to get right, counting the number of sides on a region in a grid (ended up with basically counting corners).

### General learnings / re-learnings
Some topics / notes in general:
- Remembered how to traverse 2d grids 😂
	- Definitely need to put together a grid module for next year
- Bunch of DFS/BFS and [Dijkstra](https://en.wikipedia.org/wiki/Dijkstra's_algorithm) problems
	- Same with these - used them all a lot - wrote them again each time
	- Didn't really need [A*](https://en.wikipedia.org/wiki/A*_search_algorithm) for anything, but I tried it on a couple of the problems to revisit how it works. I didn't keep it but glad that I revisited
- Had not seen the reconstruction of best paths from parent nodes in Dijkstra in a long while, took a bit to debug but I got it working
- [Bron-Kerbosch](https://en.wikipedia.org/wiki/Bron%E2%80%93Kerbosch_algorithm) for finding cliques in graphs (fully connected sub groups)
- Reminded (in great detail) of how [binary addition](https://www.geeksforgeeks.org/full-adder-in-digital-logic/) works
- There were no 3D vector problems, cycles or card games this year. I spent way too long trying to find a cycle in the stones problem
- Practice formulating recursive in different ways
  - [add or multiply problem](https://github.com/heathhenley/AOC/tree/main/2024/day7) - I did this bottom up and it worked but was slow but intuitive to figure out. I then found a bunch of examples of how to do it top down and that trims so much off the search space that it's much faster
  - [stones](https://github.com/heathhenley/AOC/blob/main/2024/day11/) - I also did this one kind of bottom up with a loop and hashtable (original solution is in there still) - but it turns out to be much cleaner recursively with memoization. I think they're kind of the same but the second is way more elegant in my opinion
  - [towels](https://github.com/heathhenley/AOC/tree/main/2024/day19) and [box pushing](https://github.com/heathhenley/AOC/tree/main/2024/day15)
- I joined a group on Discord for AOC from Twitter - it was cool to see everyone's solutions, share different approaches and get hints / vent when needed - definitely a better experience than doing it alone

In general I think this year felt easier than last year. I would love to think I just got better at it or more used to the style of the problems but I’m not THAT naïve. Both could be true I guess...let's go with that!

Grateful for [Eric Wastl](https://was.tl/) for coming up with these problems and running the show every year with a lot of effort for little return.