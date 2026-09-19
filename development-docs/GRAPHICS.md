# Graphics Subsystem

This document explains how `goro` turns Ragnarok Online map data into pixels
on screen. It assumes you can program but have never touched a graphics or
game engine before, so every concept is introduced in plain language before
we look at the code that implements it in this repository.

It intentionally does **not** cover the overall application architecture
(config, networking, sessions, the UI framework) — see
`development-docs/ARCHITECTURE.md` for that. This document goes deep on one
thing: how the world, actors, effects, and models get from data files to the
screen.

All line numbers are correct as of the commit this document was written
against; if the code has moved, use them as a starting point for a search
rather than a guarantee.

## Table of contents

1. [Graphics 101 — vocabulary used throughout](#graphics-101)
2. [The render pipeline foundation](#the-render-pipeline-foundation)
3. [Shaders](#shaders)
4. [World geometry: GND, GAT, RSW](#world-geometry-gnd-gat-rsw)
5. [3D models: RSM and GR2](#3d-models-rsm-and-gr2)
6. [Sprites and 2D actor rendering](#sprites-and-2d-actor-rendering)
7. [The effects system](#the-effects-system)
8. [Lighting, fog, and water](#lighting-fog-and-water)
9. [The camera](#the-camera)
10. [Simple overlays: cursor, damage numbers, equipment view](#simple-overlays-cursor-damage-numbers-equipment-view)
11. [CPU rasterization vs. the GPU path](#cpu-rasterization-vs-the-gpu-path)
12. [Palettes and image decoding](#palettes-and-image-decoding)
13. [How UI compositing fits in](#how-ui-compositing-fits-in)
14. [Testing graphics code](#testing-graphics-code)
15. [Where to start](#where-to-start)
16. [Glossary of RO format acronyms](#glossary-of-ro-format-acronyms)

## Graphics 101

A **mesh** is just a list of points in 3D space (**vertices**) plus a list of
triangles connecting them (**indices**, three per triangle, referencing
vertices by position in the list). The ground you walk on, a tree, a
character's body — all of it is triangles.

A **draw call** is one instruction to the GPU: "draw these triangles, using
this texture, with these blending rules." GPUs are extremely fast at drawing
triangles but relatively slow at *switching state* between draw calls (a new
texture, a new blend mode). This is why engines **batch**: group triangles
that share the same texture/settings into one buffer and issue one draw call
for all of them instead of one call per object. You'll see `goro` batch
world geometry by texture in several places below.

A **shader** is a small program that runs on the GPU for every vertex
(`vertex shader`: transform a 3D point into 2D screen space) and every pixel
covered by a triangle (`fragment shader`: decide that pixel's final color,
usually by sampling a texture). `goro` writes its shaders in **WGSL**
(WebGPU Shading Language), the shading language of the `wgpu` graphics API.

A **texture** is an image living in GPU memory that a shader can sample
(look up a color at a UV coordinate, `(0,0)`–`(1,1)` across the image). A
**texture atlas** packs many small images into one big texture so multiple
objects can share a single draw call/texture binding instead of one each —
see [the ground lightmap atlas](#lighting-fog-and-water) for a concrete
example in this codebase.

**Indexed color / a palette** is an old-school image format where each pixel
stores a small index (0–255) into a 256-color lookup table instead of
storing full RGB values. It saves space and, critically for RO, lets the
client swap the palette to recolor a sprite (different armor tints, dyed
hair) without touching the pixel data. `goro`'s `res.Palette` type is this
256-entry lookup table.

A **billboard sprite** is a flat, textured quad (two triangles forming a
rectangle) that is rotated at draw time to always face the camera — like a
cardboard cutout. RO's characters, monsters, NPCs, and most visual effects
are 2D artwork drawn this way inside an otherwise 3D world; the alternative,
a full 3D character mesh, is only used for the newer "GR2" models (see
[§5](#3d-models-rsm-and-gr2)).

**Skeletal animation** poses a 3D mesh by attaching each vertex to one or
more named **bones** with a weight, then moving the bones. A vertex's final
position each frame is a weighted blend of the transforms of the bones that
influence it (that per-vertex blend is called **skinning**). Bones move over
time by interpolating between **keyframes** — recorded bone poses at
specific points in an animation clip.

The **camera** defines what part of the 3D world is visible and how it maps
onto the 2D screen. Two pieces combine into a **view-projection matrix**: the
*view* matrix moves the world so the camera sits at the origin looking down
some axis, and the *projection* matrix (here, a **perspective** projection,
which makes far things look smaller, controlled by a field-of-view angle)
squashes 3D space into the `[-1, 1]` **clip space** cube the GPU expects.
Multiplying a vertex's world position by this matrix is how the vertex
shader decides where on screen it lands.

**Z/depth testing** (and its buffer, the **depth buffer**) is how a GPU knows
which of two overlapping triangles is in front: each pixel remembers how far
away the triangle that last colored it was, and a new triangle only wins if
it's closer. `goro`'s custom **depth bias** and "occlusion" tricks (in the
world shader) exist to work around depth-fighting between the 2D sprites and
3D ground in ways plain z-testing handles badly — more in [§3](#shaders).

## The render pipeline foundation

### The `render` package's job

Per its own `doc.go` comment, `render` "wraps the gogpu backend and provides
drawing primitives used by game... Image is CPU texture data. Frame is a
per-frame command buffer consumed by the gogpu renderer." Concretely:

- **`Image`** (`render/image.go:9`) is a thin wrapper around Go's standard
  `image.RGBA` — pixels living in normal CPU (system) memory, plus a
  `version` counter that increments whenever the pixels change
  (`render/image.go:76`). Sprite frames, composited character images, ground
  textures, lightmap atlases — all start life as an `Image`.
- **`Frame`** (`render/frame.go:11`) is *not* pixels — it's a list of
  drawing instructions ("draw commands") accumulated during one game update,
  to be handed to the GPU renderer once per frame. Game code never touches
  the GPU directly; it just appends to a `Frame` via methods like
  `DrawImage`, `DrawTriangles3D`, `DrawWorldMesh`, `DrawWorldBillboard`.

A `Frame` separates commands into several buckets (`render/frame.go:18-25`):
`commands` (flat 2D screen-space triangles — UI, sprites drawn without a 3D
camera), `worldCommands` (3D triangles submitted fresh every frame — RSM
props, dynamic effects), `worldMeshes` (references to pre-built, cached 3D
meshes — static terrain), `worldBillboards` (camera-facing sprite quads —
characters, monsters, particle effects), plus separate UI-only buckets for
rects/text/labels drawn as an overlay (see [§13](#how-ui-compositing-fits-in)).

### The frame loop

`render.Run` (`render/backend.go:292`) is the windowed entry point called
from `main.go`. It creates a `gogpu.App` (the underlying window +
GPU-context library), wires up input event fan-out (`fanoutEventSource`,
`render/backend.go:451`), and registers three callbacks on the `gogpu.App`:

- `OnResize` → tells the `Game` interface to resize (`render/backend.go:369`).
- `OnUpdate` → calls `r.update()` once per tick, which calls `game.Update()`
  and then advances the `gogpu/ui` widget tree one frame
  (`render/backend.go:377`, `render/backend.go:669`).
- `OnDraw` → calls `r.draw(ctx)` once per frame (`render/backend.go:383`).

`r.draw` (`render/backend.go:771`) is the heart of the pipeline each frame:

1. Ensure a `*Frame` (`r.screen`) exists at the current window size, creating
   the `gpuRenderer` lazily on first draw (`render/backend.go:791`).
2. `r.screen.BeginFrame()` clears last frame's command buckets
   (`render/frame.go:53`).
3. `r.game.Draw(r.screen)` — **this is where all `game`-package drawing code
   runs.** Everything in this document downstream of this call is code that
   fills the `Frame` with commands; it never touches the GPU.
4. Draw the `gogpu/ui` widget tree as a separate 2D overlay image
   (`r.drawUI`, see [§13](#how-ui-compositing-fits-in)).
5. Let the game draw further overlays (tooltips, damage floaters, cursor)
   via optional interfaces (`overlayDrawer`, `uiOverlayDrawer`,
   `render/backend.go:153-159`).
6. Hand the finished `Frame` to `r.gpu.Draw(ctx, r.screen)` — the one place
   the accumulated commands actually become GPU draw calls
   (`render/gpu_renderer.go:486`).

The `Game` interface (`render/backend.go:35`) is the seam between `render`
and everything else: `Update() error`, `Draw(*Frame)`, `Resize(w, h int)`,
`InputState() *input.State`. `app.New` constructs a value implementing this
interface (see `ARCHITECTURE.md`); `render` knows nothing about Ragnarok
Online, only about this four-method contract.

### Graphics API selection

`render.Run` picks a `gogputypes.GraphicsAPI` from the `--graphics-api`
config string via `graphicsAPI()` (`render/backend.go:432`): `vulkan`
(default), `auto`, `dx12`, `metal`, `gles`, or `software`. This is passed
into `gogpu.DefaultConfig().WithGraphicsAPI(api)` — `goro` itself never
speaks Vulkan/DX12/Metal directly; the `gogpu`/`wgpu` libraries (external
Go modules, pure-Go reimplementations of a wgpu-like GPU abstraction) own
that translation. `goro`'s own GPU code (`render/gpu_renderer.go`) is written
once against the `github.com/gogpu/wgpu` Go API and works unchanged across
backends.

### Headless mode

`render.RunHeadless` (`render/headless.go:12`) runs the exact same
`Update`/`Draw`/"submit frame" cycle on a 60Hz ticker with **no window and
no GPU at all** — it calls `game.Draw(screen)` into a `*Frame` and then just
discards it. This exists so the game logic, resource loaders, and world
simulation can run in CI and automated tests without a display or a GPU
driver (see `app/headless_test.go`, `config/headless_test.go`). It proves
`render.Frame` genuinely decouples "what to draw" from "how to draw it" —
headless mode exercises the former and skips the latter entirely.

## Shaders

`goro` has exactly three WGSL shaders, all defined as Go string constants in
`render/gpu_shaders.go`:

1. **`screenShaderWGSL`** (`render/gpu_shaders.go:3`) — flat, non-3D
   textured quads. Vertex shader maps pixel coordinates directly to clip
   space (`render/gpu_shaders.go:28-29`, no camera matrix at all — screen
   pixels in, `[-1,1]` clip space out). Fragment shader samples the texture,
   multiplies by a per-vertex color, and discards near-fully-transparent
   pixels (`render/gpu_shaders.go:39-41`). Used for UI, 2D sprite drawing
   without a camera, and general flat triangles.

2. **`worldShaderWGSL`** (`render/gpu_shaders.go:46`) — 3D world geometry
   (terrain, RSM props). The vertex shader multiplies the vertex position by
   a `Uniforms` struct holding the camera's view-projection matrix as four
   `vec4` columns (`c0..c3`, `render/gpu_shaders.go:83`). It also computes a
   **second** clip-space position from a separate `depth_pos` attribute and
   uses the *lesser* of the two depths, minus a small `depth_bias`
   (`render/gpu_shaders.go:84-89`). This is a manual depth-occlusion trick:
   it lets a triangle be drawn at one 3D position for depth-testing purposes
   while its "real" position differs slightly — used to stop thin ground
   decals and sprites from z-fighting with the terrain (see `DepthBias` in
   `render/types.go:67` and its callers). The fragment shader samples both
   the surface texture and a **lightmap texture** at a second UV set
   (`light_uv`), combines them (`color.rgb * lightmap.a +
   clamp(lightmap.rgb)`, `render/gpu_shaders.go:102`), and finally blends in
   fog color based on view-space depth using `smoothstep`
   (`render/gpu_shaders.go:106`).

3. **`worldBillboardShaderWGSL`** (`render/gpu_shaders.go:116`) — camera-
   facing quads. Instead of taking vertex positions directly, each instance
   carries a `center` (3D anchor point), `right_axis`/`up_axis` (already
   camera-aligned direction vectors, computed on the CPU per §9), and
   `params` (width/height/anchor offset). The vertex shader builds the
   final 3D position as `center + right*pixelX + up*pixelY`
   (`render/gpu_shaders.go:158-159`) — i.e. the CPU tells the GPU *which
   directions* count as "sideways" and "up" for this billboard (already
   facing the camera), and the shader just places the four corners along
   those directions. It reuses the same depth-occlusion and fog logic as
   the world shader.

There is **no runtime shader cross-compilation** in the shipped binary —
these three WGSL strings are handed straight to
`device.CreateShaderModule` (`render/gpu_renderer.go:214-237`), and the
underlying `gogpu`/`wgpu` stack translates WGSL to whatever the selected
backend needs. `github.com/gogpu/naga` (the WGSL parser/lowering/SPIR-V
codegen library) only shows up in `render/gpu_renderer_test.go:22-46`, where
a test parses the `worldShaderWGSL` source with `naga.Parse` +
`naga.Lower` + `naga.GenerateSPIRV` purely as a **validation check** that the
hand-written WGSL is well-formed — it is not part of the runtime render
path.

### Render pipelines and blend modes

For each shader `goro` builds **multiple `wgpu.RenderPipeline` objects**, one
per combination of blend mode and depth-write behavior, because a GPU
pipeline bakes in fixed settings like blending:

- `render.Blend`: `BlendSourceOver` (normal alpha blending — the new pixel
  covers the old one proportionally to its alpha), `BlendLighter` (additive
  — colors add together, used for bright glow effects like magic spells),
  `BlendSrcAlphaDstAlpha` (a symmetric alpha blend used for some overlay
  effects). See `render/types.go:21-27` and `gpuRenderer.pipeline`
  /`worldPipelineFor` (`render/gpu_renderer.go:1239-1266`) which pick the
  right pre-built pipeline for a given `Blend` value.
- Depth write vs. depth read-only: static, opaque terrain writes to the
  depth buffer (`worldAlphaWrite` etc., so later translucent things are
  correctly occluded by it); most dynamic/translucent world geometry only
  *reads* depth (`worldAlphaRead` etc.) so it doesn't block other
  translucent things behind it. Billboards additionally have a "no depth
  test at all" variant (`billboardAlphaNoDepth` etc.) for effects that
  should always render on top.

All of these pipelines are created once in `gpuRenderer.init`
(`render/gpu_renderer.go:196`) and picked per-draw-call by key — this is the
"switching GPU state is expensive" trade-off from §1: build every
combination up front, then just *select* one per batch instead of mutating
GPU state live.

## World geometry: GND, GAT, RSW

Ragnarok Online maps are described by three separate files, each parsed by
a small binary-format reader in `res/`:

- **RSW** (`res/rsw.go`, `type RSW`, `res/rsw.go:10`) — the "scene" file.
  Points at the GND/GAT files for this map (`RSWFiles`, `res/rsw.go:24`),
  and lists everything placed *in* the scene: static prop models
  (`RSWModel`, filename + position/rotation/scale, `res/rsw.go:61`), point
  lights (`RSWObjectLight`), ambient sound emitters (`RSWSound`), particle
  effect spawners (`RSWEffect`), plus global map settings — directional
  light angle/color (`RSWLight`, `res/rsw.go:40`), default water plane
  (`RSWWater`, `res/rsw.go:31`), and map bounds (`RSWGround`).
  `ParseRSW` (`res/rsw.go:101`) is version-gated field-by-field (`if
  rsw.versionAtLeast(1, 8) { ... }`) because the binary layout grew new
  fields across the RO client's history.

- **GND** (`res/gnd.go`, `type GND`, `res/gnd.go:11`) — the ground mesh: a
  `Width × Height` grid of `GNDCell`s (`res/gnd.go:49`), each holding four
  corner heights (`Heights [4]float32`) and indices into three shared
  tables: `Surfaces` (`GNDSurface`, UV coordinates + which texture +
  which lightmap, `res/gnd.go:41`), `Textures` (ground texture file names),
  and `Lightmaps` (`GNDLightmap`, an 8×8 grid of precomputed brightness +
  color per cell, `res/gnd.go:36` — see [§8](#lighting-fog-and-water)). A
  `GNDCell` can reference up to three surfaces (`Top`/`Front`/`Right`,
  `res/gnd.go:52-54`) for the top face plus the two visible side walls
  where terrain height steps down, which is how RO renders cliffs without a
  full voxel system. `GND` also optionally carries a water plane
  (`GNDWater`, `res/gnd.go:24`, present on newer map format versions).

- **GAT** (`res/gat.go`, `type GAT`, `res/gat.go:16`) — the *collision and
  pathfinding* grid, completely separate from the visual GND mesh (GAT and
  GND commonly use different cell resolutions/origins). Each `GATCell`
  carries four corner heights (used for terrain height sampling — see
  `terrainHeightAt`, `game/scene_projection.go:269`) and a `Type` bitmask
  (`GATTypeWalkable`, `GATTypeWater`, `GATTypeSnipable`,
  `res/gat.go:10-14`) computed from the raw file value by `GATCellType`
  (`res/gat.go:100`). `GAT` never touches rendering directly, only movement
  and line-of-sight logic (`world`/`game` packages) — but its heights are
  also used as a fallback terrain-height source when a map has no GND (see
  `terrainHeightAt`).

### From GND cells to GPU triangles

Drawing raw per-frame terrain triangles every frame would be wasteful —
terrain is static. `game/static_world_mesh.go` builds a **retained** (i.e.
built once, reused across frames) mesh:

1. `WorldMode.drawGNDMeshes` (`game/static_world_mesh.go:86`) keeps a
   `gndRetainedMeshCache` keyed by which `*GND`/`*RSW` is currently loaded,
   split into fixed-size **chunks** (`gndRetainedChunkSize` cells square).
   Only chunks that overlap the camera's visible ground footprint
   (`gndDrawBounds`, `game/gnd.go:64`, itself derived from projecting the
   camera frustum onto the ground plane — `cameraGroundFootprint`,
   `game/gnd.go:159`) are built or drawn.
2. Each chunk is built once by `buildGNDMeshChunk`
   (`game/static_world_mesh.go:124`): it walks every cell in the chunk's
   range and groups triangles into a `retainedMeshBuilder` **per (texture,
   lightTexture, draw options) key** — this is the batching from §1 again:
   two cells using the same ground texture end up in the same mesh, one
   draw call.
3. `buildGNDLightmapAtlas` (`game/static_world_mesh.go:285`) packs *all* of
   a map's 8×8 `GNDLightmap`s into one big `Image` atlas up front (rounded
   up to a power-of-two size) so a chunk's mesh only ever needs a single
   lightmap texture binding regardless of how many distinct lightmap
   entries its cells reference — a textbook texture-atlas use.
4. The resulting per-chunk meshes are `render.WorldMesh` objects
   (`render/world_mesh.go:3`), submitted via `screen.DrawWorldMesh(mesh)`
   (`render/frame.go:183`) — these land in the `Frame.worldMeshes` bucket.

On the GPU side, `render/gpu_world_mesh_batch.go` and
`render/gpu_world_mesh_buffer.go` add a **second** layer of batching and
caching specifically for these retained meshes:

- `gpuRenderer.depthWriteWorldMeshBatches` (`render/gpu_renderer.go:931`)
  groups the *currently submitted* `WorldMesh` commands by their draw key
  (texture/lightTexture/options) into `worldMeshBatch`es, reusing the
  previous frame's grouping when the submitted set hasn't changed
  (`worldMeshSubmissionCache.matches`, `render/gpu_world_mesh_batch.go:17`)
  — so unchanged terrain costs nothing to re-batch frame to frame.
- Each batch's vertex/index data is uploaded once into a shared, growable
  GPU buffer pool (`worldMeshBufferPage`, `render/gpu_world_mesh_buffer.go`)
  using a simple free-list allocator (`page.allocate`/`allocation.release`,
  `render/gpu_world_mesh_buffer.go:29`), and only re-uploaded when the
  batch's mesh set or size actually changes (`gpuWorldMeshBatch.matches`,
  `render/gpu_world_mesh_batch.go:57`). This is a classic "GPU buffer arena"
  pattern: allocate large pages once, hand out ranges, coalesce them back
  into the free list on release, to avoid constant buffer
  creation/destruction as the camera moves and chunks enter/leave visibility.

Contrast this with **dynamic** 3D geometry (RSM props with animated
textures/nodes, per-frame procedural effect geometry): that goes through
`screen.DrawTriangles3DOwned` (`render/frame.go:167`) into
`Frame.worldCommands` instead, which `gpuRenderer.buildWorldFrame` batches
and re-uploads to a single dynamic vertex/index buffer **every frame**
(`r.dynamicBuffer`, `render/gpu_renderer.go:1202`) — simpler, but pays an
upload cost each frame, which is fine because these buffers are typically
much smaller than the whole terrain.

### Water

Water is drawn as its own quad per visible GND cell, not part of the
retained terrain mesh (because it animates every frame). `WorldMode.
drawGNDWater` (`game/gnd.go:36`) walks the visible cell range, and for each
cell below the water threshold (`waterVisibleForCell`, `game/gnd.go:465`)
computes a slightly wavy quad (`waterHeightsForCell`, `game/gnd.go:452`,
a sine wave driven by wall-clock time) textured with one of 32 looping water
animation frames (`waterFrameForTime`, `game/gnd.go:473`, `res.
WaterTextureCandidates`, `res/image.go:125`) and submits it via
`drawTexturedSurface3DAlpha` (defined in `game/world.go:2099`, which just
builds a `render.Vertex3D` quad and calls `screen.DrawTriangles3DOwned`) —
i.e. water rides the same "dynamic world geometry" path as RSM props.

## 3D models: RSM and GR2

RO ships two unrelated 3D model formats, used for different things:

### RSM — static/animated scene props

**RSM** (`res/rsm.go`, `type RSM`, `res/rsm.go:10`) is RO's original
"3D model" format for scenery — trees, rocks, buildings, fences, anything
placed via an `RSWModel` entry. An `RSM` is a tree of named `RSMNode`s
(`res/rsm.go:25`), each with its own vertices/faces/texture references and
*optional* keyframe animation (`RSMPositionKeyframe`,
`RSMRotationKeyframe`, `RSMScaleKeyframe`) — so a windmill's blades can be
one animated child node of an otherwise static model. There is no bone
skinning here; each node moves as a rigid whole, and node transforms
compose down the hierarchy (parent/child), similar to how a scene graph or
a simple robot-arm rig works. `game/rsm_render.go` reads this per placement:

- `WorldMode.visibleRSMPlacements` (`game/rsm_render.go:96`) culls
  `RSWModel` placements outside the camera's visible ground footprint using
  a spatial grid (`rsmPlacementGrid`) for fast lookup.
- `WorldMode.rsmMeshesForPlacement` (`game/rsm_render.go:179`) builds
  **retained** `WorldMesh`es for models that have no node animation
  (`rsmHasNodeAnimation`, `game/rsm_render.go:756`) — most scenery, trees,
  buildings — reusing exactly the same retained-mesh/GPU-batch machinery
  described above for GND.
- Models that *do* animate (like windmills) instead go through
  `drawAnimatedRSMPlacement` (`game/rsm_render.go:343`) → `
  drawAnimatedRSMNodeTriangles` (`game/rsm_render.go:376`), which rebuilds
  triangles fresh every frame from the current animation-sampled node
  matrices (`buildRSMNodeMatrices`, `game/rsm_render.go:670`, computed by
  interpolating between keyframes — `sampleRSMPosition`/`sampleRSMScale`/
  `sampleRSMRotation`).

### GR2 — skeletal character models

**GR2** is the Granny3D format Gravity's later client used for some
non-player character models (see `res.IsGR2ResourceName`,
`game/sprite_assets.go:63`, which decides per-actor whether to use this path
instead of 2D sprites). Unlike RSM, GR2 supports real **bone/skeleton
skinning**.

`res/gr2.go` (1500+ lines) implements a general-purpose reader for
Granny3D's container format: `ParseGR2Container` (`res/gr2.go:49`) parses a
**self-describing binary structure** — the file embeds its own type
descriptors (`gr2Nav.typeSize`/`gr2Nav.members`, `res/gr2.go:328`,`289`),
so the parser walks a generic reflection-like `member`/`field` tree
(`gr2RawMember`, `gr2Field`, `res/gr2.go:241-254`) rather than a fixed
struct layout — this is standard for how Granny3D files work, and is by
far the most complex parser in the codebase.

Two supporting files handle compression Granny3D files can use for embedded
data:

- `res/gr2_oodle.go` implements a small range-decoder based decompressor
  for **Oodle**-compressed geometry data blocks
  (`decompressGR2Oodle`, `res/gr2_oodle.go:112`).
- `res/gr2_bink.go` implements a **Bink**-encoded texture pixel decoder
  (`decodeGR2Bink`, `res/gr2_bink.go:595`) — a wavelet-based image codec
  (Haar/coarse-detail passes, `res/gr2_bink.go:431-552`) used for textures
  embedded directly inside `.gr2` files instead of as separate image files.

Once parsed, geometry is normalized into a flatter, render-friendly shape by
`res/gr2_geometry.go` (`res.GR2Geometry`, with `Batches` grouping triangle
index ranges by texture — the same "group by texture" batching idea as
everywhere else in this codebase).

Animation and skinning live in `game/gr2_animation.go`:

- `gr2SkeletonPoseFromModel` (`game/gr2_animation.go:37`) builds a
  `gr2SkeletonPose` — each bone's bind-pose (rest position) transform.
- `gr2AnimationClipFromFile` (`game/gr2_animation.go:113`) extracts a
  `gr2AnimationClip` — per-bone animation curves (position/rotation/scale
  over time, evaluated with a B-spline-like De Boor algorithm,
  `gr2Deboor2`, `game/gr2_animation.go:405`, for smooth interpolation
  between keyframes).
- `gr2AnimationClip.skinningPalette` (`game/gr2_animation.go:145`) is the
  per-frame **skinning**: for the current elapsed animation time, evaluate
  every bone's animated local transform, compose it with its parent
  (walking the skeleton hierarchy) to get a world matrix, then combine with
  the bone's inverse bind pose to produce a final "palette" of matrices —
  one 4×4 matrix per bone, ready to be applied to vertices.
- `gr2SkinnedPoint`/`gr2SkinnedNormal` (`game/gr2_animation.go:253`,`283`)
  apply that palette to a single vertex, blending by the vertex's bone
  weights (a `res.GR2ModelVertex` carries up to 4 bone indices + weights).

The actual per-frame draw, `WorldMode.drawNonPCGR2Model3D`
(`game/gr2_model.go:38`), is intentionally simple once skinning is done: for
each texture batch in the model, walk its index range, skin each of the
three triangle vertices via `gr2ModelVertex3D` (`game/gr2_model.go:236`,
which calls the skinning functions above plus applies the actor's world
placement matrix from `gr2ActorModelMatrix`, `game/gr2_model.go:256`), and
accumulate them into an `animatedRSMDrawBatch` that flushes as ordinary
`DrawTriangles3D` calls (`game/rsm_render.go:291-341`) — GR2 models are
**not** retained/cached meshes, since their vertex positions change every
frame as the skeleton animates.

## Sprites and 2D actor rendering

Most visible things in RO — player characters, monsters, NPCs, most skill
effects, damage numbers, the mouse cursor — are **not** 3D models at all.
They're pre-rendered 2D artwork, animated frame-by-frame, and drawn as
camera-facing billboards.

### The source formats

- **SPR** (`res/spr.go`, `type SPR`, `res/spr.go:14`) is the raw pixel data:
  a list of `SPRFrame`s, each either **indexed** (`SPRFramePalette` — one
  byte per pixel, an index into a shared 256-color `Palette`, optionally
  RLE-compressed on disk — `readIndexedFrame`, `res/spr.go:157`) or
  **true-color** (`SPRFrameRGBA` — 4 bytes per pixel, used in newer sprite
  files for smoother artwork). `SPR.FrameImageWithPalette`
  (`res/spr.go:92`) decodes either kind into a normal `image.NRGBA`,
  treating palette index 0 and RO's signature magenta
  (`isROMagenta`, `res/image.go:106`, `#F800F8`-ish) as transparent.
- **ACT** (`res/act.go`, `type ACT`, `res/act.go:9`) is the *animation
  script*: a list of `ACTAction`s (one per "action", e.g. idle-facing-south,
  walk-facing-east — 8 directions × several actions, indexed by
  `ACTAction.ActionFor(action, direction)`, `res/act.go:98`), each
  containing several `ACTAnimation`s (individual animation frames played in
  sequence) made of one or more `ACTLayer`s (`res/act.go:27`) — a layer
  references one SPR frame index plus a 2D offset, rotation angle, per-
  channel color tint, and scale, so one animation "frame" can composite
  several SPR images together (e.g. a weapon layer drawn on top of a body
  layer).
- **PAL** (`res/pal.go`) is a standalone 256-color palette file, used when a
  sprite's recolor needs to come from elsewhere (e.g. character body-color
  customization) instead of the palette baked into the `.spr` file.

### Building a drawable billboard

`game/sprite_render.go` and `game/sprite_assets.go` turn ACT+SPR (+
optional PAL) into an on-screen billboard in two stages:

1. **Composite on the CPU.** `composeSingleSpriteBillboard`
   (`game/sprite_render.go:615`) walks every `ACTLayer` in one
   `ACTAnimation` and draws each referenced SPR frame onto a shared target
   `render.Image` using `render.Image.DrawImage` (ordinary alpha-blended
   2D compositing, running on the CPU — see [§11](#cpu-rasterization-vs-the-gpu-path)),
   applying the layer's own offset/rotation/scale/tint. Humanoid player
   characters are more involved — `composeHumanoidBillboard`
   (`game/sprite_render.go:543`) composites up to 8 separate layers (body,
   head, weapon, shield, headgear slots — `humanoidLayerBody` through
   `humanoidLayerShield`, `game/sprite_render.go:60-70`) from *different*
   spriteViews into one image, using an `IMF` file (`res.IMF`, referenced
   at `game/sprite_render.go:11`) to decide draw order per action/frame,
   since which layer should be in front changes depending on the pose.
   The result is cached by animation key (`spriteView.billboards`,
   keyed by `singleSpriteBillboardKey`/`humanoidBillboardKey`) so this CPU
   compositing work only happens once per distinct frame, not every draw.
2. **Draw as a billboard.** `drawSpriteBillboardTintAlpha3DWithOptions`
   (`game/sprite_render.go:278`) takes the composited `spriteBillboard`
   (a `render.Image` + anchor point) and the camera's `right`/`up`
   direction vectors for this world position
   (`projection.BillboardBasis`, `game/scene_projection.go:87` — this is
   the camera-facing math: it returns the camera's local right/up axes
   scaled so one screen pixel of sprite artwork maps to a fixed number of
   world units, `unitsPerPixel`). It then emits one
   `render.WorldBillboardCommand` (`render/frame_commands.go:22`) per
   sprite instance — these are the *instanced* draws described in
   [§3](#shaders): the vertex shader computes the quad's four corners from
   `center + right*pixel + up*pixel` at draw time, so one draw call can
   render many billboards sharing a texture via GPU instancing
   (`gpuRenderer.drawWorldBillboards`, `render/gpu_renderer.go:1022`,
   batched by texture key exactly like everything else).

### Animation playback

Which frame of an `ACTAction` to show is time-based, not tied to the
render framerate: `spriteMotionIndexWithDelay`
(`game/sprite_assets.go` via `game/sprite_render.go:1087`) divides elapsed
wall-clock time since the animation started by each frame's `DelayMS`
(from the `.act` file, `res/act.go:19`) to pick the current
`ACTAnimation` index, looping or clamping depending on whether the action
repeats (walk cycles loop; a one-shot attack animation clamps at its last
frame).

## The effects system

RO's spell/skill visual effects (explosions, auras, ground rings, numbers
popping off a hit) are driven by a large, hand-authored, **data-driven**
table in `game/effects.go` (2700+ lines) rather than by external effect
files for most of them. Each effect is a `worldEffectSpec`
(`game/effects.go:741`) — duration, camera shake, sound cues, and one or
more `worldEffectComponent`s (`game/effects.go:753`, a struct with roughly
100 optional fields covering every parameter the original client's
effect scripting language could express: position/size/angle animation
curves, random jitter ranges, orbit motion, blend mode, and so on). Each
component has a `kind` (`effectComponentKind`, `game/effects.go:713`)
picking which renderer draws it:

| Kind | File | What it draws |
|---|---|---|
| `effectComponentSTR` | `game/effects_str.go` | A pre-authored particle animation loaded from a `.str` file (see below) |
| `effectComponentSPR` | `game/effects_sprite.go` (`drawSPREffect`) | A raw ACT/SPR animation played as a billboard, same machinery as §6 |
| `effectComponent2D` | `game/effects_2d.go` | A flat, non-camera-facing 2D quad |
| `effectComponent3D` | `game/effects_3d.go` | A single textured, camera-facing billboard with procedural size/rotation/position animation curves |
| `effectComponentCylinder` | `game/effects_cylinder.go` | A procedurally generated 3D cylinder/cone band mesh (rings of expanding/collapsing fire, magic circles) |
| `effectComponentQuadHorn` | `game/effects_quadhorn.go` | Procedural "horn" quads — thin radiating shapes (spikes, beams) |
| `effectComponentFUNC` | `game/effects_func.go` | An escape hatch: a named Go function (`effectFuncAdapter`) implementing effect logic too bespoke for the declarative fields |

**STR** (`res/str.go`, `type STR`, `res/str.go:9`) is RO's actual
pre-authored particle effect format — a fixed frame rate (`FPS`) and a list
of `STRLayer`s, each with its own texture list and a list of
`STRAnimation` keyframes (`res/str.go:20`) describing, per keyframe: which
texture frame to show (`AniFrame`), a quad's four corner offsets and UVs
(`XY`, `UV` — unlike a billboard sprite this can be a distorted/skewed
quad, not just a rectangle), position, rotation, per-channel color, and
blend mode (`SrcAlpha`/`DestAlpha`). `game/effects_str.go`'s
`calculateSTRAnimation` interpolates between keyframes by elapsed time
(`strEffectKeyIndex`, `game/effects_str.go:51`) and `drawSTRAnimation`
submits the resulting quad. This is the format behind most "real" skill
effect animations (explosions, magic auras) that a game designer authored
frame-by-frame rather than the engine generating procedurally.

The `effectComponentCylinder`/`effectComponentQuadHorn` kinds, by contrast,
are **procedural geometry generated in Go at draw time** — no artist-
authored mesh exists on disk. `drawWorldCylinderBandWithBasis`
(`game/effects_cylinder.go:185`) builds a ring of quads around a
`segments`-sided circle by rotating a base vector, interpolating between a
`bottomRadius` and `topRadius` — the same technique you'd use to build a
cone or a lathe shape in any 3D modeler, just done at runtime per frame,
so skill effects can generate a cylinder of any size/shape by pure
parameters coming from `worldEffectComponent`.

Effects are triggered by network packets — e.g.
`WorldMode.applySpecialEffectNotify` (`game/effects.go:1221`) or
`applySkillCastNotify` (`game/effects.go:997`) map a server-sent effect ID
to one of these `worldEffectSpec`s and spawn a `worldEffect` instance
(`game/effects.go:725` — position, actor/target IDs, start/expire time),
then `WorldMode.drawWorldEffects` (called from `Draw`, `game/world.go:1594`)
walks all active `worldEffect`s each frame and dispatches to the component
renderer matching its `kind`.

## Lighting, fog, and water

### Lightmaps

RO's terrain lighting is **baked**, not computed live: each GND surface
references a `GNDLightmap` (`res/gnd.go:36`) — an 8×8 grid of precomputed
brightness (`Alpha`) and tinted light color (`Color`), effectively a tiny
pre-rendered "how lit is this patch of ground" texture, generated offline
by the original map tools from the map's static lights. `goro` decodes two
on-disk encodings (`decodeGNDLightmapRaw` for newer maps,
`decodeGNDLightmapIndexed` — a 5-bit-per-channel packed format — for
older ones, `res/gnd.go:215`/`231`), packs every lightmap in a map into one
atlas texture (`buildGNDLightmapAtlas`, described in §4), and the world
shader's fragment stage (`render/gpu_shaders.go:101-102`) multiplies each
ground pixel's base color by the sampled lightmap alpha and adds its RGB —
this dual role (alpha = shadow darkness, RGB = colored light tint) is why
the shader math looks like `color.rgb * lightmap.a + lightmap.rgb` rather
than a simple multiply.

Dynamic per-vertex shading (used for the 4 corner colors of a ground
surface where no lightmap texture is sampled, or for RSM prop lighting) is
computed on the CPU from the map's single directional "sun" light
(`RSWLight`, longitude/latitude angle + diffuse/ambient color,
`res/rsw.go:40`) via `sceneLighting.groundScale` (referenced throughout
`game/gnd.go`, e.g. `game/gnd.go:511`), essentially a simple Lambertian
`dot(normal, lightDirection)` term blended with an ambient floor — the
smoothed vertex normals it needs are precomputed once per map by
`buildSmoothGNDTopNormals` (`game/gnd.go:225`, averaging face normals of
neighboring cells so terrain doesn't look faceted).

### Fog

Map-level distance fog (`sceneFog`, `game/fog.go:11`) is loaded per-map from
a fog parameter table (`manager.FogParameter(mapName)`,
`game/fog.go:22`) giving a near/far distance and a color; `sceneFogFromMap`
converts these into world units (`* 240`, `game/fog.go:34-35` — RO's
distance unit conversion constant) and can be overridden per-map by special
weather effects (`sceneFogColorForMapWeather`, `game/fog.go:40`). This is
passed into the camera as `render.Fog3D` (`render/types.go:51`,
`sceneProjection.RenderCameraWithFog`, `game/scene_projection.go:68`) and
applied **in the GPU shader**, per-pixel, using view-space depth and a
`smoothstep` ramp (`render/gpu_shaders.go:106`) — this is why fog looks
smooth rather than banded, and why it correctly fades billboards and world
meshes alike (both the world and world-billboard WGSL shaders share this
exact fog block). There is also a CPU-side fog helper,
`sceneFog.mixVertexTints`/`mixColor` (`game/fog.go:91`,`51`), used to
pre-tint *per-vertex* colors (e.g. for water, drawn with baked-in vertex
colors rather than the GPU fog uniform) — so fog is applied via two
different mechanisms depending on which draw path a piece of geometry takes.

### Water

See [§4](#world-geometry-gnd-gat-rsw) — water is a per-cell animated quad,
not a lightmapped/fogged mesh in the GPU-shader sense, though its tint
still factors in the map's ambient light color for certain water types
(`waterTint`, `game/gnd.go:494`) and its vertex colors are still passed
through `fog.mixVertexTints` (`game/gnd.go:376`).

## The camera

RO's camera is a **third-person orbiting camera** fixed on the player
character, not a free-fly camera: the player rotates and zooms the camera
around a target point, but (outside of locked indoor maps) always looking
at roughly the same pitch angle down at the character.

`followCamera` (`game/camera.go:25`) owns the *target* the camera looks at:
it smoothly interpolates (`lerp`, exponential smoothing scaled by elapsed
time, `cameraFollowLerp`, `game/camera.go:70`) toward the player's current
world position each `Update` call, so camera motion is not a hard snap even
though the player can teleport/warp or move in discrete steps. It also
tracks yaw (rotation the player has dragged with the right mouse button,
`Rotate`/`updateCameraRotation`, `game/camera.go:77`,`195`), pitch (limited
tilt range via right-drag+shift, `Tilt`, `game/camera.go:81`,
clamped to `[205°, 245°]`, `game/camera.go:20-21` — RO's camera never goes
near top-down or eye-level), and zoom (mouse wheel/pinch,
`ZoomBy`/`ZoomByDelta`, clamped `[65, 165]` world units,
`game/camera.go:19`). Indoor maps and certain scripted "view points" can
lock yaw/pitch/zoom entirely (`cameraRotationLockedForMap`,
`game/camera.go:251`, reading a per-map `res.CameraViewPoint` — the
reference client's concept of an indoor map having one fixed, art-directed
camera angle).

Turning that target + yaw/pitch/zoom into an actual matrix is
`sceneCameraMatrixWithYawPitchZoom` (`game/scene_projection.go:179`):

1. Convert yaw/pitch/zoom into a 3D `eye` position orbiting the target —
   basic spherical-to-Cartesian coordinates (`horizontal = cos(pitch) *
   distance`, then `eye.x = target.x + sin(yaw)*horizontal`, etc.,
   `game/scene_projection.go:187-193`).
2. Build a **look-at** view matrix (`mat4LookAt`,
   `game/scene_projection.go:202`) — the standard technique: compute
   `forward` (target minus eye, normalized), derive `right` via a cross
   product with world-up, derive a true `up` via another cross product,
   and place those three orthonormal axes as the matrix's rows/columns,
   with the eye position folded in as a translation.
3. Multiply by a **perspective projection matrix**
   (`mat4Perspective`, `game/scene_projection.go:223` — a standard
   field-of-view/aspect/near/far perspective matrix, FOV fixed at
   `defaultSceneCameraFOV = 15°`, `game/scene_projection.go:26` — a narrow
   FOV, which combined with the long camera distance is what gives RO its
   characteristic mild "telephoto" look rather than a wide fisheye).

The resulting `sceneProjection.viewProjection` matrix
(`game/scene_projection.go:19`) is what every 3D drawing function in this
document ultimately needs: it's uploaded once per frame as the world
shader's uniform (`worldUniformBytes`, `render/gpu_renderer.go:1468`) so
the GPU can transform vertices, and it's also used **on the CPU** for
hit-testing/culling math — `sceneProjection.Project` (screen-space position
of a world point, used for cursor hover/click picking,
`game/scene_projection.go:56`), `.ScreenRay` (mouse position → 3D ray, used
for ground click targeting, `game/scene_projection.go:101`), and
`.BillboardBasis` (camera-facing right/up axes for a given world position,
`game/scene_projection.go:87`, described in §6).

## Simple overlays: cursor, damage numbers, equipment view

These three are good "second file to read" examples: small, focused, and
built entirely from primitives already covered above.

- **Damage numbers** (`game/damage_numbers.go`) are literally sprite
  billboards of individual digit glyphs, composited side-by-side into one
  image per unique number string and cached (`composeDamageNumberBillboard`,
  `game/damage_numbers.go:113` — note it reuses `composeSingleSpriteBillboard`
  from §6 per digit, then does its own simple horizontal layout with
  `render.Image.DrawImage`). Their screen motion (arcing up, fading out,
  the "combo counter" bounce) is pure math with no new rendering concepts —
  `damageFloaterPlacement` (`game/damage_numbers.go:187`) just returns an
  offset/scale/alpha curve per animation "kind" that the caller feeds into
  the same billboard-drawing functions from §6.
- **The cursor** (`game/cursor.go`) is a `roCursorState`
  (`game/cursor.go:40`) tracking which cursor action (attack, talk, pick
  up, disabled, etc.) is currently appropriate given what's under the
  mouse, each backed by its own small ACT/SPR animation
  (`cursorActionInfo`, `game/cursor.go:34`) drawn as a screen-anchored
  billboard (`roCursorState.draw`, `game/cursor.go:115`). Worth reading
  for how it picks a target to "snap" to (`cursorActorMagnetOffset`,
  `game/cursor.go:271`, projecting each nearby actor's billboard center to
  screen space via `projection.Project` and testing mouse distance).
- **Equipment view** (`game/equipment_view.go`, only 19 lines) is the
  smallest file in the graphics code — a thin helper resolving which
  weapon/shield sprite overlay layer to use, feeding directly into the
  humanoid layer compositing from §6. Good for seeing how little new code
  it takes to add "one more layer" to the sprite system.

## CPU rasterization vs. the GPU path

`render/cpu_raster.go` implements a **software triangle rasterizer**
(`Image.drawTriangle`, `render/cpu_raster.go:8`) — the classic
edge-function algorithm: for every pixel in a triangle's bounding box,
compute barycentric weights via three edge functions and only shade the
pixel if all three are non-negative (i.e. inside the triangle),
interpolating UV/color across the triangle by those weights
(`render/cpu_raster.go:16-38`).

This is **not** an alternative to the GPU renderer for drawing the game
world — it's the mechanism behind `Image.DrawImage`, used throughout §6/§7
to *composite sprite layers together on the CPU* before the result is ever
uploaded to the GPU as a texture (e.g. drawing a weapon-overlay SPR frame
on top of a body SPR frame into one `render.Image`). Compositing
pre-rendered 2D art this way is cheap and simple; it would be unusual
overkill to spin up a GPU render pass just to composite a few dozen small
sprite layers once and cache the result.

**Headless mode** (§2) is unrelated to this CPU rasterizer — it doesn't
rasterize anything at all, it just runs `Update`/`Draw` and throws the
resulting `Frame` away, for tests that need the game/world logic to run
without any display.

## Palettes and image decoding

`res/image.go` centralizes generic image loading: `LoadImage`/
`LoadImageExact` (`res/image.go:15`,`33`) try Go's standard PNG/JPEG
decoders first, falling back to a hand-rolled TGA decoder
(`decodeTGA`, referenced at `res/image.go:23`) for the Targa-format
textures RO ships for some ground/effect art. Every loaded image passes
through `applyROTransparency` (`res/image.go:62`), which treats RO's
signature magenta (`isROMagenta`, `res/image.go:106`) as transparent and
then zeroes the RGB of fully-transparent pixels (`clearTransparentRGB`,
`res/image.go:92`) — this matters because GPU texture filtering blends
neighboring pixels' colors even through zero alpha, so leftover garbage
color in "transparent" pixels can bleed a colored fringe around sprites
without this cleanup.

Palette-based (indexed) images are handled separately in `res/pal.go`
(a bare `[256][4]byte` lookup table, `res/pal.go:5`) and `res/spr.go`'s
`FrameImageWithPalette` (§6), which is what lets `goro` **swap palettes at
runtime**: the same indexed sprite pixel data, redrawn through a different
256-color table, produces a different-looking character (dyed hair, tinted
armor) without re-decoding or re-storing pixel data per variant —
`characterHeadPalette`/`loadSpritePalette`
(`game/sprite_assets.go:67`,`429`) resolve which palette file a given
character's customization should use before compositing.

## How UI compositing fits in

`gogpu/ui` (an external widget-tree UI library) draws the game's windows,
buttons, and text into its own bitmap, entirely separately from the 3D
world pipeline described above. `runner.drawUISync`
(`render/backend.go:1076`) rasterizes the current `gogpu/ui` widget tree
into an offscreen `ggcanvas.Canvas` (only when something changed —
`win.NeedsRedraw()`/dirty-rectangle tracking, `render/backend.go:1113`),
converts that into a `render.Image` (`r.updateUIImage`), and then
`drawUIPublishedImage` (`render/backend.go:1237`) draws that whole image as
one full-screen textured quad **on top of** everything `game.Draw` already
put into the `Frame` — i.e. the 3D world (and any world-billboards/sprites)
is always composited first, and the 2D UI layer is a single flat image
laid over it last. `render/ui_async.go` additionally supports rasterizing
that UI canvas on a background goroutine (`drawUIAsync`,
`render/backend.go:1178`) so a busy UI redraw doesn't stall the main
render thread — worth knowing exists, but not something you need to
understand to work on world/sprite/effect rendering.

Some overlay elements (map "banner" text, actor name labels, floating
combat text, tooltips) are **not** part of the `gogpu/ui` widget tree at
all — they're queued directly onto the `Frame` as lightweight command
structs (`UITextBoxCommand`, `UIActorLabelCommand`, `UIRectCommand` in
`render/frame_commands.go:40-85`) by game code and drawn by `render`
itself as simple flat 2D quads/text, bypassing the widget system entirely
for things that need to be positioned relative to 3D world objects (e.g. a
name label that must track an actor's screen position every frame).

## Testing graphics code

Graphics code in this repository is tested at several different levels
without requiring a GPU or display:

- **Headless app tests** (`app/headless_test.go`, `config/headless_test.go`,
  `render/headless_test.go`) exercise the full `Update`/`Draw` cycle
  through `render.RunHeadless` (§2), proving game logic and draw-call
  construction don't panic/error, without ever creating a GPU device.
- **Pure unit tests** on the math/data layer (`render/gpu_world_mesh_batch_test.go`,
  `render/gpu_world_mesh_buffer_test.go`, `render/cursor_test.go`,
  `render/window_provider_test.go`, `game/scene_projection_test.go`) test
  batching/allocator/matrix logic directly against Go values — no
  rendering happens at all.
- **CPU-rasterizer-backed tests** (`render/image_test.go`,
  `render/text_banner_test.go`, `render/input_test.go`) construct a
  `render.Frame`/`render.Image` and assert on the resulting command lists
  or pixel values (e.g. `TestBlendSourceOverPreservesStraightAlphaOnTransparentTarget`,
  `render/image_test.go:19`, blends a pixel with `Image.blendPixel`
  directly and checks the resulting `color.RGBA`) — these test the CPU
  path (§11) precisely because it's deterministic and needs no GPU.
- **Format-parser tests gated on real game data** — `res/*_real_test.go`
  (e.g. `res/gnd_real_test.go`, `res/rsm_real_test.go`, `res/gr2_test.go`)
  call `realDataManager(t)` (`res/real_data_test.go:8`), which reads the
  `GORO_DATA_DIR` environment variable and calls `t.Skip(...)` if it's
  unset (`res/real_data_test.go:11`) — these tests only run when a
  developer points them at an actual extracted RO client install; they
  don't run in ordinary CI without that data. This is why you'll often see
  two test files per resource format: a `_test.go` with small synthetic
  binary fixtures (fast, always runs) and a `_real_test.go` that
  round-trips against real client files (slow, opt-in, catches
  real-world format edge cases synthetic fixtures wouldn't).
- **GPU renderer tests** (`render/gpu_renderer_test.go`) don't spin up a
  real GPU either — as noted in §3, they validate the WGSL shader source
  text through the `naga` shader-compiler library (parse → lower →
  generate SPIR-V) as a static correctness check on the shader code, not
  an integration test of actual rendering.

There are no pixel-diff/golden-image screenshot tests in this codebase —
correctness of the *visual* result is instead verified indirectly (shader
compiles, draw commands are queued as expected, CPU compositing produces
the expected byte values) plus manual testing in the running client.

## Where to start

If you're new to this codebase and want to build a mental model before
making changes, read in roughly this order:

1. **`game/damage_numbers.go`** and **`game/equipment_view.go`** — tiny,
   self-contained, and touch the sprite-billboard system (§6) without any
   of the camera/lighting/effect-table complexity.
2. **`render/frame.go`** and **`render/types.go`** — see the full surface
   area of what game code can ask the renderer to do, with no GPU code in
   sight yet.
3. **`game/scene_projection.go`** — the camera math (§9). Small file,
   used by everything downstream of it.
4. **`game/sprite_render.go`**, starting at `drawPlayerSprite3D`
   (`game/sprite_render.go:168`) — the most common on-screen thing (a
   walking character) traced end to end.
5. **`game/gnd.go`** + **`game/static_world_mesh.go`** — the terrain
   pipeline (§4), and your first exposure to retained/batched meshes.
6. **`render/gpu_renderer.go`** — once the above make sense, this is where
   it all actually becomes GPU draw calls; read `Draw`
   (`render/gpu_renderer.go:486`) top to bottom.
7. **`game/gr2_animation.go`** and **`res/gr2.go`** last — by far the most
   involved code in the graphics subsystem (generic binary reflection +
   skeletal skinning + custom decompression codecs), best tackled once
   everything above is familiar.

## Glossary of RO format acronyms

| Acronym | Full meaning (informal) | What it stores | Parsed in |
|---|---|---|---|
| GRF | (Gravity Resource File) | An archive/container of many game files (textures, sprites, maps, sounds), RO's equivalent of a `.zip` | `res/grf.go` |
| GND | Ground | The visual terrain mesh: height grid + per-cell texture/lightmap surfaces | `res/gnd.go` |
| GAT | (Ground Altitude Table, informally) | The walkability/collision/pathfinding grid, separate from the visual mesh | `res/gat.go` |
| RSW | (Resource/Scene World) | The map "scene" file: pointers to GND/GAT, plus placed models/lights/sounds/effects and global map settings | `res/rsw.go` |
| RSM | (Resource Model) | RO's original 3D prop/scenery model format, with optional rigid per-node animation | `res/rsm.go` |
| GR2 | Granny3D file | A newer, general 3D model container format with true bone-skeleton skinning, used for some character models | `res/gr2.go` |
| ACT | Action | An animation script: which sprite frames make up each action/direction, with per-layer transform/timing | `res/act.go` |
| SPR | Sprite | The raw 2D pixel data (indexed or RGBA frames) referenced by an ACT | `res/spr.go` |
| PAL | Palette | A standalone 256-color lookup table for recoloring indexed sprites | `res/pal.go` |
| STR | (Sprite/Special effect animation format) | A pre-authored particle/visual-effect animation: keyframed quads, textures, blend modes | `res/str.go` |
| IMF | (Item/layer Motion File, informally) | Per-frame draw-order data for compositing a player character's equipment layers | referenced in `game/sprite_render.go` |
