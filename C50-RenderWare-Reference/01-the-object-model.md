# C50.1 — The object model

RenderWare is an object database with a renderer attached. Every visible thing in San Andreas is a graph of a
few RenderWare object types, and understanding those types — and which chapter decodes each — is understanding
the game's rendering. This page names them from the bottom up.

## The type stack

| Type | What it is | Decoded in San Andreas by |
|---|---|---|
| `RwEngine` | the singleton: device, plugin registry, render-state cache | (bootstrapped at startup — [C51](../C51-Executable-Lifecycle/C51-Executable-Lifecycle.md)) |
| `RwFrame` | a node in the transform hierarchy (a matrix + parent) | the frame tree atomics attach to |
| `RwCamera` | view + projection + the raster it renders to | [C39](../C39-Camera/C39-Camera.md)'s `CCamera` wraps one |
| `RpWorld` | the static world as a BSP of sectors | [C5](../C5-CWorld/C5-CWorld.md)/[C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) |
| `RpClump` | a model — a collection of atomics + frames | [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) (the DFF) |
| `RpAtomic` | one drawable piece of a clump (geometry + frame) | [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) |
| `RpGeometry` | vertices, indices, morph targets, materials | [C8](../C8-Geometry/C8-Geometry.md) |
| `RpMaterial` | a surface: colour + texture + effects | [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) |
| `RwTexture` | a named handle to a raster (+ addressing/filter) | [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) |
| `RwRaster` | the pixels on the GPU (a D3D9 surface) | [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) |
| `RwTexDictionary` | a named set of textures (the TXD) | [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) |

## How they compose into a car

Concretely, take a vehicle. It loads from a **DFF** as an `RpClump`
([C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)); the clump contains `RpAtomic`s for the body, each
wheel, the doors — every separately-moving piece. Each atomic has an `RpGeometry`
([C8](../C8-Geometry/C8-Geometry.md)) holding its vertices and an `RpMaterial` list; each material points at an
`RwTexture` resolved from the vehicle's **TXD** (`RwTexDictionary`,
[C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)). The atomics hang off `RwFrame`s in a
hierarchy, so rotating the wheel's frame spins the wheel. When the car is visible
([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)), each atomic is submitted to the D3D9 pipeline
([C44](../C44-Shaders/C44-Shaders.md)) and drawn through the `RwCamera`'s view
([C39](../C39-Camera/C39-Camera.md)). Every game object you can name is this same composition with different
data.

## The plugin extension mechanism

The reason RenderWare objects can carry game-specific data (bone weights, 2d-effects, collision) without the
core knowing about it is the **plugin** system: a plugin registers at startup and *reserves bytes* on an
object type, plus stream read/write callbacks. So an `RpAtomic` in San Andreas is larger than a stock
RenderWare atomic — it carries the `skin2`/`hanim` skinning data ([C17](../C17-IFP-Animation/C17-IFP-Animation.md)),
the `matfx` env-map flag ([C44](../C44-Shaders/C44-Shaders.md)), and Rockstar's own extensions — all appended
by plugins. This is why [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)'s DFF stream has *extension*
sections after each object: they are the plugin data. ([C50.2](02-plugins-driver-pipelines.md) inventories the
plugins.)

## Key takeaways

- Every game visual is a graph of a few RenderWare types: `RpClump`/`RpAtomic` (models) of `RpGeometry`, with
  `RpMaterial`→`RwTexture`→`RwRaster` surfaces, in an `RwFrame` hierarchy, viewed by an `RwCamera`.
- Each type is decoded by a specific chapter — DFF ([C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)),
  geometry ([C8](../C8-Geometry/C8-Geometry.md)), TXD ([C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)),
  camera ([C39](../C39-Camera/C39-Camera.md)) — so C50 is the index.
- **Plugins** reserve bytes on objects and add stream callbacks; that is how atomics carry skinning/env-map/
  collision, and why DFFs have extension sections.

**Continue:** [C50.2 — Plugins, the driver, and pipelines →](02-plugins-driver-pipelines.md)
