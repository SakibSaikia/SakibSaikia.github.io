---
layout: 	post
title:  	"Going Indirect on UE3"
date:   	2017-08-18 15:45:46 -0500
category: 	"Graphics"
published:	false
---

The following is a simple algorithm that I used to change UE3's renderer to use indirect rendering, and draw a large number of instanced meshes in a single batched draw. The key motivations were

* Support long draw distances and higher draw calls
* Reduce CPU time spent on rendering, which for the most part runs on a single thread
* Minimal artist intervention - it just works!

### Overview
1. During an offline build phase, go through all the placed assets in the level and create an aggregate *InstancedDrawBatch* structure which contains instancing info and indirect args buffer (to be populated later)
2. Launch a culling task per draw batch. This performs frustum culling, occlusion culling (using HZB), projected bounds size culling, etc. The instance info for all instances that pass culling are copied over to another buffer which will be used during draw submission. The indirect args buffer is populated at the same time.
3. Submit the indirect args buffer generated for each batch using `DrawIndexedInstancedIndirect()`

### InstancedDrawBatch
This is an aggregate struct that holds all the information required to cull and render an instance group. The following is a concise and simplified C++ representation.

``` C++
struct InstancedDrawBatch
{
	FVertexBuffer vb; 				// Vertex buffer shared by all instances in batch
	FIndexBuffer ib; 				// Index buffer shared by all instances in the batch
	FIndexBufferType ib_pos;		// Simplified IB with degenerate verts removed 
	FTypedBuffer vcolor;			// Appended vert color data for every instance in the batch
	FStructuredBuffer inst_buff;	// Data unique to each instance - transforms, vcolor index, etc.
	FStructuredBuffer inst_buff_draw;// Same as above but filled out after culling. *Not Serialized*
	FConstantBuffer cbuffer;		// Constant data for the batch
	FMaterial mat;					// Material used by each instance - uniform for entire batch
	FLightingData light_info;		// Atlassed lightmaps, shadowmaps, etc for the batch
	FStructuredBuffer args_buff;	// DrawIndirect args filled out after culling. *Not Serialized*
	size_t num_instances;			// Number of instances in the draw batch
};
``` 

The structured buffer for `inst_buff` has the following data

``` C++
struct InstanceData
{
	matrix3x3 local_to_world;			// Local-to-world transform
	vector4 lightmap_coord_scale_bias;	// Lightmap atlas transform
	vector4	wind_direction_and_speed;	// Wind parameters. GPU-simulation. *Not Serialized*
	float draw_distance;				// For distance culling
	unint32_t vert_color_index;			// Index into InstanceColorBuffer 
	uint32_t vert_lighting_index;		// Index into vertex lighting typed buffers
};
```

### Culling
The culling phase rejects all the instances in a draw batch that fail visibility tests and generates the args buffer for the idirect draw call. This is run early in the frame, before any render passes. The previous frame's z-buffer is used for occlusion culling.

The following is the skeleton structure for the culling shader

