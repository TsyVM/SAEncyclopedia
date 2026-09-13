# C38.1 — The sky is not a box

The single most important fact about the San Andreas sky is what it *isn't*. There is no cube-map, no sky
sphere, no textured dome, and no sky asset anywhere in the retail tree. The sky is drawn by two functions in
`CClouds`, and between them they never bind a texture.

## The gradient quad — `SetUpOneSkyPoly` (`0x713060`)

`SetUpOneSkyPoly` takes four corner positions and two colours — a top RGB and a bottom RGB — and writes a
four-vertex primitive into RenderWare's immediate-mode temp buffer. The colour assignment is the whole
story:

```
vertex 0 : RGBA(topRed,    topGreen,    topBlue,    0xFF)   U=0  V=0
vertex 1 : RGBA(topRed,    topGreen,    topBlue,    0xFF)   U=0  V=0
vertex 2 : RGBA(bottomRed, bottomGreen, bottomBlue, 0xFF)   U=0  V=0
vertex 3 : RGBA(bottomRed, bottomGreen, bottomBlue, 0xFF)   U=0  V=0
```

The top two vertices carry the sky-top colour, the bottom two the sky-bottom colour, every vertex fully
opaque, and every texture coordinate zero. When the hardware rasterises the quad it Gouraud-interpolates the
vertex colours down the screen — that vertical blend *is* the sky. Because `U` and `V` are both zero on all
four vertices and no raster is ever bound, the interpolation samples no texture at all. This is the proof,
in the code, that the sky is a colour ramp and nothing more.

`derive_sky.py` asserts the structural half of this (`skybox_is_untextured_gradient`): the body of
`SetUpOneSkyPoly` pushes **no** texture-name string from the `0x872xxx` block — unlike `Init` one function
over, which pushes four of them. A poly builder that references a texture name would betray a textured sky;
this one does not.

## Orienting and colouring it — `RenderSkyPolys` (`0x714650`)

`SetUpOneSkyPoly` only knows how to draw *one* band once it is handed corners and colours. `RenderSkyPolys`
is what computes them. Its opening instructions, read cold from the exe, match the reconstruction
line-for-line:

```
0x714650: mov   eax, [0xB6F03C]      ; TheCamera.m_matrix
0x71465d: cmp   eax, esi
0x714660: je    0x71467b             ; no matrix -> fall back to heading
0x714662: ...    load forward vector from the matrix
0x71467b: fld   dword [0xB6F038]     ; camera heading
0x714681: fsin                       ; norm.x = -sin(heading)
0x714683: fchs
0x714689: fld   dword [0xB6F038]
0x71468f: fcos                       ; norm.y =  cos(heading)
```

That `-sin(heading)`, `cos(heading)` pair is the camera-facing normal the sky bands are built around: the
quad always spans the screen no matter which way the player looks, which is why the gradient never appears
to rotate. The function then sets the render state for an untextured, depth-disabled draw
(`rwRENDERSTATETEXTURERASTER = NULL`, `ZTEST`/`ZWRITE` off, blend `SRCALPHA`/`INVSRCALPHA`) — the sky is
laid down first, behind everything, writing no depth — and calls `SetUpOneSkyPoly` for each band.

## The horizon is banded, not a single split

The colours are not just "top" and "bottom". `RenderSkyPolys` builds an array of six horizon heights,
`aPosZ[0..5]`, from the camera Z plus a set of `SKYP_*` offset constants:

| Band edge | Constant |
|---|---|
| above horizon (top) | `SKYP_ABOVE_HORIZON_Z` |
| horizon | `SKYP_HORIZON_Z` |
| sea horizon | `SKYP_SEA_HORIZON_Z` |
| below horizon (bottom) | `SKYP_BELOW_HORIZON_Z` |

Six edges → the sky is stacked from several gradient quads (above-horizon sky, the horizon haze band, the
below-horizon fill), which is how the game gives you a brighter haze at the horizon line than at the zenith.
The band count (six `aPosZ` entries) is structural and marked ✅; the individual `SKYP_*` float values are
not reproduced here (they are layout constants, not counts) and are left as an ⏳ detail for anyone extending
the page.

## The timecyc tie — where the colours come from

The top and bottom colours handed to `SetUpOneSkyPoly` are the timecycle's. `RenderSkyPolys` reads
`CTimeCycle::m_BelowHorizonGrey` at **`0xB7CB10`** directly — the disassembly shows
`mov ebx, [0xB7CB10]` at `0x714748`, and `derive_sky.py` asserts that reference
(`skypolys_reads_timecyc_grey`). That grey is itself computed in `CTimeCycle` as a lerp of
`m_CurrentColours.m_nSkyBottomRed/Green/Blue` toward a horizon value, and `m_CurrentColours` (@`0xB7C4A0`) is
the per-frame interpolation of the raw `timecyc.dat` tables — the very columns
[C15](../C15-Timecycle/C15-Timecycle.md) documented:

| C15 column | Raw table VA | Consumed here as |
|---|---|---|
| Sky top R/G/B | `0xB7BF78` / … | top two vertices of each sky quad |
| Sky bot R/G/B | `0xB7BD50` / … | bottom two vertices; source of `m_BelowHorizonGrey` |

This is the chapter's cross-subsystem anchor: the renderer in `0x714xxx` reads a colour that
[C15](../C15-Timecycle/C15-Timecycle.md) derived independently from a text `.dat` file. A colour that one
subsystem parses from disk and another consumes from the same global cannot be at the wrong offset — the
same standard of evidence C28.3 used to re-derive C2.

## Key takeaways

- The sky is one function that builds a vertex-colour quad (`SetUpOneSkyPoly`) and one that orients and
  colours it to the camera (`RenderSkyPolys`); neither binds a texture, so the sky is a pure gradient.
- It is banded across six horizon heights, giving the horizon-haze look, not a single top-to-bottom fade.
- Its colours are the timecyc `SkyTop`/`SkyBot` columns of [C15](../C15-Timecycle/C15-Timecycle.md), read
  from `m_BelowHorizonGrey` @`0xB7CB10` — the tie is proven by a direct global reference in the exe.

**Continue:** [C38.2 — Four sprites from particle.txd →](02-particle-txd-textures.md)
