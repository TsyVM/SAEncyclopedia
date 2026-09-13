# C20.5 — `EventVol.dat`: the File That Would Not Divide

> **The one-sentence version:** the reason 45,402 has no convincing record width is that there are no
> records — the file is a **flat array of 45,401 signed bytes indexed directly by audio event ID**, and
> the proof is that all **29** event IDs hard-coded in `gta_sa.exe` land on the 1.31 % of entries that
> carry a value.

[← Chapter 20 hub](C20-Audio.md) · [Prev: C20.4 — The loaders in the executable](04-the-loaders-in-the-executable.md)

**Confidence:** ✅ Verified (element type, flat indexing, extent, the sentinel; the census extended to
**81 accesses** and the proof to **44 / 44**) / 🟡 (the values are decibel trims) / ⏳ (the audio-event ID
namespace — now shown to be **not recoverable from the binary**, §7)

---

## 1. The wrong question

[C20.1 §4](01-the-config-tables.md) left this file open, and stated the difficulty honestly: 45,402
factorises as `2 × 3 × 7 × 23 × 47`, offering candidate record widths of 6, 14, 42, 46, 47, 69, 94 and
141, none more convincing than any other. [C20.4 §4](04-the-loaders-in-the-executable.md) then narrowed it
usefully by showing the game reads exactly `0xB159` = **45,401** bytes — which *eliminated* every clean
divisor of 45,402 at a stroke, since 45,401 = 83 × 547 and neither factor is a plausible field count.

At that point the honest conclusion was "the row width is not derivable from the file". It was also the
wrong question. A file with no plausible record width may simply have **no records** — and the way to find
out is not to keep factorising, but to watch the code use it.

## 2. Eighty-nine references, and what they all do

The loader stores the buffer's base pointer in the global at `0xBD00F8`
([C20.4 §4](04-the-loaders-in-the-executable.md)). That global is referenced **89 times** across the
executable's code sections. Disassembling each reference and reading the first typed access made through
the loaded pointer gives a startlingly uniform answer:

| Property of the typed accesses | Count |
|---|---:|
| Access width `byte` | **81 / 81** |
| Instruction `movsx` (sign-extending load) | **81 / 81** |
| Accesses using a **width-scaled** index (`[base + reg*N]`, N > 1) | **0** |
| Register-indexed (`[base + reg]`, runtime event ID) | 22 |
| Constant-displacement (`[base + ID]`, compile-time event ID) | 59 (44 distinct IDs) |

> The original write-up counted **56** by reading only the *first* typed access after each of the 89
> base-pointer loads. A wider scan — up to twelve instructions past each load, following the loaded
> register — raises the census to **81** typed accesses (`tools/derive_eventvol.py`, deep scan). The
> uniformity is unaffected: still `movsx byte` in every case, still nothing width-scaled.

```
004da1d1  mov    eax, dword ptr [0xbd00f8]
004da1d6  movsx  ecx, byte ptr [eax + 0x76]      ; event 118, sign-extended
```

```
004da0dd  mov    edx, dword ptr [0xbd00f8]
004da0f1  movsx  ecx, byte ptr [edx + ebx]       ; event id in ebx, unscaled
```

Two facts fall straight out. The element is a **signed byte** — `movsx`, never `movzx`, in all 56 cases,
so the values are meant to go negative. And the index is **never scaled**: there is no `imul`, no
`lea …*N`, no `[base + reg*4]` anywhere among them. A two-dimensional table indexed `[event][field]`
cannot be read without scaling by the row width somewhere. There is no row width because there are no
rows.

✅ *Verified:* `EventVol.dat` is `int8 volume[45401]`, indexed directly by an audio event ID.

> A limit on that claim, stated rather than hidden: this examined the typed accesses that consume each of
> the 89 base-pointer loads. It does not prove that no other code path anywhere scales an index into this
> buffer — it proves that the 81 accesses that immediately consume the pointer do not.

## 3. The proof: 44 for 44

Uniform addressing is suggestive but not conclusive — it is still, in principle, a guess about what the
bytes mean. The decisive test comes from the **constant** indices.

