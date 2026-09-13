# C48.1 — Load and boot config

Five of the uncovered files are the engine's startup manifests — line-oriented lists that tell it which
`.ide`, `.img`, `.ipl` and `.col` files to pull in, and one small set of always-present definitions. They are
the [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) family, applied to booting the game.

## `default.dat` — the master load list

`default.dat` is the boot manifest: a list of `KEYWORD path` lines the engine processes at startup, the
sibling of `gta.dat` ([C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)/[C22](../C22-Map-Zones/C22-Map-Zones.md)).
The keywords are the load directives:

```
IDE      DATA\DEFAULT.IDE          ; object/model definitions
IDE      DATA\VEHICLES.IDE
IDE      DATA\PEDS.IDE
COLFILE  0  MODELS\COLL\WEAPONS.COL ; a collision archive (C6)
```

`IDE` loads a definitions file, `IMG` mounts an archive ([C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)),
`COLFILE` loads collision ([C6](../C6-Collision/C6-Collision.md)), `SPLASH`/`TEXDICTION` set boot resources.
`default.dat` holds the entries loaded for *every* session before the map-specific `gta.dat` runs — the base
layer of the world. `derive_datasweep.py` confirms it is an `IDE`/`IMG`/`COLFILE` list (`load_config_files`).

## `gta_quick.dat` — the fast-boot variant

`gta_quick.dat` is the same format, a **shorter** manifest used for a quick startup path (development /
faster loads) — it lists the `CARREC.IMG` path record archive and the generic map IDEs but skips much of the
full world. It is a trimmed `default.dat`: same grammar, fewer entries. Its existence is a small window into
the build process — a "load less, start faster" config kept in the retail files.

## `default.ide` — the always-loaded definitions

Where `default.dat` says *which files to load*, `default.ide` *is* one of those files — the object and weapon
definitions present in every session. It uses the standard IDE section grammar
([C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)):

```
objs
320, airtrain_vlo, generic, 1, 2000, 0     ; id, model, txd, count, draw-dist, flags
end
weap
321, gun_dildo1, gun_dildo1, null, 1, 50, 0
end
```

`derive_datasweep.py` confirms the `objs` and `weap` sections. These are the low-id models the engine assumes
are always resident — generic objects and the base weapon models — which is why they live in a `default`
file rather than a map-specific one.

## `txdcut.ide` — cutscene texture parents

`txdcut.ide` is a specialised IDE holding only a **`txdp`** section — TXD *parent* assignments for cutscenes:

```
txdp
csbravura, bravura     ; cutscene txd 'csbravura' inherits from 'bravura'
cscopcarla, copcarla
```

A `txdp` entry says "texture dictionary A's parent is B", so A only needs to store the textures that *differ*
from B ([C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)). For cutscenes, the special `cs*`
character/vehicle texture dictionaries inherit from their in-game counterparts, saving memory — the cutscene
Bravura reuses the gameplay Bravura's textures. It is a small file (25 lines) but a real memory-optimisation
mechanism.

## `animviewer.dat` — a developer artifact

`animviewer.dat` is the odd one: an IDE load-list (`IDE DATA\MAPS\generic\vegepart.IDE` …) used by the
game's built-in **animation viewer** debug tool, not the shipped game path. It is a developer artifact left
in the retail files — like the `gta_quick.dat` fast-boot config, evidence of the tools the developers used,
preserved on the disc. Documented for completeness; it drives no retail gameplay.

## Key takeaways

- `default.dat` (+ the fast-boot `gta_quick.dat`) are the startup manifests — `IDE`/`IMG`/`COLFILE` lists, the
  base layer loaded before the map's own `gta.dat`.
- `default.ide` holds the always-resident object/weapon definitions; `txdcut.ide` is a `txdp` file giving
  cutscene texture dictionaries a memory-saving parent ([C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)).
- `animviewer.dat` is a preserved **developer** debug-tool config, not a retail path — coverage, not
  gameplay.

**Continue:** [C48.2 — Economy and customization →](02-economy-and-customization.md)
