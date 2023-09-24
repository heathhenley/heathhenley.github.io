---
title: "Shut the Box"
date: 2021-11-01T21:11:13-04:00
draft: false
tags: ["python3", "python"]
keywords: ["python3", "python", "algorithms"]
description: "Playing a silly 'Shut the box' table top game using Monte Carlo Simulation to test different strategies, so that I hopefully stop losing to my friends so much."
---
**TL;DR** - choose either the option containing the largest number, or the fewest
tiles and you'll be ok!

Read on to learn more...

## What am I talking about?
Over the summer at a friends house, I was presented with an “old bar game” that
I was completely unfamiliar with. It’s a wooden tray, with the numbers 1-12
printed in ascending order on little wooden tiles.

Here’s an example of what it looks like:

![Shut the box game board](/shut_the_box/shut_the_box.jpg)

The game we actually used was similar but had four sides, so that four players
could play at the same time. The goal in this case was to roll two six sided
dice, and put down number of wooden tiles that add to the sum of the die. The
person with the lowest sum of remaining, unflipped tiles wins.

So for example, on your first turn, all of the tiles, 1-12, are up. Let’s say
you roll a 10, you can flip your 1 and 9 tile, or 2 and 8, or your 5, 3, and 2
tile, etc. Then you will roll again, flip tiles with the same sum as the sum
you’ve rolled on the dice, and repeat. You continue until you cannot flip down
tiles equal to the sum you have rolled.

# Monte Carlo Simulation of Shut the Box Game

Of course I got to thinking about what the optimum strategy actually is for this
game, and workshopped a couple ideas against my friends. It seemed like I was
doing pretty well with a strategy of always taking the option containing the
highest number, but I wasn’t really sure that was the best strategy. I thought
about it a lot on the drive home. Finally, I decided it would be fun to write a
short python script to simulate playing shut the box using different strategies,
in order to put my strategy to the test. 

You can find a Gist with the Python 3 script for this [here](https://gist.github.com/heathhenley/4ac69dad35a009d0685c785ecea270e1).

It’s a bit "rough" - but the idea is that it’s a script to "roll some
dice" randomly and select the shut the box moves based on a predefined, general
strategy. I was excited to test my hypothosized best strategy of "always taking
the option with the largest number". But of course, you can see that I also
included a completely "random" strategy, the strategy of taking down the most
tiles possible, and finally strategy of taking the fewest possible tiles. The
script just plays a given number of games (100k, decided arbitrarily to be
long enough to give consistent results between runs) according to each defined
strategy and tracks the average score, so that the strategies can be compared. 

## Results

Running each of these strategies 100k times and averaging the results gives
average scores of:

| **Strategy**  | **Average Score (100k runs)** |
| ------------- | ------------- |
| Random | 49.7 |
| As many tiles as possible  | 53.8  |
| As few tiles as possible  | 35.5  |
| Option containing largest number  | 35.4 |


Based on this simple implementation, you’re definitely best off adopting either
the strategy of taking of "always taking the option containing the largest
number" or the strategy of "taking the fewest tiles possible". You definitely
want to have a plan, we can see that  randomly choosing the tiles to put down
doesn’t do so well in the long run. Just don’t plan to take down as many tiles
as possible - that won’t pan out for you! :)

## Some Details and Additional Ideas

A couple notes about the implementation of these strategies - the two best
strategies perform statistically the same with the number of trials I’ve run.

Another note, is that when there was ambiguity, after the first step in the
strategy, I implemented a random choice. So for example, if two valid options
allow 2 tiles to be dropped, and we’re playing by the "take as few tiles as
possible" strategy, I choose between those randomly. I applied the same approach
to the "option with the largest number strategy", when two or more options had
the same max number - the option played was chosen randomly. I think this is a
pretty reasonable approach for a casual player - but perhaps a composite
strategy would preform better?

So for example, we could use the max strategy to result conflicts in the "take
as few as possible" strategy. Or we could use the the "take as few as possible"
strategy to resolve some conflicts in the max strategy. I did try both of these
approaches, and I was a bit surprised to see that the did not reduce the score
in significant way. Perhaps I'll dig into this more in a follow up...

## Wrapping Up

Anyway - of course it's possible to calculate the real optimal strategy is for
crushing your friends at Shut the Box using probability, and perhaps that would
be a another good subject for a follow up post...

At least now if you ever come across this game, you’ll have a strategy to start
with!
