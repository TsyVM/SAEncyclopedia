# C0.3 — External Analysis: Corroboration & Conflicts

> **The one-sentence version:** an automated analysis dump of the same binary independently confirms
> the SecuROM finding and adds a verified RenderWare 3.6 identification — but it reports ASLR as
> *enabled* when it is not, reports an MD5 that is not this file's, and its function catalogue misses
> **488 of the 492 relocated functions**, which is C0.1 §4.1's prediction demonstrated against a real
> tool rather than argued.

**Subsystem category:** Binary substrate (pre-engine) — evidence handling
**Depends on:** [C0.1](01-the-hoodlum-layer.md), [C0.2](02-build-fingerprint-and-address-resolver.md)
**External source:** `dumpall.csv` — 596,867 rows, 45 MB, self-identified as *NiTools*
**RE status:** Documented
**Confidence:** ✅ Verified (every conflict re-derived from the binary)

---

## 1. Why an entry about a source, not a subsystem

MWEncyclopedia had to solve this exact problem when it cross-referenced an external SDK, and its rule
is the one adopted here:

> Where the two sources **agree**, that's corroboration and the open item gets closer to closed. Where
> they **conflict**, the conflict itself is a finding — state both, state why, don't silently pick one.

An encyclopedia that ingests a 45 MB machine-generated dump without auditing it does not gain 596,867
facts. It gains 596,867 claims of unknown quality, and it loses the one property that makes it worth
building — that a reader can tell how much to trust each line.

So this entry audits the dump before any of it is used, and fixes an evidence tier for it.

### Evidence tier for external automated analysis

| Marker | Meaning |
|---|---|
| ✅ **Verified (this project)** | Re-derived from the shipped bytes by this project |
| 🔷 **External (automated)** | Reported by an analysis tool; **not** re-derived. Usable as a *lead*, never as a citation |
| ⚠️ **External, contradicted** | Reported by the tool and shown false against the bytes |

The rule that follows: **nothing enters the address database at 🔷.** External output is a search
heuristic that tells you where to look. It is not evidence, and C0.2's `sdk_ready` gate already
requires a verified binding.

---

## 2. What it is

| Property | Value |
|---|---|
| Rows | 596,867 (the file is 597,767 lines — JSON fields contain embedded newlines, so a line count over-reports) |
| Size | 45 MB |
| Categories | 85 |
| Subject | `gta_sa.exe`, `FileSize = 14383616` — the same binary as C0.1 |

Largest categories: `xrefs.from` (239,340), `xrefs.to` (154,916), `functionBoundaries` (97,300),
`Strings` (57,169), `entropy` (14,047), `VTableFunctions` (4,357), `FunctionMap` (3,493).

**Address convention:** empirically determined, because the dump mixes conventions and documents none.
Sampling 3,000 `functionBoundaries` prologue rows and testing both readings against the file:

| Interpretation | Rows whose bytes look like a function prologue |
|---|---|
| file offset | 27 / 3,000 |
| **RVA** | **3,000 / 3,000** |

✅ *Verified:* `functionBoundaries`, `FunctionMap` and `signatures` addresses are **RVAs**; `VTables`,
`ctorDtor.ctors` and `heatmap.top` use **VAs**. Any consumer of this dump must normalise per category
or it will silently mis-bucket every row.

---

## 3. Corroboration

These agree with C0.1 and were re-derived independently here.

| Claim in dump | Status |
|---|---|
| `security.packer = "SecuROM"` | ✅ Confirms C0.1 §2 — independently reached |
| `iatRebuilt.isPackedLikely = true`, `"Packer detected: SecuROM"` | ✅ Confirms |
| `anomalies: W+X Section` | ✅ Confirms the RWX characteristics of sections 8–11 |
| PE timestamp `1114702282` → `2005-04-28T15:31:22Z` | ✅ Byte-identical to C0.1 §1 |
| `EntryPoint 0x424570`, `ImageBase 0x400000`, `FileSize 14383616` | ✅ Byte-identical |
| `timestamps.consistency = "Perfect"` | ✅ Consistent with a single-build image |

The SecuROM identification arriving from a second, unrelated method is worth something: C0.1 inferred
it from string counts in section 10, the tool from packer signatures and IAT reconstruction. Two
methods, one answer.

### 3.1 A genuine addition — RenderWare 3.6

The dump reports `engineDB: RenderWare (Criterion/EA), confidence 100`. This one is worth promoting,
so it was re-derived rather than accepted:

