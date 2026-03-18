---
layout: post
title: Tiled Light Culling in Vulkan
date: 2026-03-15 09:00:00
description:
tags:
categories:
thumbnail: assets/img/Blog/LightCulling/
featured: false
---

<br>
2026-03-05-LightCulling
During rasterization, the geometry is converted into fragments, and the fragment shader is executed for each fragment. 

When dealing with simple scenes consisting of few light sources, little geometry and basic materials. 
The fragment shader performs lighting calculations. The GPU computes the contribution of each light source in the scene for each fragment. 

For scenes with small number of light sources, this setup is reasonable and interactive frame-rate can be maintained. However, suppose the scene has hundreds of light sources. Traversing all light sources for each fragment will significantly reduce the frame-rate. Hence, the idea of light culling is introduced. 

<br>

The **tiled light culling** algorithm is described as follows: 
<br>
<br>
1. For each light source $$L$$, create a bounding sphere $$B(c, r)$$ such that: 

    - The center $$c$$ of the sphere is the light position. 
    
    - The radius $$r$$ of the sphere is the distance at which the light intensity $$I$$ becomes negligible $$ε$$, assuming the use of the inverse square attentuation of the light intensity 
    <br>
    <br>
    $$
    I_r = \frac{I}{r^2}
    $$
    <br>
    <br>
    That is, objects at distances $$d > r$$ will receive very little light contribution from the light source $$L$$, that we can ignore its light contribution all together and the physical accuracy of the final scene will not be noticeably affected. A reasonable value for $$ε$$ can be $$1 \; \%$$ of $$I$$. This means that 
    <br>
    <br>
    $$
    r = \sqrt{\frac{I}{0.01 * I}} = \sqrt{\frac{1}{0.01}} = \sqrt{100} = 10
    $$

2. Dissect the screen into tiles with equal resolution.

3. For each tile, compute its frustum.

3. In a seperate pass before the shading pass, iterate through all light sources. For each light sources $$L$$, test for intersection between 

    - Test the intersection between the light bounding sphere $$B$$ and each tile frustum. 
    <br>
    <br>
    - If the light bounding sphere intersects the tile frustum, record the ID of the light into the per-tile light list. 
    <br>
    <br>
4. In the main shading pass: 

    - Calculate the tile index in which the fragment/pixel belongs to.
    
    - Calculate the light contribution that the fragment receives by only going through the list of lights of that tile. 

<br>

***

<br>
#### **References** <br>
<br>

**[1]** 

<br>
