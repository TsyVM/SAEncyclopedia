# C33.1 — The Classifier

> **The one-sentence version:** every structural chapter left behind a set of class-owned static addresses
> and record strides; collected into one signature map and matched against each unnamed function's
> disassembly, they vote for a class on data-flow evidence, and a vote is promoted to STRONG only when the
> function's own location independently agrees.

**Subsystem category:** Binary substrate — method
**Depends on:** [C33 hub](C33-Attributing-The-Unnamed.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md)–[C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md)
**RE status:** Documented
**Confidence:** ✅ for the method; the per-function verdicts are in [C33.2](02-the-eleven-and-what-remains.md)

---

## 1. The signature map

Each structural chapter pinned data that belongs to exactly one class. Collected, they form the signature
table the classifier matches against:

| Signature (static range or stride) | Class | Source |
|---|---|---|
| `0x8E4CC0…0x9654B6` (streaming-info array) | `CStreaming` | [C28.3](../C28-Class-Catalogue/03-cstreaming-and-rederiving-c2.md) |
| `0x96F050…0x96FB00` (path node/link tables) | `CPathFind` | [C28.4](../C28-Class-Catalogue/04-cpathfind-and-the-28-byte-node.md) |
| `0xA48960…0xA49970` (LVAR + global var space) | `CTheScripts` | [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)/[C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md) |
| `0x965560` / stride `0x2C` | `CColStore` | [C31.1](../C31-Streaming-Slot-Tables/01-ccolstore.md) |
| `0x8E3FB0` / stride `0x34` | `CIplStore` | [C31.2](../C31-Streaming-Slot-Tables/02-ciplstore.md) |
| `0x9788C4…0x97D644`, `0x978624` | `CPickups` | [C29.1](../C29-Gameplay-Object-Pools/01-cpickups-the-pickup-pool.md) |
| `0x96C048…0x96EAC8` / stride `0xD8` | `CGarages` | [C29.2](../C29-Gameplay-Object-Pools/02-cgarages-the-garage-pool.md) |
| `0x96A71C…0x96A7E0` / stride `0x3C` | `CEntryExitManager` | [C30.1](../C30-Gameplay-Managers/01-centryexitmanager.md) |
| `0x97F838…0xA43088` (replay buffer) | `CReplay` | [C30.2](../C30-Gameplay-Managers/02-creplay.md) |
| `0xA97290…0xA97E00` (shop ledger) | `CShopping` | [C30.3](../C30-Gameplay-Managers/03-cshopping.md) |
| `0xC8A4A4…0xC8A4B8` (active-war globals) / `0xC8B2C0` (zone ownership array) | `CGangWars` | [C30.4](../C30-Gameplay-Managers/04-cgangwars.md) |
| `0xC091F0` / stride `0x10` (10-entry gang table) | `CGangs` | [C30.4](../C30-Gameplay-Managers/04-cgangwars.md) |

Each entry is a *fingerprint*: these addresses and strides were each shown, in their source chapter, to be
touched by one class's methods. Nothing else in the binary indexes `0x8E4CC0` with a `×20` stride but the
streaming code. The table grows automatically as each new structural chapter lands: C30.4's CGangWars and
CGangs entries were added when that chapter was written, bringing the map to its current twelve entries. A
re-run of `derive_attribution.py` with any newly-added rows is all that is needed to extend attribution
coverage to the functions that reference those addresses.

## 2. The two signals

For each of C27's 124 unnamed functions, the classifier:

1. **Disassembles** the body (resolver per [C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md),
   stopping at the first `ret`).
2. **Extracts** every absolute address in an operand and every `imul` stride constant.
3. **Votes**: each extracted value that lands on a signature entry is a vote for that class; the majority
   class wins.
4. **Tiers** the winner by a *second, independent* signal — whether the function's own `entry_va` sits
   inside (± 0x400) the address range spanned by that class's *named* functions (its `.text` cluster):
   - **STRONG** — data reference **and** in-cluster (two signals agree).
   - **data-only** — references the class's data but sits outside its cluster (a weaker lead: likely a
     helper or caller in a *different* class that operates on this class's data).

The data signal and the location signal are independent — one is what the function *does*, the other is
where the linker *put* it — so their agreement is corroboration in the project's usual sense, the same shape
as a field that reproduces across two files.

## 3. Why this clears C27.3's bar

[C27.3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md#4-the-remaining-124) explicitly
declined to promote its 47 proximity candidates: "proximity in `.text` is a real signal … but it is not an
address match, and C0.3's whole finding … was that plausible-looking signals which aren't independently
checked produce confident wrong names." The classifier answers that objection directly — it never rests on
proximity alone. A STRONG attribution requires the function to *manipulate the class's data*, which
proximity cannot fake, and uses proximity only as the corroborating second vote. It still does not claim a
method name (that needs the external source C27 used, which is not in-tree), so every verdict here is a
**class** attribution at 🟡, with the four spot-checked cases raised to ✅ that the reference is genuine.

To keep the result honest, `derive_attribution.py` hard-codes the expected STRONG set and the four
spot-check instruction signatures, and **refuses to write** its JSON if a code change makes the run diverge
from the documented eleven — so this page cannot drift from the classifier.

---

### Key takeaways

- The signature map is 12 class-owned static ranges/strides, each a fingerprint pinned by a C28–C32 chapter.
- Each unnamed function gets a **data-flow vote** (what statics it touches) and a **location signal** (which
  class cluster it sits in); STRONG requires both to agree.
- This supplies the second, independent signal [C27.3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md)
  said proximity leads lacked — without asserting a method name.

**Next:** [C33.2 — The eleven, and what remains](02-the-eleven-and-what-remains.md).
