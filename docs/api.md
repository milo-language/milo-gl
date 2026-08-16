# gl API

Every constructor takes the `GlContext` so the object can be registered with it;
every `free` takes it so the name can be handed back.

```milo
// context
GlContext.new()                     // after the window system's context is current
gl.live()                           // i64 — objects still outstanding
gl.free()                           // sweeps whatever is left, once

// state
Gpu.version()                       Gpu.hasExtension(name)
Gpu.viewport(x, y, w, h)            Gpu.clear(r, g, b, a, depth)
Gpu.depthTest(on)                   Gpu.depthWrite(on)
Gpu.cullBackFaces(on)               Gpu.finish()                Gpu.error()
Gpu.blendOff()                      Gpu.blendAlpha()
Gpu.blendAdd()                      Gpu.blendPremultiplied()

// shaders
Shader.compile(gl, vert, frag)      // Result<Shader, string> — the driver's log on failure
sh.bind()                           sh.loc(name)                sh.free(gl)
sh.uniformI(name, v)                sh.uniformF(name, v)
sh.uniform2F(name, a, b)            sh.uniform3F(name, a, b, c)
sh.uniform4F(name, a, b, c, d)      sh.uniformMat4(name, m)
sh.sampler(name, tex, unit)

// meshes
Mesh.new(gl, attrs)                 // attrs: component count per vertex attribute
Mesh.fullscreenQuad(gl)
mesh.upload(data)                   mesh.draw()                 mesh.free(gl)

// textures
Texture2D.srgb8(gl, w, h, pixels, smooth)     // Vec<u8>,  3 bytes per pixel
Texture2D.rgba8(gl, w, h, pixels, smooth)     // Vec<u32>
Texture2D.rgb32f(gl, w, h, pixels, smooth)    // Vec<f32>
Texture2D.r32f(gl, w, h, pixels, smooth)      // Vec<f32>
Texture2D.r32fEmpty(gl, w, h, smooth)
Texture2D.rgba16f(gl, w, h, smooth)
tex.updateRgba8(pixels)             tex.updateRgb32f(pixels)
tex.updateR32f(pixels)              tex.updateR32fRegion(w, h, pixels)
tex.setWrap(Wrap.Repeat)            tex.generateMipmaps()       tex.setAnisotropy(want)
tex.bind(unit)                      tex.free(gl)

// render targets
Target.new(gl, w, h, depth, smooth) // Result<Target, string>
tgt.bind()                          tgt.bindTexture(unit)       tgt.free(gl)
bindScreen(w, h)                    readPixelsRgba8(w, h)       // Vec<u32>
```

## Why the context owns its objects

Nothing here has a `Drop`, and that is not the usual Milo answer — `Vec` and
`string` free themselves, and no other package makes you call anything.

GL is the exception because `glDelete*` requires the context that made the
object to still be current, on the thread it was made on. A destructor is
exactly the thing whose timing you do not control. Rust libraries solve this by
giving every GL object an `Arc<Context>`, so the context is provably alive
whenever an object drops. **Milo cannot express that**: references are
second-class, so a struct can never store a `&GlContext`, and an object
therefore cannot keep its context alive.

So ownership runs the other way. A `GlContext` records every name it hands out
and deletes whatever is left, once, where you put the call.

`@noCopy` is what makes an early `free` followed by a use a **compile error**
rather than a driver-level mystery, and it is why these types are not `Copy` —
they are integers, and the all-fields-Copy rule would otherwise make `free`
consume nothing.

The two checks cover different failures and neither subsumes the other.
`@noCopy` catches use-after-free and double-free, which the context cannot see.
The context catches forgetting, which `@noCopy` cannot see. `GlContext.live()`
reports how many objects are outstanding, so a leak in a frame loop is a number
you can assert on rather than a slow climb in a memory graph.

Uploads are bounds-checked too — the driver reads `w * h` elements off a pointer
with no idea how long your `Vec` is, so a short one would be a heap over-read
from a call with no `unsafe` at the call site.

## sRGB, mipmaps and anisotropy

`srgb8` takes three sRGB bytes per pixel — what a PNG decodes to — and the
sampler decodes to linear **in hardware, before filtering**. Doing it afterwards
in the shader is both slower and wrong: a bilinear tap averages four sRGB bytes,
and the average of two sRGB values is not the sRGB of their linear average, so
edges come out too dark. A lookup table per fetch has the same flaw and costs a
dependent read.

Mipmaps are not optional for anything tiled across a 3D surface. Without them a
distant pattern samples one texel out of the dozen the pixel covers, and which
one changes as the camera moves — the ground crawls and glitters. Anisotropy
then fixes what mips alone get wrong at a grazing angle, where trilinear picks
one level from the pixel's widest axis and blurs the direction that was not
compressed.

## darwin and linux only

`milo.json` declares `"targets": ["darwin", "linux"]`, and the compiler enforces
it — building for Windows names the package rather than failing on a missing
symbol. `opengl32.dll` exports GL 1.1 only, so every 3.3 entry point here would
be undefined.

3.3 core is the floor on purpose: it is the highest version macOS ships, and old
enough that every Mesa and every driver of the last decade has it.

## A context is your job

Every call needs a current GL context, and creating one belongs to the window
system, not here — the library deliberately depends on nothing but the standard
library. With SDL2 that is `SDL_GL_SetAttribute` + `SDL_WINDOW_OPENGL` +
`SDL_GL_CreateContext`, which the
[`sdl`](https://github.com/milo-language/milo-sdl) package's `sdl/gl` module
provides; `examples/` and `tests/` use it, and they carry that dependency in
their own manifests so the published package does not. Calling into GL with no
context bound is undefined behaviour, not an error return.

## Verified bindings

Every declaration carries `@cSig`, so the signature is checked against the real
GL header at build time on any machine that has one — including each pointer
parameter's pointee width, which is what an out-param's contract actually is. A
machine with neither `OpenGL/gl3.h` nor `GL/glcorearb.h` gets a named warning,
not a silent pass.
