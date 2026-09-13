# C7.2 — Version Encoding & the RW 3.6 Confirmation

> **The one-sentence version:** a bit-packed integer in every section header decodes to RenderWare
> 3.6.0.0 on all 19,299 assets — and it proves, from the data, what C0.3 proved from the executable.

[← C7.1 — The section stream](01-the-section-stream.md) · [Chapter 7 hub](C7-RenderWare-Stream.md) ·
[Next: C7.3 — Clumps and texture dictionaries →](03-clumps-and-txd.md)

**Confidence:** ✅ Verified

---

## 1. The field

The third dword of every section header ([C7.1 §1](01-the-section-stream.md)) is the **library ID** —
the RenderWare version and build that wrote the section, packed into 32 bits.

For every asset in San Andreas it is `0x1803FFFF`.

## 2. Unpacking

```c
uint32_t version = ((libraryID >> 14 & 0x3FF00) + 0x30000) | (libraryID >> 16 & 0x3F);
uint16_t build   =  libraryID & 0xFFFF;
```

Worked, step by step, for `0x1803FFFF`:

| Step | Value |
|---|---|
| `libraryID` | `0x1803FFFF` |
| `>> 14` | `0x0000600F` |
| `& 0x3FF00` | `0x00006000` |
| `+ 0x30000` | `0x00036000` |
| `\| (libraryID >> 16 & 0x3F)` | `0x00036003` |

Then the nibbles of `0x36003` read out as the version:

```
version = 3.6.0.0
build   = 0xFFFF (65535)
```

✅ *Verified.*

The encoding is odd-looking because it packs a four-part version and a 16-bit build into one dword,
with the major/minor pair offset by `0x30000` — RenderWare 3 is the baseline, so `+ 0x30000` recovers
the leading `3`. Files written by RenderWare 2 and earlier use a different, unshifted encoding
(`libraryID << 8` with no build field); nothing in San Andreas uses it.

## 3. Uniformity across the game

| Archive | DFF + TXD | Distinct library IDs |
|---|---:|---|
| `models/gta3.img` | 15,705 | `0x1803FFFF` only |
| `models/gta_int.img` | 2,418 | `0x1803FFFF` only |
| `models/player.img` | 542 | `0x1803FFFF` only |
| `models/cutscene.img` | 634 | `0x1803FFFF` only |
| **Total** | **19,299** | **one** |

✅ *Verified.* Every asset, one version, no exceptions.

That uniformity is itself informative: the shipped content was exported through a single tool chain
version. Mixed library IDs are the normal state of a long-lived game's asset set, and their absence
here means a final re-export rather than incremental accumulation.

**Practical consequence:** any file in a GTA SA installation whose library ID is *not* `0x1803FFFF` did
not ship with the game. That is a one-dword integrity check for a modded install, and a cheap one — 12
bytes per file.

## 4. The cross-confirmation

This is the point of the page.

[C0.3 §3.1](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md) identified the
renderer as RenderWare 3.6 from **90 `$Id:` source-path strings** in `.rdata`, 77 tagged `RW36Active`,
including `world/pipe/p2/d3d9/wrldpipe.c`. That is evidence about `gta_sa.exe`.

This page identifies it as RenderWare 3.6.0.0 from **a packed integer in 19,299 asset files**. That is
evidence about the data.

| Source | Method | Result |
|---|---|---|
| `gta_sa.exe` `.rdata` | source-path strings left by the linker | RenderWare 3.6, D3D9 pipeline |
| 19,299 assets | bit-packed version field | RenderWare 3.6.0.0, build `0xFFFF` |

The two share no mechanism, no file, and no assumption. Agreement between them is as close to certainty
as this project gets without source access.

It is also a useful contrast with the failure recorded in
[C5.6 §4](../C5-CWorld/06-the-sector-arrays-closed.md), where two "independent" derivations of the
repeat-sector size turned out to share a false premise. **Independence is a property of the reasoning,
not of the count of arguments** — and here it genuinely holds: one source is code, the other is data,
and neither was used to interpret the other.

## 5. Build `0xFFFF`

The build field is `65535` — all bits set.

🟡 *Reasoned:* this is a sentinel rather than a real build number. A shipping tool chain producing build
65535 on every one of 19,299 files is far more consistent with "unset/maximum" than with an actual
build counter that happened to land on the maximum value.

⏳ **Open:** whether the loader reads the build field at all. If it only checks the version nibbles, the
build is free space a mod tool could use as a marker — but that is speculation until the parsing code
is read.

---

### Key takeaways

- The library ID packs a **four-part version and a 16-bit build** into one dword, with `+ 0x30000`
  recovering the RenderWare 3 baseline.
- `0x1803FFFF` → **RenderWare 3.6.0.0, build `0xFFFF`**, shown step by step.
- **All 19,299 assets carry the identical ID** — a single-tool-chain final export, not incremental
  accumulation.
- Therefore: **any file whose library ID is not `0x1803FFFF` did not ship with the game** — a 12-byte
  integrity check.
- This **confirms C0.3's RW 3.6 finding from a completely independent direction** — code strings versus
  a data field.
- Contrast with C5.6: independence is a property of the *reasoning*, not the number of arguments. Here
  it holds.
- Build `0xFFFF` is 🟡 a sentinel; whether the loader reads it is ⏳ open.

**Continue:** [C7.3 — Clumps and texture dictionaries](03-clumps-and-txd.md)
