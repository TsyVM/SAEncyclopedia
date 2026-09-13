# C33.3 — The Remaining 109 and What Would Close Them

> **The one-sentence version:** the 109 unnamed functions that the classifier cannot attribute fall into
> four structural failure modes — not failures of the method, but ceilings of the data the method has
> available — and each mode has a specific, concrete path to resolution as more chapters land.

**Subsystem category:** Binary substrate — open items
**Depends on:** [C33.2 — The eleven, and what remains](02-the-eleven-and-what-remains.md)
**RE status:** Documented — failure modes characterised; resolution path defined

---

## 1. The four failure modes in detail

### 1.1 Pure-compute functions (~40–50 of the 109)

These functions contain no references to any global address that a structural chapter has pinned. They are
self-contained: arithmetic, bit manipulation, string processing, floating-point math, hash functions. The
classifier has nothing to grip.

What they look like in practice: a body dominated by `fmul`, `fadd`, `fsqrt` and register-to-register
moves without any `mov eax, [absolute_address]` operands is almost certainly a math helper — a clamped
distance, a dot product, a lerp, a fast-inverse-sqrt variant. Functions with `shr`/`and`/`xor` chains over
a single argument are hash or CRC steps. Both patterns are common utility code embedded in many classes at
once, which is why the linker may have placed them anywhere in `.text`. Because they carry no data fingerprint,
proximity is their only current signal — and C27.3 already has them in the 47-candidate proximity list.

What would close them: algorithm recognition. A function whose body implements CRC-32c can be named
`CRC32c` regardless of which class it belongs to, without any external source. C27's policy (no names
without corroboration) slows this path but does not block it — a recognisable algorithm is its own
corroboration. A catalogue of recognisable algorithm signatures (the constants `0xEDB88320` for CRC-32,
`0x5F3759DF` for the fast-inverse-sqrt seed, standard LCG multipliers for PRNGs) would resolve 10–15 of
these without any additional chapter work.

### 1.2 References to unstructured globals (~30–40 of the 109)

These functions reference absolute addresses — the classifier *sees* data references — but none of those
addresses fall within any signature entry. They are touching globals that no chapter has yet characterised.
They are not pure-compute; they are simply beyond the current map.

The signal that distinguishes this group from the pure-compute group: the `derive_attribution.py` output
for these functions shows `hits: 0` on every class, but their bodies do contain `mov eax, [imm32]`
instructions at addresses in the `.data` or `.bss` sections. The hit is present; the map is absent.

What would close them: expanding the chapter coverage. Every new structural chapter that pins a class's
static addresses — a base pointer, a pool object address, a distinctive stride constant — adds to the
signature table. Functions referencing those addresses immediately become attributable on the next tool run,
with no additional analysis of the unnamed functions themselves. This is the highest-leverage path: one
chapter opens attribution for all the unnamed functions in its class's neighbourhood simultaneously.

The CGangWars example is concrete: C30.4 pinned `0xC8B2C0` (320-byte zone-ownership byte array), six
globals at `0xC8A4A4`–`0xC8A4B8`, and `0xC091F0` / stride `0x10` (the CGangs table). Any unnamed function
that byte-indexes `0xC8B2C0` is doing gang-territory work — that address is not shared with any other
class. Before C30.4, these were invisible to the classifier; now they are in the map and a re-run will
find them.

### 1.3 Off-cluster references — the four data-only leads (already in C33.2)

The four functions in C33.2 §3 reference a structured class's data but sit outside that class's address
cluster. The two-signal requirement means they are held at 🟡 "lead" status. These are the border cases —
most likely utility helpers or cross-subsystem callers rather than true class methods.

What would close them: a third signal. The call-graph is the natural choice: if a function at `0x0042F870`
is called *only* from within CPathFind's confirmed method range, then its off-cluster location is explained
(the linker placed it with callers, not with siblings) and the attribution strengthens. Reconstructing a
full call-graph for the unnamed 124 requires disassembling each body for `call imm32` instructions — more
work than the current data-flow scan, but less than reading a function in semantic depth.

For the specific lead at `0x0042F870` (352 bytes, 16 callers, references `0x96F050`): 16 call sites is
unusual for a utility — a function called that many times is likely a core path-query helper. The high
call count, combined with the CPathFind data reference, makes this the strongest of the four leads.

### 1.4 Name collision risk (a small number)

A handful of unnamed functions reference statics from *multiple* structured classes simultaneously — one
body reads from the CStreaming info array and then from a CPathFind node table. These are cross-subsystem
utilities: a function that asks "is this streaming slot loaded" before consuming the path result, for
example. Attributing any one class to such a function would be misleading.

