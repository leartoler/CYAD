---
title: "Simulated worlds are like onions"
source: "https://medium.com/@haversine02/simulated-worlds-are-like-onions-dfa18cab3d85"
author:
  - "[[Madis Kase]]"
published: 2019-11-05
created: 2026-03-27
description: "Simulated worlds are like onions Breaking down Biogrid’s GPU-accelerated world simulation A classic programming axiom: your code should only do as much work as absolutely necessary. A good mindset …"
tags:
  - "clippings"
---
[Sitemap](https://medium.com/sitemap/sitemap.xml)

*Breaking down* [*Biogrid*](https://twitter.com/BioGridGame) *’s GPU-accelerated world simulation*

A classic programming axiom: your code should only do as much work as absolutely necessary. A good mindset to follow in any computing-related field, but I think it’s especially relevant in games. There’s a scant 16ms available to you per frame and you should spend it wisely. Updating or rendering entities that are static, not visible or outside player’s interaction range is generally something to be avoided.

There’s a myriad of techniques for this type of optimization and most of them impose some limits on how dynamic your world can be. There’s a \*lot\* of upsides to knowing what’s static in your world and what’s not—most everything you need can be precomputed for static objects and thus optimized.

Those techniques are a great fit for many games — most of the game worlds really are static at their core, either for performance or budget reasons, or by the simple fact that dynamic worlds are harder to balance, bug-prone and well…unnecessary for gameplay in most genres.

But…what if you want to do everything backwards and disregard all these useful limitations and precomputation steps? What if your entire world is constantly changing and interacting with itself, every entity is equally important and the player can zoom from ground level to space in the blink of an eye?

In this post we’ll take a closer look at what it takes to simulate a fully dynamic world in [Biogrid](https://twitter.com/BioGridGame).

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*q7SYeNwTBPhdutpftdJKkw.gif)

Dynamic terrain, flowing water and atmosphere simulation

## GPU or CPU?

Biogrid simulates its world on the GPU, using DX11 compute shaders. But it wasn’t always so, the original prototype was purely CPU-based, and the switch to GPU happened only once the upsides started to outweigh the heavy penalties. The decision to simulate something on GPU should be carefully considered. While GPUs are absolute monsters at rendering and crunching through well-prepared problems, they also have several limitations.

The biggest limitation for me is the slow data transfer between CPU and GPU. Sending and receiving data from GPU can be slow enough to eat up all the gains from faster computation on GPU. Rule of thumb — you should \*keep\* your data on the GPU once it’s there.

That’s also why most of the GPU-accelerated computation work in games is purely visual with no gameplay implications. Generally you’re rendering swarms of particles or strands of hair, culling your renderers or doing post-effects, but rarely something that needs to be transferred back to CPU.

(Of course there’s also the madlad option of just moving your whole game logic to the GPU and sidestepping the data transfer issue, for an example check out [Jelly in the Sky](https://store.steampowered.com/app/593530/Jelly_in_the_sky/) )

Biogrid does sync data between CPU and GPU, but only what’s absolutely necessary, as infrequently as possible. The reason why it’s cost-effective for me comes down to the sheer scale of computations performed, as we’ll discuss below. So, to rephrase the rule — If you absolutely need to read back data from GPU, do it, but make sure the results are lightweight, and your problems tough enough to be worth the hassle.

## What’s in a kernel?

To utilize the GPU effectively, you should know what they’re good at. They’re \*very\* good at parallel processing. To me, that comes down to thread count. A CPU has what, 4–16 threads? And that’s about the limit of how much simultaneous work you can do. Each thread can work on their own data chunk in parallel, but you’re still processing only a few data points at a time.

GPUs on the other hand are used to blast a massive amount of data on your screen as quickly as possible and it’s no problem to schedule let’s say, 100 million of threads. Every thread gets their own single data point from the set and they all go through it at once, no iteration or chunking required. (Well, only about 10k of them might be simultaneously active at a time, but the difference in scale is still massive and GPUs are smart about scheduling their work.)

So, GPUs are fast, but they’re also sort of picky in what sort of work they’re willing to do. All the threads are identical and run the same function on the input data — a compute kernel. Input data should then also be uniformly structured. For example an array of RGBA pixels, represented as *float4*. Almost all the data in Biogrid is arranged using similar layout — a stack of 2D layers, represented on the GPU as [*RWStructuredBuffers*](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/sm5-object-rwstructuredbuffer) or [*RWTexture2D*](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/sm5-object-rwtexture2d).

Here’s an example of a simple compute kernel, this one blurs the 1-channel temperature map that’s used in atmosphere simulation:

```c
uniform RWTexture2D<float> groundTemperatureRTRead;
uniform RWTexture2D<float> groundTemperatureRTWrite;[numthreads(BLOCKSIZE, BLOCKSIZE, 1)]
void DiffuseTemperature(int3 threadID : SV_GroupThreadID, int3 dispatchID : SV_DispatchThreadID, uint groupID : SV_GroupIndex)
{
 const uint2 uv = dispatchID.xy;
 
 float center, left, right, top, bottom;
 
 center = groundTemperatureRTRead[uv];
 left = groundTemperatureRTRead[uv + uint2(-1, 0)];
 right = groundTemperatureRTRead[uv + uint2(1, 0)];
 top = groundTemperatureRTRead[uv + uint2(0, 1)];
 bottom = groundTemperatureRTRead[uv + uint2(0, -1)];
 
 float average = (left + right + top + bottom + center) / 5.0f;
 groundTemperatureRTWrite[uv] = average;
}
```

> As a side note:
> 
> This example illustrates an important concept in GPU programming — ping-ponging. There’s two copies of the same temperature map, one’s purely for reading, the other’s for writing the output.
> 
> As the individual threads are all running asynchronously, we don’t know how far they’ve progressed. So we might end up reading neighbouring values too late — a neighbouring thread could have already finished and overwritten the original value. These errors tend to compound and mess up your results.
> 
> If we keep the input data separate from output, we can read from anywhere we like without issues. After the computation is done, we can just flip the data buffers — output becomes the input for the next iteration.
> 
> Note: this technique is needed only if the kernel’s output depends on neighbouring values, and there might be a thread writing to them. Still, it’s a common enough problem and worth mentioning.

The example kernel above might seem familiar if you’ve used Unity’s [Job system](https://docs.unity3d.com/2018.3/Documentation/Manual/JobSystem.html) for multithreaded data processing. The basic idea is the same — if you’ve got a lot of very similarly structured simple data, you can process it faster using multiple threads, all running the same kernel.

That’s a further point to consider when deciding whether to use CPU or GPU for your data processing — you might not need to slowly shuttle your data to the GPU if you could use the Jobs system with Burst compiler on the CPU side.

Still, for Biogrid’s needs, Burst is not enough. In its simulation phase, Biogrid runs 34 different compute kernels, up to 1000 times per second. All that adds up very quickly. To its credit, Burst can run the simulation step just fine at 1x speed, but that’s about it — there’s no CPU time left for entities or other game logic. By offloading the computation to GPU, the game supports 10x fast-forward at 60FPS and leaves the CPU nearly idle for future use.

## Stacking Layers

Let’s start digging into the data buffers that make up the world, and build up the complete world representation, one layer at a time. I’ve omitted some of the internal or less important buffers for clarity’s sake.

### Terrain Data

The most important buffer of them all. It contains the amounts of 4 material layers that the game currently supports — bedrock, sand, soil and water. Those layers are assumed to be lying on top of each other in fixed order and are summed together to determine the final height of the terrain.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*rzAOMy9p8FNs8ZKusxgF8w.jpeg)

That’s the only buffer that’s regularly transferred between GPU and CPU. It’s synchronized infrequently, a slight lag between GPU and CPU representations is not noticeable. For CPU->GPU modifications I keep a difference/delta buffer. Everything that’s simulated on CPU can write the requested terrain changes to the delta buffer and all the pending changes are then uploaded at once when the CPU->GPU synchronization happens.

The tweet below shows the delta map usage in action. Boulders are simulated on the CPU using the terrain data as a collision heightmap. They also erode the underlying terrain while rolling, which is written to the delta map, and eventually merged with the GPU’s representation. Delta buffer modification is parallelized with Jobs and Burst, so it’s performant enough to support constant terrain modification by thousands of entities if needed.

### Surface Water

Flowing water uses 2 internal buffers for computation and outputs a water velocity map that’s used to render [flowmaps](http://graphicsrunner.blogspot.com/2010/08/water-using-flow-maps.html) and foam. Water is simulated using the [Virtual Pipes](https://tutcris.tut.fi/portal/files/4312220/kellomaki_1354.pdf) method. The linked paper is very through and readable if you’re interested, so I won’t be covering that one in more detail right now.

## Get Madis Kase’s stories in your inbox

Join Medium for free to get updates from this writer.

Water simulation is independent from the CPU and is not synchronized with it. Having floods that wash physical objects away would be an interesting feature, but would require regularly transferring the velocity data to the CPU.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*MMW-0d3YMjA91ooE6KTytA.jpeg)

Velocity map for the water, encoding direction and intensity of the flow

### Erosion and Sediment

Various buffers are involved in this, a dissolved sediment buffer is pictured below. Erosion depends strongly on the terrain data and water flow buffers. This feature’s also based on a [paper](http://www-ljk.imag.fr/Publications/Basilic/com.lmc.publi.PUBLI_Inproceedings@117681e94b6_fff75c/FastErosion_PG07.pdf), you can even find some [Unity projects](https://github.com/bshishov/UnityTerrainErosionGPU) floating around, if you’d like to take a look. Most of them seem to be based on the same principles and sources that I’ve been using.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Fftb-7nvIoxXUaGPssROZw.jpeg)

In general, hydraulic erosion is simulated by water flows weathering away small amounts of material, carrying it downstream and depositing it as sediment if the water’s oversaturated or moving slowly. Here’s where the separate material layers come into play - it’s possible to specify different hardness, solubility and other parameters per-layer. The results can be pretty lifelike, running water can wash away softer materials, eroding and depositing them naturally.

Every supported material type also has a specific [angle of repose](https://en.wikipedia.org/wiki/Angle_of_repose). Cliffs can support themselves as almost vertical walls, while loose sandpiles will slide and flow if they’re steeper than about 45°:

### Groundwater

Water in soil is also simulated using the virtual pipes method. There are some additional terms representing the slow diffusion of water in soil and varying zones depending on depth and material properties. Permeable materials like sand and soil have specific absorption and diffusion rates. Eventually water diffuses downwards until it reaches the bedrock. If there’s nowhere for the water table to drain, it can rise and oversaturate the soil. Very high, surface level water tables will end up seeping water from the soil and form swamps and springs.

Groundwater is an important damping and diffusing factor in the water cycle. It captures runoff from rivers and rain and forms a reservoir of water that’s slowly diffused and released over a large area. Without it, plants in Biogrid could survive only on shores or under constant rainfall — surface water tends to flow off quickly if not absorbed.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*fz0kAY6VnMEMS7FoCBkFWA.jpeg)

### Temperature

Part of the atmosphere’s fluid solver, temperature map is used to simulate the convective wind flows that transport evaporated water around the world. The average heat input, representing the sun’s thermal heating, is dependent on the elevation.

In practise the temperature map is constantly varying, due to external factors like cloud cover and rain, and also due to the the airflow itself. More precisely, the temperature map is advected by the flow field. There’s an example of advection [here](https://www.shadertoy.com/view/MllBzr) — the dye in the simulation is carried along by the underlying flowfield.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*EdDLMYKziR1QvPhmx_sBMw.jpeg)

### Wind

Wind is essentially taken straight from the state buffers of the atmosphere’s fluid solver. This is used as a velocity field by the evaporated water particles.

The solver itself is a fairly standard Navier-Stokes fluid simulation on an Eulerian grid. There are some useful modifications and optimizations in my implementation though. I’m using the fluid solver mainly for computing a rough velocity field for the particles, not for accurately advecting some data through the grid itself. So, I don’t have to care all that much about the correctness of the solver itself. Specifically, I can skip most of the time-consuming Jacobi pressure-solving steps.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*PWzPhNnSkIV6huAHuoWoGQ.jpeg)

### Water particles and clouds

Evaporated water particles are slightly unique compared to all the entirely grid-based methods, as they’re simulated with [Lagrangian particle tracking](https://en.wikipedia.org/wiki/Lagrangian_particle_tracking). They use temperature and velocity data from the previously described grid-based fluid solver step but are essentially treated as free-flying particles in 3D space.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*74vLx4zk7HgV09kZFLfyPA.jpeg)

Debug view showing water particles with varying density(size) and temperature(brightness). Glowing white signifies rain.

This brings some more challenges. For instance, writing to rain or cloud density buffers requires [atomic operations](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/interlockedadd) on the GPU — lots of particles may be attempting to write to the same grid cell simultaneously, which causes race conditions and data loss.

On the other hand, this method is entirely mass-preserving and allows interesting features like truly 3D cloud layers and collisions with terrain. Collision detection becomes rather simple, in fact — particles can just read the terrain data buffer under them and compare its height with their own.

### Cloud density

The (currently) last buffer of the stack. This is generated by the water vapor particles writing their position and size to a buffer. This buffer is then blurred and used for determining cloud coverage. On the rendering side it’s got multiple uses — it’s used for rendering normal maps for the clouds, clipping and cloud shadows on the terrain.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*nhFdYnaYh5imnRjUZK2qLw.jpeg)

### Conclusion

And there it is— a stack of kernels and buffers, making up a world. Many of them interact and feed into each other which makes balancing the whole stack an art unto itself. As an example, increasing the amount of temperature that’s added to the temperature map increases the evaporation, changes wind speed and wind patterns, which end up changing the rainfall distribution, which changes the erosion and river patterns, which changes the sediment transport…you get the idea.

What I find interesting though, is that all those interlocking systems, while complex, can actually be intuitively understood. I expect it to take some time for new players, but tinkering with your world and breaking it in multiple ways is also fun in its own way.

Unfortunately this article is already getting a bit too long, so I can’t cover the logic in the kernels themselves or the rendering side of things (which involves another stack similar to simulation).

I could cover either one of these in the next article though, let me know in the comments or on [Twitter](https://twitter.com/BioGridGame) if you have a preference or just some comments.

## Thanks for reading!

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*jl-2IA66RnO0_iNT73W-0A.jpeg)

[![Madis Kase](https://miro.medium.com/v2/resize:fill:96:96/0*hncY90AZ7H5T5TVS)](https://medium.com/@haversine02?source=post_page---post_author_info--dfa18cab3d85---------------------------------------)

[![Madis Kase](https://miro.medium.com/v2/resize:fill:128:128/0*hncY90AZ7H5T5TVS)](https://medium.com/@haversine02?source=post_page---post_author_info--dfa18cab3d85---------------------------------------)

[1 following](https://medium.com/@haversine02/following?source=post_page---post_author_info--dfa18cab3d85---------------------------------------)

## Responses (1)

Write a response[What are your thoughts?](https://medium.com/m/signin?operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40haversine02%2Fsimulated-worlds-are-like-onions-dfa18cab3d85&source=---post_responses--dfa18cab3d85---------------------respond_sidebar------------------)

```c
Why not do the physics simulation of thing like boulders in the GPU as well instead of the CPU?
```

\--