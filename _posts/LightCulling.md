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

During rasterization, the geometry is converted into fragments, and the fragment shader is executed for each fragment. 

The fragment shader computes the lighting contribution from each light source in the scene and adds them up. For simple scenes with a few light sources, this setup can be reasomable and an interactive frame-rate can be maintained. However, for scenes with hundreds and thousands of light sources, this setup would not scale well, be feasible and the frame-rates would drop. Hence, the idea of light culling was introduced. 

As we know, the intensity of the light coming from a specific light source decreases as we get further and further from the light source. In fact, we tend to model the drop in intensity using the inverse square attentuation. That is, 

$$
I_d = \frac{I}{d^2}
$$

This function has an asymptote as zero. This means it will never reach zero. The idea of light culling is that at certain great distances from the light source, the light contribution from the light source becomes so small, that we can ignore that light source completely, and it would not affect the physical accuracry of the final scene by a great deal. This way we can save alot of computational cost. 

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

3. Dispatch a compute shader In a seperate pass before the shading pass, iterate through all light sources. For each light sources $$L$$, test for intersection between 

    - Test the intersection between the light bounding sphere $$B$$ and each tile frustum. 
    <br>
    <br>
    - If the light bounding sphere intersects the tile frustum, record the ID of the light into the per-tile light list. 
    <br>
    <br>
4. In the main shading pass: 

    - Calculate the tile index in which the fragment belongs to.
    
    - Calculate the light contribution that the fragment receives by only going through the list of lights of that tile. 

<br>


```c++
#version 450

layout(local_size_x = 16, local_size_y = 16, local_size_z = 1) in;

layout(std430, binding = 0) readonly buffer LightBoundingSpheresViewSpace{
    vec4 lightBoundingSpheresViewSpace[];
};

layout(std430, binding = 1) buffer TileLightCount {
    uint tileLightCount[];
};

layout(std430, binding = 2) buffer TileLightIndices {
    uint tileLightIndices[];
};

layout(std140, binding = 3) uniform LightCullingUniformVariables{
    float nearClippingPlane;
    float farClippingPlane;
    uvec2 numTiles2D;
};

layout(std430, binding = 4) readonly buffer TileDepthMinMax{
    vec2 tileDepthMinMax[];
};

layout(std430, binding = 5) readonly buffer TilesFrustumPlanes{
    vec4 tilesFrustumPlanes[];
};

layout(std430, binding = 6) readonly buffer TilesGeometryBitMask{
    uint tilesGeometryBitMask[];
};

const uint numCellsPerTile = 32;
const uvec2 tileResolution = uvec2(16, 16);

struct LightVisibilityInfo{
    bool isVisible;
    uint minCell;
    uint maxCell;
};

uint ComputeCellIndex(float depth, float minDepth, float maxDepth)
{
    float depth01 = log(depth / minDepth) / log(maxDepth / minDepth);
    uint cellIndex = uint(depth01 * float(numCellsPerTile));
    
    return max(min(cellIndex, numCellsPerTile - 1u), 0);
}

LightVisibilityInfo IsLightVisible(vec4 lightBoundingSphere, float minDepth, float maxDepth, uint tileGeometryBitMask, uint tileIndex)
{
    LightVisibilityInfo info;
    
    vec3 lightCenter = lightBoundingSphere.xyz;
    float lightDepth = -lightBoundingSphere.z;
    float radius = lightBoundingSphere.w;
    float lightMinDepth = lightDepth + radius;
    float lightMaxDepth = lightDepth - radius;
    
    if(lightMinDepth < minDepth){
        info.isVisible = false;
        return info;
    }
    if(lightMaxDepth > maxDepth){
        info.isVisible = false;
        return info;
    }
    
    uint baseP = tileIndex * 4;
        
    for(int i = 0; i < 4; i++){
        vec4 plane = tilesFrustumPlanes[baseP+i];
        if(dot(plane, vec4(lightCenter, 1.0f)) < -radius){
            info.isVisible = false;
            return info;
        }
    }
    
    uint i1 = ComputeCellIndex(lightMinDepth, minDepth, maxDepth);
    uint i2 = ComputeCellIndex(lightMaxDepth, minDepth, maxDepth);
    
    info.minCell = min(i1, i2);
    info.maxCell = max(i1, i2);
    
    bool isDiscard = true;
    for(uint i = info.minCell; i <= info.maxCell; i++){
        if(((tileGeometryBitMask >> i) & 1u) == 1){
            isDiscard = false;
            break;
        }
    }
    
    if(isDiscard){
        info.isVisible = false;
        return info;
    }

    info.isVisible = true;
    return info;
}

void main()
{
    uint threadIndex = (gl_LocalInvocationID.y * tileResolution.x) + gl_LocalInvocationID.x;
    
    uint numLights = lightBoundingSpheresViewSpace.length();
    
    uint tileIndex = (gl_WorkGroupID.y * uint(numTiles2D.x)) + gl_WorkGroupID.x;
    
    float minDepth = tileDepthMinMax[tileIndex].x;
    
    float maxDepth = tileDepthMinMax[tileIndex].y;
    
    uint tileGeometryBitMask = tilesGeometryBitMask[tileIndex];

    uint baseC = tileIndex * numCellsPerTile;
                    
    if(threadIndex < numCellsPerTile)
        tileLightCount[baseC+threadIndex] = 0;
    
    barrier();
    
    if(threadIndex < numLights){
        vec4 lightBoundingSphere = lightBoundingSpheresViewSpace[threadIndex];
        LightVisibilityInfo info = IsLightVisible(lightBoundingSphere, minDepth, maxDepth, tileGeometryBitMask, tileIndex);
        if(info.isVisible){
            uint baseI = tileIndex * numLights * numCellsPerTile;
            for(uint j = info.minCell; j <= info.maxCell; j++){
                uint i = atomicAdd(tileLightCount[baseC+j], 1);
                tileLightIndices[baseI+(j*numLights)+i] = threadIndex;
            }
        }
    }
}
```

<br><br>

```c++
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

layout(std430, binding = 3) buffer TilesGeometryBitMask{
    uint tilesGeometryBitMask[];
};

const uint numCellsPerTile = 32;
const uvec2 tileResolution = uvec2(16, 16);
const uint numThreadsPerWorkGroup = tileResolution.x * tileResolution.y;

shared vec2 threadDepths[numThreadsPerWorkGroup];

uint ComputeCellIndex(float depth, float minDepth, float maxDepth)
{
    float depth01 = log(depth / minDepth) / log(maxDepth / minDepth);
    uint cellIndex = uint(depth01 * float(numCellsPerTile));
    
    return max(min(cellIndex, numCellsPerTile - 1u), 0);
}

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

    uint threadCellIndex = ComputeCellIndex(depth, minDepth, maxDepth);
    atomicOr(tilesGeometryBitMask[tileIndex], 1u << threadCellIndex);
}
```
