# C10.1 — The Record, and Two Corrections

> **The one-sentence version:** the section nobody could identify and the section everybody assumed
> was obvious turned out to be each other's opposite — and the difference between how the two claims
> were made is the whole lesson.

[← Chapter 10 hub](C10-2dEffect.md) · [Next: C10.2 — The effect-type census →](02-effect-type-census.md)

**Confidence:** ✅ Verified

---

## 1. The record

```c
// 2dEffect — RenderWare plugin 0x253F2F8, inside Geometry / Extension
struct TwoDEffectSection {
    uint32_t count;
    // repeated `count` times:
    struct {
        float    position[3];   // model-local
        uint32_t type;          // selects the payload layout
        uint32_t dataSize;      // payload bytes that follow
        uint8_t  data[dataSize];
    } entries[];
};
```

✅ *Verified* by walking **every DFF in `gta3.img`**:

| Measurement | Value |
|---|---:|
| DFFs scanned | 12,955 |
| DFFs carrying the section | 1,681 (13.0 %) |
| Sections walked | 1,681 |
| **Walks ending exactly on the section boundary** | **1,681 / 1,681** |
| Entries decoded | 17,395 |

The 20-byte entry header — three position floats, a type, a size — plus a self-describing payload is
what makes the array walkable without knowing any type's meaning, exactly like the section stream
itself ([C7.1 §1](../C7-RenderWare-Stream/01-the-section-stream.md)).

## 2. ✅ Correction one: `0x253F2F8` is `2dEffect`

[C8.3 §4](../C8-Geometry/03-identifying-plugins-by-size.md) recorded it as unidentified — 33
occurrences, sizes of 68 and 104 bytes, "no clean relationship to vertex or triangle counts" — and
predicted it would regress against **material count** once materials were decoded.

[C9 §9.4](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) tested that and disproved it.

Both pages were looking for the wrong shape. **The section has no fixed stride at all** — it is a
variable-length array whose entries differ in size by type. `(size − 4)` divided by the first dword gave
32, 76, 100 and 108 in different files not because the record size varies randomly, but because those
files contain *different mixtures of effect types*, each with its own payload size
([C10.2](02-effect-type-census.md)).

The failed prediction was useful: it eliminated the whole family of fixed-stride hypotheses, which is
what pointed at a variable-length structure.

## 3. ⚠️ Correction two: `0x253F2FE` is the node-name plugin

[C7.3 §2](../C7-RenderWare-Stream/03-clumps-and-txd.md) labelled `0x253F2FE` as `2dEffect`, reported 92
occurrences in 60 sampled models, and drew a conclusion from it:

> *"**`2dEffect` is the important one.** 92 occurrences in 60 models means world geometry routinely
> carries attached effect markers — lights, coronas, particle emitters, ped-attractor points. This is
> where the map's lighting actually comes from."*

**The identification is withdrawn.** Tracing where `0x253F2FE` actually sits:

```
Clump / FrameList / Extension / 0x253F2FE        525 occurrences in 300 sampled DFFs
```

and dumping its payload:

```
waterjumpx2        pier69_models04        lodpdmdocka_las2
```

Plain ASCII. It is the **frame node-name plugin** — the string that names a node in the transform
hierarchy. It has nothing to do with effects.

### What went wrong

The ID was matched against a lookup table of RenderWare plugin constants that was assembled from
memory rather than derived, and the resulting name was never checked against the section's contents or
its position in the tree. Two cheap checks were skipped:

- **Where does it live?** `FrameList/Extension` is the transform hierarchy, not geometry — effects on
  frames would have been odd.
- **What is in it?** Eleven bytes of printable ASCII is not an effect record.

Either would have caught it in seconds.

### What survives

The *conclusion* — that the map's lighting hangs off models rather than living in a separate file — is
correct, and [C10.3](03-the-light-record.md) confirms it. But it was reached from a mislabelled section,
which means it was luck rather than evidence.

That distinction matters more than the error. A right answer from wrong reasoning is not a finding; it
is a coincidence that will not generalise. This encyclopedia's value is that its ✅ claims were
*derived*, and an unverified lookup-table name dressed as ✅ is precisely the failure the tiering in
[C0.3 §1](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md) exists to prevent.

## 4. The rule this produces

Both corrections in this chapter, plus the repeat-sector error in
[C5.6 §4](../C5-CWorld/06-the-sector-arrays-closed.md), share one root cause: **a name was accepted
without asking what the bytes say.**

So, as a standing check before labelling any section:

1. **Where does it sit in the tree?** A plugin's parent constrains its purpose.
2. **What do the bytes look like?** Printable ASCII, float ranges, plausible counts — a 30-second
   sanity read.
3. **Does the size behave?** Fixed stride, or a function of a declared count, or variable — each
   implies a different structure ([C8.3 §5](../C8-Geometry/03-identifying-plugins-by-size.md)).

`0x253F2FE` fails check 2 as an effect record. `0x253F2F8` passes all three as one. Neither needed a
disassembler.

---

### Key takeaways

- `2dEffect` is `{ uint32 count; (pos[3], type, dataSize, data[dataSize]) × count }`, verified with
  **1,681 / 1,681 exact walks** across the full DFF population.
- **`0x253F2F8` is `2dEffect`** — closing C8.3 and explaining C9's failed prediction: it is
  **variable-length**, so no fixed-stride hypothesis could ever have fit.
- ⚠️ **`0x253F2FE` is the frame node-name plugin** — C7.3's label is withdrawn; its payload is plain
  ASCII in `FrameList/Extension`.
- C7.3's *conclusion* about map lighting was right, but reached from a mislabelled section — **a right
  answer from wrong reasoning is a coincidence, not a finding**.
- Standing check before naming a section: **where it sits, what the bytes look like, how the size
  behaves.** None of it requires a disassembler.

**Continue:** [C10.2 — The effect-type census](02-effect-type-census.md)
