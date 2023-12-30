---
title: "Notes From Cryptopals Challenges"
date: 2023-12-29T12:58:01-05:00
draft: false
description: "Summary and notes about learnings from the Cryptopals Crypto 
Challenges. I'm starting with knowing nothing about cryptography, except that
you can see Tux the penguin if you encrypt him with ECB mode lol. Going to
hopefully push updates as I have time to work through the challenges."
tags: ["cryptography", "notes", "software", "software-engineering"]
categories: ["python", "software", "software-engineering", "notes"]
keywords: ["python", "cryptography", "software", "software-engineering" ]
---

**TL;DR:** I'm working through the
[Cryptopals Crypto Challenges](https://cryptopals.com/), starting with knowing
nothing about cryptography. These are my (not so) random notes and takeaways 
from them.

# Notes From Cryptopals Challenges
## Set 1 - Basics
This was pretty quick and easy set to go through, but I'm glad I didn't skip it.
It introduced 'repeating key XOR' and how to break some cyphers 'statistically' 
using letter frequency.
The most interesting part of the set was
[Breaking repeating key XOR](https://cryptopals.com/sets/1/challenges/6) and
[solution](https://github.com/heathhenley/CryptoPals/blob/main/set1/6.py)

### What I learned
Well, I didn't know anything about cryptography before starting this set, it was
fun to learn about and decrypt these basic cyphers, even if they aren't relevant
(directly) to real word crypto systems.

## Set 2 - Block Crypto
This set is still basic, but starting to introduce vulnerabilities that are more
relevant to real world crypto.

Most interesting part so far was learning about byte-at-a-time ECB decryption,
challenges 12 and 14.

### Byte-at-a-time ECB decryption
- Byte-at-a-time ECB decryption (Simple) [solution](https://github.com/heathhenley/CryptoPals/blob/main/set2/12.py)
- Byte-at-a-time ECB decryption (Harder) [solution](https://github.com/heathhenley/CryptoPals/blob/main/set2/14.py)

In both of these challenges, you have a function that encrypts a user provided
string along with a secret string. The algorithm is more or less:
- Determine that ECB is being used
- Determine the block size
- Send a string of bytes that is one byte short of a full block
  - This contains the first byte of the secret string, with the rest being
    whatever you provide (I used all `A`s)
  - Then just try all possible bytes for that last byte, and match it up
    with the output of the real one. The match is the first byte of the secret
    string!
- Repeat the above, but with a string that is two bytes short of a full block
  - This contains the first two bytes of the secret string, with the rest being
    whatever you provide (I used all `A`s) - but you know the first byte from
    the previous step, again - generate all possible bytes for the last byte
    and match it up with the output of the real one. The match is the second
    byte of the secret string!
- Repeat until you have the entire secret string!

For the case with the random prefix, there's a little more work to do -
basically finding the padding length first and adjusting above to account for
it being there.

### CBC bitflipping attack
Still working on this set as I update this post, about to attempt CBC 
bitflipping!

### What I learned
- The differences between AES [ECB mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Electronic_codebook_(ECB)) and
[CBC mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_block_chaining_(CBC))
- [Padding oracle attack](https://en.wikipedia.org/wiki/Padding_oracle_attack) on CBC mode (haven't implemented this yet, but I started to think about it)
- I have always heard: "don't use ECB" and I have seen the famous picture of [Tux](https://upload.wikimedia.org/wikipedia/commons/f/f0/Tux_ecb.jpg)
 encrypted with ECB, but
  never actually really understood why it happened or implemented a crack like
  this, so it was pretty cool to see it in action.