```
regex //RenderWare/([A-Za-z0-9._]+)/rwsdk/(...)  over the whole file
```

| Measurement | Value |
|---|---|
| `$Id:` source-path strings | **90** |
| Branch tags | `RW36Active` ×77, `dev` ×13 |
| `rwsdk` modules | `src` 37, `world` 22, `plugin` 16, `driver` 9, `tool` 5, `os` 1 |
| Located in | `.rdata`, all 90 |

Representative:

```
@@(#)$Id: //RenderWare/RW36Active/rwsdk/world/baclump.c#1 $
@@@@(#)$Id: //RenderWare/RW36Active/rwsdk/world/pipe/p2/d3d9/wrldpipe.c#1 $
@@@@(#)$Id: //RenderWare/RW36Active/rwsdk/plugin/anisot/rpanisot.c#1 $
```

✅ **Verified (this project):** the renderer is **RenderWare 3.6**, and the world pipeline is the
**D3D9 platform-2 pipe** (`world/pipe/p2/d3d9/`). Promoted from 🔷 to ✅ by re-derivation. This is the
anchor the rendering chapter will build on, and it arrived as a lead from the dump — which is exactly
what 🔷 material is *for*.

---

## 4. Conflicts

### 4.1 ⚠️ ASLR — the dump is wrong

```
security.features: {"name":"ASLR","status":"Enabled","color":"success"}
```

Contradicted. `DllCharacteristics` in the optional header is `0x0000`; the `DYNAMIC_BASE` bit
(`0x0040`) is clear.

```
struct.unpack_from('<H', d, optional_header + 70)[0]  ==  0
```

✅ *Verified:* this image has **no ASLR** and loads at `0x00400000` every run.

The consequence is not cosmetic. C0.2 §5 builds on address stability. A team that trusted this row
would either add a rebasing layer for a hazard that does not exist, or — much worse — conclude that
hard-coded addresses are unsafe here and redesign around a false constraint. The tool appears to be
reporting a desirable-security-posture default rather than reading the field; note the
`"color":"success"` alongside it.

### 4.2 ⚠️ MD5 — does not identify this file

```
hashes.md5 = 71F086AF124EFC0908492753174C8E95
actual     = 170B3A9108687B26DA2D8901C6948A18
```

Every other identity field in the dump matches this binary exactly — filename, size, entry point,
image base, timestamp — so the dump is describing the right file with the wrong hash.

Candidate sub-ranges were tested to see whether the value is a hash of *something*: first 64 KB, first
1 MB, first 4 MB, the 5,189,632-byte Compact span, the PE headers, `.text`, `.HOODLUM`. **None
reproduce it.** ⏳ *Open:* what the value is a hash of is unresolved, and left stated rather than
guessed.

**The rule it forces, which matters more than the anomaly:** C0.2 §3 makes build identity a hash. This
row shows that a *reported* hash can be wrong while every other field looks right. Therefore —

> A build fingerprint is only valid if **this project computed it**. A hash read out of someone else's
> metadata is a claim, not an identity.

That sentence is now a schema constraint, not a style preference: `builds.json` records only
self-computed digests.

---

## 5. The blind spot, demonstrated

C0.1 §4.1 predicted that a naive static load of this file would be incomplete "in a way that will not
announce itself." Here is the measurement, using the relocation map as ground truth.

`.HOODLUM` contains **492 verified function bodies**. What the dump found there:

| Dump category | Rows | In `.HOODLUM` | Coverage of the 492 |
|---|---:|---:|---|
| `FunctionMap` (the function catalogue) | 3,493 | **4** | **0.8 %** |
| `functionBoundaries` (`prologue` only) | 3,426 | 4 | 0.8 % |
| `VTableFunctions` | 4,357 | **0** | 0 % |
| `ctorDtor.ctors` | 39 | **0** | 0 % |
| `heatmap.top` (500 hottest functions) | 500 | **0** | 0 % |
| `signatures` | 385 | 4 | — |

And the sharper form — matching `FunctionMap` against the map's own addresses:

| Question | Answer |
|---|---|
| Relocated **entry stubs** present in `FunctionMap` | **0 / 492** |
| Relocated **bodies** present in `FunctionMap` | **1 / 492** |

✅ *Verified.* The catalogue misses these functions on *both* sides of the relocation: it does not list
the stub at the documented address, and it does not list the body where the code actually is. Among
the missing is `CdStreamRead` — one of the most-hooked functions in GTA SA modding — and the function
with 46 direct call sites that C0.1 recorded as the most-called relocated function.

