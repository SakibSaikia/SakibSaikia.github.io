---
layout: 	post
title:  	"Screen Space Reflections in Killing Floor 2"
date:   	2016-12-25 22:23:46 -0500
category: 	"Graphics"
published:	true
---

![img1](/images/KF2-SSR-1.png)

This is a summary of the Screen Space Reflections implementation used in Killing Floor 2. I will attempt to trace back and work through each step in the process of implementing this feature. Not all of it made it to the final shipped version, but it helped shape it.

* TOC
{:toc}

### Overview
The basic algoritm is quite simple at heart - you take the view vector per pixel, reflect it about the normal and ray march along the reflected vector until you get an intersection with the depth buffer. The scene color corresponding to the intersection UV is the reflected color. See sections 4.1-4.3 of Yasin Uludag's article in GPU Pro 5 [^fn1] for a great introduction on this technique. 

The following code snippet shows the main entry point which computes the parameters required for raymarching.

``` glsl
[numthreads(WARP_SIZE, GROUP_SIZE, 1)]
void Main(uint3 DispatchID : SV_DispatchThreadID)
{
	uint2 PixelID = uint2(DispatchID.x, DispatchID.y);
	float2 PixelUV = float2(PixelID) / BufferSize;
	float2 NDCPos = float2(2.f,-2.f) * PixelUV + float2(-1.f,1.f);

	// Prerequisites
	float DeviceZ = DepthBufferTexture[PixelID];
	float3 WorldPosition = MulMatrix(ScreenToWorldMatrix, float4(NDCPos, DeviceZ, 0)).xyz;
	float3 CameraVector = normalize(WorldPosition - CameraPosition);
	float4 WorldNormal = WorldNormalBufferTexture[PixelID] * float4(2, 2, 2, 1) - float4(1, 1, 1, 0);

	// ScreenSpacePos --> (screencoord.xy, device_z)
	float4 ScreenSpacePos = float4(PixelUV, DeviceZ, 1.f);

	// Compute world space reflection vector
	float3 ReflectionVector = reflect(CameraVector, WorldNormal.xyz);

	// Compute second sreen space point so that we can get the SS reflection vector
	float4 PointAlongReflectionVec = float4(10.f*ReflectionVector + WorldPosition, 1.f);
	float4 ScreenSpaceReflectionPoint = MulMatrix(ViewProjectionMat, PointAlongReflectionVec);
	ScreenSpaceReflectionPoint /= ScreenSpaceReflectionPoint.w;
	ScreenSpaceReflectionPoint.xy = ScreenSpaceReflectionPoint.xy * float2(0.5, -0.5) + float2(0.5, 0.5);

	// Compute the sreen space reflection vector as the difference of the two screen space points
	float3 ScreenSpaceReflectionVec = normalize(ScreenSpaceReflectionPoint.xyz - ScreenSpacePos.xyz);

	float3 OutReflectionColor;
	GetReflection(ScreenSpaceReflectionVec, ScreenSpacePos.xyz, OutReflectionColor);
}
```

One thing to note is that this technique uses fully qualified pixel coordinates `(ScreenCoord.xy, Devize_Z)`. It takes advantage of the fact that Device_Z can be linearly interpolated in screen space since it is already perspective correct. See Morgan McGuire's post [^fn2] for more details and the advantages of using this space.

After that, the ray marching proceeds as follows.

``` glsl
#define MAX_REFLECTION_RAY_MARCH_STEP 0.02f
#define NUM_RAY_MARCH_SAMPLES 16

bool GetReflection(
	float3 ScreenSpaceReflectionVec, 
	float3 ScreenSpacePos,
	out float3 ReflectionColor)
{
	// Raymarch in the direction of the ScreenSpaceReflectionVec until you get an intersection with your z buffer
	for (int RayStepIdx = 0; RayStepIdx<NUM_RAY_MARCH_SAMPLES; RayStepIdx++)
	{
		float3 RaySample = (RayStepIdx * ReflectionRayMarchStep) * ScreenSpaceReflectionVec + ScreenSpacePos;
		float ZBufferVal = DepthBuffer.SampleLevel(RaySample.xy, 0).r;
				
		if (RaySample.z > ZBufferVal )
		{
			ReflectionColor = SceneColor.SampleLevel(RaySample.xy, 0).rgb;
			return true;
		}
	}

	return false;
}
```

