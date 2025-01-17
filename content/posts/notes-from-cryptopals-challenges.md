---
title: The Cryptopals Challenges
date: 2023-12-29T12:58:01-05:00
draft: false
description: Summary and notes about my learnings from the Cryptopals Crypto Challenges. I started knowing nothing about cryptography, except that you can see Tux the penguin if you encrypt him with ECB mode. I am pushing updates as I have time to work through the challenges. Picking them off one at a time, so this page will be a constant work in progress.
tags:
  - cryptography
  - notes
  - software
  - software-engineering
categories:
  - python
  - software
  - software-engineering
  - notes
keywords:
  - python
  - cryptography
  - software
  - software-engineering
mathjax: true
---

**TL;DR:** I'm working through the
[Cryptopals Cryptography Challenges](https://cryptopals.com/), starting with
knowing nothing about cryptography. These are my (not so) random notes and
takeaways from them. They generally seem to follow a pattern of introducing a
concept in cryptography (repeating key XOR, ECB mode, CBC mode, stream ciphers,
etc) and then having you break it in some way(s). I'm hoping to turn my notes on
some of these in to more detailed posts at some point, but for now, this mostly
'stream of thought' that I'm updating as I work through them.

**Most difficult so far** - The CBC padding oracle attack was hard for me to
understand and tedious to implement. 

**Most interesting so far** - The length extension attack on SHA1 keyed MACs -
in general learning about how the Merkle-Damgard hash functions work (their
state IS the hash digest) was really cool.

## Set 1 - Basics

The solutions for this set are in the [set1 solutions](https://github.com/heathhenley/CryptoPals/tree/main/set1).

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

Solutions for this set are in the [set2 solutions](https://github.com/heathhenley/CryptoPals/tree/main/set2).

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

### CBC bit flipping attack
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

The solutions for this set are in the [set3 solutions](https://github.com/heathhenley/CryptoPals/tree/main/set3)

### CBC padding oracle attack
Had a lot of trouble debugging this one because of a mistake in my padding /
unpadding functions. I relied a lot on this [blog
post](https://www.nccgroup.com/us/research-blog/cryptopals-exploiting-cbc-padding-oracles/)
to finally get it working.
The idea here is that if some helpful developer has been a bit too helpful with
the error messages returned by the server when you send it a bad ciphertext to
decrypt CBC mode - you may be able to exploit that to decrypt the entire message. In this case if the server returns an error that specifically says when the padding is bad, versus just a generic decryption error, then you can exploit this 'padding oracle' to decrypt the message.

In CBC mode, the IV or the previous block of ciphertext is XOR'd with the
plaintext before it is encrypted. And in decryption, each block of ciphertext is
decrypted and then XOR's **with the previous block of ciphertext** to get the
plaintext. That's the key to this attack - we manipulate the n-1 block of
ciphertext (which is XOR'd with the decrypted n block of the ciphertext on the
server) until we get a normal deserialization error instead of the padding
error. This means that the decrypted and XOR'd block of ciphertext had valid
padding, if this is the last byte of the block, we know that

$$
CT_{n-1}^\text{last byte} \oplus PT_{n}^{\text{last byte}} = 0x01
$$

So the the last byte of the previous block of ciphertext, XOR'd with the last
byte of the current block of plaintext is 0x01 - which we can rearrange to get
the last byte of the plaintext block:
$$
PT_{n}^{\text{last byte}} = CT_{n-1}^{\text{last byte}} \oplus 0x01
$$
Then we can repeat this process - next setting the last byte of the current
block to 0x02 and manipulating the penultimate byte of the previous block of
ciphertext until we know the padding is good, and so on, decrypting the entire
message without ever knowing the key!

### Implement and break AES CTR stream cipher mode
Stream cipher mode is another way to encrypt data - instead of encrypting blocks
of the plaintext directly, you:
- Generate a random nonce (just a number)
- Start a counter at the nonce
- Generate a random key
- For each block of plaintext:
  - Increment the counter
  - Encrypt the counter with the key using some block
    cipher (AES in this case), this is the 'keystream'
  - XOR the resulting 'keystream' with the plaintext to get the ciphertext
  - Add the ciphertext to the result
There you have it - still using the same block cipher, but to generate the
keystream for each block of plaintext, instead of to encrypt the plaintext
directly.

If the nonce is reused with the same key to encrypt different plaintexts, instead of being generated randomly each time - then each plaintext will be XOR'd with the same keystream to get the ciphertext. This is bad - it basically
becomes a big repeating key XOR, where the 'repeating key' is the keystream of
the length of the shortest plaintext you have. Repeating key XOR can be broken
just as demonstrated in set 1, challenge 6.

### Mersenne Twister PRNG
I wrote up [some notes on
MT](https://heathhenley.dev/posts/python-random-module-random-notes/), more
specifically the implementation in Python, because of this challenge and another
related challenge for a CTF.

In short, the Mersenne Twister is a PRNG that is used in Python's `random`
module and in a number of other languages and frameworks - it may be fine for
some things, but it is definitely **not** fine for anything with security. The
output is completely deterministic based on the seed and it can be cloned
(challenge 23) just by collecting enough consecutive outputs. The seed can also
be brute forced (challenge 22).

Challenge 24 introduced using MT output as the keystream for a stream cipher. If
this is done, then the attacker can use brute force to find the seed used for
the PRNG. It's a good example of why you shouldn't use MT for anything security
related (use a CSPRNG or some other secure way to get random numbers on your
system).

### What I learned
Problem 17 - I learned about how to break AES CBC mode encryption by using the
padding oracle attack, provided a 'padding oracle' is available.
From problems 18, 19 and 20 - I learned about the AES CTR mode, how it works,
and how to break it if the nonce is improperly reused.
From problems 21, 22, and 23 - I learned a lot about Mersenne Twister and how it
works specifically. I wrote up more detailed notes about it in [this
post](https://heathhenley.dev/posts/python-random-module-random-notes/).  In
short, I learned about how the Mersenne Twister works, how it's seeded in Python
specifically, how to clone it and crack it, what can happen (you get pwn'd) if
you use it for something it should be used for (security) etc.

## Set 4 - Stream Crypto and Randomness

In progress... I jumped ahead again to learn more about MACs, SHA1 and HMAC as
it helped in some other CTF challenges I was working on. So far I "completed" 28
which just involved finding a pure python implementation of SHA1 and using it.
Then 29 involves breaking a SHA1 keyed MAC using a length extension attack which
is really interesting and was used to [break the Flickr API in 2009](https://www.semanticscholar.org/paper/Flickr's-API-Signature-Forgery-Vulnerability-Duong-Rizzo/785dcbf504173746e96722e4a578ef98f1351557).

### Length extension attack

Here's my [sha1 keyed MAC crack](https://github.com/heathhenley/CryptoPals/blob/main/set4/29.py) using a length extension attack.

This was really interesting to learn about. It's possible because of how the 
[Merkle-Damgard](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction)
family of hash functions work. Basically, the hash is computed in blocks, with
the previous block's hash being used as the initial state for the next block. So
if you know the hash of a message, you can use that to set the initial state of
the hash function and continue hashing from there. With this, you can compute
the hash of some evil string to append to a message, append it, and then send it
to server - where it will actually verify!

This is how it works - assuming that we're trying to crack a SHA1 based message
authentication code (MAC). The server is using a secret key to sign the message
like: `SHA1(key || user=heath)`, and sending the hash along with the message, it
could be something like `user=heath:HASH`. So we know the message and the hash. 
It would be great if we could add something to the message and have the server
accept it as valid...maybe changing the message to something like: `user=heath&admin=true` would be neat. But if we try that, the server will receive the new
message, prepend it with the secret key, hash it, and compare it to the hash it
received - which won't match. But with the length extension attack, we can
actually get this to work!

Some things to note about this attack:
- We know the message and the hash, so we can use the hash to set the initial
  state of the SHA1 function. This just splits the 40 byte hash into the 5
  32-bit integers that SHA1 uses as the initial state of the hash function. In
  some implementations these are h0, h1, h2, h3, and h4.
- SHA1 pads the thing it's hashing to be a multiple of 512 bits. This means that
  we don't exactly have the hash for `key || user=heath`, but actually for
  `key || user=heath || padding`. This is important because now we need to make
  that padding ourselves before continuing the attack. The padding is always
  `100...0` followed by the length of the message in bits. After inevitably 
  messing this up a bunch, you will get it right and be able to generate the 
  padding to pad any message to be a multiple of 512 bits.
- We don't actually know the length of the secret key, but really it doesn't
  matter. We can just keep trying a bunch of reasonable lengths for the key,
  assuming we have some 'oracle' that will tell us if the hash we generate is
  correct.

The idea is
- we guess the length of the key
- make the padding that the server would have made in generating the hash of 
  the message `key || user=heath`.
- Then add the padding to the original message, and append our evil
  `&admin=true` - something like `user=heath || padding || &admin=true`.
- Set the state of SHA1 algorithm from the original hash supplied with the
  message.
- Hash the evil message, `&admin=true`, and record the hash - this is the new
  hash that will let us trick the server.
- Send the new message to the server, with the new hash. If it verifies, then we
  know we have the correct key length - if not, increment the key length and try
  again.

When the key length is correct, the server will hash the message we send it and
compare it to the hash we send along with the message. The first 512 bits of the
hash will be exactly what it hashed before - we just added the padding for it,
and that will result in the same hash as the original mac. However, the next
512 bits will be the evil part we added, `&admin=true` - the server will
continue hashing with it's state represented by the original hash and end up
with same hash that we calculated, without knowing the secret key!

### What I learned
- Length extension attack on SHA1 keyed MACs: I had never seen this before so
that was pretty cool to learn about. TL;DR - use HMAC which is not vulnerable
to this attack and made for this purpose.
