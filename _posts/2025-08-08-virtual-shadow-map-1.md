---
title: 'Virtual Shadow Map - 1'
date: 2025-08-08
permalink: /posts/2025/08/virtual-shadow-map-1/
tags:
  - UE5
  - Virtual Shadow Map
  - Computer Graphics
---

# prior knowledge
- Shadow mapping
- Cascade shadow mapping
- Deferred Shading

# Basic Idea
What makes shadows look bad?  
There are a lot of factors, but shadow aliasing is probably the worst offender.  

Shadow aliasing mainly occurs due to the low resolution of the shadow map.  
There are several techniques to address this, such as cascaded shadow maps, percentage closer filtering, and variance shadow maps.  
Each solution has its own pros and cons, but most of them don't fully eliminate aliasing in large scenes.  

Epic Games introduced a new shadow mapping technique called "Virtual Shadow Maps" (VSM).  
The basic idea is simple: use a 16K shadow map texture!  

![https://zhuanlan.zhihu.com/p/489550318](/images/2025-08-08-virtual-shadow-map/1k_16k_texture.webp)

A 16K texture in D32 format takes up 1 GB ($$2^{14} * 2^{14} * 4$$ bytes).  
For reference, the RTX 3080 Ti has 12GB of VRAM, which means even a high-end GPU like this can only store about 12 such 16K textures.  
To address this problem, the concept of "virtual" comes into play.  

# Virtual Memory
First, let's talk about virtual memory.  
If you studied computer science, you're probably already familiar with this concept.  
Virtual memory lets programs use more memory than what's physically installed in the system.  

What makes it possible is virtual allocation.  
The system assigns virtual addresses from a large address space, and only loads the data that's actually being used into physical memory.  
If data that's not currently in physical memory is requested, the operating system loads it from disk. This triggers a page fault, which can slow down the system.  
Because of the OS, applications can use virtual memory as though they have much more RAM than is actually installed.  

That’s a quick summary of how virtual memory works.  

# Getting back to VSM
VSM has a similar mechanism. But it works with different hardware - specifically, VRAM.  
Also, unlike virtual memory in the OS, "page fault" doesn’t happen at the moment data is requested.  
Instead, VSM checks and loads the required data before rendering the shadow map.  

How can a program predict which part of shadow map needed be rendered?  
in deferred shading, Shadow map is used in during the lighting pass to determine a pixel is occluded or not.  
This lighting pass is typically dispatched as a compute shader with a (width × height) thread group, matching the screen resolution.

So, we only need to calculate (width × height) pixels, which means we'll be accessing the same number of pixels in the shadow map.  
If we have a depth buffer, we can reconstruct the world position for each pixel. Using this information, we can predict which parts of the shadow map each pixel will affect.

Here’s a brief overview of the process:
- Render the depth buffer.
- At screen resolution, determine which parts of the shadow map each pixel will affect.
- Render only the necessary parts of the virtual shadow map.
- Perform the lighting pass at screen resolution.