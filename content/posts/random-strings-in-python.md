---
title: "Efficiently Generating Random Strings in Python"
date: 2024-06-21T20:00:00
draft: False
---

The other day I had a need to generate a random eight character string in python to use as an request identifier when interacting with another system. Since it was a request identifier, I needed it to be random to minimize the chance of two identifiers being the same. Additionally, I had the requirement that it needed to consist of a case insensitive alphanumeric string. Due to the eight character limit, UUIDs were out, so I had to find another way to generate the identifier.

The first thing that came to mind was just to generate a four byte value and convert it to hex. That would give me about four billion possible values. The second way I thought of was to generate a random number and use it to select from one of 36 possible values (10 digits plus 26 letters). This gets me 36^8 = 2.8 trillion possible values. This second method reminded me of base-32 binary encoding, a close cousin of base-64 encoding that uses 0-9 and A-T to encode values, so I thought it'd be neat to try this as a method as well. To get an eight character string (stripping off the padding), you end up passing it five bytes for a grand total of about one trillion possible values.

All three are pretty simple to implement:

```python
import base64
import random
import string

# Method 1) Generate a 4 byte hex string
random.randbytes(4).hex()

# Method 2) use the random.choice method to
"".join(random.choices(string.digits+string.ascii_uppercase, k=8))

# Method 3) Generate a 5 byte value and base32 encode it
base64.b32encode(random.randbytes(5)).rstrip(b"=")
```

I was curious about the timing, so I used `timeit` to compare. This ran on a 10 year old Intel Macbook on Python 3.12.2.

``` python
import timeit

# Took 0.7197803930030204
timeit.timeit(
    'random.randbytes(4).hex()',
    setup='import random'
)

# Took 3.2596622809651308
timeit.timeit(
    'base64.b32encode(random.randbytes(5)).rstrip(b"=")',
    setup="import random; import base64",
)

# Took 2.2838933810126036
timeit.timeit(
    '"".join(random.choices(ALL_DIGITS, k=8))',
    setup="import random; import string; ALL_DIGITS=string.digits+string.ascii_uppercase",
)
```

It would seem that the `base64` method is not the way to go as it takes the longest and generates fewer distinct values than the hex approach. The `random.choices` approach presents a tradeoff of about three times more latency for three orders of magnitude more possible values. In my case, that tradeoff made sense, but different applications may prefer the performance.