``` glsl
// Write buffers
RWStructuredBuffer<PerInstanceInfo> DrawInstanceBuffer : register(u0);
globallycoherent RWByteAddressBuffer DrawArgsDstBuffer : register(u1);

// Read buffers
StructuredBuffer<PerInstanceInfo> InstanceBuffer : register(t2);

[numthreads(64, 1, 1)]
void Main(uint3 DTid : SV_DispatchThreadID)
{
	uint i = DTid.x;

	// Intitialize num draw instances to 0
	if (i == 0)
	{
		uint dummy;
		DrawArgsDstBuffer.InterlockedExchange(4, 0, dummy);
	}

	// Sync
	GroupMemoryBarrierWithGroupSync();

	// Generate draw args and instance data
	if (i < NumInstances)
	{
		// Expand bounding-box corner vertices into translated world space
		float3 CornerVerts[8];
		float3x4 LocalToWorld = GetLocalToWorld(i);

		CornerVerts[0] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(-1.0f, -1.0f, -1.0f), 1.0f));
		CornerVerts[1] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(-1.0f, -1.0f, 1.0f), 1.0f));
		CornerVerts[2] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(-1.0f, 1.0f, -1.0f), 1.0f));
		CornerVerts[3] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(-1.0f, 1.0f, 1.0f), 1.0f));
		CornerVerts[4] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(1.0f, -1.0f, -1.0f), 1.0f));
		CornerVerts[5] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(1.0f, -1.0f, 1.0f), 1.0f));
		CornerVerts[6] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(1.0f, 1.0f, -1.0f), 1.0f));
		CornerVerts[7] = mul(LocalToWorld, float4(BoundsOrigin + BoundsExtent * float3(1.0f, 1.0f, 1.0f), 1.0f));

		// Compute new (scaled) radius
		float3 WorldBoundsOrigin = 0.5f * (CornerVerts[0] + CornerVerts[7]);
		float ScaledRadius = length(CornerVerts[7] - WorldBoundsOrigin);

		// Distance cull
		[branch]
		if (DistanceCull(WorldBoundsOrigin, BoundsExtent, InstanceBuffer[i].MaxDrawDistance))
			return;

		// Frustum test
		[branch]
		if (FrustumCull(WorldBoundsOrigin, ScaledRadius))
			return;

		// Screen bounds
		float MaxZ = 0.0;
		float4 SBox;
		bool bValidScreenBounds = GetScreenBounds(CornerVerts, MaxZ, SBox);

		[branch]
		if (bValidScreenBounds)
		{
			// Calculate projected area of the object bounds in screen space
			float4 SBox_VP = SBox * ScreenSize.xyxy;
			float ProjectedBoundsArea = (SBox_VP.z - SBox_VP.x) * (SBox_VP.w - SBox_VP.y);

			// Projection area cull (catches primitives that weren't handled by distace cull)
			[branch]
			if (ProjectedBoundsArea < PIXEL_CULL_THRESHHOLD)
				return;

			// HZB occlusion test
			[branch]
			if (OcclusionCull(MaxZ, SBox))
				return;

			bEnableWorldPositionOffset = ProjectedBoundsArea > WORLD_POSITION_OFFSET_THRESHHOLD;
		}

		// Passed all culling tests, so append the instance data
		uint writeIndex;
		DrawArgsDstBuffer.InterlockedAdd(4, 1, writeIndex);
		DrawInstanceBuffer[writeIndex] = InstanceBuffer[i];

		// Wind
		DrawInstanceBuffer[writeIndex].WindDirectionAndSpeed = SimulateWind(WorldBoundsOrigin.xyz);
	}
}

```

The following snippet is used to get screen bounds, and perfrom HZB occlusion culling. The Z-Tests are reversed because we are using a reverse Z floating point depth buffer.

``` glsl
bool GetScreenBounds(float3 CornerVerts[8], out float MaxZ, out float4 SBox)
{
	// Screen rect from bounds
	float3 RectMin = float3(10000, 10000, 10000);
	float3 RectMax = float3(-10000, -10000, -10000);
	UNROLL for (int i = 0; i < 8; i++)
	{
		float4 PointClip = mul(ViewProjectionMatrix, float4(CornerVerts[i], 1.0));

		/* Behind camera check. Bail out and trivially accept if any of the bounding box verts
		are behind the camera as in that case you might end up with a projected area that is negative. This case usually happens when large meshes are viewed very close up, which would most likely pass culling anyway! */
		[branch]
		if (PointClip.w < 0)
			return false;

		float3 PointScreen = PointClip.xyz / PointClip.w;

		RectMin = min(RectMin, PointScreen);
		RectMax = max(RectMax, PointScreen);
	}

	MaxZ = RectMax.z;
	SBox = saturate(float4(RectMin.xy, RectMax.xy) * float2(0.5, -0.5).xyxy + 0.5).xwzy;

	return true;
}
```

``` glsl
bool OcclusionCull(float MaxZ, float4 SBox)
{
	// Calculate HZB level.
	float4 SBox_HZB = SBox * HZBSize.xyxy;
	float2 Size = (SBox_HZB.zw - SBox_HZB.xy) * 0.5;
	uint Level = ceil(log2(max(Size.x, Size.y)));

	// If the projected bounds is greater than the lowest res HZB, trivially accept
	if (Level >= NumHZBMips)
		return false;

	// Sample 4x4 from HiZ
	float2 Scale = (SBox.zw - SBox.xy) / 3;
	float2 Bias = SBox.xy;

	float4 MinDepth = 1.f;
	UNROLL for (int i = 0; i < 4; i++)
	{
		float4 Depth;
		Depth.x = HZBTexture.SampleLevel(HZBSampler, float2(i, 0) * Scale + Bias, Level).r;
		Depth.y = HZBTexture.SampleLevel(HZBSampler, float2(i, 1) * Scale + Bias, Level).r;
		Depth.z = HZBTexture.SampleLevel(HZBSampler, float2(i, 2) * Scale + Bias, Level).r;
		Depth.w = HZBTexture.SampleLevel(HZBSampler, float2(i, 3) * Scale + Bias, Level).r;
		MinDepth = min(MinDepth, Depth);
	}
	MinDepth.x = min(min(MinDepth.x, MinDepth.y), min(MinDepth.z, MinDepth.w));

	return MaxZ < MinDepth.x;
}

```