Of the 81 accesses, 59 use a literal displacement rather than a register, across **44 distinct event IDs**:
52, 53, 76, 77, 78, 84, 90, 91, 92, 93, 96, 99, 100, 102, 103, 107, 108, 109, 110, 111, 113, 114, 115, 116,
117, 118, 119, 120, 121, 122, 125, 128, 129, 141, 143, 144, 145, 151, 154, 157, 158, 164, 1113 and 1118.
These are event IDs the engine names at compile time — the sounds some specific piece of code always plays.
(The original write-up listed the 29 the narrower scan reached; the wider scan of §2 adds 15 more, and they
behave identically.)

Now look them up in the file. Recall that **44,804 of 45,401 entries are `0x80`** and only **597** carry a
value — a base rate of **1.31 %**. If the array interpretation were wrong in any way — wrong base, wrong
element width, wrong meaning for `0x80` — those 29 indices would fall on default entries about as often
as chance dictates.

| Index | 52 | 53 | 76 | 77 | 78 | 128 | 129 | 145 | 158 | 1113 | 1118 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Value | −6 | −2 | −26 | −3 | −6 | −22 | +20 | +14 | −28 | −16 | −12 |
| Default? | no | no | no | no | no | no | no | no | no | no | no |

✅ **44 of 44 carry a value. None is the default.** (The table shows a representative slice; the full 44
are in `RE-Data/data/eventvol_events.json`.)

At a 1.31 % base rate, 44 independent hits would occur by chance with probability `0.0131⁴⁴ ≈ 10⁻⁸³`. The
code and the data are indexing the same array, in the same units, from the same base — and the wider census
only sharpens what the original 29 already established.

This is the same shape of argument the rest of the chapter uses — recover a hypothesis cheaply, then
confirm it with a test a wrong answer could not pass. Here the weak oracle is "the addressing mode looks
flat" and the strong test is the 44-for-44 coincidence that isn't one.

## 4. The sentinel, and a byte of good luck

`0x80` as a signed byte is **−128**, the most negative value the type can hold, and it occupies 98.69 % of
the array. Read as a volume trim, "attenuate by 128 dB" is silence — which is exactly the right default
for a table where the overwhelming majority of entries are simply not configured.

✅ *Verified:* `0x80` (−128) is the "no value set" sentinel: 44,804 entries.

The trailing byte settles a loose end from [C20.4](04-the-loaders-in-the-executable.md). The file is
45,402 bytes and the game reads 45,401, so byte 45,401 is never read — and that byte is itself **`0x80`**.
The file was evidently written by dumping a buffer pre-filled with the default, one byte longer than the
array it holds. The unread byte is padding of the same fill, so nothing is lost by not reading it.

## 5. What the values are

The 597 configured entries are small signed integers:

| Property | Value |
|---|---|
| Range | **−28 … +20** |
| Negative / zero / positive | 313 / 22 / 262 |
| Most common | `+6` (232), `−2` (145), `−10` (35), `−6` (27), `−5` (24) |

🟡 *Reasoned:* these are **decibel trims** applied per audio event. Three things point the same way: the
range is the range a mixing engineer works in, not a raw amplitude; the type is signed and both signs are
well represented, which suits a relative adjustment and not an absolute level; and the sentinel is the
type's floor, which reads naturally as silence. The file's own name — `AUDIO\CONFIG\EVENTVOL.DAT`, taken
from the string in the executable rather than from any external source — says volume.

⏳ *Open:* the **unit is not proven**. Nothing in the shipped data forces decibels over some other scale,
and this chapter does not promote it.

The configured entries are not spread evenly. Two thirds sit in the first 15,000 IDs, with pronounced
clusters at 0–5,000 (299 entries) and 10,000–15,000 (234), and only a scattering above 20,000. Gaps of 1
dominate — 528 of the 596 gaps — so the values arrive in contiguous runs, which is what one expects of an
ID space allocated in blocks per subsystem.

## 6. Why this matters beyond one file

`EventVol.dat` was the only file in the chapter that resisted the method, and it resisted because the
method was being applied to the wrong question. Every other file in `audio/CONFIG/` announces its record
width through residue-free division, and four of them are confirmed by a divide-by-constant in the loader
([C20.4](04-the-loaders-in-the-executable.md)). This one has no divisor **because a one-byte element needs
no division** — `[base + index]` is the whole addressing calculation, which is precisely why the loader
contains no arithmetic to find.

