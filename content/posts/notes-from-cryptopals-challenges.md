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
Basically, you can manipulate the ciphertext to change what the resulting
plaintext is when it's decrypted when using CBC mode. So for example, in
this challenge the attack was to change the plaintext so that it would
decrypt to contain `;admin=true;`. 

In CBC mode, when decrypting a block of ciphertext it is first decrypted as
in ECB mode, then XOR'd with the previous block of ciphertext. The second part
is what was used in this attack. If we know then plaintext - eg we provide it,
then we can manipulate the block immediately before it so that when it's XOR'd
in decryption, it will result in the plaintext we want.

My solution to this one is [here](https://github.com/heathhenley/CryptoPals/blob/main/set2/16.py). I always seem to have trouble with the
bit manipulation stuff so this took a bit to get working right, but the idea
is pretty clear now.

### What I learned
- The differences between AES [ECB mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Electronic_codebook_(ECB)) and
[CBC mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_block_chaining_(CBC))
- CBC Bitflipping attack -  this SO answer helped me a bit: https://crypto.stackexchange.com/a/66086/113345
- I have always heard: "don't use ECB" and I have seen the famous picture of [Tux](https://upload.wikimedia.org/wikipedia/commons/f/f0/Tux_ecb.jpg)
 encrypted with ECB, but
  never actually really understood why it happened or implemented a crack like
  this, so it was pretty cool to see it in action.
- To be sure that the message hasn't been tampered with like we did in the
  CBC bitflipping attack, we can use a [Message Authentication Code](https://en.wikipedia.org/wiki/Message_authentication_code)
  (MAC) to sign the message, if it were HMAC'd the modified ciphertext would
  cause the MAC change and we would know it was tampered with.

## Set 3 - Block and Stream Crypto

In progress... I've completed 21, 22, and 23 so far from this set - all the
challenges introducing the 32 bit version of the Mersenne Twister PRNG.

### What I learned
I jumped ahead to learn more about Mersenne Twister and how it works because
it was relevant to another challenge I was trying to solve. I wrote up more
detailed notes about it in [this post](https://heathhenley.github.io/posts/python-random-module-random-notes/). In short, I learned about how the Mersenne Twister works, how it's seeded in Python specifically, how to clone it and crack, etc.
