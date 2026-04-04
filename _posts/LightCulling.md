---
layout: post
title: Tiled Light Culling in Vulkan
date: 2026-03-29 09:00:00
description:
tags:
categories:
thumbnail: assets/img/Blog/LightCulling/
featured: false
---

<br>
### **Introduction**
<br>

During rasterization, the triangles in the scene are converted into fragments, and the fragment shader is executed for each fragment. The fragment shader adds up the lighting contribution from each light source in the scene, and computes the final color of the fragment. 

<figure class="col-md-12 text-center theme-img repo-img-light">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Dark/1.png" class="scaled-img70" %}
</figure>
<figure class="col-md-12 text-center theme-img repo-img-dark">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Light/1.png" class="scaled-img70" %}
</figure>

This is the typical setup that exists in most real-time graphics applications. It can achieve interactive frame-rate for simple scenes with a few light sources. However, it does not scale well with the increase in the number of light sources in the scene. That is, it will fail to maintain adequate frame-rate with more complex scenes with hundreds or thousands of light sources. Hence, the idea of light culling was introduced. 

Light culling is an optimization technique in rendering. The idea of light culling is that at certain great distances from the light source, the light contribution from that light source becomes so small, because of attentuation, that ignoring the light source completely in the fragment shading computation will not affect the physical accuracy of the final scene by a great deal, and, in return, save us significant computational cost.

The model that is most commonly used to account for the attentuation of the light sources in computer graphics is the **inverse square law**. This model states that the light intensity $$I$$ is inversely proportional to the square of the distance $$d$$ from the light source. That is, the intensity of the light source at distance $$d$$; call it $$I_d$$, is

$$
I_d = \frac{I}{d^2}
$$

<br>
### **Tiled Light Culling Algorithm**
<br>

The **tiled light culling** algorithm is described as follows: 
<br>
<br>
1. For each light source $$L_k$$, create a bounding sphere $$B_k$$ such that: 

    - The center $$c_k$$ of the sphere is the light position. 
    
    - The radius $$r_k$$ of the sphere is the distance at which the maximum color component of the light intensity $$I_k$$ equals a user-defined scalar value, let's call it $$I_{min}$$, assuming the use of the **inverse square attentuation** of the light intensity $$I_k$$. That is, 

    $$
    r = \sqrt{\frac{max \; (I.r, \; I.g, \; I.b)}{I_{min}}}
    $$

2. Dissect the screen into tiles $$t_{ij}$$ with equal resolution $$R \times R$$.

3. For each tile $$t_{ij}$$, compute its frustum $$F_{ij}$$.

4. For each tile $$t_{ij}$$, initialize the list $$V_{ij}$$. This list will contain the IDs of the light sources that are inside the frustum $$F_{ij}$$.

5. Iterate through all the light sources $$L_k$$ and test the intersection between the light bounding sphere $$B_{k}$$ and each tile frustum $$F_{ij}$$. If $$B_{k}$$ intersects or is inside the frustum $$F_{ij}$$, record the ID of the light source $$L_k$$ into the list $$V_{ij}$$.

6. In the fragment shader of the main shading pass: 

    - Locate the tile $$t_{ij}$$ that the fragment resides in. This is computed as follows: 
    
    $$
    t_{ij} = \frac{gl\_FragCoord.xy}{R}
    $$
        
    - Compute the lighting of the fragment by only considering the light sources that are inside the list $$V_{ij}$$.

<br>
### **Light Bounding Sphere**
<br>

Below is the code for the function that computes the bounding sphere for a light source: 

```c++
glm::vec4 ComputeBoundingSphere(glm::vec3 position, glm::vec3 color, float minIntensity)
{
    float radius = sqrt(max{color.r, color.g, color.b} / minIntensity);
    
    return glm::vec4(position, radius);
}
```

The data structure used to represent the bounding sphere is **vec4**. The **XYZ** component of the vector will contain the center of the sphere and the **W** component will contain the radius of the sphere. 

<figure class="col-md-12 text-center theme-img repo-img-light">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Dark/vec4.png" class="scaled-img50" %}
</figure>
<figure class="col-md-12 text-center theme-img repo-img-dark">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Light/vec4.png" class="scaled-img50" %}
</figure>

<br>
### **Depth Prepass**
<br>


