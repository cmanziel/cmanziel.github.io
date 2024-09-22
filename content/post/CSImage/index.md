---
title: CSImage
description: Version of ImageEditor using GLSL compute shaders
image: effects.png
categories:
    - Image Processing
tags:
    - OpenGL
weight: 3
---

This project is a refactor of the [ImageEditor](https://github.com/cmanziel/ImageEditor) project. The first version of the project was mainly relying on the CPU for all the drawing and effects processes. This one uses compute shaders for every step of the way. [Source code](https://github.com/cmanziel/CSImage).

## Drawing

Each drawing style the brush can have has its own shader. The shader samples the canvas texture to retrieve the background color through GLSL's *imageLoad* function, and stores the color being drawn into the render target texture.
After the off-screen rendering is done, the render texture is mapped to the vertices of a quad which is rendered through the usual vertex and fragment shader.

For every compute shader, a *brush radius* * *brush radius* number of work groups is dispatched. The *local_size* of every work group is 1 x 1 x 1, so the image can be accessed using the *gl_GlobalInvocationID.xy* built-in variable as the UV coordinates.

```C++
void Execute()
{
    Dispatch(m_BrushRadius, m_BrushRadius, 1);
}
```

```C++
void Dispatch(unsigned int xGroups, unsigned int yGroups, unsigned int zGroups)
{
    glDispatchCompute(xGroups, yGroups, zGroups);
    glMemoryBarrier(GL_SHADER_IMAGE_ACCESS_BARRIER_BIT | GL_SHADER_STORAGE_BARRIER_BIT);
}
```

## Brush Effects

### Edge Detection (Sobel Operator)

[Two kernels](https://en.wikipedia.org/wiki/Sobel_operator) are defined to process every pixel in area covered by the brush based on its surrounding pixels.
The effect is applied on everything that has been drawn onto the canvas and not only on the original canvas.

To prevent *imageStore* operations to affect *imageLoad* operations in the same shader dispatch a separate canvas texture image is used.
This prevents an *imageLoad* operation to get the value of another pixel in the grid which was just modified by the same operation that is being done on the current one.

```C
float kernel_mult(ivec2 pixel_uv, int ch, mat3 kernel)
{
    float kernel_sum = 0;
    ivec2 dims = imageSize(edited);

    for(int y = -1; y < 2; y++)
    {
        for(int x = -1; x < 2; x++)
        {
            ivec2 grid_uv = pixel_uv + ivec2(x, y);

            vec4 current_color = imageLoad(sobelCanvas, ivec2(grid_uv.x, grid_uv.y));

            kernel_sum += int(current_color[ch] * kernel[x + 1][1 - y]);
        }
    }

    return kernel_sum;
}

vec4 edge_pixel(ivec2 uv)
{
    // imageLoad with an out of boundary coordinate returns an all zeroes vec4
    // so for the edge pixels of the imagedo the kernel multiplication with out of bounds values as for the other pixels

    vec4 result;
    float sum = 0;

    // for each of the rgb channels compute the kernel calculation
    for(int i = 0; i < 3; i++)
    {
        float Gx = kernel_mult(uv, i, sobel_kernel_Gx);
        float Gy = kernel_mult(uv, i, sobel_kernel_Gy);

        // sqrt is a floating point operation
        result[i] = sqrt(Gx * Gx + Gy * Gy / SOBEL_CLAMP_FLOAT_VALUE);
        sum += result[i];
    }

    result.xyz = vec3(sum / 3);

    return result;
}
```

### Blur (Box Blur)

A [Box Blur](https://en.wikipedia.org/wiki/Box_blur) algorithm is applied to every pixel inside the brush area. The result is the average value between a 3x3 grid around each pixel. The effect is applied on everything that has been drawn onto the canvas and not only on the original canvas.

```C
vec4 blur_pixel(ivec2 uv, ivec2 dims)
{
    vec4 result;
    // if the pixel is on the edge of the image, return it unmodified
    if(uv.x < 1 || uv.y < 1 || uv.x + 1 == dims.x || uv.y + 1 == dims.y)
        return imageLoad(blurCanvas, uv);
    for(int y = -1; y < 2; y++)
    {
        for(int x = -1; x < 2; x++)
        {
            if(x == 0 && y == 0)
                continue;
            ivec2 grid_uv = uv + ivec2(x, y);
            vec4 current_color = imageLoad(blurCanvas, ivec2(grid_uv.x, grid_uv.y));
            result += current_color;
        }
    }
    
    result.xyz /= 9;
    return result;
}
```

<!-- ## Syntax

```markdown
![Image 1](1.jpg) ![Image 2](2.jpg)
```

## Result

![Image 1](1.jpg) ![Image 2](2.jpg)

> Photo by [mymind](https://unsplash.com/@mymind) and [Luke Chesser](https://unsplash.com/@lukechesser) on [Unsplash](https://unsplash.com/) -->