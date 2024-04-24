---
title: "Notes: 'Hacking' on S63"
date: 2024-04-24T17:15:41-04:00
draft: false
description: "A previous post showed how the S63 encryption
  scheme works. This is an old scheme that is soon to be replaced.
  It's still in use though, here's a couple simple ways it could
  be broken depending on the information you have."
tags: ["python", "ENCs", "S63", "S57", "GIS", "cryptography"]
categories: ["python", "ENCs", "S63", "S57", "GIS", "cryptography"]
keywords: ["python", "ENCs", "S63", "S57", "GIS", "cryptography",
  "IHO S63 standard", "User Permit Number", "Hardware ID"]
---

TL;DR: The S63 encryption scheme is old and can be broken pretty easily
depending on the information you have. These are some notes on how with a
simple Python [script](https://gist.github.com/heathhenley/2f9ad47cf8b818f3ddb592569a013c87) to show what I mean.

So this is an old standard that will soon be replaced by S100, but
it's still in place and used today. The encryption part of the
scheme is mostly to allow data suppliers to license their data to
users in a standardized way and collected payment for access to that
data.

A [previous post](/posts/how-does-s63-enc-format-work/) walks through how the
whole scheme works. In short, in involves an encryption step that mostly makes
sure the that the data provider can charge for the data / avoid pirating, and a
signature step that makes sure the data hasn't been tampered with and that it is
actually from an IHO approved data provider. I haven't looked at the signing
algorithm yet, here we'll just look at the encryption part.

If you recall from the previous post, the data client, who wants to use charts
from a data provider, need to have a User Permit Number (UPN), specific to each
physical install, and send it to the data provider. The UPN is an encrypted way
to communicate the an installation specific hardware ID to the data provider.

The first case: assume we have a UPN - these are generally not too hard to get a
hold of. This UPN is the hardware id encrypted with an "M_KEY" that is specific
to the manufacturer of the data client's software (whatever is trying to
display charts)

Well - the M_KEY is only 5 bytes, and based on the spec the character set is
pretty restricted (ascii 0-9) - we can just brute force this. Keep decrypting
the UPN ciphertext with generated M_KEYs until we get a result with (1) valid
padding, and (2) three bytes of padding. This is a pretty good indication that
we've found the right key, it looks like this:

```python

# Here's a UPN we know some how, it contains the encrypted hwid
# but we don't know the mkey used to encrypt it...
upn = "66B5CBFDF7E4139D5B6086C23130"

# The encrypted hw id is the first 16 chars
encrypted_hw_id = upn[:16]

# the crc check is the next 8:
crc = upn[16:24]

# we can make sure it matches to ensure there was no error / data
# integrity issues:
assert crc == f"{binascii.crc32(encrypted_hw_id.encode()):x}".upper()

# The crc checks out, time to try to decrypt it, even without the key
for possible_key in ascii_generator_tr(5, "0123456789"):
  cipher = Blowfish.new(possible_key.encode(), Blowfish.MODE_ECB)
  decrypted_hw_id = cipher.decrypt(bytes.fromhex(encrypted_hw_id))
  # the real hw id will have valid padding:
  if decrypted_hw_id[-1] == 3 and valid_padding(decrypted_hw_id):
    # only the first 5, the rest is padding
    print(f"Found the HW_ID: {decrypted_hw_id[:5].decode()}")
    print(f"The M_KEY of the manufacturer: {possible_key}")
    break

```

This gives us the hardware id, which is specific to the install - but also the
M_KEY which is supposed to be 'secret' and specific to the manufacturer of the
client software...

The second case: assume we have a random cell permit file from a data provider,
maybe we found it on the internet or it was a permit for a different system. For
example, the attacker could have a permit file for a different system, but maybe
they don't want to buy another license for their second system.

The cell permit is the key used to encrypt the actual data, but it's encrypted
with the user's hardware id (obtained from the UPN). If we were legit, we would
just use our known hardware id to decrypt the cell permit, and then decrypt our
chart data. But we don't have the hardware id, so we'll just brute force it
again - this time with a bit more work because the hardware id is 5 bytes but
can use ascii 0-9 and A-F (any valid hex character).

```python

# What if we have a permit, but we don't know anything else? For example,
# it was made for another system, etc...
full_permit = "NO4D051220040826F7B3814E59C84805D150D571B9BE53A642E5B05951975E9C"

# you can download this file from: https://heathhenley.github.io/s63/NO4D0512_encrypted.000
# or from the IHO test kit
with open("NO4D0512_encrypted.000", "rb") as f:
  encrypted_s57 = f.read()

# extract encrypted cell key, just check the first one:
eck1 = full_permit[16:32]

# we don't know the hw_id that was used to encrypt the permit - so we have to
# guess it - basically the same idea as above, except the permits are encrypted
# wih a 6 byte key, the normal 5 byte hw_id and the first byte appended again.
for possible_key in ascii_generator_tr(5, "0123456789ABCDEF"):
  key = possible_key + possible_key[0]
  cipher = Blowfish.new(key.encode(), Blowfish.MODE_ECB)
  decrypted_cell_key = cipher.decrypt(bytes.fromhex(eck1))
  # the real key will have valid padding:
  if decrypted_cell_key[-1] == 3 and valid_padding(decrypted_cell_key):
    ck1 = decrypted_cell_key[0:5]
    print(f"Possible cell key: {ck1}")
    # try to decrypt S63 data with this cell key we found:
    cipher = Blowfish.new(ck1, Blowfish.MODE_ECB)
    decrypted_s57_zip = cipher.decrypt(encrypted_s57)
    if decrypted_s57_zip[0:2] == b"PK": # zips start with this
      if b"NO4D0512.000" in decrypted_s57_zip[:64]:
        print(f"Found cell key for this ENC: {ck1.hex().upper()}")
        print(f"Hardware ID: {possible_key}")
        break

```

We're using the same approach here - check for three bytes of padding. Here it
goes a step further and when the key is found, it tries to decrypt the S57 data
file. If the decryption is successful, it will be a zip file, and the first two
bytes will be "PK", and the cell name (also in the file name) will be in the
first 64 bytes of the decrypted data.

Neither of these really pose a threat to anyone except the data provider's
bottom line - but it's interesting to play around with.

Finally if we know nothing, we would have to brute the cell key, this is a
5 byte (40 bit) key and it can any byte value, so it's more work, but not
secure by any means. Anyone serious about cracking this would not have an issue,
though if we try to do it in our little python script in serial, it would take
1 year and half to enumerate all possible keys 😂.

Maybe I'll look at the signing algorithm next time, but I suspect that aspect
has held up to the test of time a bit better than the encryption part. In the
new standard, the keys are all longer, with fewer restrictions on the possible
bytes. For example AES is used instead of Blowfish, and the hardware id is 16
bytes instead of 5, and the cell key is 16 bytes as well.

Anyway, all the code is in this [gist](https://gist.github.com/heathhenley/2f9ad47cf8b818f3ddb592569a013c87)