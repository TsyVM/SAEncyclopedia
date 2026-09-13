# C40.3 — Where the pipeline sits in the frame

`CRenderer` ([C40.1](01-the-four-render-lists.md), [C40.2](02-the-pass-order.md)) is one stage of a larger
frame. This page places it: what comes before the world is drawn, and what comes after, so the rendering
chapters (C38 sky, C39 camera, C40 pipeline) connect into a single picture of a San Andreas frame.

## The frame, stage by stage

A San Andreas frame runs, in order:

1. **Update** — game logic, physics, streaming, AI advance the world state (not a render stage).
2. **Camera** — [C39](../C39-Camera/C39-Camera.md)'s `CCamera` fixes the view matrix for this frame; the
   active `CCam` writes `TheCamera`'s matrix at `0xB6F03C`.
3. **Begin scene** — RenderWare opens the immediate-mode frame; the back buffer and Z-buffer are cleared.
4. **Sky** — [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)'s `RenderSkyPolys` lays the gradient
   with Z-write **off**, so it fills the background behind everything.
5. **Build the render lists** — `ScanWorld` + `ConstructRenderList` sort the visible world into the four
   lists ([C40.1](01-the-four-render-lists.md)), reading the same camera matrix.
6. **Draw the world** — `RenderRoads` → `RenderEverythingBarRoads` → `RenderFadingInEntities` drain the
   lists, with the LOD/super-LOD lists filling the distance.
7. **Effects & particles** — coronas, the sun/moon, `effects.fxp` particles
   ([C21](../C21-Particles/C21-Particles.md)), water.
8. **HUD & menus** — the [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) fonts/HUD sprites draw in screen space over
   the finished 3D scene.
9. **Fade & present** — screen fades, then the frame is presented.

The three rendering chapters occupy stages 2, 4 and 5–6: the **camera** sets the view, the **sky** fills the
background, the **pipeline** draws the world between them. All three read `TheCamera` at `0xB6F028`+offset,
which is why they compose cleanly rather than each carrying its own notion of "where the eye is."

## Immediate mode, not a scene graph

San Andreas draws through RenderWare's immediate-mode path, the same `RwIm3D*`/`RwRenderStateSet` machinery
[C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) showed for the sky. There is no retained scene graph
that the engine walks; instead, each frame `CRenderer` **rebuilds** the four lists from scratch
([C40.1](01-the-four-render-lists.md)) and re-submits every visible atomic. That is why the lists are fixed
arrays reset every frame rather than a persistent structure — the render state is transient, the geometry
comes from the streamed [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) clumps/atomics, and the passes
set device state through the RenderWare vtable at `[0xC97B24]` per pass.

> ⚠️ **Correction (see [C44](../C44-Shaders/C44-Shaders.md)).** An earlier version of this section claimed
> San Andreas has "no per-object shader system … the pipeline is fixed-function RenderWare." That is **wrong**
> and is corrected here per the house rule. The *im3d* path described on this page (sky, HUD, coronas) is
> indeed fixed-function immediate mode — but the **world / atomic / skin** path is not: every building,
> vehicle and ped is drawn by a RenderWare **D3D9 vertex-shader** pipeline (`nodeD3D9*AllInOne`), whose
> vertex-shader assembly is generated at runtime and assembled by D3DX9. So the accurate statement is: **the
> immediate-mode path (this chapter's passes) is fixed-function, but vertex processing for world geometry is
> programmable** — see [C44](../C44-Shaders/C44-Shaders.md) for the pipelines, the register map, and the
> transform/skinning proof. Pixel processing does stay largely fixed-function (per-pass render states), so the
> im3d framing here is right for what it covers; it was the *scope* of the "no shaders" claim that was wrong.

## What this closes, and what it opens

**Closes** the rendering arc's framing question: the sky is a gradient (C38), the camera is one manager over
three `CCam` objects (C39), and the world is four tiling entity lists drained by an ordered pass sequence
(C40) — all driven by one camera global. There is no monolithic "renderer" mystery left; there is a list
builder and a pass order.

**Opens** the natural follow-ons:

- ⏳ **Render-state decode** — name every `RwRenderStateSet` pair each pass issues (the `rwRENDERSTATE*`
  enum), giving the exact Z/cull/blend configuration per pass.
- ⏳ **The LOD distance model** — the thresholds in `ConstructRenderList` that assign an entity to full /
  LOD / super-LOD, and how `ms_lodDistScale` (1.2) drives them.
- ⏳ **Water and reflections** — a sibling render stage not covered here.
- ⏳ **Effects/particle submission** — how [C21](../C21-Particles/C21-Particles.md)'s `effects.fxp` systems
  enter the frame at stage 7.

## Key takeaways

- The render pipeline is stage 5–6 of a nine-stage frame: **camera → begin scene → sky → build lists → draw
  world → effects → HUD → fade → present**.
- The three rendering chapters compose because they share one camera (`TheCamera` @`0xB6F028`): C39 sets it,
  C38 and C40 read it.
- San Andreas is **immediate-mode fixed-function RenderWare** — the lists are rebuilt every frame and the
  "shader program" is the ordered passes and their render-state setup, which is why there is no per-object
  shader system to extract.

**Continue:** [back to the C40 hub →](C40-Render-Pipeline.md) · or the rendering arc:
[C38 Sky](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) · [C39 Camera](../C39-Camera/C39-Camera.md).
