---
title: Minecraft-like terrain
description: Minecraft-like terrain generation using OpenGL
image: terrain_noise.png
categories:
    - Graphics Programming
tags:
    - OpenGL
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

This project aims to recreate a Minecraft-like terrain generation without using an engine. Made using OpenGL and the GLFW, GLM and GLEW libraries. [Source code](https://github.com/cmanziel/mncrft).

## Main Classes

These are the main classes used in the project:

### Terrain

The terrain is composed of chunks, which are generated around the player according to his position in space.

### Chunk

A single chunk is a collection of Blocks, every chunk allocates in the memory its own blocks, giving them a local position and a world position based on its own. From every chunk is created a Mesh, which is the actual collection of vertices that will be sent to the shaders to be drawn on the screen.

### Block

A Block is the smallest component of the terrain. Each block has a different ID according to its local y coordinate in the chunk. The ID determines which texture will be applied to the block's faces and if the block is to be considered solid or not when the mesh of its chunk is being created. Block IDs are *air*, *dirt*, *grass*, *cobblestone*.

### Mesh

The mesh is the collection of vertices of a chunk that will eventually be drawn to the screen. It is created by evaluating if the blocks' faces would be visible by the player, if not there's no need to draw them on the screen. The mesh vertices will then be used as the buffer in the function glBufferData or glBufferSubData.

### Shader and Buffer

These are the classes that wrap around the opengl functions. They are responsible of creating a shader program, compiling it and deleting it, allocating the gl buffers and update them with data from the meshes.

## Terrain generation

- In the Terrain class constructor every Chunk class instance is allocated and stored row-by-row along the z axis in a bidimensional array.
- The chunks grid consists of a number of CHUNK_RADIUS chunks created along the positive and negative z and x axis from the player
- Then for every chunk the function GenerateMeshes is called: the function checks if the current chunk being processed is in front of the camera to determine if the mesh should be generated or not
- After the call of GenerateMeshes the value of m_CurrentChunk is incremented to process, in the next frame, the next chunk in the grid
- When the player crosses the edge of the chunk it's in, the terrain around it is regenerated: chunks that are distant more then CHUNK_RADIUS from its position in the grid are deleted and new ones are generated in the direction it's moving
- When this occurs, m_CurrentChunk is reset to 0 because all the meshes should be generated again according to the new chunks' disposition that affects some of the chunks' surroundings

![Terrain generated with the use of Perlin noise to determine the height of every column of blocks.](terrain_noise.png "terrain with noise")

## Mesh generation

- every chunk has a m_LowestSolidHeight field that holds the lowest y coordinate value of the block whose ID is not *air*. The m_Blocks array is iterated through starting from the lowest solid block's index
- the condition checked first is if the block's world position lies inside the camera's frustum. 
- for every face of the blocks is checked if there is a neighbouring block which is solid. If it is not, then the vertices of the face should be added to the mesh because they will be visible
- if a face is at the edge of a chunk, then the program checks if the neighbouring chunk exists and if it has a solid block adjacent to the face
- By doing this only the vertices that make up the external profile of the chunk will be rendered
- three vectors store the data that's needed to correctly render a block face: m_Faces,m_TexCoords and m_ModelMats

*m_Faces*: each face of a block has its array of data in the header file Renderer.h. For every face added to the mesh one of the values of the enum variable "sides" is pushed back to the m_Faces vector. So when the process of mesh generation is done m_Faces is a collection of indexes that determine which of the six arrays that represent a face will be used in the shader.

*m_TexCoords*: for every vertex added to the mesh the corresponding 2D textures coordinates are pushed back to the m_TexCoords vector.
The texture coordinates referer to the texture atlas image and depend on the block's ID

*m_ModelMats*: for every face added to the mesh, a mat4 4x4 matrix is pushed back to the m_ModelMats vector. These are the matrices that will compute the correct 3D position of a vertex.
When a face is deemed to be part of the chunk's mesh a model matrix is calculated from its block's world position and in the vertex shader every position of the vertices of that face will be trasnformed by the matrix

![A chunk of blocks before evaluating its mesh.](no_shell.png "a chunk of blocks before evaluating its mesh")

![A chunk of blocks after evaluating its mesh, every vertex inside the chunk is not rendered.](shell.png "a chunk of blocks before evaluating its mesh")

## Textures

To every visible face of a block is applied a texture according to its ID. All the different textures are stored in a single texture atlas, handled by the *TextureAtlas* class, and are retrieved through the correct offset based on the block's ID.

![Textures applied to the blocks' faces.](textures.png "Textures")

![Different textures for different block IDs based on the height they're at.](texture_ids.png "Noise")

## Block Breaking

A ray is sent from the player's camera into the world. If it intersects a block whose ID isn't *air* the block's edges are highlighted. On mouse input, the block's ID is set to *air* so that when the mesh is regenerated its faces won't be rendered, conveying the effect of breaking the block.
The highlighting of the edges is done by giving every vertex of a face *barycentric coordinates* and then checking them in the fragment shader to determine if the vertex is at the edge of a face.

![Highlighting the block pointed by the player.](block_pointed.png "Block pointed")
