---
layout: 	post
title:  	"A Token Based Approach To Ordering Transitions"
date:   	2021-09-15 23:09:46 -0500
category: 	"Graphics"
published:	true
---

Low level graphics APIs brought the promise of multi-threaded command recording and issue, however one of the biggest pain points of doing that effectively is dealing with resource transitions - specifically the need to specify a [BeforeState](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_transition_barrier) when transitioning a resource. 

A common case is that you have a resource that needs be transitioned in command lists that will be recorded in different threads. A couple of approaches of handling this are:

1. Hardcoding the resource states. This requires intimate knowledge of your render passes and is also somewhat rigid in the sense that if certain render passes are changed, the transitions need to be fixed up.
2. Using a [framegraph](https://www.gdcvault.com/play/1024612/FrameGraph-Extensible-Rendering-Architecture-in) approach that figures out all the dependencies ahead of time.

This article outlines an approach to solving this problem that is more flexible than hardcoding but more light-weight than a full fledged framegraph. The basic idea is:
1. Cache the current resource state within the resource object. 
- The transition call specifies the DestinationState only. 
- The cached resource state provides the BeforeState.
2. Before kicking off a render pass in another thread or job, we obtain a token for every resource that will be transitioned within that pass.
3. A later transition call must specify the token value obtained in step 2.
- Transition calls are executed in order of the token value by waiting on a fence.
- The cached resource state is updated once the transition call finishes. 
- The transition is completed by signalling the fence which notifies that it is the turn of the next token in line.

The following example code should make this clear. These will focus on D3D12 but the same concepts should apply to Vulkan as well.

This is how a resource class looks. It has a cached state and a synchronization fence. The transition fence value keeps account of the tokens, and the process of obtaining a token is simply getting an incremented fence value.

```C++
struct FResource
{
	ID3D12Resource* m_d3dResource;
	D3D12_RESOURCE_STATES m_cachedState;
	ID3D12Fence* m_transitionFence;
	size_t m_transitionFenceValue;

	size_t GetTransitionToken() { return ++m_transitionFenceValue; }
	void Transition(FCommandList* cmdList, const size_t token, const D3D12_RESOURCE_STATES destState);
};
```

The code below shows an example of kicking off a render job. I'm using PPL tasks here. The transition token obtained synchronously on the main/dispatcher thread is captured by value and passed onto the render job, which then uses that token when performing the transition.

```C++
size_t token = rt->GetTransitionToken();
concurrency::create_task([=]
{
	FCommandList* cmdList = RenderBackend12::FetchCommandlist(D3D12_COMMAND_LIST_TYPE_DIRECT);
	rt->Transition(cmdList, token, D3D12_RESOURCE_STATE_RENDER_TARGET);
	return cmdList;
});
```

And, this is what the implentation of the transition call looks like with the token-based wait.

```C++
void FResource::Transition(FCommandList* cmdList, const size_t token, const D3D12_RESOURCE_STATES destState)
{
	// The expected value for a transition to process is that the completed value is 1 less than tokenValue.
	// If the value difference is more than 1, it means that some other CL has reserved the right to transition
	// this resource first, and we must wait!
	const size_t completedFenceValue = m_transitionFence->GetCompletedValue();
	const size_t wait = token > 0 ? token - 1 : 0;
	if (completedFenceValue < wait)
	{
		SCOPED_CPU_EVENT("transition_wait", PIX_COLOR_DEFAULT);
		HANDLE event = CreateEventEx(nullptr, nullptr, 0, EVENT_ALL_ACCESS);
		if (event)
		{
			m_transitionFence->SetEventOnCompletion(wait, event);
			WaitForSingleObject(event, INFINITE);
		}
	}

	// Make sure multiple threads do not access the following section at the same time since the before state is shared data
	static std::mutex transitionMutex;
	std::lock_guard<std::mutex> scopeLock{ transitionMutex };

	// Handle case where we are requesting a transition to the same state that the resource is already at
	if (m_cachedState == destState)
	{
		m_transitionFence->Signal(token);
		return;
	}

	// Actual API call for the transition
	D3D12_RESOURCE_BARRIER barrierDesc = {};
	barrierDesc.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
	barrierDesc.Transition.pResource = m_d3dResource;
	barrierDesc.Transition.StateBefore = beforeState;
	barrierDesc.Transition.StateAfter = destState;
	barrierDesc.Transition.Subresource = subresourceIndex;
	cmdList->m_d3dCmdList->ResourceBarrier(1, &barrierDesc);
	

	// Update the CPU-side tracking
	m_cachedState = destState;

	// Signal that it is the turn of the next token
	m_transitionFence->Signal(token);
}
```

The code above only shows transitions at a resource-level, but subresource transitions can be achieved with minor modifications.

Here it is in action with the transition waits highlighted. All the render jobs are kicked off at around the same time.


[<img src="/images/token-based-transitions/transition-wait.png">](/images/token-based-transitions/transition-wait.png)

Now, you may be wondering .. _What if I don't want to wait for the previous transitions to be recorded?_ And you are right, this can indeed cause a lot of these render jobs to effectively be stuck on a waiting state. For starters, you should delay transitions as late as you can, which is a good idea anyway. Additionally, you could cache the token value to render state mapping per render pass and just reuse those as long as they aren't invalidated. The token generation would need to restart every frame instead of being strictly monotonically increasing. This would get you closer to a framegraph, a framegraph-lite if you will.

As a bonus, I also use this to implement fire-and-forget type render jobs that automatically submit the command lists after they have finished recording. The submission order is enforced using a render token in this case.

```C++
size_t renderToken = jobSync.GetToken();

return concurrency::create_task([=]
{
	FCommandList* cmdList = RenderBackend12::FetchCommandlist(D3D12_COMMAND_LIST_TYPE_DIRECT);
	
	// Record some commands

	return cmdList;

}).then([&, renderToken](FCommandList* recordedCl) mutable
{
	// Wait for turn and submit the commandlist
	jobSync.Execute(renderToken, recordedCl);
});

```

Image below shows the corresponding waits and ExecuteCommandLists calls.

[<img src="/images/token-based-transitions/submission-wait.png">](/images/token-based-transitions/submission-wait.png)
