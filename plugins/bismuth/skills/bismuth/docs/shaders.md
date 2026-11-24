# Shaders in Bismuth

Status: implemented (issue #17). Shader stages are Bismuth functions that bic
checks against a subset of the language and compiles to SPIR-V (Vulkan),
MSL (Metal) and HLSL (Direct3D 11 and 12) with no external toolchain. The
same functions stay ordinary functions: CPU code calls them, and `bic run
--interp` runs them in the IR interpreter. This is stage 3 of
[graphics.md §3](graphics.md#3-shaders).

```bismuth
import std.gfx;

struct Vertex { vec2 pos; vec2 uv; Rgba8 color; }          // a vertex buffer: one attribute per field
struct Varyings { @position vec4 pos; vec2 uv; vec4 color; } // the vertex stage's result
struct Globals { mat4 proj; }                              // a uniform block

@vertex
fn spriteVs(Vertex v, @uniform Globals g) : Varyings {
    Varyings { pos: g.proj * vec4(v.pos, 0.0, 1.0), uv: v.uv, color: v.color.toVec4() }
}

@fragment
fn spriteFs(Varyings i, Texture2D tex, Sampler smp) : vec4 {
    tex.sample(smp, i.uv) * i.color
}

let vs = core.shader(spriteVs);   // ShaderCode: SPIR-V, MSL, HLSL and the interface, built at compile time
```

## 1. Stages and their interfaces

A function becomes a shader stage with one attribute:

| Attribute | Stage | Result |
|---|---|---|
| `@vertex` | vertex | a struct with one `@position vec4` field (clip space) and the varyings |
| `@fragment` | fragment | a `vec4` color (target 0), or a struct of `vec4` colors (targets in field order) and an optional `@depth f32` |
| `@compute(x, y, z)` | compute, workgroup size x × y × z (missing sizes are 1) | nothing: results go to storage buffers |

A stage must be a free, non-generic function. Its parameters say where each
input comes from, by type and attribute; there is no separate binding
declaration and no reflection step:

| Parameter | Meaning |
|---|---|
| a struct (vertex stage) | a vertex buffer. Slots in parameter order, one attribute per field, locations numbered across the buffers in order. `@instance` makes it per-instance |
| a struct (fragment stage) | the varyings: the vertex stage's result type. Its `@position` field reads the fragment's window position (`SV_Position`, `[[position]]`, `FragCoord`) |
| `@uniform T`, `@uniform(slot: n) T` | a uniform block holding a struct of your own (`view` is allowed: it only avoids the copy warning). Its slot, which `enc.setUniforms(slot, ...)` fills in every stage of the pipeline that has it, is its place among the stage's blocks, or `n` (0 to 15), so that a fragment stage's block can have a slot of its own beside the vertex stage's; two blocks of a stage can't share one |
| `Texture2D` | a texture binding |
| `DepthTexture2D` | a texture binding of a depth image, read through a comparison sampler (shadow maps) |
| `TextureCube` | a texture binding of a cube image (`gfx.ImageKind.Cube`), sampled by direction |
| `Texture2DArray` | a texture binding of an array image (`gfx.ImageKind.D2Array`), sampled by uv and layer |
| `StorageImage2D<T>` | compute stages only: a storage image binding, an image the stage reads and writes (`gfx.ImageDesc.storage`), of `vec4`, `f32` or `u32` texels; counted with the textures |
| `Sampler` | a sampler binding |
| `SamplerCmp` | a comparison sampler binding (`gfx.SamplerDesc.compare`) |
| `MutSlice<T>` / `Slice<T>` (or `view Slice<T>`) | a read-write / read-only storage buffer of T |
| `@vertexIndex i32`, `@instanceIndex i32` | vertex stage system values |
| `@frontFacing bool` | fragment stage |
| `@globalId ivec3`, `@localId ivec3`, `@groupId ivec3`, `@localIndex i32` | compute stage |

Fields of interface structs take `@position`, `@depth` and `@flat` (no
interpolation; integer varyings are flat anyway).

Vertex attribute fields are `f32`, `vec2`–`vec4`, `i32`, `u32`, `ivec2`–`ivec3`
or `Rgba8` (four normalized bytes, read in shader code with `toVec4()`, as
in Dear ImGui's 20-byte `ImDrawVert`). Varyings are the same types except
`Rgba8`. A pipeline can store the attributes otherwise
(`gfx.PipelineDesc.vertexLayout`, [graphics.md](graphics.md)): any
`ShaderVertexFormat` of the field's kind, such as `Snorm16x2`, `Half4` or
`Unorm1010102` for a float vector, or `Sint16x4` for an `ivec`, which the
GPU converts as it reads them.

### Layouts come from the types

The CPU struct is the layout: field offsets, sizes and strides are the ones
`sizeof`/`offsetof` report, and the GPU reads those bytes unchanged.

- **Vertex buffers**: stride = `sizeof(T)`, each attribute at its field's
  offset with a format from its type (`Float2`, `Unorm8x4`, ...).
- **Uniform blocks** are std140 on Vulkan, a cbuffer with `packoffset` on D3D
  and a packed struct with explicit padding on Metal, all at the CPU
  offsets. bic checks that those offsets are valid GPU offsets and says
  what to change when they are not: a `vec3`/`vec4`/`mat4` must start on 16
  bytes (`@align(16)`), arrays hold 16-byte elements (`vec4`, `mat4`), and
  `bool` (1 byte on the CPU, 4 on GPUs) is refused in favor of `u32`.
- **Storage buffers** are std430 on Vulkan (structured buffers on D3D,
  device pointers on Metal); the element stride is `sizeof(T)` and bic
  rejects elements whose GPU stride would differ (`Slice<vec3>` is 12 bytes
  a element on the CPU and 16 on GPUs).

### Bindings

Each resource parameter gets one binding per backend, recorded in the
`ShaderCode` (see §3), so a renderer reads them instead of following a
convention:

| Backend | Uniform block | Texture | Sampler | Storage buffer | Storage image | Vertex buffer i |
|---|---|---|---|---|---|---|
| Vulkan | set S, binding k (`UNIFORM_BUFFER`) | set S, binding k (`SAMPLED_IMAGE`) | set S, binding k (`SAMPLER`) | set S, binding k (`STORAGE_BUFFER`) | set S, binding k (`STORAGE_IMAGE`) | binding i |
| Metal | `[[buffer(n)]]` | `[[texture(n)]]` | `[[sampler(n)]]` | `[[buffer(n)]]` | `[[texture(n)]]` | `[[buffer(16 + i)]]` via `[[stage_in]]` |
| D3D | `register(bN)` | `register(tN)` | `register(sN)` | `register(uN)`, read-only: `register(tN)` | `register(uN)` | input slot i, semantic `TEXCOORD<location>` |

S is 0 for vertex and compute stages and 1 for fragment stages, so a
pipeline's two stages never collide; k is the resource's position among
the stage's resource parameters. Metal and D3D number each class from 0 in
parameter order (uniform and storage buffers share Metal's buffer table,
textures and storage images its texture table; read-write storage buffers
and storage images share D3D's u registers). A storage image's slot counts
with the textures': a stage's k-th texture or storage image reads
`images[k]` in std.gfx's bindings.

### Conventions shared by the three outputs

- Clip space has y up and depth 0..1, as in D3D and Metal. The SPIR-V
  vertex stage negates `position.y`, so Vulkan renders the same image with
  a normal (positive-height) viewport. Winding is unaffected: Vulkan's
  front-face test measures area in framebuffer coordinates with y down
  and a sign that makes on-screen counter-clockwise "counter-clockwise",
  so a triangle that is counter-clockwise on screen is counter-clockwise
  to Vulkan, D3D and Metal alike. `std.gfx`'s `frontFace: .Ccw` means
  that on every backend (`tests/gpu/gfxdraw.bi` checks it).
- Texture coordinates start at the top-left texel; fragment positions at
  the top-left pixel (pixel centers at .5).
- Matrices are column-major (`mat4.m[col * 4 + row]`), as uniform data and
  in code; `m * v` is the matrix times a column vector everywhere.
- Every output's entry point is `<function>_main`.

## 2. The shader subset

Shader code is the stage function and every function it calls,
transitively, including std functions (which compile from their Bismuth
bodies unless they map to a GPU built-in). The checker reports anything
outside the subset at the offending expression, prefixed with
`shader <stage>:`.

**In the subset**: `bool`, `i32`, `u32`, `f32`; `int`/`i64`/`u64` as local
values (computed in 32 bits on the GPU: loop counters and indices);
`vec2`–`vec4`, `ivec2`–`ivec3`, `mat4`; structs of these, generic ones
included; fixed arrays `[N]T`; plain enums; `const`s; `let`/`var`,
assignment and compound assignment; `if`/`else` (as statements and values),
`?:`, `&&`/`||` (lazy), `while`, `for` over ranges and fixed arrays,
`break`, `continue`, `return`; struct and array literals; numeric
conversions (`f32(i)`); operator methods (`opAdd`, ...) and ordinary
methods taking `self` or `view self`; calls of other non-stage functions;
`discard()` in fragment stages; `sizeof`/`offsetof`.

**Not in the subset** (each with its own error): `str` and string
interpolation (so `print`), `f64` (Metal has no doubles), 8- and 16-bit
integers and `char`, the heap (`new`, `Shared`, `List`, `Map`, arenas),
closures and function values, nullable types, results and `?`, pointers and
`unsafe`, `async`/`await`/`spawn`, `parallel for`, `defer`, `match` (use `if`
for now), tuples, enums with data, ranged types (their checks don't run on
GPUs), recursion, `ref` parameters and `ref self` methods (for now:
return the new value), C functions other than the math ones std uses, and a
storage buffer's `len()` (Metal has no buffer lengths: pass counts in a
uniform block).

**What differs from the CPU**: `int` wraps at 32 bits on the GPU and at
64 on the CPU (32-bit types wrap the same on both); an out-of-range array
or buffer index or a failed conversion panics on the CPU (with checks on)
and is undefined on the GPU; float results differ in the last bits (and
transcendental functions more, depending on the driver); `discard()`
panics on the CPU; `ddx`, `ddy` and `fwidth` are 0 on the CPU, which has
no neighbouring pixels.

### Built-ins

Shader code uses the same library as CPU code. The prelude's vector and
matrix methods (`+ - * /`, `dot`, `cross`, `length`, `distance`, `lerp`,
`min`, `max`, `abs`, swizzles like `v.xy` and `v.zyx`, `mat4 * vec4`,
`mat4 * mat4`, `transposed`, ...) map to the GPU languages' operators and
built-in functions; the rest compiles from their Bismuth bodies
(`normalized`, `mat4.translate`, `mat4.perspective`, `sin(Angle)` ...). The
prelude's `std/core/shader.bi` adds, with CPU implementations:

- `Texture2D` (`size()`, `load(ivec2)`, `sample(Sampler, vec2)`,
  `sampleLevel(Sampler, vec2, f32)`), built on the CPU from texels
  (`Texture2D.new(w, h, slice)`), and `Sampler` (`filter: Nearest/Linear`,
  `wrap: Clamp/Repeat`, which the CPU implementation follows; on the GPU the
  renderer's sampler object decides).
- `DepthTexture2D` (`size()`, `load(ivec2) : f32`, `sampleCompare(SamplerCmp,
  vec2 uv, f32 ref) : f32`, `sampleCompareLevelZero`), built on the CPU from
  depths (`DepthTexture2D.new(w, h, slice)`), and `SamplerCmp` (`compare`,
  `filter`, for the CPU implementation): `sampleCompare` compares `ref` with
  the texels around uv (`ref <= texel` for `LessEqual`) and blends the
  results bilinearly, the hardware's 2x2 percentage-closer filter. Outside
  fragment stages it samples level 0. SPIR-V: a `Depth` image and
  `OpImageSampleDrefImplicitLod`/`ExplicitLod`; HLSL: `Texture2D<float>`,
  `SamplerComparisonState`, `SampleCmp`/`SampleCmpLevelZero`; MSL:
  `depth2d<float>` and `sample_compare`. A stage's textures of every kind
  count together (`images[k]`), and so do its samplers (`samplers[k]`).
- `TextureCube` (`size() : ivec2`, a face's; `sample(Sampler, vec3 dir)`,
  `sampleLevel(Sampler, vec3 dir, f32)`), built on the CPU from six faces
  of texels (`TextureCube.new(size, slice)`, +X, -X, +Y, -Y, +Z, -Z), which
  the CPU implementation picks as the GPUs do. SPIR-V: a `Cube` image;
  HLSL: `TextureCube<float4>`; MSL: `texturecube<float>`. There is no
  `load`: no API fetches from a cube.
- `Texture2DArray` (`size() : ivec3`, a layer's size and the layer count;
  `load(ivec2, i32 layer)`, `sample(Sampler, vec2 uv, i32 layer)`,
  `sampleLevel(Sampler, vec2, i32 layer, f32)`), built on the CPU from
  layers of texels one after another (`Texture2DArray.new(w, h, layers,
  slice)`). SPIR-V: an arrayed 2D image, the layer a float third
  coordinate; HLSL: `Texture2DArray<float4>`; MSL: `texture2d_array<float>`.
- A texture binding records its kind (`ShaderBinding.textureKind()`, in
  `size`), and std.gfx panics when a draw binds an image of another kind
  to it (a `D2` image to a `TextureCube`, say).
- `StorageImage2D<T>` (`size() : ivec2`, `load(ivec2) : T`, `store(ivec2,
  T)`, and `atomicAdd(ivec2, T) : T` on `u32` images), a 2D image a compute
  stage reads and writes, as bgfx's `IMAGE2D_RW`/`UIMAGE2D_RW` and GLSL's
  `imageLoad`, `imageStore` and `imageAtomicAdd`: built on the CPU from a
  `MutSlice<T>` of texels (`StorageImage2D<T>.new(w, h, slice)`), whose
  `atomicAdd` is atomic there too. Its texels are `vec4` (`Rgba8`,
  `Rgba16F` and `Rgba32F` images), `f32` (`R32F`) or `u32` (`R32U`); a
  binding records which (`ShaderBinding.texel()`, a `ShaderTexel`, in
  `size`), and std.gfx panics when a dispatch binds an image of another
  format or one not made with `ImageDesc.storage`. `load` takes `u32` and
  `f32` images: shader model 5.0 has no typed loads of other formats, so a
  `vec4` image is written, and read back as a `Texture2D` by a later pass.
  `atomicAdd` returns the texel before, so invocations can count and claim
  slots. A texel outside the image panics on the CPU and is undefined on
  the GPU. SPIR-V: an image with `Sampled` 2, format `R32ui` or `R32f`
  (`Unknown` for `vec4`, with `StorageImageWriteWithoutFormat`),
  `OpImageRead`, `OpImageWrite`, `OpImageQuerySize` and `OpAtomicIAdd` on
  an `OpImageTexelPointer`; HLSL: `RWTexture2D<T>`, `[]` and
  `InterlockedAdd`; MSL: `texture2d<T, access::read_write>` (`access::write`
  for `vec4`), `read`, `write` and `atomic_fetch_add`, which takes MSL 3.1
  (macOS 14). A store is not seen by a load in the same invocation on
  Metal without a texture fence: read what a dispatch stored in a later one.
- `MutSlice<T>.atomicAdd(int i, T v) : T` on a stage's `MutSlice<u32>` or
  `MutSlice<i32>` storage buffer: adds atomically and returns the element
  before (`OpAtomicIAdd` on the element, `InterlockedAdd`,
  `atomic_fetch_add_explicit` with relaxed order). A read-only buffer
  (`Slice<T>`, or a `view`) has none. On the CPU it is atomic as well
  (32-bit elements through their 8-byte word).
- `Rgba8` and `toVec4()`; `discard()`.
- `ddx(x)`, `ddy(x)` and `fwidth(x)` for `f32` and `vec2`–`vec4`:
  screen-space derivatives (how much x changes to the next pixel right
  and down, and the sum of their sizes), in fragment stages only (the
  checker reports them elsewhere): `OpDPdx`/`OpDPdy`/`OpFwidth`, HLSL's
  `ddx`/`ddy`/`fwidth` and MSL's `dfdx`/`dfdy`/`fwidth`. Coarse and fine
  variants are not there yet.
- `fract`, `mix`, `step`, `smoothstep`, `saturate`, `exp`, `exp2`, `ln`,
  `log2`, `inverseSqrt`, `trunc`, `atan`; and on `vec2`–`vec4`: `floor`,
  `ceil`, `round`, `fract`, `sqrt`, `clamp`, `saturate`, `step`, and
  `reflect` on `vec3`; `vec4(vec2, f32, f32)`.

## 3. Getting the code into a program: `core.shader`

`core.shader(f)` is a compile-time intrinsic. Its argument names a stage
function (`core.shader(spriteVs)`, `core.shader(gfx.uiFs)`), and its value
is a `ShaderCode` (prelude, `std/core/shader.bi`) whose strings point at
constant data in the executable: nothing is allocated at run time, and no
tool is needed at build time. Where the platform's shader compiler is on
the build host it also holds that backend's binary (§3.1), so the program
compiles no shader source when it starts.

```bismuth
struct ShaderCode {
    ShaderStage stage;      // Vertex, Fragment, Compute
    str name;               // the Bismuth function
    str entry;              // entry point in all three languages: <name>_main
    str spirv;              // SPIR-V 1.3 module bytes (Vulkan 1.1)
    str msl;                // MSL 2.0 source
    str hlsl;               // HLSL source for vs_5_0/ps_5_0/cs_5_0
    str dxbc;               // the HLSL as DXBC, when bic ran on Windows; else ""
    str metallib;           // the MSL as a Metal library, when bic ran on a Mac with Xcode; else ""
    i32 groupX; i32 groupY; i32 groupZ;            // compute workgroup size
    i32 bufferCount;  [8]ShaderVertexBuffer buffers;   // stride, instance
    i32 attrCount;    [16]ShaderVertexAttr attrs;      // location, buffer, offset, format
    i32 bindingCount; [16]ShaderBinding bindings;      // kind, slot, size, set, binding, metal, hlsl
}
```

`code.binding(ShaderBindingKind.Texture, 0)` finds a binding by kind and
slot.

### 3.1 Precompiled binaries

Each backend gets bytecode it can create a shader from without a compiler
when the program has it:

| Backend | Binary | Made by bic where | Without it |
|---|---|---|---|
| Vulkan | `spirv` | everywhere (bic emits SPIR-V itself) | — |
| D3D11, D3D12 | `dxbc` (SM 5.0) | Windows: `D3DCompile` from `d3dcompiler_47.dll`, `/O3`, strict | HLSL compiled at load, then kept in the shader cache |
| Metal | `metallib` | macOS with Xcode: `xcrun metal` (`-mmacosx-version-min=11.0`) and `metallib` | `newLibraryWithSource` at load |

`bic build` and `bic run` precompile for the target when the host can
(a Windows build of a Windows program, a macOS build of a macOS one). A
stage that fails to compile is a warning and loads from source. bic keeps
what it compiled under the user's cache folder (`bismuth/shaders`), keyed
by the compiler, its options and the source, so a rebuild compiles only
the stages that changed. `BIC_SHADER_BINARIES=0` turns it off.

At load, std.gfx takes the binary, else the source:

- **D3D11/D3D12**: `dxbc` when it is a whole DXBC container; else the
  shader cache, `BISMUTH_SHADER_CACHE` (a folder, or `off`) or
  `%LOCALAPPDATA%\bismuth\shader-cache`, whose files hold the compiled
  source as well as the bytecode, so a hash collision is a miss; else
  `D3DCompile` with bic's options, saved to the cache. A cross-compiled
  Windows program compiles its HLSL on its first run only.
- **Metal**: `newLibraryWithData:error:` on `metallib`; if there is none,
  or the device refuses it, `newLibraryWithSource`.
- `BISMUTH_SHADER_BINARIES=0` ignores `dxbc` and `metallib`, to test the
  fallback.

`dev.sourceCompiles()` counts the shaders a device compiled from source;
a build made on Windows and on macOS keeps it at 0 (`tests/gpu/gfxdraw.bi`
checks it on every backend CI runs).

### 3.2 Shader packages: `bic shader`

`bic shader [-o dir] [--entry fn] file.bi` writes each stage as
`<fn>.bishader`, a package with everything `ShaderCode` holds, and as
`<fn>.spv`, `<fn>.metal` and `<fn>.hlsl` (with `<fn>.dxbc` and
`<fn>.metallib` where it made them), for tools like spirv-val, RenderDoc
or Xcode's shader debugger. It precompiles for every backend the host
can, whatever the target. `gfx.loadShader(path) : ShaderCode ! GfxError`
reads a package (`gfx.parseShader(bytes)` one in memory), for shaders a
program was not compiled with: mods, tools, hot reload.

```bismuth
let vs = gfx.loadShader("shaders/waterVs.bishader")?;
let fs = gfx.loadShader("shaders/waterFs.bishader")?;
let pip = dev.pipeline({ vertex: vs, fragment: fs })?;
```

A package is little-endian: `BISHADER`, version 1 (u32), the stage; the
name, entry, SPIR-V, MSL, HLSL, DXBC and metallib, each a u32 length and
its bytes; the workgroup size; then the vertex buffers (stride, instance),
attributes (location, buffer, offset, format) and bindings (kind, slot,
size, set, binding, metal, hlsl), each a u32 count and i32s, with
`ShaderCode`'s enum values.

### In the renderer (#16)

A `std.gfx` pipeline takes two `ShaderCode`s and builds everything from
them ([graphics.md §2](graphics.md#the-api-as-built-16)):

```bismuth
let pip = dev.pipeline({ vertex: core.shader(gfx.spriteVs), fragment: core.shader(gfx.spriteFs), blend: .Alpha })?;
```

- **Vulkan**: `vkCreateShaderModule(vs.spirv)`, entry `vs.entry`; vertex
  input bindings from `vs.buffers` (binding i, stride, input rate) and
  attributes from `vs.attrs` (location, binding, format, offset); one
  descriptor set layout per stage (set 0 from `vs.bindings`, set 1 from
  `fs.bindings`, with the stage's visibility), uniform blocks as
  `UNIFORM_BUFFER_DYNAMIC` so draws only change offsets, storage
  buffers as `STORAGE_BUFFER`.
- **D3D11/D3D12**: `vs.dxbc`, or the HLSL compiled (§3.1), once per
  `dev.shader` (pipelines made with `dev.pipelineWith` share it);
  input layout elements `TEXCOORD`, semantic index = location, input
  slot = buffer, per-instance from `buffers[i].instance`; registers from
  `bindings[k].hlsl`. The D3D12 root signature has a root constant buffer
  view per uniform block and a root SRV or UAV per storage buffer, then
  per stage a descriptor table of textures (a range per texture at its t
  register, since read-only storage buffers take t registers too) and one
  of samplers (s0..). Direct3D 11 binds storage buffers as structured
  buffers' views; their stride is `bindings[k].size`.
- **Metal**: `newLibraryWithData(vs.metallib)`, or
  `newLibraryWithSource(vs.msl)` (§3.1), function `vs.entry`;
  `MTLVertexDescriptor` attribute `location` at `bufferIndex 16 + buffer`;
  buffers, textures and samplers at `bindings[k].metal`.
- **Compute** (#63): `dev.computePipeline({ compute: core.shader(f) })`
  takes a `@compute` stage, with its bindings as a vertex stage's (set 0
  on Vulkan); `groupX..Z` is the threadgroup size Metal needs at dispatch
  (Vulkan and D3D compile it into the shader). A stage's storage
  parameters, of either kind in parameter order, read
  `BindingsDesc.storage`; its storage images (`StorageImage2D<T>`) read
  `BindingsDesc.images`, counted with its textures.

The built-in shaders are in `std/gfx/shaders.bi`: `colorVs`/`colorFs`
(`ColorVertex` = `vec3 pos; vec4 color`, uniform `Transform { mat4 mvp }`),
`spriteVs`/`spriteFs` (`SpriteVertex` = `vec2 pos; vec2 uv; vec4 color`,
`Transform`, texture and sampler) and `uiVs`/`uiFs` (`UiVertex` = `vec2 pos;
vec2 uv; Rgba8 color`, 20 bytes; `UiUniforms { vec2 scale; vec2 offset }`
with `UiUniforms.forScreen(w, h)`; an atlas texture and sampler) for the
GUI (#21, `std.ui`: see [graphics.md §5](graphics.md#std-ui-the-first-slice)).
`mipVs`/`mipFs` (a triangle over the target, from `@vertexIndex`, and a
`sampleLevel` of the level above) make mip levels on Direct3D 12, which
has no blit that filters.

## 4. How bic compiles them

```
checked AST of the stage and its callees ──builder──▶ shader IR ──▶ SPIR-V (binary)
     (compiler/sbuild.bi)                  (compiler/mshader) ├──▶ MSL (text)
                                                              └──▶ HLSL (text)
```

The stage is checked like any function; after the whole program is
checked, `checkShaders` builds a small typed and structured IR from the
checked AST of each stage and everything it calls, then runs the three
emitters. `core.shader` lowers to a `ShaderCode` constant.

**Why not from bic's SSA IR**, as §3 of graphics.md first proposed: that IR
is the right input for CPUs and the wrong one for GPUs. By the time code
is in it, vectors and matrices are byte offsets into stack slots, struct
fields are `PtrAdd`s, and the optimizer has turned loops and branches into
an arbitrary CFG. SPIR-V for Vulkan needs typed values, typed access
chains and structured control flow (selection and loop merges), and the
text languages need the same structure to print. The shader IR keeps those
(vector, matrix, struct and array values, member and element accesses,
`if`/`loop`/`break`/`continue`), and recovering them from the general IR
would be a decompiler. The CPU side still goes through the SSA IR: the
same functions are compiled by the normal pipeline and run in the IR
interpreter.

**Mapping.** `vecN`/`ivecN`/`mat4` become native vectors and matrices.
Calls of prelude vector and matrix methods become operators and built-ins;
the C math functions std's bodies call (`sinf`, `floorf`, `powf`, ...)
become GLSL.std.450 instructions, `sin`, `floor`... Other functions become
helper functions in the output, callees first. Expression blocks, `if`
values and lazy operands that need statements are spilled into locals. In
SPIR-V, locals are `Function` variables (drivers promote them), `&&`, `||`
and `?:` are selection constructs joined by `OpPhi`, and uniform and
storage data have their own explicitly laid-out types (`Offset`,
`ArrayStride`, `MatrixStride`) converted member by member to the logical
types code uses. The text emitters do the same with `_layout` structs and
conversion functions, `make_T` constructors in HLSL, and statements in
place of HLSL's eager `?:`/`&&` when an operand can have effects.

**Verification.** The compiler's unit tests
(`compiler/tests/comp_shader.bi`) validate every output of a set of shader
programs (and the built-ins) with `spirv-val --target-env vulkan1.1` and
HLSL with `glslangValidator -D` (when installed), and check MSL's shape
(no Metal compiler on Linux; macOS runs come with the renderer). A
compute shader runs natively on the CPU, in the IR interpreter and through
Vulkan (lavapipe in CI) from the SPIR-V bic wrote, and the three results
are compared; and std.gfx's color and GUI shaders draw the same triangles
in a software rasterizer that calls them as functions and through Vulkan
into an offscreen image, pixel for pixel (`tests/rt_gpu.bi`, with small
dlopen-based harnesses in `tests/gpu`). On Windows the HLSL is compiled by
D3DCompile when bic builds (§3.1) and drawn through D3D11 and D3D12 by the
same tests (`tools/ci/check-gui.sh`); the MSL is compiled by Xcode's
`metal` on the opt-in macOS run, which has not run yet (#26).

## 5. Later

- `match` on plain enums and integers, `ref` parameters (SPIR-V function
  pointers, HLSL `inout`, MSL references), `ref self` methods.
- Workgroup memory and barriers for compute; atomics other than `atomicAdd`
  (min, max, exchange, compare-exchange); storage images of other formats
  and kinds (arrays, cubes, 3D). (A depth image is read through a
  `Texture2D`, its depth in `.r`, or through a `DepthTexture2D` with a
  comparison sampler; cube and array images through `TextureCube` and
  `Texture2DArray`; storage images through `StorageImage2D<T>`.)
- Push constants / root constants for small per-draw data.
- DXIL (SM 6) through dxc, when std.gfx needs SM 6 features.
- Specialization constants (pipeline-time `const`s).
