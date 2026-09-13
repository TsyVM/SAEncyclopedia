# C40.2 — The pass order

The four lists ([C40.1](01-the-four-render-lists.md)) are filled and drained by an ordered sequence of named
functions. This page reads that sequence out of the exe: who fills the lists, who drains them, and in what
order — including the moment the camera hands the pipeline its view.

## The functions

| Pass | VA | Role |
|---|---|---|
| `ScanWorld` | `0x554FE0` | walk the world sectors the camera can see; collect candidate entities |
| `ConstructRenderList` | `0x5556E0` | read the camera matrix; sort candidates by distance into the four lists |
| `SetupScanLists` | `0x553540` | prepare the scan/sector working lists |
| `PreRender` | `0x553910` | per-entity pre-render setup over the visible lists |
| `RenderRoads` | `0x553A10` | draw the road network first |
| `RenderEverythingBarRoads` | `0x553AA0` | draw every visible non-road entity |
| `RenderFadingInEntities` | `0x5531E0` | draw entities fading in from streaming, with alpha |
| `RenderOneRoad` / `RenderOneNonRoad` | `0x553230` / `0x553260` | the per-entity draw calls |

## Construction reads the camera — the C39 tie

`ConstructRenderList` opens by fetching the camera's matrix, and the disassembly is unambiguous:

```
0x5556E0: push ecx
0x5556E1: mov  eax, [0xB6F03C]     ; TheCamera.m_matrix
0x5556E9: cmp  eax, ebx            ; null?
0x5556F3: mov  eax, 0xB6F02C       ;   fall back to TheCamera.m_placement
0x5556F8: mov  edx, [eax]          ; read the camera position vector...
```

`0xB6F03C` is `TheCamera + 0x14` — the exact field [C39](../C39-Camera/C39-Camera.md) identified as the
camera matrix and [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)'s sky renderer also read. So the
render list is built *from the camera's frame*: what the camera sees decides what goes in the lists.
`derive_renderer.py` asserts this reference (`construct_reads_camera_matrix`). Three rendering chapters —
sky, camera, pipeline — now all reference the same global at consistent offsets, which is the strongest kind
of corroboration this project has: independent derivations that cannot disagree.

## The passes set render state and walk a count

`RenderEverythingBarRoads` shows the standard shape of a draw pass — set up RenderWare render states, then
loop the visible-entity count:

```
0x553AA0: push ecx
0x553AA1: mov  eax, [0xC97B24]         ; the RW raster/device object
0x553AA8: push 1 ; push 0x0E           ; RwRenderStateSet(state 0x0E, 1)
0x553AAA: call [eax + 0x20]
0x553AB5: push 1 ; push 0x0C           ; RwRenderStateSet(state 0x0C, 1)
...
0x553AE5: mov  eax, [0xB76844]         ; ms_nNoOfVisibleEntities  <-- the loop bound
0x553AEC: xor  edi, edi                ; i = 0
```

The pass configures the pipeline (Z-test, culling, blending — the `RwRenderStateSet` calls through the device
vtable at `[0xC97B24]`) and then iterates `ms_nNoOfVisibleEntities`, drawing each entry of
`ms_aVisibleEntityPtrs`. That the *count* variable proven in [C40.1](01-the-four-render-lists.md) is read
right here as the loop bound closes the loop between the two pages: the array this chapter sized is the array
this pass draws. `derive_renderer.py` asserts the pass reads the count (`pass_reads_visible_count`).

## The order, and why it is that order

Reading the passes in sequence:

1. **`ScanWorld` / `SetupScanLists`** — determine visibility from the camera's sectors.
2. **`ConstructRenderList`** — bucket visible entities into the four lists by distance.
3. **The sky** — [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)'s gradient goes down first, with
   Z-write disabled, so it sits behind everything.
4. **`RenderRoads`** — the road network, drawn before the props on it.
5. **`RenderEverythingBarRoads`** — every other visible entity, loop-bounded by `ms_nNoOfVisibleEntities`.
6. **`RenderFadingInEntities`** — streaming pop-in, drawn last among world geometry so its alpha blends over
   the settled scene.

Roads before everything-bar-roads is not cosmetic: props, peds and vehicles sit *on* roads, so the road
surface must be laid first for correct overdraw and for the alpha-blended details on top to composite
correctly. The fading pass runs last because it blends new streaming geometry ([C1](../C1-Streaming/C1-Streaming.md))
in over an already-drawn frame.

## Open items

- ⏳ The exact `RwRenderStateSet` state/value pairs each pass sets (the `push 0x0E`/`0x0C`/`0x14` immediates
  are RenderWare `rwRENDERSTATE*` enum values) — decode the enum to name each state.
- ⏳ The distance thresholds inside `ConstructRenderList` that choose LOD vs full-detail vs super-LOD (they
  scale off `ms_lodDistScale` = 1.2 and `ms_lowLodDistScale` = 1.0).

## Key takeaways

- The pipeline is an ordered set of named functions: `ScanWorld`/`SetupScanLists` (visibility) →
  `ConstructRenderList` (bucket by distance) → `RenderRoads` → `RenderEverythingBarRoads` →
  `RenderFadingInEntities`.
- `ConstructRenderList` reads `TheCamera.m_matrix` (`0xB6F03C`) — the same camera global as C38/C39, so the
  view drives the list.
- Each draw pass sets RenderWare render states through the device vtable and loops the matching count;
  `RenderEverythingBarRoads` reads `ms_nNoOfVisibleEntities` as its bound, tying the pass to the array C40.1
  sized.

**Continue:** [C40.3 — Where the pipeline sits in the frame →](03-where-the-frame-fits.md)
