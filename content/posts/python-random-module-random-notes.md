---
title: "Random Notes About Python's Random Module"
date: 2023-12-26T12:10:44-05:00
draft: false
description: "Some notes on Python's random module, and why you should use the
`secrets` module or `os.urandom` for cryptographic applications instead of
`random`."
tags: ["python", "notes", "software", "software-engineering"]
categories: ["python", "software", "software-engineering", "notes"]
keywords: ["random", "os.urandom", "secrets", "python", "software", "software-engineering"]
---

**TL;DR:** Use the functions in the `random` module for modeling, simulations,
games, sampling, etc. but use `os.urandom`, `secrets`, or 
`random.SystemRandom` for cryptographic applications. I know very little about 
cryptography and security, these are just my notes about stuff I recently 
learned.

# Random Notes About Python's Random Module

It uses the [Mersenne Twister algorithm](https://en.wikipedia.org/wiki/Mersenne_Twister), which is a pseudorandom number 
generator with a period of 2^19937-1. It is one of the most widely used
PRNGs in the world, and suitable for many applications. But it is not
cryptographically secure and not suitable for cryptographic applications.

There are two reasons for this:
1. It's possible to predict the next number in the sequence given a relatively small
  set of previous numbers (624)
1. It is set to fallback on choosing a seed based on the system time if there
  is no source of system randomness available (/dev/urandom on linux, etc.)

## Predicting the Next Number in the Sequence
The internal state of the generator is used to produce the next number in the
sequence. Although the numbers generated won't cycle for an astronomically
large amount of time on current computers (the period is 2^19937-1, a number
with 6000 digits!), reading a sequence of 624 numbers from the generator
allows you to reconstruct the current internal state of the generator and to
predict the next number in the sequence. For example, there is a Python module
available called [RandCrack](https://github.com/tna0y/Python-random-module-cracker) (and I'm sure a host of others) that demonstrate
this. After reading 624 values output by the generator, it basically becomes a
clone of it, with the same internal state, so it will produce the same sequence
of numbers as the original generator.

This alone should make it clear that it's not suitable for cryptographic use.

## Fallback to System Time
The second reason is that if there is no source of system randomness available,
the [seed is initialized using](https://github.com/python/cpython/blob/main/Modules/_randommodule.c#L263):
- the current system time
- the process ID
- the current system monotonic time

as the seed. Those things have a lot less entropy than use a true random source
to chose the initial state. For example, if a system were not using
`/dev/urandom` or similar, you can make some reasonable assumptions about the 
system time, pid, and monotonic time that might have been used as the seed and 
use that brute-force the possible seed choices. You would have a reasonable chance of ending up with your generator in the same state as the target.

There's still a lot of possibilities, and you won't know the exact state of the
generator unless you know how many times it's been called since it was seeded,
but it's still a lot less entropy than reading the whole state in from
/dev/urandom.

## What can you use for a cryptographic applications?
Python has a [module called `secrets`](https://docs.python.org/3/library/secrets.html) that provides a cryptographically secure
source of randomness, and that's the one that should be used to generate random
bytes or bits for tokens, etc. There is also the lower level [`os.urandom`](https://docs.python.org/3/library/os.html#os.getrandom), which is a
direct wrapper around the system's source of randomness (e.g. /dev/urandom on 
linux) - it reads information from 'device drivers and other sources of
environmental noise' and uses that instead.

There is also the `random.SystemRandom` class in the `random` module. It
provides the same interface as `random` (all the functions are really just 
calling methods a hidden instance of `random.Random`). It implements the 
same methods as are available as functions in `random`, but instead of using
the Mersenne Twister algorithm PRNG under the hood, it calls `os.urandom` to 
generate random numbers using the system's source of randomness. See the docs
on that here: https://docs.python.org/3/library/random.html#random.SystemRandom


## Conclusion
The Python `random` module is great for generating random numbers for
modeling, simulations, games, etc. but it's not suitable for cryptographic
use. Its state can be obtained by observing a relatively small number
of consecutive numbers from the generator, or in some cases by brute forcing
the reduced space of possible seeds if the system is not using seed based on
'non-deterministic sources of randomness' from the OS. So, for cryptographic 
applications, use the `secrets` module or `os.urandom` instead.

Why not always use `secrets` or `os.urandom`? They are slower than the `random`
module, and they are, of course not deterministic. If you want to be able to 
reproduce the same sequence of random numbers - for example in a simulation, 
game, or for testing, you cannot do that with `os.urandom` (the higher level
`secrets` and `random.SystemRandom`).