#### Binary Search
This simple progressive ray march is unlikely to give you good results as it can miss details due to the fixed step size. In an initial attempt, this was followed up with a binary search to converge on the actual intersection point.

{% highlight glsl %}
#define NUM_BINARY_SEARCH_SAMPLES 6

if (bFoundIntersection)
{
	float3 MinRaySample = PrevRaySample;
	float3 MaxRaySample = RaySample;
	float3 MidRaySample;
	for (int i = 0; i < NUM_BINARY_SEARCH_SAMPLES; i++)
	{
		MidRaySample = lerp(MinRaySample, MaxRaySample, 0.5);
		float ZBufferVal = DepthBuffer.SampleLevel(MidRaySample.xy, 0).r;

		if (MidSample.z > ZBufferVal)
			MaxRaySample = MidSample;
		else
			MinRaySample = MidSample;
	}
}
{% endhighlight %}

#### Deferred Color Lookup
One change that was made to this algorithm was for the SSR shader to return Screen Space UV's of the reflected color instead of the color itself. The UV's were used to look up and filter the scene color when the reflection was applied later in post process. This had a couple of benefits

1. It allowed us to kick off the SSR shader early in the frame (as soon as we had depth and normal buffers)
2. By the time the scene color was sampled, it already had color information from the translucency pass. So we get reflections from particle systems for free. 

*Note - The particle reflections aren't accurate and rely on having successful depth intersections from an occluder behind it. However, in most scenarios this works quite well and looks better than not having those reflections at all*

The image below shows reflections generated from the flamethrower.

![img3](/images/KF2-SSR-3.png)

The output of the SSR pass now looks like this.

![img4](/images/KF2-SSR-4.jpg)

#### Applying Reflections
In addition to the reflection UV, the SSR shader also outputs a `ReflectionBlurRadius` value based on roughness/gloss of the material and a `ReflectionContribution` value based on attenuation (discussed below). The following code shows the reflection apply shader. The output is alpha blended with the scene color `SceneColor = Alpha * ReflectionColor + (1 - Alpha) * SceneColor `

``` glsl
// Sample and filter scene color at ReflectionUV using the blur radius
ReflectionColor = BoxFilter(ReflectionUV, ReflectionBlurRadius);

// Use ReflectionContribution for alpha blending. Modulate Reflection Alpha 
// with ReflectionColor to fix black halos from low intesity reflection colors
OutColor = float4(ReflectionColor, ReflectionContribution * length(ReflectionColor));
```

### Limitations
This technique, however, has a host of limitations [^fn3] which need to be handled gracefully. This amounts to constructing attenuation values for each of these edge cases and combining these into a single attenation value which is output from the SSR shader as the `ReflectionContribution`. 

```
NetAttenuation = Att_A * Att_B * Att_C ... 
```

#### Viewer Facing Reflections
The reflection information is constrained to what is rendered on screen. Any reflection vectors pointing back at the viewer need to be dropped (with a proper fade out)

``` glsl
// This will check the direction of the reflection vector with the view direction,
// and if they are pointing in the same direction, it will drown out those reflections 
// since we are limited to pixels visible on screen. Attenuate reflections for angles between 
// 60 degrees and 75 degrees, and drop all contribution beyond the (-60,60)  degree range
float CameraFacingReflectionAttenuation = 1 - smoothstep(0.25, 0.5, dot(-CameraVector, ReflectionVector));

// Reject if the reflection vector is pointing back at the viewer.
[branch]
	if (CameraFacingReflectionAttenuation <= 0)
		return;
```

#### Reflections Outside the Viewport
Similar to above, any rays that march outside the screen viewport will not have any valid pixel information. These need to be dropped too.

``` glsl
UVSamplingAttenuation = smoothstep(0.05, 0.1, RaySample.xy) * (1 - smoothstep(0.95, 1, RaySample.xy));
UVSamplingAttenuation.x *= UVSamplingAttenuation.y;

if (UVSamplingAttenuation.x > 0)
{
	// sample z-buffer and perform intersection check
}
```

