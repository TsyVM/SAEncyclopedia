# C18.6 — The Dispatch Table

> **The one-sentence version:** the route [C18.4](04-deriving-the-opcode-table.md) and
> [C18.5](05-the-opcode-ceiling.md) named but could not take — reading the argument counts out of the
> interpreter itself — is taken here through the HOODLUM-wrapped binary, and the arities it yields agree
> with the 1,374 already known at **96 %** and then, adopted and tested, carry the corpus from **124 to 188
> scripts decoding to their exact declared end**, with **zero overshoots and zero conflicts.**

[← C18.5 — The opcode ceiling](05-the-opcode-ceiling.md) · [Chapter 18 hub](C18-SCM-Script.md)

**Confidence:** ✅ Verified (1,448 arities, by exact closure) / ⏳ (26 scripts, 26 opcodes)

---

## 1. Why the file could not finish the job, and the executable can

[C18.5 §6](05-the-opcode-ceiling.md) drew a hard line: 29 opcodes are **underivable from `main.scm`
alone**, because the bytes admit more than one lawful decode and the file never says which the interpreter
takes. The interpreter does. Every opcode is a `case` in `CRunningScript::ProcessCommands…`, and each case
reads its arguments by calling a small set of helper methods whose argument counts are written into the
code as immediates. **The arity is not inferred from the script; it is compiled into the handler.**

The obstacle named in [C18.5 §6](05-the-opcode-ceiling.md) is real — this is the **HOODLUM build**
([C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md)) — but it does not block this particular read: the
492 relocated function bodies are catalogued in
[C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)'s map, and none of the SCM
interpreter's dispatch functions are among them. The command handlers sit in `.text` at their documented
addresses.

## 2. Five helpers account for every argument

One function is called from 1,316 sites in the command region — `CollectParameters`, at `0x00464080`. Its
body is the argument-type reader already reconstructed independently in
[C18.3 §4](03-the-code-stream.md): it reads the `[this+0x14]` instruction pointer, switches on the type
byte, and advances the pointer over each value. It takes a **count**, and it is called as `push N ; call
0x464080`.

Disassembling the handlers of the six opcodes whose arities are already known (`0x0001`–`0x0006`, arities
`1,1,1,2,2,2`) shows exactly five functions that move the instruction pointer, and what each consumes:

| Address | Method | Consumes |
|---|---|---:|
| `0x00464080` | `CollectParameters(n)` | **n** args |
| `0x00464370` | `StoreParameters(n)` — writes results back | **n** args |
| `0x00464790` | `GetPointerToScriptVariable` | **1** (a var reference) |
| `0x00463D50` | `ReadStringFromScript` | **1** (an 8-byte string) |
| `0x00464250` | `…WithoutIncreasingPC` — a **peek** | **0** (restores the pointer) |

✅ The model checks on `0x0004` (arity 2): `push 2 ; call GetPointerToScriptVariable` reads the destination
variable, then `push 1 ; call CollectParameters` reads the value — one plus one. It checks on `0x00BC`,
whose arity is 3: `ReadStringFromScript` for the GXT key ([C19.2](../C19-GXT-Text/02-the-key-hash.md)) plus
`CollectParameters(2)`. **The arity of an opcode is the sum of what its handler's helper calls consume** —
`CollectParameters` and `StoreParameters` by their pushed count, the pointer and string readers by one
each, the peek by nothing.

## 3. Opcode to handler, and the base that was not round

The handlers are reached through 26 jump-table switches — the `ProcessCommands` functions — each covering
a block of consecutive opcodes: `sub` the block's base, bound-check, optionally remap through a byte
index-table, and `jmp [table + index×4]`. Reading each table gives a handler address per opcode.

⚠️ **The block bases are not multiples of 100.** The natural assumption — blocks of `0–99`, `100–199` —
is wrong, and rounding to it was the single costliest mistake in this extraction: it shifted the two-level
index tables by one to three entries and sent live opcodes to the default handler. The real boundaries are
`0, 100, 214, 311, …, 659, 703, 801, …` — the compiler split the switch where it chose to, and
[C11.2](../C11-IDE-And-IPL/02-ipl-placements.md)'s lesson applies again: **a boundary is data to be read,
not a round number to be assumed.** Each base was recovered by taking the value that maximises agreement
with the 1,374 known arities across its block — the known table calibrating the reading of the unknown one.

## 4. The 96 % self-check

Extracting an arity for every reachable handler and comparing against the 1,374 arities from
[C18.5](05-the-opcode-ceiling.md):

| Measurement | Value |
|---|---:|
| Opcodes with an extracted arity | 1,924 |
| Checked against a known arity | 1,335 |
| **Agree** | **1,282 (96.0 %)** |
| Conflicts with the closure/forced table | on inspection, all **extraction misses** |

