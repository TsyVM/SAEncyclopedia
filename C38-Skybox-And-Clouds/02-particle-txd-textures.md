# C38.2 — Four sprites from particle.txd

If the sky is an untextured gradient ([C38.1](01-the-sky-is-not-a-box.md)), then everything textured in the
sky — the clouds, the moon — must come from somewhere else. It comes from **four textures**, loaded by name
in `CClouds::Init` at `0x7138D0`, all resident in `models/particle.txd`. There is no `sky.txd`, no
`clouds.txd`, no `moon.txd` in the retail tree; a `find` over `references/GTASA` for any such name returns
nothing. That absence is the first half of the finding.

## `CClouds::Init` (`0x7138D0`), read from the exe

The loader's body is a straight run of `RwTextureRead` calls, each preceded by its name string(s) pushed on
the stack. Read cold:

```
push 0 ; push 0x872874("cloud1")        ; call RwTextureRead  -> [0xC6AA78] gpCloudTex
push 0 ; push 0x872868("cloudmasked")   ; call RwTextureRead  -> [0xC6AA7C]
push 0x872860("lunarm") ; push 0x872858("lunar")    ; call RwTextureRead -> [0xC6BFC8] gpMoonMask
push 0x87284c("cloudhighm") ; push 0x872840("cloudhigh") ; call RwTextureRead -> [0xC6AA74] ms_vc.texture
```

`RwTextureRead(name, maskName)` is the RenderWare two-argument texture reader — the second push is a mask
name, `NULL` for the first two textures and a real mask (`lunarm`, `cloudhighm`) for the moon and the high
cloud. So the four sky textures are:

| Texture | Mask | Global | Role |
|---|---|---|---|
| `cloud1` | — | `0xC6AA78` (`gpCloudTex`) | the scrolling low clouds |
| `cloudmasked` | — | `0xC6AA7C` | masked cloud variant |
| `lunar` | `lunarm` | `0xC6BFC8` (`gpMoonMask`) | **the moon** |
| `cloudhigh` | `cloudhighm` | `0xC6AA74` (`ms_vc.texture`) | the high / volumetric cloud |

`derive_sky.py` proves this two ways: `sky_texture_names_present` reads the NUL-terminated string at each of
the six name addresses and confirms it equals the expected name, and `init_loads_sky_textures` confirms
`cloud1` and `cloudmasked` are actually pushed inside the `Init` body (so the names are *used by the loader*,
not merely present in `.rdata`).

## The moon is a masked sprite, sized by one global

There is no separate moon texture file and no phase-frame strip. The moon is the `lunar` sprite drawn through
its `lunarm` alpha mask, and its on-screen size is a single global read in `CClouds::Render`:

```
moonSz = screenSize × (MoonSize × 2 + 4)
```

`MoonSize` lives at **`0x8D4B60`** and reads **3** in the shipped image (`derive_sky.py`:
`moon_size_default_3`), giving a default `moonSz = screenSize × 10`. The game advances `MoonSize` as the
in-game days pass, which is how the moon appears to wax and wane — not by swapping textures, but by scaling
one sprite. (The phase *stepping* logic lives in the day/night update path, not in `CClouds`, and is left as
an ⏳ for a follow-on; what is proven here is that the moon is one masked sprite scaled by this global.)

## Why `particle.txd` and not a dedicated sky TXD

The same TXD holds the coronas (`coronastar`, `coronamoon`, `coronaringb`, `coronaheadlightline` …), which
is why the sun, the moon-glow, headlight flares and the sky clouds are all drawn by the same corona/cloud
machinery: they are neighbours in one texture dictionary. This is the rendering-side echo of what
[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) found for the HUD — a *named* sprite dictionary, not a grid — and
it is why "extract the skybox" has no answer: the sky's textures are cloud and moon sprites filed with the
particle effects, and the sky itself has none.

## Key takeaways

- `CClouds::Init` @`0x7138D0` loads exactly four textures by name from `particle.txd` —
  `cloud1`, `cloudmasked`, `lunar` (+`lunarm`), `cloudhigh` (+`cloudhighm`) — into four fixed globals.
- The moon is the `lunar` sprite masked by `lunarm`, scaled by a single `MoonSize` global (@`0x8D4B60`,
  default 3); phases are a scale animation, not a texture swap.
- No dedicated sky texture ships; the sky sprites are filed with the particle effects, corroborating the
  C38.1 finding that the sky proper is untextured.

**Continue:** [C38.3 — Stars, fog and volumetric cloud counts →](03-stars-fog-volumetric.md)
