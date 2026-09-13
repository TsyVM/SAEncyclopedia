# C18.3 — The Code Stream

> **The one-sentence version:** the instruction stream cannot be walked without a table that is not in the
> data — but 215 known-good instruction positions, one self-identifying opcode and a statistical pairing
> between two opcodes and their argument types recover five parameter widths and every script name in the
> game.

[← C18.2 — The mission and external tables](02-mission-and-external-tables.md) ·
[Chapter 18 hub](C18-SCM-Script.md)

**Confidence:** ✅ Verified (what is measured) / ⏳ (the opcode table)

---

## 1. Why the stream cannot simply be walked

Every other format in this encyclopedia could be walked because each record declared its own length. SCM
does not:

```
04 00   02 90 9d   04 00
^^^^^   ^^^^^^^^   ^^^^^
opcode  global var  int8
        (3 bytes)   (2 bytes)
```

The instruction is seven bytes, but **nothing in those seven bytes says so**. The length is the sum of the
argument widths, the argument *types* are self-describing, but the argument **count** is a property of the
opcode — and that count lives in a jump table inside `gta_sa.exe`, not in `main.scm`.

⚠️ **So a wrong opcode length does not fail loudly here.** Unlike the IFP stride error in
[C17.2 §3](../C17-IFP-Animation/02-frames.md), which desynchronised on 132/132 files immediately, a
mis-walked SCM stream keeps producing byte sequences that decode as *valid-looking* instructions. This
format has the failure mode of a text format wearing binary clothing, and that is the single most
important thing to know before writing a tool for it.

**This page therefore measures only from positions that are known to be correct**, and says so each time.

## 2. 215 positions that are known-good

Three sources give instruction offsets that cannot be wrong, because they come from tables verified in
[C18.2](02-mission-and-external-tables.md):

| Source | Positions |
|---|---:|
| Start of the main script | 1 |
| Mission offsets (segment 3) | 135 |
| Start of each external script (`script.img`) | 79 |
| **Total** | **215** |

Reading the opcode at each:

| Opcode | Main | Missions | Externals |
|---:|---:|---:|---:|
| `0x03A4` | 1 | **112** | **71** |
| `0x0050` | — | **23** | — |
| `0x0004` | — | — | 4 |
| `0x0005` | — | — | 3 |
| *(artefact)* | — | — | 1 |

✅ Verified. **Four distinct opcodes account for all 215 entry points**, and the artefact is `aaa.scm`
([C18.1 §5](01-the-segment-chain.md)), which is not a script.

## 3. `0x03A4` names itself

The main script begins:

```
a4 03  09  4d 41 49 4e 00 00 00 00      →   03A4  "MAIN"
^^^^^  ^^  ^^^^^^^^^^^^^^^^^^^^^^^
opcode  |  eight bytes, NUL-padded
        parameter type 0x09
```

Mission 0 begins `03A4 09 "INITIAL"`. Mission 1, `"INITIL2"`. Mission 2, `"INTRO"`.

**The opcode declares the name of the script it opens** — `SCRIPT_NAME` in every reasonable reading — and
the parameter type `0x09` introduces a **fixed 8-byte string**. Both facts are proved together by the
external scripts: 71 of them carry a name here, and **38 of those reproduce their own filename exactly**.

A field that reproduces the filename of the file it is stored in cannot be at the wrong offset, and a
string width that yields clean names 160 times cannot be the wrong width. Same evidence class as
[C12.1 §3](../C12-Path-Network/01-nodes-dat.md) and [C17.1 §3](../C17-IFP-Animation/01-anp3-container.md).

| Measurement | Value |
|---|---:|
| `03A4 09` sites with a clean name | **160** |
| Distinct names among them | **160** |
| Name lengths observed | **4 – 7 characters** |
| Names occupying all 8 bytes | **0** |

✅ **No name fills the field**, so the string is always NUL-terminated inside it — a reader never has to
handle the unterminated case. That is worth knowing precisely because the neighbouring
[IFP](../C17-IFP-Animation/01-anp3-container.md) format *does* require it.

### ⚠️ The names are not the filenames

Of the 71 named external scripts, 38 match their filename and **33 do not** — because the field is eight
bytes and the filenames are not:

| File | Internal name |
|---|---|
| `slot_machine.scm` | **`BANDIT`** |
| `wheelo.scm` | **`WOF`** |
| `zero_ambience.scm` | **`RCSHOP`** |
| `shopkeeper.scm` | **`SKBRAIN`** |
| `home_brains.scm` | **`HMLES`** |
| `customer_panic.scm` | **`FFPNC`** |
| `bcesar3.scm` | **`COKEC`** |
| `basketb.scm` | **`BBALL`** |

These are **hand-chosen abbreviations, not truncations** — and several are more informative than the
filename. `slot_machine` → `BANDIT` is the one-armed bandit, and segment 2's model list carries
`KB_BANDIT_U` for the same object ([C18.1 §4](01-the-segment-chain.md)); `zero_ambience` → `RCSHOP`
identifies whose shop it is; `wheelo` → `WOF` is the wheel of fortune.

🟡 *Reasoned:* the internal name is what the script author called the thing and the filename is what the
build system called it, so where they disagree the internal name is closer to the design. **The
abbreviation is not lossy — it is a second, independent label**, and reading both gives more than reading
either.

## 4. Two opcodes that prove five parameter types

The eight external scripts that do not begin with `03A4` begin with assignments instead:

```
05 00 | 02 c4 95 | 06 00 00 00 00        ammu.scm
04 00 | 02 90 9d | 04 00                 carmod1.scm
```

Opcodes `0x0004` and `0x0005` differ by one and are obviously a pair. Censusing what parameter type
follows a global-variable destination for each — over the whole code region, restricted to references
that pass the divisibility and range test from [C18.1 §3](01-the-segment-chain.md):

| Following type | after `0x0004` | after `0x0005` |
|---:|---:|---:|
| `0x04` (int8) | **4,048** | 0 |
| `0x01` (int32) | 194 | 0 |
| `0x05` (int16) | 191 | **1** |
| **`0x06` (float32)** | 20 | **2,880** |
| everything else | 3,679 | 17 |
| **total matches** | **8,132** | **2,898** |

The `0x0005` column is the load-bearing one: **2,880 of 2,898 matches are float32, against exactly one
integer-typed counterexample.** In the other direction, float appears after `0x0004` twenty times against
4,433 integer-typed matches.

✅ `0x0004` is `SET_VAR_INT` and `0x0005` is `SET_VAR_FLOAT` — derived by *correlating two fields against
each other*, the same method that produced the bone-ID bands in
[C17.4](../C17-IFP-Animation/04-the-bone-id-table.md), rather than assumed from a table.

### ⚠️ The correction that produced this table

The first version of this page said **"the two sets are disjoint"** and printed zeros in both off-diagonal
cells. The verification pass failed it: the off-diagonals are 20 and 1, not 0 and 0.

Worse, the "everything else" row was written as `~180` and `~13`. The real values are **3,679 and 17** —
so the first draft understated its own noise floor by a factor of twenty in the `0x0004` column, where
noise is *45 % of all matches*.

The asymmetry has a cause, and it is the interesting part. This is a **pattern scan, not a walk**, so
matches can land mid-instruction — and `04 00` is not an arbitrary byte pair, it is itself a valid
parameter encoding: *int8, value zero*. Any instruction carrying a zero byte argument followed by a global
reference produces a false match. `05 00` (int16 zero) is far rarer, which is exactly why the `0x0005`
census is clean and the `0x0004` census is not.

**A scan's noise floor is a property of what the pattern means, not of how long the pattern is.** Reporting
`~180` was a guess dressed as a measurement, and it is the same failure as the "multiples of 256" claim
corrected in [C11.2](../C11-IDE-And-IPL/02-ipl-placements.md) — a top-N table read as a distribution.

The five parameter types established so far, each from evidence rather than a table:

| Type | Width | Evidence |
|---:|---:|---|
| `0x01` | 4 (int32) | the six segment `GOTO`s ([C18.1 §1](01-the-segment-chain.md)) |
| `0x02` | 2 (global ref) | divisible by 4, bounded by 43,800 ([C18.1 §3](01-the-segment-chain.md)) |
| `0x04` | 1 (int8) | 4,048 pairings with `SET_VAR_INT` |
| `0x06` | 4 (float32) | 2,880 of 2,898 `SET_VAR_FLOAT` matches, against 1 integer |
| `0x09` | 8 (string) | 160 clean names, 38 matching filenames |

