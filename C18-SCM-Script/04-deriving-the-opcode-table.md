# C18.4 — Deriving the Opcode Table

> **The one-sentence version:** the constraint route proposed in C18.3 §6 was run and it *partially*
> works — **1,225 opcode arities validated by 124 scripts decoding end-to-end to the exact declared byte**
> — and the four attempts that failed first are more instructive than the one that worked.

[← C18.3 — The code stream](03-the-code-stream.md) · [Chapter 18 hub](C18-SCM-Script.md)

**Confidence:** ✅ Verified (1,225 opcodes, by exact closure) / ⏳ (the remaining 90 scripts)
**Partially closes:** [C18.3 §6](03-the-code-stream.md)

---

## 1. The result

| Measurement | Value |
|---|---:|
| Scripts decoding to their **exact** declared end | **124 / 214 (57.9 %)** |
| Instructions decoded end to end | **187,788** |
| Code bytes covered | **1,601,459 / 3,522,255 (45.5 %)** |
| **Opcode arities validated by an exact closure** | **1,225** |
| Table entries not yet exercised by a closure | 176 |

✅ **Every one of those 1,225 arities is confirmed by at least one script whose entire instruction stream
walks from its first byte to its last with no residue.** That is a stronger check than it sounds: a single
wrong arity anywhere in a script desynchronises the rest of it, and the walk then lands somewhere other
than the declared boundary.

The 124 closures are **independent tests** — separate files and separate mission slices, each with its own
boundary from a table verified in [C18.2](02-mission-and-external-tables.md).

The table is written to [`RE-Data/data/scm_opcode_arity.json`](../RE-Data/data/scm_opcode_arity.json),
split into `validated` and `unvalidated` so the two are never confused.

## 2. What the validated table looks like

| Arity | Opcodes | | Arity | Opcodes |
|---:|---:|---|---:|---:|
| **2** | **494** | | 8 | 29 |
| **1** | **304** | | 9 | 16 |
| 3 | 80 | | 10 | 5 |
| 4 | 78 | | 11–16 | 8 |
| 0 | 68 | | **18** | **2** |
| 6 | 61 | | **variadic** | **2** |
| 5 | 48 | | | |
| 7 | 30 | | | |

✅ **Two arguments is the mode and 65 % of opcodes take one or two.** The tail is long but thin: only 34
opcodes take more than eight arguments, and **only two are variadic** — far fewer than the format's
`0x00`-terminator mechanism would suggest.

The most-executed opcodes across the 187,788 decoded instructions:

| Opcode | Arity | Executions |
|---:|---:|---:|
| `0x004D` | 1 | **26,148** |
| `0x00D6` | 1 | **25,964** |
| `0x0006` | 2 | 13,050 |
| `0x0002` | 1 | 9,010 |
| `0x0039` | 2 | 7,925 |

🟡 *Reasoned:* `0x00D6` and `0x004D` at nearly equal counts are the condition-block opener and the
conditional jump — they pair, which is why the counts track within 0.7 %. Together they are **28 % of all
instructions in the game's script**, and with `0x0006` (assignment) and `0x0002` (`GOTO`) the top four
account for over 40 %. San Andreas's mission logic is overwhelmingly *test, branch, assign*.

## 3. The method that worked

The constraint is the one [C18.3 §6](03-the-code-stream.md) identified: a walk must land **exactly** on the
script's declared end. Turning that into a solver:

**Backward path-counting.** For each script, compute `bwd[p]` = the number of ways to reach the end from
position `p`, allowing every candidate arity at every unknown opcode. Then compute `fwd[p]` forward from
the start the same way.

**Adopt only forced transitions.** A transition at `p` with arity `k` reaching `q` lies on *every* path iff

```
fwd[p] × bwd[q] == total_paths
```

If a transition is on every path, its arity is the right one **regardless of which path is real** — the
truth is one of those paths, and it uses this transition. That criterion is sound, and it showed it:
across all 214 scripts the propagation produced **zero conflicts**, meaning no two scripts ever forced the
same opcode to two different arities.

Then iterate — each newly fixed arity prunes the graph and forces more transitions.