```cpp
#version 450

layout(location = 0) in vec3 vertex;

layout(binding  = 0) uniform VertexShaderUniformVariables{
    mat4 cameraViewMatrix;
    mat4 cameraProjectionMatrix;
} vs;

void main()
{
    mat4 modelMatrix = mat4(1.0f);
    gl_Position = vs.cameraProjectionMatrix * vs.cameraViewMatrix * modelMatrix *  vec4(vertex, 1.0);
}
```



<br>
### **Tile Frustum**
<br>

A frustum consists of $$6$$ planes: **top**, **bottom**, **right**, **left**, **near**, and **far**. 

<figure class="col-md-12 text-center theme-img repo-img-light">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Dark/frustum.png" class="scaled-img40" %}
</figure>
<figure class="col-md-12 text-center theme-img repo-img-dark">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Light/frustum.png" class="scaled-img40" %}
</figure>

To be able to check for intersection between the bounding sphere of the light source and the tile frustum, we first need to construct the frustum of the tile. We can compute the top, bottom, right, and left plane using the tile index on the screen and the projection and the view matrix as follows: 

However, for the near and far plane of the tile frustum, we cannot use the near and far clipping plane of the camera. It can work. However, it would be too loose. For better performance, we should instead use the minimum depth and maximum depth of all pixels inside the tile as the near and far plane of the frustum. For better performance, we should compute the minimum and maximum depth in each tile using a compute shader, in a seperate pass before the light culling pass. 

Below is the code for the function that computes the frustum for each tile: 

```cpp
std::vector<glm::vec4> ExtractTilesFrustumPlanes(glm::uvec2 numTiles2D, glm::mat4 cameraProjectionMatrix)
{
    std::vector<glm::vec4> tilesFrustumPlanes(numTiles2D.x * numTiles2D.y * 4);
        
    glm::vec4 row1 = glm::vec4(cameraProjectionMatrix[0][0], cameraProjectionMatrix[1][0], cameraProjectionMatrix[2][0], cameraProjectionMatrix[3][0]);
    glm::vec4 row2 = glm::vec4(cameraProjectionMatrix[0][1], cameraProjectionMatrix[1][1], cameraProjectionMatrix[2][1], cameraProjectionMatrix[3][1]);
    glm::vec4 row4 = glm::vec4(cameraProjectionMatrix[0][3], cameraProjectionMatrix[1][3], cameraProjectionMatrix[2][3], cameraProjectionMatrix[3][3]);
        
    float stepX = 2.0f / (float)numTiles2D.x;
    float stepY = 2.0f / (float)numTiles2D.y;
        
    for(int x = 0; x < numTiles2D.x; x++){
        float offsetX = (float)x * stepX;
        float xMin = -1.0f + offsetX;
        float xMax = xMin + stepX;
        for(int y = 0; y < numTiles2D.y; y++){
            std::array<glm::vec4, 4> tileFrustumPlanes;
            tileFrustumPlanes[0] = row1 - (row4 * xMin);
            tileFrustumPlanes[1] = (xMax * row4) - row1;
                
            float offsetY = (float)y * stepY;
            float yMin = -1.0f + offsetY;
            float yMax = yMin + stepY;
            tileFrustumPlanes[2] = row2 - (row4 * yMin);
            tileFrustumPlanes[3] = (yMax * row4) - row2;
                
            for(int i = 0; i < 4; i++)
                tileFrustumPlanes[i] /= glm::length(glm::vec3(tileFrustumPlanes[i]));

            int baseIndex = (y * numTiles2D.x * 4) + (x * 4);
            for(int i = 0; i < 4; i++)
                tilesFrustumPlanes[baseIndex + i] = tileFrustumPlanes[i];
        }
    }
        
    return tilesFrustumPlanes;
}
```

<br>

<br>
### **Tile's Minimum and Maximum Depth**
<br>

We can compute the minimum and maximum depth in each tile in **parallel** using the following algorithm: 

<blockquote>
Let \(N\) be the total number of tiles on the screen and \(R \times R\) be the resolution of a tile. Do the following:
<br>
<br>