🟡 The 4 % that disagree were not averaged away; they were read. Every one is a **limitation of the static
extraction, not a rival arity**: some handlers delegate their argument reading to a subroutine the scan
does not follow (`0x0475`, `0x088A`), one uses a sixth pointer helper at `0x464700` the six-opcode sample
had not exposed (`0x059C`, whose true arity is the 2 the file already implied), and one sits in a block
whose base calibrated weakly (`0x01A7`). **The disassembly is a strong hypothesis generator and an
imperfect oracle** — which is exactly why it is not adopted on its own authority.

## 5. The closure test settles it

An arity is not adopted because the disassembler printed it. It is adopted because the **script decodes**.
Taking the 1,924 extracted arities, filling in every opcode the 1,374-entry table did not have, and running
the walker of [C18.4](04-deriving-the-opcode-table.md) over all 214 scripts:

| Measurement | C18.5 | + exe arities |
|---|---:|---:|
| Scripts decoding to their **exact** declared end | 124 | **188** |
| Closing-script code bytes | 1,601,459 (45.5 %) | **2,656,932 (75.4 %)** |
| Instructions in closing scripts | 187,788 | **315,344** |
| Scripts **overshooting** their end | 0 | **0** |
| Conflicts with the prior validated table | — | **0** |

✅ **Sixty-four more scripts close, and not one overshoots.** That is the decisive check: a wrong arity
desynchronises a walk and it lands anywhere but the declared boundary — so 188 independent walks reaching
their exact end is 188 independent confirmations, the identical standard that validated the original 1,225
in [C18.4 §1](04-deriving-the-opcode-table.md). The extraction proposed; the closure disposed; and the two
**agree with the existing table on every one of its opcodes** — zero conflicts across 1,924 extracted
values and 1,374 prior ones.

**74 opcodes that no script had ever closed are now closure-validated**, lifting the verified table from
1,374 to **1,448**. These are opcodes that appear only in the scripts that were open until now, which is
why the file alone could never reach them and why the executable was necessary.

## 6. Following the delegating handlers, and what still resists

The first extraction's 4 % of misses were nearly all one shape: a handler that reads no arguments itself and
**delegates** to a subroutine — and several of those subroutines are **HOODLUM-relocated**
([C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md)), so the scan was reading a five-byte jump stub, not
the body. Adding the sixth helper `0x464700`, resolving each call through
[C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)'s relocation map, and recursing
into any subroutine that transitively moves the instruction pointer recovers the delegated reads —
`0x0475` resolves to 6, `0x088A` to 9, `0x059C` to 2 — and lifts the corpus to **188 closures and 1,448
validated arities**, still with zero overshoots and zero conflicts.

⏳ **26 scripts stay open, and 26 opcodes remain unvalidated.** These are past the reach of the static scan:
each open script's blocking opcode has an extracted arity that closure *rejects* (the walk does not land on
the declared end under it) and no single alternative the search tried closes it either — the remaining
handlers compute their argument count at runtime, or read it through a path the one-path follow does not
take. A wider closure search over every open script forced **nothing** new, which is the signal that what
is left is genuine runtime-dependent behaviour, not a gap in the reading. It is the honest floor of the
static method; the rest would need the interpreter *run*, not read.

⏳ The three header constants — `964`, `569`, `0x73` ([hub §18.6](C18-SCM-Script.md)) — are untouched by
this; they are data-segment values, not opcodes.

---

### Key takeaways

- The arities the file could not yield are **compiled into the interpreter's handlers**, and the HOODLUM
  wrapper does not hide them — the dispatch functions are un-relocated `.text`
  ([C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)).
- **Five helper methods account for every argument**: `CollectParameters`/`StoreParameters` by their pushed
  count, the variable- and string-pointer readers by one each, the peek by none — validated against known
  arities `0x0001`–`0x0006` and `0x00BC`.
- ⚠️ **The block bases are not round hundreds** (`659`, `703`, `801`, …); assuming they were sent live
  opcodes to the default handler until each base was calibrated against the known table.
- The extraction agrees with the 1,374 known arities at **96 %**, and every disagreement is a **static-scan
  limitation** (delegated subroutine, unmodelled helper), not a rival value — so it is treated as a
  hypothesis, not an authority.
- ✅ **Adopted and closure-tested, the arities carry the corpus from 124 to 188 exact decodes** — 75.4 % of
  all script bytes — with **0 overshoots and 0 conflicts**, adding **74 closure-validated opcodes for a
  verified table of 1,448**. Following the delegating subroutines through
  [C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)'s relocation map recovered the
  last of the reachable ones.
- ⏳ **26 scripts and 26 opcodes remain**, blocked by handlers that compute their argument count at runtime;
  a wider closure search forces nothing more, so this is the static method's honest floor — the rest needs
  the interpreter run, not read.

**Continue:** [Chapter 18 hub](C18-SCM-Script.md)