The absence of a divide instruction in the loader was, in hindsight, the clue. It is recorded here so the
next unresolved table is approached from the use sites sooner.

## 7. The namespace: what the binary can and cannot give

> ⚠️ **Corrected by [C20.6](06-audioevents-namespace-recovered.md).** This section's conclusion that the
> namespace is "not recoverable from the shipped files alone" is **wrong** and is corrected there. It is true
> that the *binary* has no names (that part stands), but a shipped **data file** — `data/AudioEvents.txt` —
> names 7,071 audio events and, in fact, names **447 of the 597** configured `EventVol` entries below. This
> section only checked `gta_sa.exe`, never `data/`. Read [C20.6](06-audioevents-namespace-recovered.md) for
> the corrected result; the paragraphs below are left as written (with this notice) so the mistake and its
> fix are both on the record.

The obvious next step is to name the events — to turn "event 76, trimmed −26" into "event 76 is *this*
sound". Following the 44 constant-index sites through the executable settles what is derivable, and it is a
bounded, slightly disappointing answer worth stating plainly.

Each event ID reaches the code as a **bare integer immediate**. At a site like `0x504BE6` the engine reads
`byte [ptr + 0x91]` (event 145), converts it through a dB-to-gain curve, and hands the result to a
sound-report call — but the identifier `145` is never accompanied by a name. There is **no id → name
table** parallel to the array, and **no strings** at any of the sampled access-site functions (checked at
`0x504BE6`, `0x5068D5`, `0x4DA1D1`, `0x4DC726`, `0x4EC746`, `0x4F99D4` — none references an ASCII string).
The audio-event enumeration was a compile-time C++ `enum` whose names did not survive into the shipped
binary.

So the canonical namespace is **not recoverable from the shipped files alone**. It could only be supplied
from outside — community-maintained enum listings — and this project does not adopt those
([house rules](../handoff.md)). What *is* derivable, and now is, is everything the binary actually contains:
the complete set of 44 hard-coded IDs, each one's trim value, and the code site that consumes it. The
`RE-Data/data/eventvol_events.json` file records the IDs and values; `tools/derive_eventvol.py` re-derives
and re-checks all of it.

This is a **negative result, and it is a result** — the same disposition [C20](C20-Audio.md) took toward
the per-sound trim field and the track header's rate-like word. The chapter can now say precisely *why*
event 76 cannot be named from the executable, rather than leaving it as work not yet attempted.

---

### Key takeaways

- ✅ `EventVol.dat` is **`int8 volume[45401]`, indexed directly by audio event ID** — not a record table.
  The file has no convincing divisor because it has **no records**.
- ✅ All **81** typed accesses through the base pointer at `0xBD00F8` are `movsx byte` with **no width
  scaling** — a signed one-byte element and flat indexing (22 register-indexed, 59 constant-displacement).
- ✅ The proof is **44 / 44**: every event ID hard-coded in the executable lands on a configured entry,
  against a **1.31 %** base rate. Chance probability ≈ 10⁻⁸³. (Extends the original 29 / 29.)
- ✅ `0x80` (**−128**) is the "no value set" sentinel — **44,804 of 45,401** entries. **597** are
  configured, ranging **−28 … +20**.
- ✅ The one byte the game never reads is itself `0x80`; the file is a default-filled buffer dumped one
  byte long.
- 🟡 The values are decibel trims (name, range, sign distribution and sentinel all agree); ⏳ the unit is
  not proven.
- ✅/⏳ The **event-ID namespace is not recoverable from the binary** — the IDs are bare integer immediates
  with no id → name table and no strings at the access sites (§7). The 44 IDs and their values are
  enumerated; canonical names would require external data the project does not adopt.
- 🧭 The general lesson: when a table has no plausible record width, stop factorising and **watch the code
  index it**. A missing divide instruction in the loader is evidence, not an obstacle.

**Continue:** [← Chapter 20 hub](C20-Audio.md)