#### Reflections of Back Faces
Any part of an object that doesn't face the camera won't be rendered. As such any reflection vector that tries to sample from such a location won't have valid information. If we do not handle this, it will use the information from the front faces making it look incorrect and broken as shown below.

``` glsl
// This will check the direction of the normal of the reflection sample with the
// direction of the reflection vector, and if they are pointing in the same direction,
// it will drown out those reflections since backward facing pixels are not available 
// for screen space reflection. Attenuate reflections for angles between 90 degrees 
// and 100 degrees, and drop all contribution beyond the (-100,100)  degree range
float4 ReflectionNormalColor = tex2Dlod(WorldNormalBufferTexture, float4(ReflectionUV, 0, 0));
float4 ReflectionNormal = ReflectionNormalColor * float4(2, 2, 2, 1) - float4(1, 1, 1, 0);
float DirectionBasedAttenuation = smoothstep(-0.17, 0.0, dot(ReflectionNormal.xyz, -ReflectionVector));
```

|![img5](/images/KF2-SSR-5.png)|![img6](/images/KF2-SSR-6.png)|
|:----------------------------:|:----------------------------:|
|*Incorrect Reflections*       | *Fixed*                      |

#### Foreground Reflections
Unless the world is rendered first without a foreground pass (first person weapon), there will be missing information for all the world pixels that the foreground occludes. These will need to be masked and handled appropriately.

``` glsl
// Attenuate any reflection color from the foreground. 
// The GBuffer normal color for foreground objects is (0,0,1).
float ForegroundAttenuation = step(0.0001f, ReflectionNormalColor.r * ReflectionNormalColor.g);
```

#### Banding and Self-Intersection
Although the effect is minimal, initially some banding was noticed for the reflection color. A 4x4 Bayer matrix was used to dither the raymarch samples (trading noise for banding). Also, a constant bias was applied to the RaySample to prevent any self-intersections

|0 |8 |2 |10|
|12|4 |14|6 |
|3 |11|1 |9 |
|15|7 |13|5 |

``` glsl
#define RAY_MARH_BIAS 0.001f

// Dithered offset for raymarching to prevent banding artifacts
float DitherOffset = DitherTexture.SampleLevel(InUV * DitherTilingFactor, 0).r * 0.01f + RAY_MARH_BIAS;
float3 RaySample = ScreenSpacePos.xyz + DitherOffset * ScreenSpaceReflectionVec;
```

### HZB Optimization
A naive implementation is quite slow as progressive ray marching can result in a ton of texture lookups. Enter the Hi-Z buffer.

A Hi-Z buffer (or HZB) is computed by taking the min on each 4x4 tile of the depth buffer and storing it in the next mip level. We stop when the mip resolution falls below 8x8. The following images show the 6 highest mips of a HZB.

|![img7](/images/HZB1.png) |![img8](/images/HZB2.png) |
|![img9](/images/HZB3.png) |![img10](/images/HZB4.png)|
|![img11](/images/HZB5.png)|![img12](/images/HZB6.png)|

Here is the code snippet.

``` glsl
[numthreads(WARP_SIZE, GROUP_SIZE, 1)]
void DownsampleCS(uint3 DispatchId : SV_DispatchThreadID)
{
	float Depth0 = DepthTexIn[2 * DispatchId.xy];
	float Depth1 = DepthTexIn[2 * DispatchId.xy + uint2(1, 0)];
	float Depth2 = DepthTexIn[2 * DispatchId.xy + uint2(1, 1)];
	float Depth3 = DepthTexIn[2 * DispatchId.xy + uint2(0, 1)];

	DepthTexOut[DispatchId.xy] = min(min(Depth0, Depth1), min(Depth2, Depth3));
}
```

The HZB is used to quickly converge on the ray intersection by skipping empty space in the world. The following series of images taken from SIGGRAPH 2015 Real Time Rendering course by Tomasz Stachowiak [^fn4] best illustrates how HZB ray marching works.

![img13](/images/HZB-Raymarch.gif)

The following code shows the HZB accelerated ray marching. A max iteration count can be used to bail out if something goes wrong and the HZB ray march does not converge.