## 4. ⚠️ Four attempts that failed, and why

**Attempt 1 — greedy forward learning.** Walk from a known-good start; on an unknown opcode, pick the
arity whose successor "looks like a valid opcode." Result: 106 opcodes, **0 scripts closing**.

The flaw is that *"looks like a valid opcode" is almost no constraint at all*. Any 16-bit word ≤ `0x0A8C`
qualifies, which is 1 in 24 at random — so nearly every wrong arity passed. It assigned `0x016A` an arity
of 0 on the very first mission, and the walk died 13 bytes into the main script.

**Attempt 2 — depth-first search with backtracking.** Correct in principle, and it reported "no solution"
for 209 of 214 scripts. ⚠️ **It had not found no solution — it had run out of node budget**, and my code
returned an empty result for both cases identically. A timeout that is reported as a negative result is
worse than a crash, because a crash cannot be mistaken for evidence. The smallest script *did* have a
solution; I later decoded it by hand in ten minutes.

**Attempt 3 — backward reachability with intersection.** For each opcode, intersect the viable arities
over every position where it appears live. Result: **890 opcodes, 0 closures**, and the "variadic" set
filled with nonsense values like `0x0703` and `0x0903` — which are not opcodes at all but parameter type
bytes read as instruction starts.

The flaw: **the live set contains positions from wrong paths**, so intersecting over it computes a
statistic over the wrong population. That is the same error as the "multiples of 256" claim corrected in
[C11.2](../C11-IDE-And-IPL/02-ipl-placements.md) and the noise-floor mistake corrected one page ago in
[C18.3 §4](03-the-code-stream.md) — three times now, in three different disguises. **The population a
number is computed over is part of the number.**

**Attempt 4 — sound forcing, wrong search bound.** The path-counting criterion of §3, with candidate
arities capped at 12. Result: sound but stalled, and **86 of 214 scripts had no reachable path at all** —
not "no path found", *no path exists* under the model.

## 5. ⚠️ The unlock: an opcode with 18 arguments

Eighty-six scripts being unreachable *in principle* means the model was wrong, not the search. Tracing the
smallest of them — `OTBTILL`, 815 bytes — the wall sits at offset 259:

```
71 08                          opcode 0x0871
   03 03 00                    local var 3
   04 01   04 00               int8, int8
   01 b0 fe ff ff              int32
   04 01   01 be fe ff ff      int8, int32
   04 ff   01 b0 fe ff ff      int8, int32      ×6 more pairs
                               ────────────────
                               18 arguments, ending exactly on the next instruction
```

**`0x0871` takes eighteen arguments.** With the cap at 12 it could never be parsed, so every script
containing it was unreachable by construction.

Raising the cap dropped unreachable scripts from **86 to 15** in one change.

🟡 *Reasoned:* the trailing `int32, int8` pairs are a variable-length list flattened into a fixed arity —
🟡 plausibly a dialogue or cutscene sequence with `(line, speaker)` pairs, padded with `-1`. That would
explain why the arity is fixed at 18 rather than being variadic: the compiler always emits the maximum
and fills unused slots with the sentinel, the same `-1` convention seen in
[C17.4 §5](../C17-IFP-Animation/04-the-bone-id-table.md) and
[C11.2 §4](../C11-IDE-And-IPL/02-ipl-placements.md).

**A search bound is a hypothesis.** Twelve arguments felt generous, so I never questioned it, and it
silently made a third of the corpus insoluble. The tell was there in the diagnostics from the first
run — *dead=86* — and I read it as a search failure for three attempts before reading it as a model
failure.

## 6. ⚠️ Where the derivation is not sound, and what it cost

Once forcing runs out, further progress needs a *choice* rather than a deduction. I first resolved
ties by taking the smallest surviving arity. That reached **1,650 opcodes and 149 / 214 closures** — and
then **65 scripts failed with parse errors on opcodes like `0x0400`, `0x0300` and `0x0104`**.

Those are not opcodes. They are parameter type bytes — `04 00` is *int8 zero* — read as instruction
starts by a walk that had already desynchronised. **The table had been poisoned, and the symptom was
opcodes that are really arguments.**

