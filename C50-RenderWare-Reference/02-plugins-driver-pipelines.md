# C50.2 — Plugins, the driver, and pipelines

The 88 linked RenderWare modules split into a **core**, a set of **plugins** that extend it, a **Direct3D 9
driver** that talks to the GPU, and the **P2 pipelines** that turn objects into draw calls. This page
inventories them from the exe and points each at its game chapter.

## The module inventory

Grouped by the `$Id` source paths in the exe:

| Subsystem | Modules | Role |
|---|---:|---|
| core (`plcore`) | 14 | `RwEngine`, plugin registry, streams, memory |
| core objects (`src`) | 14 | frames, cameras, rasters, images, matrices |
| world (`world`) | 11 | `RpWorld` BSP, sectors, lights |
| D3D9 pipelines (`world/pipe/p2/d3d9`) | 10 | the AllInOne atomic/world/skin pipelines ([C44](../C44-Shaders/C44-Shaders.md)) |
| P2 core (`src/pipe/p2`) | 7 | the pipeline-2 framework |
| D3D9 driver (`driver/d3d9`) | 6 | device, render states, textures, 2D |
| plugins (`plugin/*`) | ~12 | skin2, hanim, matfx, uvanim, anisot |
| tools (`tool/*`) | 5 | quat/slerp (anim math), png/bmp (image), anim |

**21 modules touch D3D9** (driver + pipelines), which is the concrete weight of the
[C44](../C44-Shaders/C44-Shaders.md) finding that San Andreas is a real programmable-vertex D3D9 renderer, not
a thin wrapper.

## The plugins

Five RenderWare plugins are linked, and each maps to a game feature:

| Plugin | Extends | Game feature | Chapter |
|---|---|---|---|
| `skin2` | atomics with bone weights/indices | skinned peds | [C17](../C17-IFP-Animation/C17-IFP-Animation.md) / [C44.2](../C44-Shaders/02-register-map-and-transform.md) |
| `hanim` | frames with a bone hierarchy | the skeleton animations play on | [C17](../C17-IFP-Animation/C17-IFP-Animation.md) |
| `matfx` | materials with effects | vehicle **environment mapping** | [C44.1](../C44-Shaders/01-the-pipelines.md) |
| `uvanim` | materials with UV animation | scrolling textures (e.g. conveyor belts, water) | — |
| `anisot` | textures with anisotropic filtering | sharper textures at grazing angles | — |

The plugin list is itself a map of San Andreas's rendering features: peds are skinned (`skin2`+`hanim`),
vehicles reflect (`matfx`), some surfaces scroll (`uvanim`), and textures filter anisotropically (`anisot`).
Anything **not** in the plugin list, the game does not do at the RenderWare level — there is no shadow-volume
plugin, no normal-map plugin, which is consistent with the era.

## The P2 pipeline framework

RenderWare's "pipeline 2" (P2) is a node-graph framework for turning objects into draw calls, but San Andreas
uses the **`AllInOne`** collapsed pipelines ([C44](../C44-Shaders/C44-Shaders.md)) — one node per object class
(world sector, atomic, skinned atomic) that does the whole transform-and-submit in one step. The framework
(`src/pipe/p2`, 7 modules) provides the plumbing; the D3D9 realisation (`world/pipe/p2/d3d9`, 10 modules)
provides the concrete vertex-shader pipelines [C44](../C44-Shaders/C44-Shaders.md) decoded. The `im3d`
immediate-mode path ([C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)/[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md))
is the P2 framework's simpler sibling for un-batched geometry.

## The driver

The `driver/d3d9` modules (6) are RenderWare's thin abstraction over Direct3D 9 — the device, render-state
cache, texture upload, and 2D rendering. This is the layer that issues the actual `IDirect3DDevice9` calls,
and it is what the [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) passes ultimately drive through the
device vtable at `[0xC97B24]`. It also handles device-lost/reset (the alt-tab black screen), format selection,
and the swap chain — the boundary between RenderWare and the OS graphics stack.

## Open items

- ⏳ The exact plugin **byte reservations** on each object type (how many bytes `skin2`/`matfx` add).
- ⏳ The full **render-state cache** behaviour in the driver (what `RwRenderStateSet` batches vs flushes).
- ⏳ `uvanim`/`anisot` usage sites in the game data.

## Key takeaways

- The 88 modules split into **core / world / D3D9 driver / P2 pipelines / plugins / tools**; **21** touch
  D3D9, quantifying [C44](../C44-Shaders/C44-Shaders.md)'s "real D3D9 renderer".
- Five **plugins** map one-to-one to game features: `skin2`+`hanim` (skinned peds,
  [C17](../C17-IFP-Animation/C17-IFP-Animation.md)), `matfx` (vehicle env-map,
  [C44](../C44-Shaders/C44-Shaders.md)), `uvanim` (scrolling UVs), `anisot` (filtering) — the feature set is
  the plugin set.
- The **`AllInOne` P2 pipelines** over the **D3D9 driver** are the [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)/[C44](../C44-Shaders/C44-Shaders.md)
  render path; `im3d` is the simpler immediate-mode sibling.

**Continue:** [back to the C50 hub →](C50-RenderWare-Reference.md) · or [C44 — Shaders](../C44-Shaders/C44-Shaders.md) (the D3D9 pipelines in detail).
