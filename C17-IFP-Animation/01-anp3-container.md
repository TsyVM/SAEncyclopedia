# C17.1 — The ANP3 Container

> **The one-sentence version:** three levels of nesting, every header exactly 36 bytes with the same
> shape, and a package name that reproduces its own filename on all 132 files.

[← Chapter 17 hub](C17-IFP-Animation.md) · [Next: C17.2 — Frames →](02-frames.md)

**Confidence:** ✅ Verified over all 132 files

---

## 1. The layout

```c
struct IfpHeader {           // 36 bytes
    char     magic[4];       // 'ANP3'
    uint32_t fileSize;       // bytes following this field
    char     packageName[24];
    uint32_t numAnimations;
};

struct AnimHeader {          // 36 bytes
    char     name[24];
    uint32_t numObjects;
    uint32_t frameBytes;     // total frame payload for this animation — §5
    uint32_t always1;        // 1 in all 1,557 animations
};

struct ObjectHeader {        // 36 bytes
    char     name[24];       // bone name
    int32_t  frameType;      // 3 or 4 — C17.2
    int32_t  numFrames;
    int32_t  boneId;         // NOT the object index — C17.3 §4
};
```

✅ *Verified.* **Every header is 36 bytes and every one is `name[24]` followed by three dwords.** The
file header substitutes `magic + fileSize` for the first eight bytes of what would otherwise be a name,
which is why it lands on the same width.

That uniformity is unusual and useful: a reader can advance 36 bytes at every level without
special-casing.

## 2. `8 + fileSize` is the true end

`fileSize` counts the bytes *after* itself, so the payload ends at `8 + fileSize`.

| Check | Result |
|---|---|
| `8 + fileSize` fits inside the archive slot | **132 / 132** ✅ |
| Full recursive walk terminates exactly there | **132 / 132** ✅ |

The second is the real result. A walk that descends into 1,557 animations and 38,825 objects, choosing
between two frame sizes 38,825 times, arrives at the declared byte on every file. Any error in any
header width or frame size would desynchronise and miss.

## 3. The package name reproduces the filename

| File | `packageName` |
|---|---|
| `airport.ifp` | `AIRPORT` |
| `attractors.ifp` | `Attractors` |
| `bar.ifp` | `BAR` |
| `benchpress.ifp` | `benchpress` |
| `bf_injection.ifp` | `BF_injection` |

✅ **132 / 132 match the filename stem**, case-insensitively — casing is inconsistent (`AIRPORT` vs
`benchpress`), which is itself a hint that the field was typed by hand rather than generated.

This is the same class of self-identification as `areaId` reproducing its filename in
[C12.1 §3](../C12-Path-Network/01-nodes-dat.md), and it serves the same purpose here: a field that
independently reproduces external information **cannot be at the wrong offset**.

## 4. `3dsmax5` — a name field full of leftover memory

The first animation in `airport.ifp` has this in its 24-byte name field:

```
offset 36:  74 68 72 77 5f 62 61 72 6c 5f 74 68 72 77 00   "thrw_barl_thrw\0"   (15 B)
offset 51:  33 64 73 6d 61 78 35 00                        "3dsmax5\0"          (8 B)
offset 59:  3d                                             '='                  (1 B)
                                                           ────────────────────
                                                           24 bytes
```

The animation is named `thrw_barl_thrw`. The remaining 9 bytes are **not padding** — they contain a
second complete string, `3dsmax5`, and a stray `=`.

🟡 *Reasoned:* this is **uninitialised buffer content**. The exporter wrote a 24-byte field without
clearing it first, so whatever was in that memory — here, the tool's own version string — was
serialised into the file.

It is the same artefact class as the stale heap pointer in
[C12.1 §1.1](../C12-Path-Network/01-nodes-dat.md): **parts of these files are memory dumps, not designed
formats.** And like that one, it is accidentally informative — it names the export tool.

**For readers:** always truncate at the first NUL. Anything after it in a fixed-width name field is
noise, and in this format it is *specifically* noise that looks like a valid string.

Many object names also carry a **leading space** — ` Pelvis`, ` Spine` — so names must be stripped as
well as truncated.

## 5. The frame-byte field validates the whole decode

The animation header's third dword is the total frame payload for that animation:

```
frameBytes == Σ over objects of (frameSize[type] × numFrames)
```

| Hypothesis | Matches |
|---|---:|
| `frameBytes == total frame bytes` | **1,557 / 1,557** ✅ |
| `frameBytes == frame bytes + 36 per object header` | 0 |

This is the single strongest check in the chapter, because it is **sensitive to the frame sizes**. If
type 3 were 8 bytes instead of 10, every animation containing a type-3 track would disagree — and
35,791 of 38,825 tracks are type 3.

That is exactly how the frame sizes were found ([C17.2 §3](02-frames.md)): an initial guess of 8 bytes
failed on all 132 files, and 10 succeeded on all of them.

The fourth dword is **`1` in every one of the 1,557 animations** — ⏳ constant, meaning unknown.

## 6. Reading it

```python
FRAME_SIZE = {3: 10, 4: 16}

def ifp(path_bytes):
    b = path_bytes
    assert b[:4] == b'ANP3'
    end = 8 + struct.unpack_from('<I', b, 4)[0]
    package = b[8:32].split(b'\0')[0].decode('latin-1')
    off = 36
    for _ in range(struct.unpack_from('<I', b, 32)[0]):
        name = b[off:off+24].split(b'\0')[0].decode('latin-1').strip()
        nobj, frame_bytes, _one = struct.unpack_from('<3I', b, off+24)
        off += 36
        counted = 0
        for _ in range(nobj):
            bone = b[off:off+24].split(b'\0')[0].decode('latin-1').strip()
            ftype, nframes, bone_id = struct.unpack_from('<3i', b, off+24)
            off += 36 + FRAME_SIZE[ftype] * nframes
            counted += FRAME_SIZE[ftype] * nframes
        assert counted == frame_bytes        # holds on all 1,557 retail animations
    assert off == end                        # holds on all 132 retail files
```

Both assertions hold on unmodified retail data, so both are safe to keep. Together they catch any
misreading of any header field or frame size — which is more than the field-count checks that were the
only defence in the text formats ([C15.3 §2](../C15-Timecycle/03-the-fifth-defect.md)).

**Binary formats with redundant size fields are far more verifiable than text ones.** IFP declares its
own totals at two levels; `timecyc.dat` declares nothing, which is why a missing value there could hide.

---

### Key takeaways

- **Three nested headers, all exactly 36 bytes**, all `name[24]` + three dwords.
- The payload ends at **`8 + fileSize`**, and the full recursive walk lands there on **132 / 132** files.
- **`packageName` reproduces the filename on 132/132** — a field that cannot be at the wrong offset.
- ⚠️ Name fields contain **uninitialised buffer content**: `thrw_barl_thrw\0` is followed by `3dsmax5\0`
  in the same 24 bytes. Truncate at NUL *and* strip — many names have a leading space.
- **`frameBytes` matches the computed sum on 1,557/1,557** — the check that proves the frame sizes,
  because it is sensitive to them.
- The fourth animation dword is **`1` in all 1,557** — ⏳ unknown.
- **Binary formats with redundant size fields verify themselves**; the text formats in C13–C16 could not.

**Continue:** [C17.2 — Frames](02-frames.md)