⏳ `0x05` (int16) is strongly indicated by its 191 pairings with `0x0004` but is not independently
confirmed. The remaining types are underived.

## 5. The sign of a jump target is its addressing mode

23 missions open with opcode `0x0050` and an int32 parameter — and **every one of those 23 values is
negative**:

```
mission   3 @285,597    50 00 01 f0 ff ff ff       target −16
mission   5 @313,807    50 00 01 bd ff ff ff       target −67
```

| Check | Result |
|---|---:|
| Targets negative | **23 / 23** ✅ |
| `|target|` < that mission's size | **23 / 23** ✅ |
| Resolved position holds a valid opcode | **23 / 23** ✅ |

Resolving `missionStart + |target|` lands on `0x0004` nineteen times, `0x0006` three times and `0x0A0E`
once — assignments, exactly what a subroutine prologue would contain.

✅ **A negative target is relative to the base of the current script; a positive target is an absolute
offset in the flat address space** of [C18.2 §3](02-mission-and-external-tables.md).

🟡 *Reasoned:* this is what makes missions relocatable. A mission is loaded into a shared buffer at an
address that is not its build-time address, so every branch *inside* it must be base-relative — while
branches into the always-resident main script can stay absolute. **The sign bit carries the distinction at
zero cost**, which is why the offsets are stored signed in a file where no offset is ever genuinely
negative.

⚠️ A tool that reads these as unsigned gets 4,294,967,280 instead of −16 and will not notice, because the
value is never dereferenced during a size check.

## 6. What would close the opcode table

Three routes, in increasing cost:

**Bootstrap from the disjoint-type method of §4.** Any opcode whose argument types are consistent across
thousands of occurrences reveals its arity the same way `0x0004` and `0x0005` did. Run over the whole
stream this becomes a constraint-satisfaction problem — one length per opcode, such that a walk from
each of the 215 verified entry points terminates on that script's declared boundary. **The 135 mission
sizes are the constraint**, and they are exact ([C18.2 §2](02-mission-and-external-tables.md)), so a
candidate table is testable 135 times independently.

**Read the dispatch table in the executable.** The interpreter switches on the opcode, so the argument
counts are encoded in the handler bodies. C0.2's address resolution
([C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)) is the prerequisite.

**Adopt a community table.** Complete tables exist. Per
[C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md) they are **not adopted here** — the
`0x253F2FE` correction happened precisely because a name was accepted without asking what the data said,
and an opcode table is 1,200 opportunities to repeat that.

The first route is the right one *and* it is the one that validates the third: a derived table and a
community table that agree over 135 missions would settle the question far more firmly than either alone.

---

### Key takeaways

- ⚠️ **SCM cannot be walked without an argument-count table**, and a wrong walk **fails plausibly rather
  than loudly** — the opposite of IFP ([C17.2 §3](../C17-IFP-Animation/02-frames.md)).
- **215 instruction positions are known-good** from the verified tables, and four opcodes account for all
  of them.
- **`0x03A4` names its own script**; 160 clean names, all 4–7 characters, **none filling the 8-byte
  field**.
- ⚠️ **33 internal names are hand-chosen abbreviations, not truncations** — `slot_machine` → `BANDIT`,
  `zero_ambience` → `RCSHOP`. The internal name is often closer to the design than the filename.
- **`0x0004` and `0x0005` take near-disjoint argument types** — `0x0005` is float on **2,880 of 2,898**
  matches against **one** integer — which *derives* `SET_VAR_INT` / `SET_VAR_FLOAT` and four parameter
  widths rather than assuming them.
- ⚠️ The first draft of that table claimed the sets were **exactly** disjoint and understated its noise
  floor twentyfold. **A scan's noise floor depends on what the pattern means**: `04 00` is itself a valid
  parameter encoding, so it matches mid-instruction constantly.
- ✅ **The sign of a jump target is its addressing mode** — negative is script-relative, and all 23
  observed cases resolve to a valid opcode inside their own mission.
- ⏳ The opcode table remains open; the **135 exact mission sizes make any candidate table testable 135
  times over**, which is the route to closing it.

**Continue:** [Chapter 18 hub](C18-SCM-Script.md)