That table was discarded and the run restarted from the last sound state, using a guarded rule: adopt an
arity only if it **strictly increases** the number of exactly-closing scripts. The result is smaller —
1,401 entries against 1,650 — and 124 closures against 149.

**The smaller table is the better one.** The 149-closure version cannot be trusted anywhere, because a
wrong arity in a shared opcode corrupts scripts that appear to close by coincidence. Fewer verified
results beat more unverified ones, and the honest comparison is not 149 versus 124 but *0 trustworthy
versus 1,225 trustworthy*.

## 7. What remains, and why it is hard

⏳ **90 scripts, 54.5 % of code bytes, and 176 table entries not yet exercised by a closure.**

The guarded climb has reached a **local optimum**: no single arity assignment increases the closure count,
because the remaining scripts each need *several* unknown opcodes resolved simultaneously before they
close. Single-variable hill-climbing cannot cross that gap.

> **Update — [C18.5](05-the-opcode-ceiling.md).** The stall was partly noise, not only a local optimum.
> Adding one already-proven constraint to the forcer — a real opcode is ≤ `0x0A8C`, so a candidate whose
> successor exceeds the ceiling is not on the true path — lifts the validated table from **1,225 to
> 1,374** with **zero conflicts**, resolving 147 of the entries below and confirming them against the
> earlier guess. The closure count still holds at 124: the forcing extends the *certain prefixes* of the
> open scripts but cannot make their ambiguous tails unique, which is the honest edge of what the data
> proves. See [C18.5](05-the-opcode-ceiling.md).

Three routes remain, in increasing cost:

**Joint search over small opcode groups.** Take a script blocked on three unknowns and search the product
of their candidate sets. Bounded and mechanical.

**Read the dispatch table in the executable.** The interpreter's handlers encode the argument counts
directly, and 1,225 already-validated arities make an excellent correctness check on the extraction —
this is the route where the encyclopedia's own result *validates* the disassembly rather than depending
on it.

**Compare against a community table.** Now genuinely worth doing, because there is something to compare
*with*: 1,225 independently derived arities, each backed by an exact closure. Per
[C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md) a community table is still not adopted — but
agreement across 1,225 values derived by a different method would be strong mutual confirmation, and any
disagreement would point at a specific opcode worth examining.

---

### Key takeaways

- ✅ **1,225 opcode arities are validated** by **124 / 214 scripts decoding to their exact declared end** —
  187,788 instructions, 45.5 % of all script bytes.
- The sound criterion is **path counting**: a transition on *every* reaching path is correct regardless of
  which path is real. It produced **zero conflicts across 214 scripts.**
- **Two arguments is the mode**; 65 % of opcodes take one or two, only 34 take more than eight, and **only
  two are variadic**.
- **`0x00D6` + `0x004D` are 28 % of all executed instructions** — the script is overwhelmingly test,
  branch, assign.
- ⚠️ **A search bound is a hypothesis.** Capping arity at 12 made 86 scripts insoluble *in principle*
  because `0x0871` takes **18** arguments — and the diagnostic said so for three attempts before I read it.
- ⚠️ **A timeout reported as a negative result** cost an entire attempt; "no solution found" and "no
  solution exists" must not share a return value.
- ⚠️ Intersecting over live positions **computed a statistic over the wrong population** — the third
  appearance of that same error in this encyclopedia.
- ⚠️ An unsound tiebreak reached 149 closures and a **poisoned table**; the giveaway was "opcodes" like
  `0x0400` that are really parameter bytes. **1,225 trustworthy entries beat 1,650 untrustworthy ones.**
- ⏳ 90 scripts remain, blocked at a **local optimum** needing joint resolution of several opcodes at once.
- ➡️ **[C18.5](05-the-opcode-ceiling.md) reopens this:** the sound opcode-ceiling prune forces **1,374**
  arities (from 1,225), confirms 147 of them against the earlier guess with 0 conflicts, and shows 124 is
  the data's honest closure ceiling.

**Continue:** [C18.5 — The opcode ceiling](05-the-opcode-ceiling.md)
