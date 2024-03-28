---
layout: post
title: "random (python)"
author: "melon"
date: 2023-10-20 20:07
categories: "2023"
tags:
  - python
---

### # introduction to RNG and PRNG
system RNG, which provides a source of random numbers based on system-level entropy.  
system RNG, like /dev/random or /dev/urandom, utilizes entropy sources from the operating system or hardware to generate random numbers

pseudo-random number generator is an algorithm that generates a sequence of numbers that appear to be random but are actually deterministic.  
basic features about PRNG are: deterministic generation, periodicity after certain deadline,
pseudorandomness but not truly random (can be predicted by seed), seed selection (to determine the sequence).

<hr>

### # random vs urandom
the main distinction between random(PRNG) and urandom(system RNG) is one of use cases.

random implements deterministic PRNGs (pseudo-random number generators), there are scenarios where you want exactly those.
for instance when you have an algorithm with a random element which you want to test, and there's need to make the test to be repeatable.
in that case you want a deterministic PRNG which you can seed.

urandom on the other hand cannot be seeded and draws its source of entropy from many unpredictable sources,
making it more random.

true random is something else yet and you'd need a physical source of randomness like something that measures atomic decay;
that is truly random in the physical sense, but usually overkill for most applications.

<hr>

### # code in action
```text
import random
import string

# generate str in given scope
def get_str(len=8, scope=""):
    if not scope or not isinstance(scope, str):
        ret = "".join(random.SystemRandom().choice(string.ascii_lowercase + string.digits) for _ in range(len))
    else:
        ret = "".join(random.SystemRandom().choice(scope) for _ in range(len))
    return ret

# generate num at a specific len if no fro/to given; or generate num in scope [fro,to] without len
def get_int(len=5, fro=None, to=None):
    if fro == None or to == None:
        ret = "".join(random.SystemRandom().choice(string.digits) for _ in range(len))
    else:
        ret = random.randint(fro, to)
    return int(ret)


if __name__ == "__main__":
    s = get_str(8, "0123456789ABCDEF")
    print(s)                           # AD6A1198
    k = get_int(fro=3, to=800)
    print(k)                           # 142
    l = get_int(len=2)
    print(l)                           # 82
```
ref: hostfw/pkg_host/utils/myrandom.py
