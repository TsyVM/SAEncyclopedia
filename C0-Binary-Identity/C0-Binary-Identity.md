# Chapter 0 — Binary Identity: What `gta_sa.exe` Actually Is

> **Goal of this chapter:** establish the ground truth the entire encyclopedia rests on. Before any offset
> can mean anything, you must know *which* binary you are reading and *where its code really lives*. The
> `gta_sa.exe` in this tree is the retail **1.0 US** image wrapped by **SecuROM** and cracked by **HOODLUM**,
> and the crack does something that breaks naive reverse engineering: **492 function bodies do not live at
> their documented addresses** — they are relocated into a `.HOODLUM` section and reached by a 5-byte `jmp`
> planted at each original entry. This chapter defines the build fingerprint, the address model every other
> chapter uses (an address is a *pair*, not a number), and the resolver that makes the other 48 chapters
> possible.

**Subsystem category:** Executable identity & the address-resolution foundation
**Depends on:** nothing — this is the root chapter; every other chapter depends on *it*
**Ties:** [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md), [C33](../C33-Attributing-The-Unnamed/C33-Attributing-The-Unnamed.md), [C37](../C37-Verifying-The-Catalogue/C37-Verifying-The-Catalogue.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md)
**RE status:** ✅ Documented
**Confidence:** ✅ Verified — MD5, section table, the 492 relocations and the `jmp`-thunk model are all
proven from the file; external analysis is admitted only at 🔷 (lead, not evidence)
**Data artifact:** [`RE-Data/data/hoodlum_relocation_map.json`](../RE-Data/data/hoodlum_relocation_map.json)
— the 492 `entry_va → body_va` relocations (input to C27/C28)

---

## Deep-dive pages

- [C0.1 — The SecuROM wrapper & the HOODLUM layer](01-the-hoodlum-layer.md): this `gta_sa.exe` is retail
  1.0 US + SecuROM (sections 8–10) + HOODLUM (section 11); **492 function bodies live in `.HOODLUM`**, reached
  by a 5-byte `jmp` at each documented entry. Every signature scanner and prologue-copying hook that trusts
  the documented address is reading a thunk, not the function.
- [C0.2 — The build fingerprint & the address resolver](02-build-fingerprint-and-address-resolver.md): because
  two files both called "1.0 US" disagree about where those 492 bodies live, **an address is a *pair*** —
  `entry_va` (call here) and `body_va` (read code here) — resolved against an *identified* build. This page
  specifies the identification scheme, the pair model, and the resolver the whole encyclopedia is built on.
- [C0.3 — External analysis: corroboration & conflicts](03-external-analysis-corroboration-and-conflicts.md):
  an automated dump of the same binary confirms the SecuROM finding and the RenderWare 3.6 identification —
  but reports ASLR wrongly, an MD5 that isn't this file's, and a catalogue that **misses 488 of the 492**
  relocations. External analysis enters at **🔷 lead, not evidence**.

---

## 0.0 The result first

| Claim | Value | Evidence |
|---|---|---|
| Build | retail **1.0 US** | MD5 `170b3a9108687b26da2d8901c6948a18` |
| Protection | **SecuROM** wrapper (sections 8–10) | section table |
| Crack | **HOODLUM** (section 11 `.HOODLUM`) | section table + relocations |
| Relocated function bodies | **492** | 5-byte `jmp` thunks at documented entries |
| Address model | a **pair** `(entry_va, body_va)` | two files disagree on body location |
| RenderWare version | **3.6** (RW36) | corroborated (C0.3, later confirmed C44) |

## 0.1 Why this chapter is the foundation

Every other chapter cites addresses — `imul …,0x238`, `mov [esi+0x548]`, a global at `0xB6F028`. Those numbers
are only meaningful because C0 pins down *which binary* they are in and *how to resolve a documented address
to the code that actually runs*. The HOODLUM relocation is not a footnote: it is why a signature scan for a
function's prologue fails (the prologue is a `jmp`), why a hook that copies the first bytes corrupts control
flow, and why this project models an address as `(entry_va, body_va)`. The 492 relocations, captured in
[`hoodlum_relocation_map.json`](../RE-Data/data/hoodlum_relocation_map.json), are the raw material the
[C27](../C27-Function-Catalogue/C27-Function-Catalogue.md) function catalogue names, the
[C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) class catalogue structures, and the
[C37](../C37-Verifying-The-Catalogue/C37-Verifying-The-Catalogue.md) verification pass checks. Get C0 wrong and
everything downstream is reading the wrong bytes.

## 0.2 The house discipline starts here

C0 also sets the evidentiary tone the whole encyclopedia keeps: claims are tiered (✅ proven from the file /
🟡 reasoned / ⏳ open / 🔷 external lead), external analysis is *corroboration, never evidence*
([C0.3](03-external-analysis-corroboration-and-conflicts.md)), and a wrong "1.0 US" is caught by fingerprint,
not assumed away. The MD5 stamped in every derive tool's `source_exe_md5_expected` field traces back to this
chapter.

---

## Key takeaways

- This `gta_sa.exe` is **retail 1.0 US + SecuROM + HOODLUM**, MD5 `170b3a9108687b26da2d8901c6948a18`; the
  crack relocates **492 function bodies** into `.HOODLUM`, reachable only through the `jmp` thunks at their
  documented entries.
- An address in this encyclopedia is a **pair** `(entry_va, body_va)` resolved against the identified build —
  the model [C0.2](02-build-fingerprint-and-address-resolver.md) defines and every later chapter uses.
- C0 is the **root of the dependency graph**: [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md)/[C28](../C28-Class-Catalogue/C28-Class-Catalogue.md)/[C37](../C37-Verifying-The-Catalogue/C37-Verifying-The-Catalogue.md)
  and every exe-derived fact resolve through its map; external analysis is admitted only as a 🔷 lead
  ([C0.3](03-external-analysis-corroboration-and-conflicts.md)).

**Continue:** [C0.1 — The SecuROM wrapper & the HOODLUM layer →](01-the-hoodlum-layer.md)

## See also (forward links)

The 492 HOODLUM-relocated functions this chapter identifies are named in [C27 — Function Catalogue](../C27-Function-Catalogue/C27-Function-Catalogue.md), structured in [C28 — Class Catalogue](../C28-Class-Catalogue/C28-Class-Catalogue.md), and verified in [C37](../C37-Verifying-The-Catalogue/C37-Verifying-The-Catalogue.md). The shipped input/render backends (DirectInput 8, RenderWare D3D9) are covered in [C46 — Input](../C46-Input-Devices/C46-Input-Devices.md) and [C44 — Shaders](../C44-Shaders/C44-Shaders.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C1](../C1-Streaming/C1-Streaming.md), [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md), [C33](../C33-Attributing-The-Unnamed/C33-Attributing-The-Unnamed.md), [C37](../C37-Verifying-The-Catalogue/C37-Verifying-The-Catalogue.md), [C44](../C44-Shaders/C44-Shaders.md)
- **Known bugs / gotchas:** naive signature scanners read the 5-byte jmp thunk, not the function (the whole point of the chapter).
- **Modding:** the entry->body resolver is the base every hook/mod-loader tool needs on this build; MD5-gate before patching.
- **Performance:** one-time load; no runtime cost.
