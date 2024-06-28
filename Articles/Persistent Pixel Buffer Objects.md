# Persistent Pixel Buffer Objects

# Introduction

When questions are asked on forums related to Pixel Buffer Objects (PBO), replies usually direct the reader to the article [OpenGL Pixel Buffer Object (PBO)](http://www.songho.ca/opengl/gl_pbo.html) by Song Ho, which showcases double buffering PBOs for texture uploads/downloads. In a talk by [Nivida during Steam Dev Days 2014](https://www.youtube.com/watch?v=-bCeNzgiJ8I), Cass Everitt and John McDonald describe using Persistent Mapped Buffers with triple buffering to reduce driver overhead. There are a decent number of articles describing Persistent Mapped Buffers, but none describe their use with Pixel Buffer Objects. In this article, I want to showcase how one might use Persistent Mapped Buffers for Pixel Buffer Objects.

# Algorithm

The general algorithm is as follows:

1. Allocate a buffer with 3x of the intended size. This should guarantee enough buffer room for data in flight (GPU), data staged by the driver, and data you're writing (CPU).
2. Write to section 1 of the buffer. Create a fence sync object after issuing all commands that read from the buffer.
3. In the next frame, write to section 2 of the buffer. Like the previous step, create a fence sync object after issuing all commands that read from the buffer. 
4. Repeat for section 3 of the buffer.
5. On the fourth frame, before writing to section 1 of the buffer, you must check the sync object to see if it has been completed before you can start writing.

> [!WARNING]
> You must keep each section’s sync object separate!

Here is a pseudo-code adapting the above algorithm for writing color data to the PBO and copying its data to a texture it as a texture to be used in drawing a textured quad.

```
data = create_buffer_and_map_range()
buffer_id = 0

fn update:
    wait_for_sync(buffer_id)
    
    for each pixel in data do:
        pixel = white_color
    
    tex_id = copy_data_to_texture(data, buffer_id)
    
    draw_quad_with_texture(tex_id)
    
    place_fence(buffer_id)
    buffer_id = (buffer_id + 1) mod 3
```

# Creating a PBO

Creating a PBO requires three steps:

1. Create a new buffer object with glCreateBuffers().
2. Specify flags and allocate memory to the buffer object with glNamedBufferStorage().
3. Save a pointer to the data.

```cpp
glCreateBuffers(1, &pbo_handle);
constexpr GLbitfield flags = GL_MAP_WRITE_BIT | GL_MAP_PERSISTENT_BIT | GL_MAP_COHERENT_BIT;
glNamedBufferStorage(pbo_handle, DATA_SIZE_BYTES * 3, nullptr, flags);
void *buffer = glMapNamedBufferRange(pbo_handle, 0, DATA_SIZE_BYTES * 3, flags));
```

> [!NOTE]
> The `GL_MAP_COHERENT_BIT` flag automatically makes your changes in the memory visible to the GPU. Without this flag, you have to set a memory barrier manually. See Easy Synchronization on the [OpenGL Wiki](https://www.khronos.org/opengl/wiki/Buffer_Object#Persistent_mapping) for more information.


# Releasing the PBO

Make sure to unmap the buffer before deleting it.

```cpp
glUnmapNamedBuffer(pbo_handle);
glDeleteBuffers(1, &pbo_handle);
```

# Example

This code snippet demonstrates how to write color data for an OpenGL texture object using a PBO.

```cpp
enum {
    TEX_WIDTH = 32,
    TEX_HEIGHT = 32,
    PIXEL_COUNT = TEX_WIDTH * TEX_HEIGHT,
    PIXEL_COUNT_BTYES = PIXEL_COUNT * 4 // 4 bytes per pixel
};

struct Color {
    unsigned char r, g, b, a;
};

struct BufferRange {
    uintptr_t offset;
    GLsync sync;
} buffer_ranges[3];

GLuint pbo_handle;
GLuint tex_handle;
Color *data;
unsigned char buffer_id;

void init() {
    buffer_id = 0;
    buffer_ranges[0].offset = 0;
    buffer_ranges[1].offset = DATA_SIZE;
    buffer_ranges[2].offset = DATA_SIZE * 2;
    
    glCreateBuffers(1, &pbo_handle);
    constexpr GLbitfield flags = GL_MAP_WRITE_BIT | GL_MAP_PERSISTENT_BIT | GL_MAP_COHERENT_BIT;
    glNamedBufferStorage(pbo_handle, PIXEL_COUNT_BTYES * 3, nullptr, flags);
    data = (Color*)glMapNamedBufferRange(pbo_handle, 0, PIXEL_COUNT_BTYES * 3, flags);
    
    glCreateTextures(GL_TEXTURE_2D, 1, &tex_handle);
    glTextureStorage2D(tex_handle, 1, GL_RGBA8, TEX_WIDHT, TEX_HEIGHT);
}

void update() {
    // Wait for fence
    GLsync &sync = buffer_ranges[buffer_id].sync;
    if (sync) {
        while (true) {
            GLenum const result = glClientWaitSync(sync, GL_SYNC_FLUSH_COMMANDS_BIT, 1);
            if (result == GL_ALREADY_SIGNALED || result == GL_CONDITION_SATISFIED)
                return;
        }
    }
    
    // Write color data to buffer
    Color *buffer = data + buffer_sections[buffer_id].offset;
    for (unsigned int i = 0; i < PIXEL_COUNT; ++i) {
        buffer[I] = Color{ 255, 255, 255, 255 };
    }
    
    // Copy PBO data to texture
    glBindBuffer(GL_PIXEL_UNPACK_BUFFER, pbo_handle);
    glTextureSubImage2D(
        tex_handle, 0, 0, 0, TEX_WIDTH, TEX_HEIGHT, GL_RGBA, GL_UNSIGNED_BYTE,
        (void*)buffer_sections[buffer_id].offset * sizeof(Color));
    glBindBuffer(GL_PIXEL_UNPACK_BUFFER, 0);
    
    // Draw something here with the texture.
    /* drawing code... */
    
    // Place fence
    if (sync)
        glDeleteSync(sync);
    sync = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
    buffer_id = (buffer_id + 1) % 3;
}

void clean_up() {
    glDeleteTextures(1, &tex_handle);

    glUnmapNamedBuffer(pbo_handle);
    glDeleteBuffers(1, &pbo_handle);
}
```

# References

- [Pixel Buffer Object - OpenGL Wiki](https://www.khronos.org/opengl/wiki/Pixel_Buffer_Object)
- [Buffer Object Streaming - OpenGL Wiki](https://www.khronos.org/opengl/wiki/Buffer_Object_Streaming)
- [Beyond Porting: How Modern OpenGL Can Radically Reduce Driver Overhead - Nvidia](http://media.steampowered.com/apps/steamdevdays/slides/beyondporting.pdf)
- [Persistent Mapped Buffers - Ferran Sole](https://ferransole.wordpress.com/2014/06/08/persistent-mapped-buffers/)
