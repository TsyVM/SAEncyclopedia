# C17.2 — Frames

> **The one-sentence version:** two frame sizes, 10 and 16 bytes, found by a wrong guess that failed on
> all 132 files and a right one that succeeded on all 132 — with an incrementing time field visible in
> the raw hex that made the stride obvious once looked at.

[← C17.1 — The ANP3 container](01-anp3-container.md) · [Chapter 17 hub](C17-IFP-Animation.md) ·
[Next: C17.3 — The skeleton →](03-the-skeleton.md)

**Confidence:** ✅ Verified (sizes, layout) / ⏳ (quantisation scale)

---

## 1. The two types

```c
struct FrameType3 {          // 10 bytes — rotation only
    int16_t  rotation[4];    // quaternion, quantised
    uint16_t time;
};

struct FrameType4 {          // 16 bytes — rotation + translation
    int16_t  rotation[4];
    uint16_t time;
    int16_t  translation[3];
};
```

| Type | Size | Tracks | Share |
|---:|---:|---:|---:|
| **3** | **10 B** | 35,791 | **92.2 %** |
| **4** | **16 B** | 3,034 | 7.8 % |

✅ *Verified.* Only these two values appear across all 38,825 tracks in the game.

## 2. The time field gives away the stride

The stride did not need to be guessed. Reading raw bytes from the first track of `airport.ifp`:

```
34 fd ec 00 96 0a a1 0b | 00 00 | fe ff 15 00 70 ff
29 fd e2 00 96 0a 9e 0b | 04 00 | f3 ff 15 00 6f ff
15 fd d0 00 98 0a 99 0b | 08 00 | e9 ff 15 00 6f ff
e0 fc 9f 00 9b 0a 8b 0b | 10 00 | d2 ff 15 00 6b ff
                          ^^^^^
```

**A little-endian `uint16` at offset 8 incrementing by a constant** — `0x0000`, `0x0004`, `0x0008`,
`0x0010`. Locating the repeat of that pattern gives the record length directly: 16 bytes.

The same read on a type-3 track shows the increment at offset 8 with a 10-byte repeat.

**A monotonically increasing field is the cheapest possible stride detector.** It is the same technique
as the `0x7FFE` marker scan in [C12.1 §2](../C12-Path-Network/01-nodes-dat.md) — find something that
recurs predictably, measure the gap.

## 3. ⚠️ How the wrong size was caught

The first attempt assumed type 3 was **8 bytes** — rotation only, no time field, which is the obvious
guess for "rotation-only frame."

Result: **0 of 132 files parsed.** The walk desynchronised inside the first animation and reported 67
different "unknown frame type" values — garbage read from the middle of frame data.

Changing 8 to 10 gave **132 of 132 exact**.

Two things are worth extracting:

**The failure was loud, not quiet.** A wrong stride in a nested binary format produces immediate
nonsense, because the next header read lands mid-record. Compare the text formats of C13–C16, where a
wrong assumption produced *plausible* output — the shifted timecycle row that passed a range check
([C15.3 §2](../C15-Timecycle/03-the-fifth-defect.md)), or the parse artefact that manufactured a
non-existent vehicle bug ([C13.2 §4](../C13-Vehicle-Data/02-cars-section-repaired.md)).

**Binary formats fail safely; text formats fail plausibly.** That is a general property worth naming,
and it is why the confidence in this chapter is higher than in any of the four preceding it despite
none of the frame *semantics* being derived.

**The `frameBytes` field made it a one-line test.** Because each animation declares its own frame-byte
total ([C17.1 §5](01-anp3-container.md)), a candidate size can be checked without walking the whole
file — sum the hypothesis, compare to the declared value, done.

## 4. Root motion

Type 4 carries three extra `int16` values. Where it appears is the tell:

| Measurement | Value |
|---|---:|
| Animations whose **first** object is type 4 | **1,336 / 1,557 (85.8 %)** |
| Animations with **exactly one** type-4 track | 1,146 (73.6 %) |
| Animations with **no** type-4 track | 205 |
| Type-4 tracks at object index 0 | 1,336 |

✅ *Verified.*

🟡 *Reasoned:* this is the standard skeletal-animation economy. **One bone translates — the root — and
every other bone only rotates**, because a skeleton's shape is fixed and only its joint angles change.
Storing translation for all 26 bones would cost 60 % more with nothing to show for it.

The 205 animations with no type-4 track are in-place motions with no root displacement; the 74
animations with three type-4 tracks are 🟡 plausibly multi-part objects rather than characters.

The first object's name is usually `Root` or `normal` ([C17.3](03-the-skeleton.md)), consistent with the
reading.

## 5. Storage economy

| | Bytes |
|---|---:|
| Total keyframes | 868,337 |
| At 10 B (type 3) | 35,791 tracks |
| At 16 B (type 4) | 3,034 tracks |

The whole animation set — 1,557 animations across 132 packages — fits in **10.7 MB** of archive
(`anim.img`). At 868,337 keyframes that is roughly **12 bytes per keyframe including all headers**.

For comparison, storing rotation as four 32-bit floats plus a float time would be 20 bytes per frame
before any translation — so the `int16` quantisation is saving roughly half, and it is the reason the
whole animation set streams from a 10 MB archive rather than a 20 MB one.

## 6. What is not derived

⏳ **The quantisation scale.** Rotation is four `int16` values, which is a quaternion at some fixed
divisor — 4096 and 32767 are both conventional. Reconstructing an actual pose and checking which
divisor produces a unit quaternion would settle it in one pass; that was not done.

⏳ **The time unit.** Sampled deltas between consecutive frames were **4, 6, 24 and 32** — so the step
is *variable*, not fixed, and the field is a timestamp rather than a frame index. What one unit
represents in seconds is not established.

⏳ **Translation scale**, likewise — three `int16` values at an underived divisor.

Per [C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md), community values exist for all three
and are **not adopted here**. The structural facts — sizes, field order, counts — are verified on the
full population; the scales are not, and the difference is stated rather than blurred.

---

### Key takeaways

- **Two frame types only**: 10 bytes (rotation `int16[4]` + time `uint16`) and 16 bytes (+ translation
  `int16[3]`). **92.2 % are type 3.**
- The stride was found from an **incrementing time field visible in raw hex** — the cheapest stride
  detector there is.
- ⚠️ An initial 8-byte guess **failed on 132/132 files**; 10 bytes succeeded on 132/132.
- **Binary formats fail safely; text formats fail plausibly.** The wrong stride here produced immediate
  garbage, where C13–C16's wrong assumptions produced believable output.
- **Type 4 is root motion** — first object in 85.8 % of animations, exactly one per animation in 73.6 %.
- The set averages **~12 bytes per keyframe including headers**; `int16` quantisation roughly halves
  what floats would cost.
- ⏳ Rotation scale, translation scale and the time unit are **underived** — community values exist and
  are deliberately not adopted.

**Continue:** [C17.3 — The skeleton](03-the-skeleton.md)
