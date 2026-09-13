# C9.2 — Texture References

> **The one-sentence version:** a material does not contain its texture — it contains the texture's
> *name*, and that one indirection is the reason models and textures stream independently.

[← C9.1 — The material list](01-the-material-list.md) · [Chapter 9 hub](C9-Materials-And-Textures.md) ·
[Next: C9.3 — TextureNative →](03-texturenative-payload.md)

**Confidence:** ✅ Verified

---

## 1. The reference

```c
// Texture (0x06), inside a Material when isTextured != 0
//   Struct (0x01):  exactly 4 bytes
struct TextureStruct {
    uint16_t filterFlags;
    uint8_t  addressingUV;   // low nibble U, high nibble V
    uint8_t  _pad;
};
//   String (0x02):  texture name
//   String (0x02):  mask name (usually empty)
//   Extension (0x03)
```

✅ *Verified:* the struct is exactly 4 bytes on **1,124 of 1,124** sampled textures.

Names recovered from the sample read exactly as you would expect of a 2004 art pipeline:

```
planks01        Aascaff128       greywallc128     plasticdrum1_128
jumpside1_256   jumpside2_256    gen_chrome       rustyboltpanel
```

Note the embedded resolution hints (`128`, `256`) and the `gen_` prefix for shared generic textures —
naming conventions carrying information the format does not.

## 2. The indirection is the whole point

There is **no pointer, offset, or ID** from a material to its pixels. There is a string.

At load time the engine resolves that string against the currently-bound texture dictionary
([C7.3](../C7-RenderWare-Stream/03-clumps-and-txd.md)). Four consequences follow, and together they
explain most of the streaming architecture:

**Models and textures can stream independently.** A DFF can load while its TXD is still absent, and
vice versa. This is why they occupy *separate ranges* in the ID space
([C3.1](../C3-Model-Stores/01-the-partition.md)) rather than textures being embedded per model.

**One TXD serves many DFFs.** With ~35,000 textures against 15,325 models
([C7.3 §5](../C7-RenderWare-Stream/03-clumps-and-txd.md)), heavy reuse is the norm — the same wall
texture appears in hundreds of models, stored once.

**A missing texture is not fatal.** Name resolution fails, the material renders untextured, and the
game continues. This is the mechanism behind the familiar white/untextured buildings when a TXD fails
to stream, and it is a deliberate degradation rather than a bug.

**Texture replacement is name-based.** Any mod that swaps a texture works by supplying a dictionary
entry with the same name. Nothing in the model file needs editing — which is why texture mods are the
lowest-risk category of GTA modding, and why they compose so cleanly.

## 3. The mask name

Every texture section carries a **second** string for a mask texture. In the sampled data it is
consistently empty.

🟡 *Reasoned:* alpha in San Andreas is carried in the texture's own alpha channel (DXT3 for the 48
textures that need graded alpha, DXT1's 1-bit alpha otherwise —
[C9.3](03-texturenative-payload.md)), so a separate mask texture is unnecessary. The field is a
RenderWare capability the game does not use.

A reader must still parse the second string, because it occupies space whether or not it is populated.
Skipping it desynchronises the section walk — the same class of error as assuming a single root in
[C7.1 §3](../C7-RenderWare-Stream/01-the-section-stream.md).

## 4. Filter and addressing

The 4-byte struct packs the sampler state:

- `filterFlags` — RenderWare's filter mode enum (nearest / linear / mip variants).
- `addressingUV` — two 4-bit fields, U in the low nibble and V in the high.

⏳ **Open:** the per-value census. This pass verified the struct's *size* on all 1,124 samples and its
field layout, but did not tabulate which filter and addressing modes the shipped content actually uses.
That is a cheap follow-up — one pass over the same sample — and is queued rather than claimed.

The distinction matters for a texture-replacement tool: addressing mode is per-*reference*, not per-
texture, so the same texture can tile in one material and clamp in another. A tool that rewrites
textures without preserving these four bytes will change how existing models sample them.

---

### Key takeaways

- The texture reference struct is **exactly 4 bytes** on **1,124/1,124** — filter flags plus packed
  U/V addressing.
- A material names its texture by **string**, not by pointer or ID.
- That single indirection explains: **independent streaming ranges**, **TXD sharing**, **graceful
  degradation to untextured**, and **name-based texture modding**.
- The **mask-name string is always empty** but must still be parsed, or the section walk desynchronises.
- ⏳ The filter/addressing value census is queued, not claimed — and matters because addressing is
  per-reference, so the same texture may tile in one material and clamp in another.

**Continue:** [C9.3 — TextureNative: the pixel payload](03-texturenative-payload.md)
