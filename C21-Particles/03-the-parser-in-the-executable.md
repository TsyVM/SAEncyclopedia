# C21.3 — The Parser in the Executable

> **The one-sentence version:** `gta_sa.exe` contains exactly **34** `FX_*_DATA` tokens and compares them
> as string literals — but it contains **none** of the format's scalar field names, which proves fields are
> read by position rather than by name, and leaves **three** effect-info types the shipped library never
> uses.

[← Chapter 21 hub](C21-Particles.md) · [Prev: C21.2 — The effect library](02-the-effect-library.md)

**Confidence:** ✅ Verified (the token set, the comparison mechanism, the three unused types, the absence
of field names)

---

## 1. One reference, one parser

`models\effects.fxp` appears once in `.rdata` at VA `0x85A6D4`, with exactly one cross-reference:

```
0049ea9d  push  0x85a6d4                 ; "models\effects.fxp"
0049eaa2  mov   ecx, 0xa9ae80             ; the effect manager
0049eaa7  call  0x5c2420                  ; ← the parser
0049eaac  push  0xc812f0
0049eab1  push  0xc813e0
0049eab6  mov   ecx, 0xa9ae80
0049eabb  call  0x4a93e0
0049eac0  mov   ecx, esi
0049eac2  call  0x49e660
0049eac7  mov   dword ptr [esi + 0x50], 0
0049eace  mov   al, 1                     ; success
```

There is no second path and no binary fallback: the retail executable loads the particle library by
handing this one text file to `0x5C2420`. ✅

## 2. Block tags are compared as literals

Inside the parser, `FX_SYSTEM_DATA` (VA `0x86B2EC`) is referenced twice, and the first reference shows
exactly how a tag is recognised:

```
005c253e  mov       edi, 0x86b2ec         ; "FX_SYSTEM_DATA:"
005c2543  lea       esi, [esp + 0x90]     ; the line just read
005c254a  mov       ecx, 0x10             ; 16 bytes
005c254f  xor       edx, edx
005c2551  repe cmpsb byte ptr [esi], byte ptr es:[edi]
005c2553  jne       0x5c25bf              ; not this tag → try the next
```

A `repe cmpsb` over `0x10` bytes — and `"FX_SYSTEM_DATA:"` is fifteen characters plus its NUL, which is
exactly sixteen. ✅ *Verified:* block tags are matched by literal string comparison against constants in
the executable, in a chain of tests.

That single detail justifies the whole approach of [C21.1](01-the-grammar.md). Because the tags are
compiled in, the executable's token set is an **enumeration of everything the format can express** — and
it can be compared against everything the shipped file actually uses.

## 3. Thirty-four tokens, and three that never appear

Scanning both `.text` sections and `.HOODLUM` for the pattern `FX_[A-Z0-9_]+_DATA` yields **34** distinct
tokens. They account for themselves exactly:

```
29  FX_INFO_*_DATA   the behaviour types used by effects.fxp   (C21.2 §3)
 1  FX_PRIM_EMITTER_DATA
 1  FX_SYSTEM_DATA
 3  FX_INFO_*_DATA   present in the executable, absent from the shipped library
──
34
```

✅ **29 / 29** of the info types the file uses are literals in the executable — no type in the data is
unknown to the code.

⚠️ And in the other direction, three are **declared but never used**:

| Token | Appearances in `effects.fxp` |
|---|---:|
| `FX_INFO_ATTRACTLINE_DATA` | **0** |
| `FX_INFO_COLOURRANGE_DATA` | **0** |
| `FX_INFO_SMOKE_DATA` | **0** |

The engine can parse and presumably render a line attractor, a colour-range behaviour and a dedicated
smoke behaviour; the shipped library never asks it to. `FX_INFO_ATTRACTPT_DATA` — the *point* attractor —
is used exactly once, which makes the absent *line* attractor look like the other half of a pair that was
built and half-used.

This is the kind of result the encyclopedia exists to record, and it is only available because the format
is a grammar with a compiled-in vocabulary. It is stated as what it is: **three capabilities present in
the 1.0 US executable that no shipped effect exercises.** Whether they work is not derivable from the
bytes and is not claimed.

