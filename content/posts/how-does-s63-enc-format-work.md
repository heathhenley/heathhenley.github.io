---
title: "How Does S-63 Electronic Navigational Chart (ENC) Format Work?"
date: 2024-01-30T19:18:13-05:00
draft: false
metathumbnail: "/s63/thumbnail.png"
description: "A high level overview of the IHO S-63 standard, and how to implement the main parts in Python. This post talks about the S-63 standard, a data protection scheme for official Electronic Navigational Charts (ENCs). It's a process for encrypting and signing S-57 data, and it's used to make sure that the data is official, and that it hasn't been tampered with. It also makes it possible for data providers to license sections of the charts (cells) and be sure that they are not being shared illegally / pirated. All of the examples are implemented in python and available in a Gist. S-57 will eventually be replaced by the S-100 standard - but it will probably be around for a while."
tags: ["python", "ENCs", "S63", "S57", "GIS", "cryptography"]
categories: ["python", "ENCs", "S63", "S57", "GIS", "cryptography"]
keywords: ["python", "ENCs", "S63", "S57", "GIS", "cryptography",
  "IHO S63 standard", "User Permit Number", "Hardware ID"]
---

TL;DR: S-63 is a standard used to encrypt official Electronic Navigational Charts (ENCs). Encryption and
signing allow clients to be sure that they have official data and that it has
not been tampered with. It also make it possible for data providers to license 
sections of the charts (cells) and be sure that they are not being shared 
illegally / pirated. This post gives an overview of the process and shows how 
to implement the key steps. All of the examples are implemented in python and available
in a [Gist](https://gist.github.com/heathhenley/7ecf13aec81eb09399d204e240306e00).

## Vector Charts and S-57 Format
The International Hydrographic Organization (IHO) has defined a standard for
vector navigational charts called [S-57](https://iho.int/uploads/user/pubs/standards/s-57/31Main.pdf).
The standard defines how to store the data, what attributes to use, etc,
and the format is open so that anyone can implement it. NOAA even publishes all
their ENC data in S-57 format, you can [download the data](https://charts.noaa.gov/ENCs/ENCs.shtml) or 
[view it in a web viewer](https://www.nauticalcharts.noaa.gov/enconline/enconline.html).
Ships over a certain size are required to have
official ENCs on board (due to [SOLAS](https://www.imo.org/en/About/Conventions/Pages/International-Convention-for-the-Safety-of-Life-at-Sea-(SOLAS),-1974.aspx)) - they previously used paper charts.

Still though, how does the captain know that the data it has is official and 
hasn't been tampered with or corrupted? And do the providing agencies have any 
way to control who can access the data, so they can get their money? The answer 
that the IHO came up with is the S-63 standard, a process for encrypting and 
signing the S-57 ENC data.

## IHO S-63 Standard
The IHO S-63 standard defines a process for encrypting and signing S-57 data,
you can find the full standard here in [IHO Publication S-63](https://iho.int/uploads/user/pubs/standards/s-63/S-63_2020_Ed1.2.1_EN_Draft_Clean.pdf). If you
like to dig though standards like that, and want even more detail than what
is given in this blog, I recommend you go straight to the source. The whole
process is defined and if you're into that sort of thing you can
skip the rest of this post and check that out to get an idea of it instead.
I'm going to attempt to explain the process in a more approachable way, and 
include some code examples that complete the most important parts.

### Why - What Problem Does S-63 Solve?
The IHO "surveyed" the hydrographic offices of the world (the ones that 
participate in the IHO at least) and found that there was a general consensus 
that they wanted to
establish a security scheme for their electronic charts. They were concerned
about piracy of their data due to the implications that may have for navigation 
safety and, for their bottom line (second part is not in the official 
documentation 😉). This is what they came up with.

### First - who's involved?
Sticking to the naming used in the official documentation, here are the main 
players in the S-63 process:
- Scheme Administrator - the one, the only, you guessed it, the IHO
- Data Server - provides the official data, there are lot of these, most
  associated with the hydrographic offices or other official distributors of
  the data
- Data Client - the one who wants to use the data, usually a ship or other
  vessel
- Original Equipment Manufacturer (OEM) - this is where I got all my experience
with this lovely process. This is the company that makes a navigation system,
like a chart plotter, or other software (voyage planning, etc) that wants to let
their users pull in official ENC data. 

Maybe a picture will help:

![S-63 Players](/s63/players.png)

It's basically a central 'authority' (the IHO) that defines the process, and then
the data servers get the official data and OEMs set up their software to display
it for the user.

### What's the process?

#### Overview
As a high level overview, here's what happens:
- the data server gets the official data from the hydrographic office or another
  official distributor, whoever is making the actual charts from the survey data
- the data is in standard S-57 format (the standard for vector charts)
- the data server encrypts the data with a random key (we'll see more later)
- the data server sends out the encrypted data to any users who might want it
- a user who wants to use the data must send a User Permit Number (UPN) to the
  data server, and probably some money too (to license the data for a certain
  period of time, or for a certain geographic area, etc)
- the data server gladly takes the user's money, and User Permit Number, and then
  sends the user a "permit" file, which contains the key that was used to
  encrypt the parts of the data that the user has licensed

The tricky part here is that the User Permit Number (UPN) is installation specific,
so every client for a given OEM will have a different one. They need be able to send a unique
hardware identifier to the data server, and then receive the key(s) needed to
decrypt the data in a way that is secure. So the rest of this post is about how
that actually happens, and how to implement some of the main parts in Python.

Is it really secure? I don't know, I'm not a security expert. I do know that 
the scheme was
put in place years ago though. It only uses a 40-bit encryption key for
the data. It's not unreasonable to brute force 40-bits (we'll probably be trying
that in another blog). The scheme might not stop a determined attacker from
accessing the data if for some reason they really want some
of that juicy navigation info. However, the encrypted cells are also
signed so that their origin can be confirmed, and if the signature checks out (and the correct public key is
used checking the signature), the user can trust the origin of the data.

A new [S-100 standard](https://iho.int/uploads/user/pubs/standards/s-100/S-100_5.1.0_Final_Clean.pdf) is set to replace S-57 and S-63,
but it will likely be a while before it replaces the old standards. In the new
standard, AES in CBC block mode is used for encryption instead of Blowfish. The big change is
that the key size can be 128, 192, or 256 bits. That's a big improvement over the 40-bit
keys in this standard...

Either way, let's look at how to implement each piece, and maybe see how they 
fit together a bit better.

#### The Manufacturer ID and Private Key
The first step for the OEM who wants to show S-63 data in their software is to
register with the IHO. Once approved, they will be assigned a "manufacturer id"
(`M_ID`) and a "manufacturers key" (`M_KEY`). The list of OEM `M_ID` and 
`M_KEY` pairs is shared with the official data servers, but not with users,
other OEMs, or anyone else. The `M_ID` is a 2 character alphanumeric code, but
represented using it's ASCII representation in hexadecimal. So for example, the
`M_ID` `01` is represented as `3031`. The `M_KEY` is a five character hexadecimal
number, but also often represented using it's ASCII representation in hexadecimal. So
the `M_KEY` `123AB` is represented as `3132334142`. This needs to be kept secret
by the OEM. The IHO shares this with the data servers by a secure means that
is no doubt well documented, so that they have access to it in later steps.

We can flip these around in python by encoding the ASCII representation to 
bytes and then dumping to hex:
  
```python
# ascii str of hex chars to hexstr
"01".encode().hex() # --> '3031'
"123AB".encode().hex() # --> '3132334142'

# back the other way
bytes.fromhex("3031").decode() # --> '01'
bytes.fromhex("3132334142").decode() # --> '123AB'
```

#### The Hardware ID
So to really get started - we want each installation to have a unique 
identifier, so that the data server can license the data to the specific 
users / specific installations. We know that each approved OEM has an `M_ID`
and `M_KEY` assigned by the IHO. Next, the OEM generating this installation 
specific identifier is the first step in the real process. 

As mentioned, the install specific identifier is called the Hardware ID, and 
each OEM has their own process for generating it. They could come from serial
numbers of some piece of hardware, be randomly generated, etc, they just need to be unique. It's not meant to be 
shared with the user in raw format. This is a five character string of 
hexadecimal characters, and it's recommended that it is not generated 
sequentially.

For example, a valid hardware id is "A79AB" - this is the one used in the S-63 
docs as an example.

What's our goal now? We want to find a way to give the data server the hardware
id (`HW_ID`) for a specific installation. They can then use that to encrypt the
key that they used to encrypt the cell of S-57 data, and send the cell key back to the user. 
Then the user can use the same hardware id to decrypt the cell key, and then use 
that cell key to decrypt the S-57 data that they have licensed!

But how do we get the hardware id to the data server in a secure way? We don't 
want to just send it and risk it being intercepted, corrupted, or tampered 
with. We need to encrypt it somehow...

#### The User Permit Number
The User Permit Number (UPN) is the way that the user can send the hardware id
to the data server in a secure way. The OEM software the user is using takes the
hardware id and encrypts it with the `M_KEY` (the one that the IHO assigned to
the OEM). This results in 16 hex characters. Then a CRC32 checksum is calculated
and added to it (8 hex characters). Finally the `M_ID` is added to the end, in
hex string format (4 hex characters). So we end up with a 28 character string
that looks like:

![User Permit Illustration](/s63/upn.png)

So let's look at an example of how to make a User Permit Number. We'll use the
the IHO's [implementation test kit](https://iho.int/uploads/user/Services%20and%20Standards/ENC_ECDIS/data_protection/S-63_Test_Data_Implementation_Guide_v1.1.pdf), with the numbers: `M_ID` is `10` and the 
`M_KEY` is `10121`. The hardware id for this "installation" is `12345`.

First, we need to encrypt the hardware id (`12345`) using the [Blowfish
algorithm](https://en.wikipedia.org/wiki/Blowfish_(cipher)) using the `M_KEY` as the key. We can use the `pycryptodome` package
(`pip install pycryptodome`) to do this:

```python
from Crypto.Cipher import Blowfish

# Assigned to manufacturer (OEM) by IHO:
m_id = "10"
m_key = "10121".encode()

# Generated by manufacturer (OEM) for each installation:
# - padded to 8 bytes
hw_id = "12345".encode() + b"\x03" * 3

# Encrypt hardware id with blowfish and m_key
cipher = Blowfish.new(m_key, Blowfish.MODE_ECB)
encrypted_hw_id = cipher.encrypt(hw_id).hex().upper()

print(f"Encrypted HW_ID: {encrypted_hw_id}")
# --> Encrypted HW_ID: 66B5CBFDF7E4139D
```

Next, we need to calculate the CRC32 checksum of the encrypted hardware id. In
python, we can use the `binascii` package to do this:

```python
import binascii

# Compute the CRC32 checksum of the encrypted hw_id:
crc = binascii.crc32(encrypted_hw_id.encode())
crc = f"{crc:x}".upper().zfill(8)
print(f"CRC32: {crc}")
# --> CRC32: 5B6086C2
```

Finally we can make the actual User Permit Number by combining the [CRC32](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)
checksum, and the `M_ID`:

```python
upn = encrypted_hw_id + crc + m_id.encode().hex().upper()
print(f"UPN: {upn}")
# --> UPN: 66B5CBFDF7E4139D5B6086C23130
```

There we go, we have the final User Permit Number of: 
`66B5CBFDF7E4139D5B6086C23130` that contains (1) the encrypted `HW_ID` (
encrypted with the `M_KEY`), (2) the CRC32 checksum, and (3) the `M_ID`.

#### That's the UPN - what's next?
This User Permit Number is obviously unique for each installation, as it's
based on the `HW_ID`. This is the number that the user will send to the data
server - remember that they have a list of all the super secret `M_ID` and
`M_KEY` pairs, so they can use the `M_ID` at the end of the UPN to look up the
corresponding `M_KEY`, and use it to decrypt the encrypted `HW_ID` out of the 
UPN.

They now know the secret hardware id (`HW_ID`) for a specific installation.
They can use that hardware id to encrypt the random key that they used to
encrypt one cell (usually some defined geographic region) of the S-57 data.
They send that back to the user as a "cell permit".

#### The Permit File
The data server creates a "cell permit" in order to communicate the key used to 
encrypt the S-57 data back to the user. The cell key (well actually two keys) is 
encrypted with the hardware id (`HW_ID`) that the user sent in the UPN as 
described above, and then put in the permit for the user. The user's software
as we said above, can then decrypt the key (it's knows the `HW_ID` that was
used to encrypt it) and finally, use that key to decrypt the S-57 data.

So what does that permit look like?

It contains the identifier of the ENC cell that it corresponds to, so that's
usually a geographic region at a particular scale. The cell names look
something like: `NO4D0512`, or `NO5F1615`. Finally, the resulting permit file
(`permit.txt` by naming convention) is going to look something like this:

```
:DATE 20030909 09:02
:VERSION 1
:ENC
NO4D051220040826F7B3814E59C84805D150D571B9BE53A637BD0B34F1F8F091,0,3,0,
NO5F1615200408262324DAD11E4C2BDC6CCC1D301FC162755946447034895300,0,4,0,
:ECS
```

where the lines following `:ENC` are the cell name, the expiration date of the 
permit, the key used to encrypt the data (but encrypted with our `HW_ID`), an 
extra key (the 'next' key that will be used to encrypt the data when the 
current one rotates), another CRC32 checksum, and finally some other metadata 
for the cell (version, etc).

So the actual permit (the part we care about mostly) for each cell is broken 
down like so for the cell `NO4D0512` in the file above:


![Cell Permit Illustration](/s63/permit.png)

So we (we're the data client now) already had the encrypted S-57 data ("S-63 
data"). Now we have the permit file from the data server with cell permits like 
the one above, with one row for each cell we have paid for. They had to 
generate the permit file using the hardware id (`HW_ID`) we sent them in the 
UPN to encrypt the keys used for this particular ENC cell. So how did they do
that?

Let's look at how the data server makes a permit file for a cell. We'll use the 
same `M_ID`, `M_KEY`, and `HW_ID` as above so that we can use the S-63 test kit 
data. We'll stick with our `NO4D0512` cell name - the key used to encrypt the 
cell data is a random 5 byte number. There is a second cell key sent along too, 
it can be the next key the server will use if it rotates keys, or it could be 
same as the first, that's a detail and it's up to each particular data server. Either way, 
let's say we've already generated them (maybe using `os.urandom(5).hex()` or 
some other method) and we have: `9C467D359D` and `27737811B4` for the first and
second keys for the cell.

The user sent us (we're the data server now) the UPN `66B5CBFDF7E4139D5B6086C23130`, so we've got to use
that to get their `HW_ID`, then use that to encrypt the keys for the cell. Those
last 4 characters of the UPN are the `M_ID` - here that's `3130`, we can use
that to look up the secret `M_KEY` in the official list of OEMs from the IHO. We
get the `M_KEY` `10121` for the `M_ID` `3130`. Then we can use that to decrypt
the `HW_ID` from the UPN:
  
```python
import binascii
from Crypto.Cipher import Blowfish

upn = "66B5CBFDF7E4139D5B6086C23130"

# We looked up the MKEY using the MID at the end of the UPN (3130)
MKEY = "10121"

# the encrypted hw id is the first 16 chars
encrypted_hw_id = upn[:16]

# the crc check is the next 8:
crc = upn[16:24]

# we can make sure it matches to ensure there was no error / data
# integrity issues:
assert crc == f"{binascii.crc32(encrypted_hw_id.encode()):x}".upper()

# The crc checks out, so lets decrypt the hw_id using the m_key:
cipher = Blowfish.new(MKEY.encode(), Blowfish.MODE_ECB)

decrypted_hw_id = cipher.decrypt(bytes.fromhex(encrypted_hw_id))
# only the first 5, the rest is padding
print(f"HW_ID: {decrypted_hw_id[:5].decode()}")
# --> HW_ID: 12345
```

So we've got the client's `HW_ID` now, and we can use that to encrypt the keys
for this cell:

```python
hw_id = "12345" # --> got this from the UPN as above
# append first byte of hw_id to end of hw_id (in 9.6.2 of standard)
hw_id = hw_id + hw_id[0]

# padding to 8 bytes
padding = b"\x03" * 3

# the cell keys that were randomly generated, and used to encrypt the cell
cell_key1, cell_key2 = "9C467D359D", "27737811B4"

cipher = Blowfish.new(hw_id.encode(), Blowfish.MODE_ECB)
eck1 = cipher.encrypt(bytes.fromhex(cell_key1) + padding).hex().upper()
eck2 = cipher.encrypt(bytes.fromhex(cell_key2) + padding).hex().upper()

print(f"Encrypted key 1: {eck1}\nEncrypted key 2: {eck2}")
# --> Encrypted key 1: F7B3814E59C84805
# --> Encrypted key 2: D150D571B9BE53A6
```

Finally, we can add the cell name, the expiration date, and the encrypted keys
together, and compute the CRC32 checksum for the permit file:

```python
# these are fixed for the cell
cell_name = "NO4D0512"
expiration_date = "20040826"

# add the cell name, expiration date, and encrypted keys together
permit = f"{cell_name}{expiration_date}{eck1}{eck2}".upper()

# compute the crc32 checksum of the permit
hex_crc = f"{binascii.crc32(permit.encode()):x}".upper()
bytes_crc = bytes.fromhex(hex_crc)

# encrypt it with the hw_id (with the special "first byte appended" thing)
encrypted_crc = cipher.encrypt(bytes_crc + b"\x04" * 4).hex().upper()

# add the crc to the permit
permit += encrypted_crc
print(permit})
# --> NO4D051220040826F7B3814E59C84805D150D571B9BE53A637BD0B34F1F8F091
```

So there we go, we've got the permit for the cell `NO4D0512` that will be sent
to the user. The user's software can then, you guessed it, decrypt the keys
in the permit using they this system's `HW_ID`, and finally use them to decrypt
the S-57 data for the `NO4D0512` cell and display it.

That process starts with the reverse of what we just id, then the key is used to decrypt the chart data, and finally the data uncompressed and can be displayed.

#### Decrypting the S-57 Data
So we reverse the process we just implemented to get the cell keys:

```python
full_permit = "NO4D051220040826F7B3814E59C84805D150D571B9BE53A642E5B05951975E9C"

# extract encrypted cell key
eck1 = full_permit[16:32]

# go backwards to get the cell key from the permit
hw_id = "12345"
hw_id6 = hw_id + hw_id[0]

# decrypt the cell key
cipher = Blowfish.new(hw_id6.encode(), Blowfish.MODE_ECB)
ck1 = cipher.decrypt(bytes.fromhex(eck1))[:5]
```

and then we can use the cell key to decrypt the S-57 data for the cell:

```python

import zipfile

# encrypted s57
with open("NO4D0512.000", "rb") as f:
  encrypted_s57 = f.read()

# decrypt the s57 data with the cell key
cipher = Blowfish.new(ck1, Blowfish.MODE_ECB)
decrypted_s57_zip = cipher.decrypt(encrypted_s57)

# check the header (the zip file header starts with PK)
assert decrypted_s57_zip[0:2] == b"PK", "invalid zip"

# save a copy of the decrypted chart
with open("NO4D0512_decrypted.000", "wb") as f:
  f.write(decrypted_s57_zip)
  
# extract
with zipfile.ZipFile("NO4D0512_decrypted.000") as z:
  z.extractall("NO4D0512_decrypted_unzipped.000")
```

There you go! We have an unencrypted S-57 file that we can now display in any
chart viewer that supports the format. There's also a GDAL driver for it, so
it could be converted into other GIS formats, or rasterized, etc. Here's the
cell we just decrypted, displayed in QGIS:

![NO4D0512](/s63/s57.png)

We can see all the vector layers, and while QGIS is not representing the data
as a chart plotter would, it's all there! It was just a quick way to check that
the decryption / extraction worked.

In the real scheme, you would want to check the second key if the first one
fails. The permit checksum should also be checked before using the keys, but
I didn't include that here (we've already seen how to do it). Of course the
expiration data needs to be checked too to make sure the permit is still valid,
and a few other details that I didn't include here.

The only significant part of the process that we didn't cover here is that the
compressed, encrypted S-57 data is also sent with a signature that is validated
to confirm the origin of the data. Maybe that's a topic for another post!

## Conclusion
If you read this far, thanks! Hope you learned something, let me know if you
have any questions or comments.

The full code for this post is available in a [Gist](https://gist.github.com/heathhenley/7ecf13aec81eb09399d204e240306e00).