Note also `functionBoundaries`: of 97,300 rows, **77,450 (79.6 %)** carry addresses outside every
section of the image, and 93,873 are `call_target` heuristics rather than detected prologues. The
signal-to-noise ratio of that category is poor enough that it should not be mined without filtering.

**This is the empirical justification for C0.2 in one table.** Not "signature scanning could break" —
a real tool, run on this exact file, produced a function map that omits 99.2 % of the relocated
functions and gave no indication that it had done so.

---

## 6. A second caution: template-generated method names

`VTableFunctions` (4,357 rows) carries entries such as:

```
CarPaintJobs::QueryInterface        class=CarPaintJobs index=0   Kind: "COM IUnknown"
UnknownClass_004B85A8::AddRef
UnknownClass_00457668::Release
```

Of 3,493 `FunctionMap` names, 2,333 are `sub_*` placeholders and most of the remaining 1,160 follow
the `UnknownClass_<addr>::{QueryInterface,AddRef,Release}` shape.

🟡 *Reasoned:* the tool is applying a COM `IUnknown` template to vtable slots 0/1/2. GTA SA's engine
classes are not COM objects; `CarPaintJobs` has no `QueryInterface`. The **vtable addresses** and
**slot indices** here are probably sound and worth mining; the **method names** are generated from a
template and carry no recovered information.

Ingesting these names would inject 4,357 confident-looking, wrong symbol names into the database —
the precise failure mode the tiering in §1 exists to prevent.

---

## 7. What the dump is good for

Kept, at 🔷, as leads to be verified before use:

- **`Strings` (57,169)** — the single most valuable category. Strings are directly checkable against
  the file, so promotion from 🔷 to ✅ is cheap. This is how the RenderWare result was won.
- **`VTables` (299)** + slot indices — structurally plausible; names need independent confirmation.
- **`xrefs` (394,256)** — useful for ranking what to reverse first, *provided* the consumer knows the
  graph is blind to `.HOODLUM`.
- **`Imports` (203)**, `Resources`, `entropy`, `Sections` — low-risk, mechanical, easy to re-derive.

Not used:

- `security.features` — demonstrably unreliable (§4.1).
- `hashes` — demonstrably unreliable (§4.2).
- `FunctionMap` / `functionBoundaries` names — 99.2 % blind to the relocated set (§5) and largely
  templated (§6).
- `mitre`, `capabilities`, `yara` — malware-triage output. This is a game executable; "Timing
  Anti-Debug" and "DLL Injection" fire on `timeGetTime` and `LoadLibraryA`. Not meaningful here.

---

## 8. Open items

- ⏳ **The MD5 provenance** (§4.2) — unexplained, bounded, recorded.
- ⏳ **Why 4 `.HOODLUM` functions were found and 488 were not.** Whether those 4 were reached by a
  fallback scan or by chance is unknown; understanding it would say something about what any tool must
  do differently to see the section.
- 🟡 **The 299 vtables** are the most promising unmined asset in the dump and the natural input to the
  class-catalogue work, once names are independently established.

---

### Key takeaways

- External automated analysis enters the encyclopedia at **🔷 — lead, not evidence**. Nothing reaches
  the address database without re-derivation.
- **Corroborated:** SecuROM packing, W+X sections, and every PE identity field.
- **Promoted to ✅ by re-derivation:** the renderer is **RenderWare 3.6**, D3D9 platform-2 world
  pipeline — 90 `$Id:` strings, 77 tagged `RW36Active`, all in `.rdata`.
- **Contradicted:** ASLR is reported *Enabled* but `DllCharacteristics == 0x0000` — no ASLR. A design
  built on that row would defend against a hazard that does not exist.
- **Contradicted:** the reported MD5 is not this file's, while every other identity field is correct.
  Hence the schema rule: **only self-computed digests are build identities.**
- **C0.1 §4.1 is now measured, not argued:** the dump's function catalogue contains **0/492 relocated
  entry stubs and 1/492 bodies** — 99.2 % blind, silently.
- 4,357 vtable method names are COM-template guesses, not recovered names; the addresses are usable,
  the names are not.

**Next:** `C1 — Streaming: CdStream, CStreaming & the IMG Model` — beginning from two ✅ facts already
in hand (`CdStream` array `0x008E3FFC`, stride `0x30`) and one the dump cannot help with, because
`CdStreamRead` is among the 488 it cannot see.
