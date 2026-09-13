# C7.4 — Modding DFF and TXD files

## Why DFF/TXD modding is the foundation of vehicle and map mods

Every visual mod for San Andreas begins with a DFF (geometry/clump) and a TXD (texture dictionary). Understanding the RenderWare stream structure from C7.1–C7.3 gives the modder the "why" behind the tools and the errors they produce.

## The DFF pipeline for a vehicle mod

Adding or replacing a vehicle model involves:

1. **Name matching:** the DFF's internal clump name must match the model name in `vehicles.ide`. The name is the null-terminated string in the FrameList's first frame — this is what the loader uses to bind the DFF to the model ID. A name mismatch causes the model to silently fail to load (returns to the original model).

2. **TXD binding:** the DFF references textures by name only (not by path or ID). The loader resolves the name against the TXD that the `cars` row in `vehicles.ide` specifies as the TXD column. The TXD must be in the same `.img` archive (or loaded by the same streaming request).

3. **COL file:** the collision model (C6) is not in the DFF — it is a separate `.col` file. A new vehicle needs both the DFF replacement and a matching `.col` update. Vehicles without collision have no solid body (fall through the world).

4. **Atomic count:** the engine expects a specific atomic structure per vehicle type. `CAutomobile`-derived vehicles typically have a main body atomic plus door/wheel frame children. Extra atomics for `extra_1`, `extra_2`, `col_OK`, `dam_OK` etc. are optional but must be named correctly if present.

## The TXD pipeline

A TXD is a count header + N native texture sections (C7.3). Each native texture section contains:
- A D3D9 format identifier (always `0x15` = D3DFMT\_DXT / 0x16 = D3DFMT\_RGBA etc.)
- Mip-map chain
- The raw compressed pixel data

**Format compatibility:** SA uses D3D9's DXT1, DXT3, and RGBA8888. DXT1 (4 bits/pixel, no alpha) is the most common format in the stock game. DXT3 (8 bits/pixel, explicit alpha) is used for transparent surfaces. DXT5 (interpolated alpha) is present in some vehicle TXDs.

**Mip-map chains:** the engine expects mip maps for distant rendering. A TXD with only one mip level loads and renders but shows shimmering at distance (the absence of the mip chain causes the GPU to sample from the base level for all distances, producing aliasing).

**VRAM budget:** each loaded TXD occupies GPU VRAM proportional to its texture size × mip chain. A 512×512 DXT1 TXD = ~170 KB. The streaming system (C1) manages this automatically, but adding many large TXDs can exhaust the streaming buffer allocated at startup.

## The streaming hook for img replacement

The standard mod workflow:
1. Pack the new DFF and TXD into `gta3.img` (replacing existing entries)
2. The streaming system (C1) will load the new file when the model is requested

The `.img` directory is indexed at startup from `gta3.img`'s header. The directory entry for each file is a 32-byte record: `[sectorOffset (4B)][sectorSize (4B)][name (24B)]`. The offset is in 2048-byte sectors from the start of the `.img`. The streaming system uses this offset to issue the read.

**Adding new entries:** tools that add new `.img` entries must rewrite the directory, which shifts all sector offsets. The streaming system recalculates offsets from the directory at startup, so this is safe as long as the modified `.img` is consistent.

## Plugin compatibility

C7.3 established that the Extension section carries plugin data. The plugins that matter for modding:
- **`skin` (0x116)**: bone weights and bone indices for ped/vehicle skeletons — absent on static geometry
- **`hanim` (0x11E)**: the hierarchical animation controller — required for animated skeletons to work
- **`matfx` (0x120)**: material effects (reflection maps, dual-UV sets) — absent in most non-vehicle geometry
- **`uvanim` (0x135)**: UV animation channels — used on some special effects and signs
- **`2dfx` (0x100)**: 2dEffect records (lights, particle generators) attached to a model — covered in C10

A tool that strips unknown extensions will silently remove these plugins, breaking skinning, reflections, or lights. The safest approach is to preserve all extension payloads verbatim and only modify the specific geometry data.

## Common mod errors and their RW-stream root causes

| Error | Root cause in the RW stream |
|---|---|
| Model reverts to stock | Internal name mismatch between DFF frame and `vehicles.ide` model name |
| White/untextured model | TXD not in the same streaming archive, or texture name mismatch |
| Model has no collision | COL file not updated — separate from the DFF |
| Animation broken | `skin` / `hanim` plugin stripped or corrupted |
| Game crashes on model load | Malformed section header (wrong size or type) |
| Distant shimmering | Single-mip TXD — mip chain absent |

**Previous:** [C7.3 — Clumps and texture dictionaries](03-clumps-and-txd.md)  
**Up:** [C7 — RenderWare Stream](C7-RenderWare-Stream.md)
