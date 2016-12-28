---
layout: post
title:  "Screen Space Reflections in Killing Floor 2"
date:   2016-12-25 22:23:46 -0500
---

![img1](/images/KF2-SSR-1.png)

This is a summary of the Screen Space Reflections implementation used in Killing Floor 2. I will attempt to trace back and work through each step in the process of implementing this feature. Not all of it made it to the final shipped version, but it helped shape it.

### Overview
The basic algoritm is quite simple at heart. See sections 4.1-4.3 of Yasin Uludag's article in GPU Pro 5 [^fn1] for a great introduction on this technique. 

The following code snippet shows the main entry point which computes the parameters required for raymarching.

``` glsl
[numthreads(WARP_SIZE, GROUP_SIZE, 1)]
void Main(uint3 DispatchID : SV_DispatchThreadID)
{
	uint2 PixelID = uint2(DispatchID.x, DispatchID.y);
	float2 PixelUV = float2(PixelID) / BufferSize;
	float2 NDCPos = float2(2.f,-2.f) * PixelUV + float2(-1.f,1.f);

	// Prerequisites
	float3 WorldPosition = MulMatrix(ScreenToWorldMatrix, float4(NDCPos, 1, 0)).xyz;
	float DeviceZ = DepthBufferTexture[PixelID];
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

![img1](/images/KF2-SSR-3.png)

The output of the SSR pass now looks like this.

![img1](/images/KF2-SSR-4.jpg)

This technique, however, has a host of limitations [^fn3] which need to be handled gracefully. Also, a naive implementation is quite slow as progressive ray marching can result in a ton of texture lookups.

# References

[^fn1]: "Hi-Z Screen-Space Cone-Traced Reflections", Yasin Uludag, GPU Pro 5
[^fn2]: [Screen Space Ray Tracing](http://casual-effects.blogspot.com/2014/08/screen-space-ray-tracing.html)
[^fn3]: [The Future of Screen Space Reflections](https://bartwronski.com/2014/01/25/the-future-of-screenspace-reflections/)
[jekyll-talk]: https://talk.jekyllrb.com/
