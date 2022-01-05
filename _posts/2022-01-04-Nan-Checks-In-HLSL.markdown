---
layout: 	post
title:  	"NaN Checks In HLSL"
date:   	2022-01-04 00:00:00 -0500
category: 	"Graphics"
published:	false
---

The topic of detecting NaN in HLSL shaders comes across quite frequently. NaN(s) can create a mess, specially when using previous frames' data as they propagate across the screen. 

Renderdoc has a great overlay for displaying NaN(s) but other tools may not have that and sometimes it is best to catch these at runtime using your own debug visualization.

HLSL has intrinsics to check for NaN but [these can get optimized away unless you force IEEE strictness](https://twitter.com/_Humus_/status/1074973351276371968?s=20).

The following includes a few code snippets with explanation on how to check for NaN using your own hand-rolled code.


```C++
bool IsNan(float val)
{
    return !(val < 0.f || val > 0.f || val == 0.f);
}
```

```C++
bool IsNan(float val)
{
    uint exponent = (asuint(val) & 0x7F800000) >> 23;
    uint mantissa = (asuint(val)) & 0xffffff;
    return exponent == 0xff && mantissa != 0;
}
```
[IEEE-754 Representation](http://www.fredosaurus.com/notes-java/data/basic_types/numbers-floatingpoint/ieee754.html)

```C++
bool IsNaN(float x)
{
    return (asuint(x) & 0x7FFFFFFF) > 0x7F800000;
}
```

