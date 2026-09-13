# C18.5 — The Opcode Ceiling

> **The one-sentence version:** the derivation of [C18.4](04-deriving-the-opcode-table.md) stalled at
> 1,225 arities because its sound forcer was drowning in wrong walks — and adding **one already-proven
> fact, that a real opcode is ≤ `0x0A8C`**, as a pruning constraint lets the *same* forcing criterion
> reach **1,374 arities with zero conflicts**, independently confirming 147 previously-unvalidated values
> and resolving all but 29 — while the closure count holds at 124, which turns out to be the honest
> ceiling of what the data can prove.

[← C18.4 — Deriving the opcode table](04-deriving-the-opcode-table.md) · [Chapter 18 hub](C18-SCM-Script.md)

**Confidence:** ✅ Verified (1,374 arities, by exact forcing + cross-derivation) / ⏳ (29 opcodes, 90 scripts)
**Closes further:** [C18.4 §7](04-deriving-the-opcode-table.md)

---

## 1. Where C18.4 stopped, and why it was not the end

[C18.4 §3](04-deriving-the-opcode-table.md) established the sound criterion: a transition that lies on
*every* path reaching a script's declared end is correct **regardless of which path is real**, because the
true walk is one of those paths and it uses that transition. Run to a fixpoint, that forcer validated
1,225 arities and 124 exact closures, then stalled.

The stall was read in [C18.4 §7](04-deriving-the-opcode-table.md) as a *local optimum* — the remaining
scripts each needing several opcodes resolved at once. That reading was half right. The deeper problem was
the **population the forcer was counting over**: it enumerated every type-consistent walk, and the vast
majority of those walks are false. A false walk reads an argument byte as an opcode — `04 00` is *int8
zero*, not opcode `0x0004` — and because SCM "fails plausibly rather than loudly"
([C18.3 §1](03-the-code-stream.md)), these false walks keep decoding and frequently reach the declared end
by coincidence. Every false closing walk is another path the forcer must intersect over, and two paths
that disagree force nothing. **The forcer was not at a local optimum so much as buried in noise.**

## 2. The constraint that was already proven

The fact needed to cut the noise was sitting in [C18.4 §4](04-deriving-the-opcode-table.md), used there
only to explain *why* attempt 1 failed:

> Any 16-bit word ≤ `0x0A8C` qualifies as a plausible opcode, which is 1 in 24 at random.

Turned from a lament into a constraint: **a real instruction never begins with a word above `0x0A8C`.**
The highest opcode anywhere in the validated table is `0x0A45`; the documented ceiling is `0x0A8C`. So any
candidate arity whose successor position lands on a word above the ceiling cannot be on a real walk —

✅ **and pruning it is sound, because the true path never lands there.** Removing walks that step onto an
impossible opcode cannot remove the one walk that is real. The intersection over the *surviving* walks
therefore still contains the truth, and an opcode constant across the survivors is still forced — the
[§3](04-deriving-the-opcode-table.md) argument is untouched.

What changes is only the noise floor. A false walk that reads a parameter byte as an opcode tends, a step
or two later, to land on an impossible word and die. The ceiling makes it die *immediately* instead of
wandering far enough to reach the declared end by accident.

## 3. The effect

The prune was added to the forcer and nothing else changed — same walker, same masking of the `0x8000`
negation bit ([C18.4 baseline](04-deriving-the-opcode-table.md)), same intersection-over-all-closing-paths
adoption rule.

| Measurement | C18.4 | + ceiling |
|---|---:|---:|
| Opcode arities validated by forcing | 1,225 | **1,374** |
| Scripts whose search completes (not budget-bound) | 158 / 214 | **213 / 214** |
| Conflicts (an opcode forced two ways) | 0 | **0** |
| Exact closures | 124 | **124** |

✅ **+149 arities, still zero conflicts across all 214 scripts.** The single most expensive symptom of the
old run — 56 of 90 blocked scripts exhausting the node budget before their search completed — collapsed to
one, because most of the branching the budget was being spent on was false walks the ceiling now kills at
birth.

Two of the 149 are opcodes the earlier run had never placed at all (`0x06D2` → 2 args, `0x0912` → 3). The
other 147 were sitting in the table's `unvalidated` half — proposed by the earlier unsound tiebreak but
never confirmed by a closure.

## 4. The cross-checks

A method that suddenly validates 149 new values is exactly the kind of result [C18.4 §6](04-deriving-the-opcode-table.md)
warns is most likely poisoned. Four checks, each of which the earlier failures taught us to run:

**The cap is not doing the work.** The search bounds candidate arity at some maximum, and
[C18.4 §5](04-deriving-the-opcode-table.md) is a whole section on a wrong bound (`12`, when `0x0871` needs
`18`) silently making a third of the corpus insoluble. So the entire derivation was run twice, at cap
**24** and cap **48**, and the two tables were compared:

| Check | Result |
|---|---:|
| New arities at cap 24 | 149 |
| New arities at cap 48 | 149 |
| Disagreements between them | **0 / 149** |

✅ The forced set is **independent of the cap**. A value that is forced is forced by the structure of the
walks, not by where the search was told to stop.

**The values agree with the independent guess.** The 147 that came from the `unvalidated` half were
proposed by a *different* method — the earlier guarded hill-climb. Sound forcing and that earlier heuristic
now agree:

| Check | Result |
|---|---:|
| Resolved-from-`unvalidated` | 147 |
| Sound-forced value == earlier proposed value | **147 / 147** ✅ |

