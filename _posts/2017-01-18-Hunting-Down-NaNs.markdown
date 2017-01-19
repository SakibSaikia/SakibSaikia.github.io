---
layout: post
title:  "Hunting Down NaNs"
date:   2017-01-18 22:18:46 -0500
---

I recently updated an old project from DX9/SM3 to DX11/SM5. One of the things that I noticed immediately was that there were suddenly a lot of NaNs showing up which didn't exist earlier. This was a little surprising since the shaders hadn't changed. So, the difference must be in the HLSL intrinsics. A simple google search didn't return what I was looking for. So, I decided to look at the generated assembly for differences.

The following sample programs where compiled with fxc's /Fc option for the specific profile such as

```
fxc test.ps /T ps_5_0 /Fc test_sm5.asm
```

Here is a shader that takes in the XY components of a vector and reconstructs Z assuming that the vector was normalized in the first place. This is usually used when decoding normals.

``` glsl
float2 NormalXY;

float4 main() : SV_Target
{
	float NormalZ = sqrt(1.f - dot(NormalXY,NormalXY));
	return float4(NormalXY, NormalZ, 0.0);
}
```

Here is the SM5 assembly. This is mostly expected - take the dot product of the NormalXY vector, subtract from 1.0, and take the square root to get NormalZ.

``` asm
ps_5_0
dcl_globalFlags refactoringAllowed
dcl_constantbuffer CB0[1], immediateIndexed
dcl_output o0.xyzw
dcl_temps 1
dp2 r0.x, cb0[0].xyxx, cb0[0].xyxx
add r0.x, -r0.x, l(1.000000)
sqrt o0.z, r0.x
mov o0.xy, cb0[0].xyxx
mov o0.w, l(0)
ret 
```

and here is the corresponding SM3 assebly. The dot product and subraction are still there (although it is rolled into one `dp2add` instruction). However, instead of taking the sqare root, it computes the inverse square root (`rsq`) and then the reciprocal (`rcp`) of that. Whoa!

``` asm 
ps_3_0
def c1, 1, 0, 0, 0
mov r0.xy, c0
dp2add r0.z, r0, -r0, c1.x
rsq r0.z, r0.z
rcp oC0.z, r0.z
mul oC0.xyw, r0.xyzx, c1.xxzy
```

Turns out there is a good reason for that - you can compute an approximate inverse quare root much faster than an actual square root. See [^fn1] and [^fn2]. 

Additionally, SM3 takes the absolute value of the argument [^fn3] before computing `rsq()` which swallows up any NaNs resulting from when the argument is negative.

Another source of NaN I ran into was `pow()`, which is used all over the place in lighting code. Take this program as an example.

``` glsl
float x,y;

float4 main() : SV_Target
{
	float res = pow(x,y);
	return float4(res, res, res, 0.0);
}
```

This is what the generated assembly looks like

``` asm
ps_5_0
dcl_globalFlags refactoringAllowed
dcl_constantbuffer CB0[1], immediateIndexed
dcl_output o0.xyzw
dcl_temps 1
log r0.x, cb0[0].x
mul r0.x, r0.x, cb0[0].y
exp o0.xyz, r0.xxxx
mov o0.w, l(0)
ret 
```

The thing to note here is that `pow(x,y)` has been changed to `exp(y * log(x))`. This is because exp() and log() are quarter rate instructions[^fn5],[^fn6]. Which means that x needs to be a positive non-zero value, otherwise the result is undefined. Turns out SM3 implementation of log() already performs this high level logic [^fn4] for us which suppresses any NaNs.

``` glsl
float v = abs(src);
if (v != 0)
{
    dest.x = dest.y = dest.z = dest.w = 
        (float)(log(v)/log(2));  
}
else
{
    dest.x = dest.y = dest.z = dest.w = -FLT_MAX;
}
```

In short, for SM5 the intrinsic functions do just what you ask them to do - nothing more. It is upto you to feed correct values to these functions or make sure you `saturate()` or `clamp()` your values to the correct range.

# References

[^fn1]: [Fast inverse square root](https://en.wikipedia.org/wiki/Fast_inverse_square_root)
[^fn2]: [0x5f3759df](http://h14s.p5r.org/2012/09/0x5f3759df.html)
[^fn3]: [rsq - ps](https://msdn.microsoft.com/en-us/library/windows/desktop/bb147345(v=vs.85).aspx)
[^fn4]: [log - ps](https://msdn.microsoft.com/en-us/library/windows/desktop/bb174712(v=vs.85).aspx)
[^fn5]: [lInverse trigonometric functions GPU optimization for AMD GCN architecture](https://seblagarde.wordpress.com/tag/gpu-performance/)
[^fn6]: [Low-Level Shader Optimization for Next-Gen and DX11](http://www.humus.name/Articles/Persson_LowlevelShaderOptimization.pdf)