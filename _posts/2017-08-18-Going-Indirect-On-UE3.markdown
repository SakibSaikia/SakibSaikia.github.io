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