<ol>
<li>Launch \(N\) workgroups. Each workgroup should have a total of \(R^2\) threads; a grid of \(R \times R\) threads. Each thread is responsible for one pixel in that tile.</li>
<br>
<li> Each thread reads the depth value of the pixel within the tile it is responsible for.</li>
<br>
<li> After all the threads have read the depth values from the depth texture, start a loop. Each iteration of the loop corresponds to one level of the tree. The number of iterations of the loop is \(log_{2}N\). The starting iteration index is \(0\). The iteration index is \(k\). For each iteration </li>
<br>
<ul>
<li> Thread \(i\) will team up with thread \(i + 2^{k}\). </li>
<br>
<li> Thread \(i\) will compare its depth value with the depth value of thread \(i + 2^{k}\). The maximum and minimum of both values will be stored in slot \(i\) of the array \(D\). Thread \(i + 2^{k}\) will retire, and thread \(i\) will continue on to the next level of the tree. </li>
<br>
<li> Enforce a barrier to make sure each pair of threads has finished comparing their depth values and have stored their max and min values in the array \(D\). </li>
</ul>
<br>
<li> After the loop is finished, slot \(0\) in the array \(D\) will contain the minimum and maximum depth of the entire tile. </li>
</ol>

</blockquote>
  
<br>

<figure class="col-md-12 text-center theme-img repo-img-light">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Dark/DepthMinMaxTree.png" class="scaled-img70" %}
<figcaption>Computing minmax depth in parallel using 16 threads; or a grid of \(4 \times 4\) threads</figcaption>
</figure>
<figure class="col-md-12 text-center theme-img repo-img-dark">
    {% include figure.html loading="lazy" path="assets/img/Blog/LightCulling/Light/DepthMinMaxTree.png" class="scaled-img70" %}
<figcaption>Computing minmax depth in parallel using 16 threads; or a grid of \(4 \times 4\) threads</figcaption>
</figure>

<br>

Below is the code for the compute shader that computes the minimum and maximum depth for each tile, assuming the resolution of each tile is $$16 \times 16$$, and hence, the use of a grid of $$16 \times 16$$ threads in each workgroup.

<br>

```cpp
#version 450

layout(local_size_x = 16, local_size_y = 16, local_size_z = 1) in;

layout(binding = 0) uniform sampler2D depthTexture;

layout(std140, binding = 1) uniform DepthMinMaxUniformVariables{
    float cameraNearPlane;
    float cameraFarPlane;
    uvec2 numTiles2D;
};

layout(std430, binding = 2) buffer TileDepthMinMax{
    vec2 tileDepthMinMax[];
};

const uvec2 tileResolution = uvec2(16, 16);
const uint numThreadsPerWorkGroup = tileResolution.x * tileResolution.y;

shared vec2 threadDepths[numThreadsPerWorkGroup];

float LinearizeDepth(float depth)
{
    float z = depth * 2.0 - 1.0;
    return (2.0 * cameraNearPlane * cameraFarPlane) /
           (cameraFarPlane + cameraNearPlane - z * (cameraFarPlane - cameraNearPlane));
}

void main()
{
    uint threadIndex = (gl_LocalInvocationID.y * tileResolution.x) + gl_LocalInvocationID.x;
    
    uint tileIndex = ((gl_WorkGroupID.y * uint(numTiles2D.x)) + gl_WorkGroupID.x);
        
    uvec2 textureCoordinates = (gl_WorkGroupID.xy * tileResolution) + gl_LocalInvocationID.xy;
    
    float depth = LinearizeDepth(texelFetch(depthTexture, ivec2(textureCoordinates), 0).r);
    
    threadDepths[threadIndex] = vec2(depth);
    
    barrier();
    
    for(uint stride = 1; stride < numThreadsPerWorkGroup; stride*=2){
        if(threadIndex % (stride*2) == 0){
            float minZ = min(threadDepths[threadIndex].x, threadDepths[threadIndex + stride].x);
            float maxZ = max(threadDepths[threadIndex].y, threadDepths[threadIndex + stride].y);
            threadDepths[threadIndex] = vec2(minZ, maxZ);
        }
        barrier();
    }
    
    float minDepth = threadDepths[0].x;
    float maxDepth = threadDepths[0].y;
    
    if(threadIndex == 0)
        tileDepthMinMax[tileIndex] = vec2(minDepth, maxDepth);
}
```

<br>

