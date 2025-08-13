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
<br/><br/>

# Basic Idea
What makes shadows look bad?  
There are a lot of factors, but shadow aliasing is probably the worst offender.  

Shadow aliasing mainly occurs due to the low resolution of the shadow map.  
There are several techniques to address this, such as cascaded shadow maps, percentage closer filtering, and variance shadow maps.  
Each solution has its own pros and cons, but most of them don't fully eliminate aliasing in large scenes.  

Epic Games introduced a new shadow mapping technique called "Virtual Shadow Maps" (VSM).  
The basic idea is simple: use a 16K shadow map texture!  

![1k vs 16k](/images/2025-08-08-virtual-shadow-map/1k_16k_texture.webp)
*reference : https://zhuanlan.zhihu.com/p/489550318*
<br/><br/>

A 16K texture in D32 format takes up 1 GB ($$2^{14} * 2^{14} * 4$$ bytes).  
For reference, the RTX 3080 Ti has 12GB of VRAM, which means even a high-end GPU like this can only store about 12 such 16K textures.  
To address this problem, the concept of "virtual" comes into play.  
<br/><br/>

# Virtual Memory
First, let's talk about virtual memory.  
If you studied computer science, you're probably already familiar with this concept.  
Virtual memory lets programs use more memory than what's physically installed in the system.  

What makes it possible is virtual allocation.  
The system assigns virtual addresses from a large address space, and only loads the data that's actually being used into physical memory.  
If data that's not currently in physical memory is requested, the operating system loads it from disk. This triggers a page fault, which can slow down the system.  
Because of the OS, applications can use virtual memory as though they have much more RAM than is actually installed.  

That’s a quick summary of how virtual memory works.  
<br/><br/>

# Getting back to VSM
VSM has a similar mechanism. But it works with different hardware - specifically, VRAM.  
Also, unlike virtual memory in the OS, "page fault" doesn’t happen at the moment data is requested.  
Instead, VSM checks and loads the required data before rendering the shadow map.  

How can a program predict which part of shadow map needed be rendered?  
in deferred shading, Shadow map is used in during the lighting pass to determine a pixel is occluded or not.  
This lighting pass is typically dispatched as a compute shader with a (width × height) thread group, matching the screen resolution.

So, we only need to calculate (width × height) pixels, which means we'll be accessing the same number of pixels in the shadow map.  
If we have a depth buffer, we can reconstruct the world position for each pixel. Using this information, we can predict which parts of the shadow map each pixel will affect.
<br/><br/>

Here’s a brief overview of the process:
- Render the depth buffer.
- At screen resolution (with depth buffer), determine which parts of the shadow map each pixel will affect.
- Render only the necessary parts of the virtual shadow map.
- Perform the lighting pass at screen resolution.
<br/><br/>

# Implementation Detail
Let's assume we have a directional light and only one 16K shadow map for it, just to keep things simple.
The key concept here is the idea of a "page", which is similar to the concept of pages in virtual memory.

![VSM page](/images/2025-08-08-virtual-shadow-map/VSM_Page.png)
*reference : https://zhuanlan.zhihu.com/p/489550318*
<br/><br/>
The image above shows a 16K shadow map.  
First, the entire shadow map is divided into units called "pages".  
By default, each page is 128 × 128 pixels, so a 16K shadow map contains 128 × 128 pages in total.  
We set a "used" flag on each page to indicate whether it will be accessed during the lighting pass.  

![VSM page mapping](/images/2025-08-08-virtual-shadow-map/VSM_Page1.png)
*reference : https://zhuanlan.zhihu.com/p/489550318*
<br/><br/>

As mentioned earlier, we can determine this by checking the depth buffer.  
The depth stores the depth value for each pixel, which allows us to reconstruct the world position of every pixel.  
![VSM page mapping](/images/2025-08-08-virtual-shadow-map/VSM_Page2.png)
*reference : https://zhuanlan.zhihu.com/p/489550318*
<br/><br/>

Using the world position and the shadow projection matrix, we can calculate the shadow-space position, which tells us exactly which shadow map page each pixel will use.

![VSM page mapping](/images/2025-08-08-virtual-shadow-map/VSM_Page3.png)
*reference : https://zhuanlan.zhihu.com/p/489550318*
<br/><br/>

After setting the flags, we know that we only need to render green pages marked "used".
So, we collect all the green pages and allocate phyiscal memory for them.
This allocated memory is called the "physical page pool.

In a real game, the physical page pool might look like this.
![physical page pool](/images/2025-08-08-virtual-shadow-map/VSM_Page3.png)

if we have a maximum of 1024 pages, we allocate 128 × 8 pages, resulting in a physical page pool with a size of 16,384 × 1,024 pixels.  
(We won't get into the details of how the width and height are determined here.)  
But is 1,024 pages enough?
If the scene is large and complex, with a wide range of depth values across screen pixels, you may need more than 1,024 pages.  
In this case, you might see an error like:  
_"Virtual Shadow Map Page Pool overflow (%d page allocations were not served), this will produce visual artifacts (missing shadows). Increase the page pool limit or reduce the resolution bias to avoid this."_  
When this happens, any pages beyond the limit are not rendered.