``` glsl
int MipLevel = 0;
while (	MipLevel > -1 && MipLevel < (NumDepthMips - 1))
{
	// Cross a single texel in the HZB for the current MipLevel
	StepThroughCell(RaySample, ScreenSpaceReflectionVec, MipLevel);

	// Constrain raymarch UV to (0-1) range with a flalloff
	UVSamplingAttenuation = smoothstep(0.05, 0.1, RaySample.xy) * (1.f-smoothstep(0.95, 1, RaySample.xy));

	if (any(UVSamplingAttenuation))
	{
		float ZBufferValue = DepthBuffer.SampleLevel(RaySample.xy, MipLevel).r; 

		if (RaySample.z < ZBufferValue)
		{
			// If we did not intersect, perform successive test on the next
			// higher mip level (and thus take larger steps)
			MipLevel++;
		}
		else
		{
			// If we intersected, pull back the ray to the point of intersection (for that miplevel)
			float t = (RaySample.z - ZBufferValue) / ScreenSpaceReflectionVec.z;
			RaySample -= ScreenSpaceReflectionVec * t;

			// And, then perform successive test on the next lower mip level.
			// Once we've got a valid intersection with mip 0, we've found our intersection point
			MipLevel--;
		}
	}
	else
	{
		break;
	}
}
```

Given below is the code to step across a single texel of the HZB.

``` glsl
// Constant offset to make sure you cross to the next cell
#define CELL_STEP_OFFSET 0.05

void StepThroughCell(inout float3 RaySample, float3 RayDir, int MipLevel)
{
	// Size of current mip 
	int2 MipSize = int2(BufferSize) >> MipLevel;

	// UV converted to index in the mip
	float2 MipCellIndex = RaySample.xy * float2(MipSize);

	//
	// Find the cell boundary UV based on the direction of the ray
	// Take floor() or ceil() depending on the sign of RayDir.xy
	//
	float2 BoundaryUV;
	BoundaryUV.x = RayDir.x > 0 ? ceil(MipCellIndex.x) / float(MipSize.x) : 
		floor(MipCellIndex.x) / float(MipSize.x);
	BoundaryUV.y = RayDir.y > 0 ? ceil(MipCellIndex.y) / float(MipSize.y) : 
		floor(MipCellIndex.y) / float(MipSize.y);

	//
	// We can now represent the cell boundary as being formed by the intersection of 
	// two lines which can be represented by 
	//
	// x = BoundaryUV.x
	// y = BoundaryUV.y
	//
	// Intersect the parametric equation of the Ray with each of these lines
	//
	float2 t;
	t.x = (BoundaryUV.x - RaySample.x) / RayDir.x;
	t.y = (BoundaryUV.y - RaySample.y) / RayDir.y;

	// Pick the cell intersection that is closer, and march to that cell
	if (abs(t.x) < abs(t.y))
	{
		RaySample += (t.x + CELL_STEP_OFFSET / MipSize.x) * RayDir;
	}
	else
	{
		RaySample += (t.y + CELL_STEP_OFFSET / MipSize.y) * RayDir;
	}
}
```

### Final Thoughts
The HZB optimization allowed us to run SSR at full res (1080p) in ~2.5ms on current gen console and comparabe PC hardware. The HZB generation itselft took ~0.5ms. The SSR shader was run using async compute right after the G-buffer pass (and before lighting), since the color lookup was [decoupled from it](Screen-Space-Reflection-in-Killing-Floor-2.html#deferred-color-lookup). 

Going forward, having a coarse SSR pass precdeing the HZB raymarching that can reject tiles we don't want to perform ray marching on might be beneficial. ~~I plan to do another post later with more detailed perf analysis.~~

# References

[^fn1]: "Hi-Z Screen-Space Cone-Traced Reflections", Yasin Uludag, GPU Pro 5
[^fn2]: [Screen Space Ray Tracing](http://casual-effects.blogspot.com/2014/08/screen-space-ray-tracing.html)
[^fn3]: [The Future of Screen Space Reflections](https://bartwronski.com/2014/01/25/the-future-of-screenspace-reflections/)
[^fn4]: [Stochastic Screen-Space Reflections](http://www.frostbite.com/2015/08/stochastic-screen-space-reflections/)