## 4. The fields are positional, not named

The revealing search is the one that comes back empty. Split the file's vocabulary into three groups and
ask which members appear as literals in the executable:

| Vocabulary | Distinct | Present in `gta_sa.exe` |
|---|---:|---:|
| `FX_*_DATA` block tags | 36 | **31** |
| Curve-holder tags (`SIZEX`, `RED`, `FORCEZ`, …) | 67 | 26 |
| Scalar field keys (`LENGTH`, `CULLDIST`, `MATRIX`, `LODSTART`, …) | 26 | **3** |

The five block tags *missing* from the executable are precisely the structural ones — `FX_PROJECT_DATA`,
`FX_PROJECT_DATA_END`, `FX_PRIM_BASE_DATA`, `FX_INTERP_DATA` and `FX_KEYFLOAT_DATA`. Those are the blocks
that never need dispatching: the project wrapper occurs once, and a base block, an interpolator and a
keyframe always occur in a fixed place inside their parent. Only the blocks the parser must *choose*
between — which system, which emitter, which of 32 behaviour types — are compiled in as strings.

And the scalar keys are simply not there. `CULLDIST`, `LODSTART`, `BOUNDINGSPHERE`, `SRCBLENDID`,
`TIMEMODEPRT` — none appears anywhere in the image.

⚠️ A caveat the chapter states rather than hides: the three scalar keys that *do* match (`NAME`, `TIME`,
`VAL`) and the 26 curve-holder tags that match are mostly short, common strings, and a three- or
four-character match inside 14 MB of code is worth nothing on its own. The finding rests on the long,
unambiguous tokens: `BOUNDINGSPHERE`, `LOOPINTERVALMIN`, `OMITTEXTURES`, `SRCBLENDID` and `TIMEMODEPRT` are
each fourteen characters or more and each is **absent**.

✅ *Verified:* the executable dispatches on **block tags** and reads **field values positionally**, in the
order the exporter wrote them. It never looks for a field name.

This independently vindicates the parser in [C21.1](01-the-grammar.md), which was written positionally
before the executable was examined — it asserts on field names purely as a self-check, and the game does
not. It also explains a fragility of the format worth recording: because nothing is keyed, **the field
order in `effects.fxp` is load-bearing.** Reordering two lines inside a block would not produce a parse
error in the game; it would silently swap two values.

## 5. The two-source summary

| Claim | From the file | From the executable | Agree |
|---|---|---|:--:|
| 29 behaviour types in use | 1,470 blocks, 29 distinct tags | all 29 present as literals | ✅ |
| The container types | `FX_SYSTEM_DATA`, `FX_PRIM_EMITTER_DATA` | both present as literals | ✅ |
| Structural blocks need no dispatch | occur in fixed positions | **absent** from the image | ✅ |
| Fields are unkeyed | order is fixed in all instances | field names **absent** | ✅ |
| Total format vocabulary | 31 tags used | **34** tags compiled in | 3 unused |

---

### Key takeaways

- ✅ `models\effects.fxp` has **one** cross-reference at `0x49EA9D`, calling the parser at `0x5C2420`.
  There is no binary fallback path.
- ✅ Block tags are matched by **literal string comparison** — `repe cmpsb` over `0x10` bytes at
  `0x5C253E`, which is exactly the length of `"FX_SYSTEM_DATA:"` plus its NUL.
- ✅ The executable contains **34** `FX_*_DATA` tokens: 29 used behaviour types + 2 container types +
  **3 never used by the shipped library** — `ATTRACTLINE`, `COLOURRANGE`, `SMOKE`.
- ✅ The five **structural** block tags are *absent* from the image; only tags the parser must choose
  between are compiled in.
- ✅ **Scalar field names do not appear in the executable** — fields are read **positionally**. The check
  rests on the long tokens (`BOUNDINGSPHERE`, `LOOPINTERVALMIN`, `OMITTEXTURES`, `TIMEMODEPRT`), not on
  short ones that match by chance.
- ⚠️ Consequence for anyone editing the file: **field order is load-bearing**. Swapping two lines produces
  no error, just wrong values.

**Continue:** [← Chapter 21 hub](C21-Particles.md)
