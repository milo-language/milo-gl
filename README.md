# gl

This is a package for the [Milo language](https://milo-language.github.io/milo/).

## Overview

OpenGL 3.3 core bindings, plus a safe layer over them: shaders, meshes,
textures and framebuffers, with the raw pointers kept out of your program. Drop
to `gl/raw` for an entry point it does not wrap.

You supply the context. Creating one belongs to the window system rather than
here, so `examples/` and `tests/` get theirs from the
[`sdl`](https://github.com/milo-language/milo-sdl) package's `sdl/gl` module.
darwin and linux only.

Nothing here has a `Drop`, which is not the usual Milo answer. The context owns
every object it hands out and frees whatever is left when you call `gl.free()`.
Freeing one yourself early is a move, so touching it afterwards is a compile
error rather than a driver-level mystery.

Why it works that way, and every function and method:
[docs/api.md](docs/api.md).

## Installation

```bash
milo add github.com/milo-language/milo-gl            # latest release
milo add github.com/milo-language/milo-gl@v0.2.0     # or pin a tag
```

```milo
from "gl" import { GlContext, Gpu, Shader, Mesh, Texture2D }
```

## Examples

### Drawing a frame

```milo
from "gl" import { GlContext, Gpu, Shader, Mesh }

fn main(): i32 {
    // A GL context must already be current: creating one is the window
    // system's job, not this package's.
    var gl = GlContext.new()

    var sh = Shader.compile(gl, VERT, FRAG)!
    var quad = Mesh.fullscreenQuad(gl)

    Gpu.clear(0.0, 0.0, 0.0, 1.0, true)
    sh.bind()
    sh.uniformF("time", 0.5)
    quad.draw()

    gl.free()   // sweeps anything still outstanding
    return 0
}
```

`Shader.compile` returns a `Result` carrying the driver's own log, so a shader
that does not compile tells you why.

### Textures on 3D geometry

The constructors start clamped, unfiltered and mip-less, which is right for a
texture that is a picture of the whole frame. A texture laid over 3D geometry
wants the other three settings:

```milo
from "gl" import { GlContext, Texture2D, Wrap }

fn uploadGround(gl: &mut GlContext, w: i64, h: i64, bytes: &Vec<u8>): Texture2D {
    let ground = Texture2D.srgb8(gl, w, h, bytes, true)
    ground.setWrap(Wrap.Repeat)      // UV is world position over a period, not 0..1
    ground.generateMipmaps()         // after the upload: it derives the chain from level 0
    ground.setAnisotropy(16.0)       // no-op without GL_EXT_texture_filter_anisotropic
    return ground
}
```

`srgb8` takes three sRGB bytes per pixel, what a PNG decodes to, and the sampler
decodes to linear in hardware before filtering. Doing that in the shader
afterwards is both slower and wrong. Mipmaps are not optional for anything tiled
across a 3D surface, and anisotropy fixes what mips alone get wrong at a grazing
angle. [docs/api.md](docs/api.md) explains both.

### Freeing an object early

Anything replaced mid-run should be freed the moment you are done with it rather
than waiting for teardown. `free` takes the context, so the name can go back:

```milo
let t = Texture2D.rgba8(gl, 16, 16, pixels, false)
t.free(gl)
t.bind(0)
```

```
error: use of moved variable 't'
  ──> texture.milo:9:5
  │
9 │     t.bind(0)
  │     ^
  hint: 't' is a @noCopy handle, so transferring it ended its life here — copying
        one would let the same resource be released twice. Borrow it (pass it to
        a '&Texture2D' parameter) instead of transferring, or reorder so the
        transfer is last.
```

That is a compile error, not a crash at some later frame. `GlContext.live()`
reports how many objects are still outstanding, so a leak in a frame loop is a
number you can assert on rather than a slow climb in a memory graph.

A full spinning-cube program, window and all: `examples/gpucube.milo`.
