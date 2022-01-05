---
layout: 	post
title:  	"NaN Checks In HLSL"
date:   	2022-01-04 00:00:00 -0500
category: 	"Graphics"
published:	true
---

The topic of detecting NaN in HLSL shaders comes across quite frequently. NaN(s) can create a mess, specially when using previous frames' data as they propagate across the screen. 

Renderdoc has a great overlay for displaying NaN(s) but other tools may not have that and sometimes it is best to catch these at runtime using your own debug visualization.

HLSL has the `isnan()` intrinsic to check for NaN but [these can get optimized away unless you force IEEE strictness with `-Gis`](https://twitter.com/_Humus_/status/1074973351276371968?s=20).

The following includes a few code snippets with explanation on how to check for NaN using your own hand-rolled code.


```C++
bool IsNan(float x)
{
    return !(x < 0.f || x > 0.f || x == 0.f);
}
```
This one is self-explanatory and one of the first things that pop up when you google NaN check. I've tested it and it works, but it requires some level of trust. Perhaps we can do better..

HLSL float(s) use [IEEE-754](http://www.fredosaurus.com/notes-java/data/basic_types/numbers-floatingpoint/ieee754.html) 32-bit single precision representation, albeit [with some of the rules relaxed](https://docs.microsoft.com/en-us/windows/win32/direct3d11/floating-point-rules). 

There is a single sign bit followed by a 8 bit exponent and a 23 bit mantissa or significand.

[<img src="/images/nan-checks/IEEE-754.png">](/images/nan-checks/IEEE-754.png)

In this representation, a NaN is identified by a very specific bit pattern -- **an exponent which consists of all 1's and a mantissa whose value is non-zero**. 

An exponent of all 1's is the bit pattern
`0111 1111 1000 0000 0000 0000 0000 0000` or `0x7f800000` in hex. 

Let's rewrite the Nan check to look for this.

```C++
bool IsNan(float x)
{
    // 1. Extract the bits in the float by interpreting it as an unsigned int
    // 2. Bitwise AND to isolate the exponent and right shift to erase the mantissa bits
    uint exponent = (asuint(x) & 0x7f800000) >> 23;

    // 1. Extract bits in float as earlier
    // 2. Bitwise AND to extract the 23 bits of the mantissa
    uint mantissa = (asuint(x)) & 0x7fffff;

    // Check that exponent is all 1 and mantissa is non-zero
    return exponent == 0xff && mantissa != 0;
}
```
The above code can be condensed down to the following. I'll leave you to figure out that this does the same thing.

```C++
bool IsNaN(float x)
{
    return (asuint(x) & 0x7fffff) > 0x7f800000;
}
```

Pretty neat, huh?