What would close them: accepting that they are *utilities*, not class methods, and naming them descriptively
(`Util_StreamingAwarePathCheck`) rather than by class ownership. C27's in-tree naming policy — external
source or disassembly — applies here, which means these wait for a dedicated read of their individual
bodies. They are a small fraction of the 109 and cannot be resolved by the classifier by design.

---

## 2. How the map has grown, and what that means now

The signature map had 10 entries when `derive_attribution.py` produced the current `attribution.json`. It
now has 12. The additions from C30.4:

| New entry | What it fingerprints |
|---|---|
| `0xC8A4A4…0xC8A4B8` — six war-state globals | `CGangWars` — active-war boolean, attacker/defender IDs, progress, wave count, safe-house flag |
| `0xC8B2C0` — 320-byte zone ownership array | `CGangWars` — gang territory, one byte per zone (0 = no gang, 1–9 = gang ID) |
| `0xC091F0` / stride `0x10` — gang table | `CGangs` — 10-entry static table, model sets and RGBA colour per gang |

The zone-ownership array is particularly useful as a fingerprint: it is a 320-byte block at a unique
address, and indexing it with a loop counter in the range 0–319 is a pattern nothing else in the binary
shares. Any unnamed function that walks `0xC8B2C0` with a `cmp …, 0x140` (= 320) loop bound is territory
code. When the tool is re-run, this will attribute it automatically.

---

## 3. Priority path for reducing the 109

Given the four failure modes and the current state of the map, the highest-leverage actions in order:

**1. Re-run `derive_attribution.py` with the current 12-entry map.**  
Zero additional chapter work needed. CGangWars and CGangs are already in the map; a re-run will attribute
any unnamed functions that reference `0xC8B2C0`, `0xC8A4A4`, or `0xC091F0`. This is the lowest-cost next
step.

**2. Structure one additional heavy class that is already touched by the unnamed functions.**  
The best candidates are classes whose statics appear in the unstructured-globals group (failure mode 1.2):
the audio system, the vehicle-spawning pool, or the world entity pool. Each is referenced somewhere in the
`.text` region but has no chapter yet. A single chapter that pins the base address and record stride opens
attribution for its entire neighbourhood.

**3. Build a call-graph pass for the 124 unnamed functions.**  
This resolves the off-cluster leads by supplying the third signal. It is a moderate time investment —
disassemble each of the 124 for their `call imm32` instructions and record the callee set — but it does
not require semantic reading of any body. The result would promote at least some of the 4 data-only leads
to STRONG.

**4. Algorithm-signature catalogue for pure-compute.**  
A flat list of constant patterns (CRC polynomial, PRNG multiplier, fast-inverse-sqrt seed, sine LUT base
address) applied against each pure-compute body. This is independent of the class structure and can be
done in a separate pass with no chapter dependency.

---

## 4. The honest ceiling

The data-reference classifier is deterministic, fast, and scales automatically — every new structural
chapter adds rows to the map and immediately extends attribution coverage. But two failure modes (pure-
compute and name-collision) are fundamental: no amount of map expansion will attribute a function that
touches no class-owned global, or one that touches multiple classes at once. The realistic ceiling is
something like 30–40 of the current 109, reachable as C34–C52 land and the map fills out.

The 11-out-of-124 figure is a current snapshot. As the remaining chapters are written and the tool re-run,
that number will rise automatically. The pure-compute and multi-reference groups will require individual
function reads regardless — and the project's approach of disassembling on demand means they will be
addressed as interest in those specific functions arises, not in a batch.

---

### Key takeaways

- **~40–50** of the 109 are pure-compute: no data fingerprint to match; algorithm recognition is the path.
- **~30–40** reference unstructured globals: map expansion (new chapters) opens them automatically on re-run.
- **4** off-cluster leads need a call-graph third signal to promote to STRONG.
- **A small number** are cross-class utilities that belong to no single class by design.
- The **CGangWars/CGangs** entries added to the map in C30.4 represent the immediate next re-run opportunity.
- The realistic ceiling for the classifier is ~30–40 more attributions as C34–C52 are structured and the
  map grows — the pure-compute and multi-class groups are the irreducible remainder.

**Previous:** [C33.2 — The eleven, and what remains](02-the-eleven-and-what-remains.md)  
**Up:** [C33 — Attributing the Unnamed](C33-Attributing-The-Unnamed.md)
