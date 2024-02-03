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

It uses the [Mersenne Twister algorithm](https://en.wikipedia.org/wiki/Mersenne_Twister), which is a pseudorandom number 
generator with a period of 2^19937-1. It is one of the most widely used
PRNGs in the world, and suitable for many applications. But it is not
cryptographically secure and not suitable for cryptographic applications.

There are two reasons for this:
1. It's possible to predict the next number in the sequence given a relatively small
  set of previous numbers (624)
1. It is set to fallback on choosing a seed based on the system time if there
  is no source of system randomness available (/dev/urandom on linux, etc.)

It's also pretty simple to implement, and there is a really detailed pseudocode
implementation in the [Wikipedia article](https://en.wikipedia.org/wiki/Mersenne_Twister#Pseudocode). Here's what it might look like in [Python](https://github.com/heathhenley/CryptoPals/blob/main/set3/21.py#L13):

```python
class MT19937:

  # buncha constants
  f = 1812433253
  w, n, m, r = 32, 624, 397, 31
  a = 0x9908B0DF
  u, d = 11, 0xFFFFFFFF
  s, b = 7, 0x9D2C5680
  t, c = 15, 0xEFC60000
  l = 18

  def __init__(self, seed: int):
    # init 
    self.MT = [0] * self.n  # state

    # masking for 32 bit ints 
    self.lower_mask = (1 << self.r) - 1
    self.upper_mask = (1 << self.r)

    # index
    self.index = self.n + 1

    # init state for the first time
    self.seed_mt(seed)
  
  def seed_mt(self, seed: int):
    # Initialize the generator from a seed
    self.index = self.n
    self.MT[0] = seed
    for i in range(1, self.n):
      self.MT[i] = self.f * (
        (self.MT[i-1] ^ (self.MT[i-1] >> (self.w-2))) + i)
      self.MT[i] &= self.lower_mask

  def extract_number(self):
    if self.index >= self.n:
      if self.index > self.n:
        raise Exception("Generator was never seeded")
      self._twist() 

    y = self.MT[self.index]
    y ^= (( y >> self.u) & self.d)
    y ^= (( y << self.s) & self.b)
    y ^= (( y << self.t) & self.c)
    y ^= ( y >> self.l)
    self.index += 1
    return y & self.lower_mask

  def _twist(self):
    for i in range(self.n):
      x = (self.MT[i] & self.upper_mask) | (
        self.MT[ (i+1) % self.n] & self.lower_mask )
      xA = x >> 1
      if x % 2 != 0:
        xA ^= self.a
      self.MT[i] = self.MT[(i + self.m) % self.n] ^ xA
    self.index = 0
```

## Predicting the Next Number in the Sequence
The internal state of the generator is used to produce the next number in the
sequence. Although the numbers generated won't cycle for an astronomically
large amount of time on current computers (the period is 2^19937-1, a number
with 6000 digits!), reading a sequence of 624 numbers from the generator
allows you to reconstruct the current internal state of the generator and to
predict the next number in the sequence. For example, this is pretty 
[straightforward to do](https://github.com/heathhenley/CryptoPals/blob/main/set3/23.py).

Basically the generator produces a new number by taking the next number in its
624 uint32 internal state and 'tempering' it with a bunch of bitwise operations:

```python
 def temper(y) -> int:
    y ^= (( y >> MT19937.u) & MT19937.d)
    y ^= (( y << MT19937.s) & MT19937.b)
    y ^= (( y << MT19937.t) & MT19937.c)
    y ^= ( y >> MT19937.l)
    return y & ((1 << 32) - 1)
```
Where all the constants are defined in the class. The `temper` function is
reversible, so it's a little tedious but you can undo all the operations and
get back to the original number:

```python
def untemper(y: int) -> int:
  """ Untemper the output of the MT19937 RNG.
  Used the excellent explanation here:
    https://occasionallycogent.com/inverting_the_mersenne_temper/index.html
  to wrap my head around all this bit manipulation.
  """
  smask = (1 << MT19937.s) - 1
  umask = (1 << MT19937.u) - 1
  lower_mask = (1 << MT19937.w) - 1
  y ^= (y >> MT19937.l)
  y ^= ((y << MT19937.t) & MT19937.c)
  y ^= ((y << MT19937.s) & MT19937.b & (smask << MT19937.s))
  y ^= ((y << MT19937.s) & MT19937.b & (smask << (MT19937.s * 2)))
  y ^= ((y << MT19937.s) & MT19937.b & (smask << (MT19937.s * 3)))
  y ^= ((y << MT19937.s) & MT19937.b & (smask << (MT19937.s * 4)))
  y ^= (y >> MT19937.u) & (umask << (MT19937.u * 2))
  y ^= (y >> MT19937.u) & (umask << MT19937.u)
  y ^= (y >> MT19937.u) & umask
  return y & lower_mask
```

I used the
[blog post](https://occasionallycogent.com/inverting_the_mersenne_temper/index.html) linked in the docstring of the `untemper` function to help me wrap my head around all the bit twiddling,
check it out for a more detailed explanation and some cool diagrams showing
what's actually going on at each step.

So basically, after reading 624 values output by the generator, you can `untemper` them to get back to the original internal state of the generator. 
Then you can use that make a clone of the original generator, with the same 
internal state, so it will produce the same  sequence of numbers as the 
original generator.

There is even a Python module available called [RandCrack](https://github.com/tna0y/Python-random-module-cracker) (and I'm sure a host of others) that clone the
state and then offer the same interface as the `random` module.
This alone should makes it obviously not suitable for cryptographic use.

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

Whether or not the Mersenne Twister should be used as the default general
purpose PRNG in so many languages and compilers is called into question in this
[review article](https://arxiv.org/pdf/1910.06437.pdf). The author demonstrates a number of statistical tests which
this family of algorithms fail, and suggests alternatives. To quote from the conclusion:

> The current, dangerous ubiquity of the Mersenne Twister as basic PRNG in many
environments is a historical artifact that we, as a community, should take care of.

Something to keep in mind next time you write you own programming language or
compiler! Or really, when relying on a PRNG for anything important!