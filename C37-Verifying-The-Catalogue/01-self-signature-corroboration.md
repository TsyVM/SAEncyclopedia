# C37.1 — Self-signature Corroboration

> **The one-sentence version:** the promotion rule is narrow on purpose — a 🔷 method promotes to ✅ only if
> it references a static address (or record stride) that a C28–C36 chapter proved belongs to *its own*
> class, so the external name and the byte-level data-flow have to point at the same subsystem, and touching
> a *different* class's data earns no credit.

**Subsystem category:** Binary substrate — method
**Depends on:** [C37 hub](C37-Verifying-The-Catalogue.md), [C33.1](../C33-Attributing-The-Unnamed/01-the-classifier.md)
(the signature map)
**RE status:** Documented
**Confidence:** ✅ for the method

---

## 1. The rule

For each function that C27 named `Class::method` where `Class` is one of the 13 classes structured in
C28–C36, and whose confidence is still 🔷 (not already ✅ from C27.3):

1. Disassemble the body (resolver per [C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)).
2. Collect the static addresses it references and the `imul` strides it uses.
3. **Promote to ✅** iff at least one of those lands on a signature entry **for `Class` itself** — the same
   `Class` the name claims.

The signature entries are the class-private globals and record strides pinned by the structural chapters
(the [C33.1 §1](../C33-Attributing-The-Unnamed/01-the-classifier.md#1-the-signature-map) map): e.g.
`CStreaming` → the streaming-info array `0x8E4CC0` and the IMG/render region; `CGarages` → the pool at
`0x96C048` and stride `0xD8`; `CPathFind` → the node/link tables at `0x96F050…` and stride `0x1C`;
`CRunningScript`/`CTheScripts` → the variable blocks `0xA48960`/`0xA49960` and the model lists.

The **self** restriction is the whole point. A function named `CGarages::DeActivateGarage` that writes
`0x96C048` is doing garage-pool work under a garage name — agreement. If instead it only touched, say, the
streaming-info array, the rule gives **no** promotion: touching some *other* class's data neither confirms
nor denies a `CGarages` label, so the method stays 🔷. There is no cross-crediting.

## 2. Why it is not circular

The external name and the signature address are **independent inputs**. The name came from `gta-reversed`'s
hook database ([C27.1](../C27-Function-Catalogue/01-methodology-the-hook-harvest.md)) — a source that never
saw this analysis. The signature address came from *this project* disassembling the class's *other* methods
in C28–C36. When the two coincide on a function neither derivation used to reach the other, that is
corroboration, not assumption: the label says "garage," the bytes independently say "operates the garage
pool," and they were produced by different processes.

A false positive would require `gta-reversed` to have mislabelled a function as `CGarages::X` *and* that
function to coincidentally manipulate the exact global this project independently identified as the garage
pool. That conjunction is the same low-probability event [C27.3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md)
relied on when it argued a wrong match would have to "coincidentally produce plausible-looking code for its
specific claimed behavior." C37 simply mechanises the cheapest, strongest slice of that check — *does it
touch the right class's memory* — and applies it to all 195.

## 3. The contrast with C33, in one line

[C33](../C33-Attributing-The-Unnamed/C33-Attributing-The-Unnamed.md) asks the **unnamed**: *which class's
data do you touch?* → attribute a class. C37 asks the **named**: *do you touch the data of the class you
claim?* → confirm the name. Same signature map, opposite direction; C33 is inference, C37 is verification.
Where C33 also used address-cluster proximity as a second signal, C37 does not need proximity — the name
*is* the class hypothesis, and the data either matches it or does not.

## 4. Regression safety

`tools/derive_name_verification.py` hard-codes the expected promotion count (**119**) and three spot-check
addresses, and **refuses to write** its JSON if a code change makes the run diverge — so the C37.2 result
cannot silently drift from the classifier, the same discipline every `derive_*` tool in the project uses.

---

### Key takeaways

- A 🔷 method promotes to ✅ **iff** it references a signature static or stride of **its own** named class —
  name and data-flow agreeing on the subsystem.
- Touching a *different* class's data earns **no** promotion; there is no cross-crediting.
- It is not circular: the name (external `gta-reversed`) and the signature address (this project's C28–C36
  disassembly) are independent inputs that happen to coincide.
- C33 *infers* a class for the unnamed; C37 *confirms* the class for the named — same map, opposite
  direction.

**Next:** [C37.2 — The 119, and what's left at 🔷](02-the-119.md).