🟡 This is the mutual confirmation [C18.4 §7](04-deriving-the-opcode-table.md) said was worth having: two
methods that fail in different ways agreeing on 147 values is far stronger than either alone. It is the
same evidence class as a derived table agreeing with a community table — except here the second witness is
the encyclopedia's own earlier, honestly-quarantined guess.

**Nothing that already closed was disturbed.** A newly forced arity is for an opcode that, by definition,
never appeared in a fully-closing script — otherwise it would already be validated. The check confirms it:

| Check | Result |
|---|---:|
| Closing scripts (base) ⊆ closing scripts (extended) | **124 ⊆ 124** ✅ |
| Base-closing scripts that use any newly added opcode | **0** ✅ |

**The additions are not the fakes.** The opcodes that *block* the remaining scripts are `0x0802`,
`0x0304`, `0x0404`, `0x0300` — parameter-byte pairs read as instructions, precisely the poison of
[C18.4 §6](04-deriving-the-opcode-table.md). Not one of them is in the 149. ✅ The set that was adopted and
the set that signals corruption are disjoint.

## 5. ⏳ Why the closure count did not move

149 more arities and **not one additional script closes to its exact end.** That is not a disappointment;
it is the finding.

A script closes only when its walk is *unique* — one path from first byte to last. The ceiling prune
removed a mountain of false paths, but for 90 scripts it did not remove them all: each still admits **two
or more** type-consistent, ceiling-consistent walks that both reach the declared end. Where two such walks
disagree on an opcode, that opcode stays unforced and the script stays open. The forcing captured
everything the walks *agree* on — which is why coverage rose sharply even though closures did not:

| Measurement | C18.4 | + ceiling |
|---|---:|---:|
| Bytes on a fully-forced prefix (all 214 scripts) | 1,838,785 (52.2 %) | **2,038,364 (57.9 %)** |
| Instructions on those prefixes | 216,073 | **239,613** |

✅ An additional **199,579 bytes and 23,540 instructions** now decode with certainty — they simply live in
scripts whose *tails* remain ambiguous.

The residual ambiguity is real and it is structural. Of the 90 open scripts, 24 admit no valid walk at all
under the sound model — their true decode needs a fact the model does not have — and the other 66 admit
several. The competing walks route through parameter-bytes-as-opcodes, and **no amount of walk-counting can
rule those out, because they are genuine alternative parses of the same bytes.** Distinguishing them needs
a witness outside `main.scm`. That is the boundary [C18.4 §6](04-deriving-the-opcode-table.md) drew when it
chose 1,225 trustworthy entries over 1,650 poisoned ones, and the ceiling has moved the boundary to 1,374
without crossing it.

## 6. What is left, and the one route that crosses the line

⏳ **29 opcodes remain unvalidated** — down from 176 — and **90 scripts remain open.** The three header
constants `964`, `569` and `0x73` are still unnamed ([hub §18.6](C18-SCM-Script.md)).

Of the three routes in [C18.4 §7](04-deriving-the-opcode-table.md), the data-only one is now exhausted:
sound forcing, given every constraint the file itself supplies, has produced everything it can. The
remaining 29 arities are not underived for want of effort — they are **underivable from `main.scm` alone**,
because the bytes admit more than one lawful reading and the file never says which the interpreter takes.

The interpreter does say, though. **The argument counts are encoded in the opcode handlers inside
`gta_sa.exe`** — the route [C18.4 §7](04-deriving-the-opcode-table.md) called the one that both closes the
table and validates itself, since 1,374 independently forced arities are now an enormous correctness check
on any extraction. ⚠️ The obstacle is that this binary is the **HOODLUM build** dissected in
[C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md): its `.text` is wrapped, so reading the dispatch
handlers is a resolver problem before it is a disassembly problem, and it belongs to C0's address space,
not this chapter's. It is named here as the derived-and-checkable next step, not adopted.

Per [C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md), a community opcode table is still **not
adopted** — but the case for *comparing against* one is now overwhelming: 1,374 arities, each on every
surviving path of at least one script, are 1,374 chances for an external table to agree or to point at a
specific opcode worth a second look.

---

### Key takeaways

- ✅ **1,374 opcode arities are now validated** — up from 1,225 — by the same exact-forcing criterion, with
  **zero conflicts across all 214 scripts.**
- The unlock was **a constraint already proven and not yet used**: a real opcode is ≤ `0x0A8C`, so a
  candidate whose successor exceeds the ceiling cannot be on the true path and pruning it is sound. It
  works by **lowering the noise floor**, killing false walks before they reach the end by accident.
- ✅ **The result is cap-independent** — identical at arity caps 24 and 48 — so it is a property of the
  walks, not the search bound that [C18.4 §5](04-deriving-the-opcode-table.md) was burned by.
- 🟡 **147 of the 149 match the earlier `unvalidated` guess exactly**, 0 disagreements — sound forcing and
  the earlier heuristic confirming each other.
- ✅ **No closing script was disturbed** and **none of the parameter-byte fakes** (`0x0802`, `0x0404`, …)
  entered the table — the adopted set and the corruption signal stay disjoint.
- ⏳ **Closures held at 124** while forced-byte coverage rose 52.2 % → 57.9 %: the new arities extend the
  *certain prefixes* of open scripts without making their ambiguous tails unique. That is the honest edge
  of what `main.scm` can prove.
- ⏳ **29 opcodes remain**, underivable from the file alone; the handlers in the **HOODLUM-wrapped
  `gta_sa.exe`** ([C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md)) are the derived, self-checking
  route across the line.

**Continue:** [C18.6 — The dispatch table](06-the-dispatch-table.md)
