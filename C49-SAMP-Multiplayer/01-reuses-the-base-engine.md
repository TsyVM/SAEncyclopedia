# C49.1 — SA:MP reuses the base engine

The most instructive thing about SA:MP's asset folder is how *unremarkable* it is. A total multiplayer
conversion, and every content file in `SAMP/` is a format this encyclopedia already decoded for the base
game. SA:MP added content by using the engine's own extensibility, not by inventing anything.

## The `SAMP/` folder, format by format

| File | Format | Documented in | Content |
|---|---|---|---|
| `SAMP.img` | **VER2 IMG** archive | [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) | 1,673 entries (models/textures) |
| `SAMP.ide` | standard **IDE** | [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) | object definitions, IDs 18631+ |
| `SAMP.ipl` | standard **IPL** | [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) | object placements |
| `SAMPCOL.img` | **COL** archive | [C6](../C6-Collision/C6-Collision.md) | collision for the added objects |
| `samaps.txd`, `blanktex.txd` | **TXD** | [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) | texture dictionaries |
| `custom.img` / `CUSTOM.ide` | IMG / IDE (empty stub) | [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)/[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) | user-content slots |
| `samp.saa` | proprietary SA:MP archive | — | ⏳ opaque |

`derive_samp.py` confirms `SAMP.img` opens with the **`VER2`** magic and declares **1,673** entries — the
identical archive format [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) reads for `gta3.img`. No new
container was needed.

## The model-ID trick

The one genuinely clever thing in the data is *where* SA:MP's objects live in the model-ID space. The base
game uses IDs up to ~20,000 with large unused gaps; `SAMP.ide` places its added objects **above 18631**
(e.g. `18631, NoModelFile`, `18632, FishingRod`, …). `derive_samp.py` confirms the IDE adds a large block of
objects in the 18631+ range (`sampide_adds_high_ids`).

This is the whole extensibility mechanism in one number. Because the [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
IDE system is just "id → model/txd/params" and the [C2](../C2-CStreaming/C2-CStreaming.md) streaming-info
array is indexed by id, an add-on can claim a free id range, define models there, and the engine streams them
exactly like Rockstar's own — no engine change, no format change. SA:MP's server can then reference those ids
to spawn its objects. The base game's **26,316-entry streaming-info array** ([C2](../C2-CStreaming/C2-CStreaming.md))
has room for them precisely because the id space was left sparse.

## Why this matters for the encyclopedia

SA:MP is external, but this page is a *validation* of the base-game chapters. Every claim
[C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)/[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)/[C6](../C6-Collision/C6-Collision.md)
make about the container formats is confirmed by a completely independent producer using them: SA:MP's tools
wrote a VER2 IMG and a valid IDE that the retail engine loads. If the format documentation were wrong, SA:MP's
content would not load. It is the same "independent producers agree" evidence the project prizes — here the
second producer is the SA:MP team rather than a second Rockstar file.

## Key takeaways

- Every content file in `SAMP/` is a **base-engine format** already documented — VER2 IMG
  ([C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)), IDE/IPL ([C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)),
  COL ([C6](../C6-Collision/C6-Collision.md)), TXD ([C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)).
- SA:MP extends the game by claiming **model IDs above 18631** in a standard IDE — the
  [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)/[C2](../C2-CStreaming/C2-CStreaming.md) id-space mechanism,
  no engine change required.
- That an independent producer's IMG/IDE loads in the retail engine **corroborates** the base-game format
  chapters. Only `samp.saa` uses a proprietary format (⏳).

**Continue:** [C49.2 — The network client →](02-the-network-client.md)
