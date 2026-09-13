# C37.3 — The 76 Remaining 🔷 Methods and What Would Promote Them

> **The one-sentence version:** the 76 methods in structured classes that did not promote to ✅
> fall into four structural failure modes — utility methods that route through received pointers
> (no direct class-global reference), register-mediated indirect loads (static analysis cannot
> trace the value), cross-class utility functions, and a small set of likely-mislabelled entries
> — and each failure mode has a concrete, achievable resolution path, so the 119-of-195 result
> is explicitly a **floor**, not a ceiling.

**Subsystem category:** Binary substrate — verification
**Depends on:** [C37.2](02-the-119.md), [C33.1](../C33-Attributing-The-Unnamed/01-the-classifier.md)
(the shared signature map)
**RE status:** Documented — four failure modes characterised; upgrade path for each defined
**Confidence:** ✅ for the four failure-mode categories (exhaustive enumeration of the 76)
· 🟡 for the specific breakdown by mode (approximate counts, not individually catalogued)

---

## 1. Failure mode A: utility methods that don't reference the class's own statics (~25–30 of 76)

The self-signature rule (C37.1 §1) requires a direct reference to the class's own static address
or stride constant. The most common reason a method fails this test is that it operates entirely
through a **pointer received as an argument or via `this`** rather than through an absolute
address.

**Example:** `CGarages::GetGaragePointer(int index)` would compute
`this->m_aGarages + index * sizeof(CGarage)` using a `this`-relative offset, not the global
`0x96C048`. The disassembly sees `mov eax, [ecx + offset]` (indirect through `ecx = this`), not
`mov eax, [0x96C048]`. The signature map has `0x96C048` as the fingerprint for `CGarages`;
a method that only dereferences `[ecx]` never produces that immediate, so it does not match.

This failure mode is inherent to the class's calling convention, not to the method's correctness.
These are genuine `CGarages` methods — they are simply utility helpers that receive a pre-computed
base pointer from their callers, who *do* reference the global. The external name from
`gta-reversed` says `CGarages`; the bytes neither confirm nor deny — they are silent, not
contradictory.

**Upgrade path:** infer from callers. If every call site of `CGarages::GetGaragePointer` (as
identified by the C27 name) passes a value derived from `0x96C048`, that is indirect corroboration
— a call-graph argument. This is a moderate time investment (disassemble each call site for the
argument-computation expression) but does not require reading the target method's full semantic
body. The upgrade would move these from 🔷 to a new tier (🟡 caller-corroborated) rather than
✅, since caller-derived evidence is weaker than the method touching the static directly.

---

## 2. Failure mode B: register-mediated indirect loads (~20–25 of 76)

A more subtle case: the method *does* reference the class's static, but through a register that
was loaded from the static in a preamble, before the instructions that `derive_name_verification.py`
scans:

```asm
; preamble: load static into register
mov  esi, dword ptr [0x96C048]   ; esi = CGarages pool base
...
; body of the method (what the verification script scans for its window):
mov  eax, [esi + ecx * 4]        ; no immediate address here — only register + offset
cmp  eax, -1
```

A narrow-window static analysis tool that looks for `mov reg, [imm32]` patterns will miss the
`[esi + ecx * 4]` access if it only sees the method body past the preamble. `derive_name_verification.py`'s
current implementation scans for immediate addresses in the form `[0xXXXXXX]`, and register-relative
accesses are transparent to it.

**What "data-flow analysis" means here.** A full data-flow pass would trace `esi` backward from
the `[esi + ecx * 4]` use to its definition (`mov esi, [0x96C048]`), then record `0x96C048`
as "indirectly referenced" by the method. This is the same algorithm compilers use for constant
propagation; implementing it for the verification tool is a few hours of work but requires
extending the tool from a pattern-match scanner to a simple abstract interpreter over the x86
instruction set. The result would promote 20–25 additional methods.

---

## 3. Failure mode C: methods in classes not yet structured (~15–20 of 76)

When C37.2 ran, the signature map covered 13 classes from C28–C36. Methods named as belonging
to classes that subsequent chapters will structure (C38 Skybox, C39 Camera, C40 Render Pipeline,
C41 Ped AI, C42 Vehicle Physics, C44 Shaders, C47 Vehicle Dynamics…) were not tested, because
those classes had no signature entries yet.

**The re-run guarantee.** This is the most important upgrade path: `derive_name_verification.py`
is deterministic and re-runs the full 492 on every invocation. Every new chapter that pins a
class's static addresses adds rows to the signature map; the next tool run automatically includes
those methods in the promotion sweep. No code changes to the tool are required.

The projected benefit: of the 297 remaining 🔷 methods (492 − 195 in structured classes), the
majority belong to classes that will be structured in C38–C53. As those chapters land, the
promotion count will climb automatically. The 15–20 methods in this failure mode within the
already-tested 195 are a lower bound on the per-chapter gain.

---

## 4. Failure mode D: mismatched or wrapper-function entries (~10–15 of 76)

A minority of methods fail verification for a different reason: the static-analysis result is
correct, and the method genuinely does not manipulate the class its name claims. Three sub-cases:

**D1: Naming errors in `gta-reversed`.** The external source occasionally maps a function address
to the wrong class name. The self-signature check is then trying to confirm, say, `CPathFind`
behavior in a function that actually manipulates `CStreaming` data — it fails, correctly. These
are real data-quality issues in the input. The remedy is individual disassembly of the suspect
function: if it manipulates a different class's data, the `gta-reversed` name is wrong and should
be flagged.

**D2: Virtual-dispatch thunks.** SA uses single-inheritance C++ with vtables for polymorphic
entities (`CEntity`, `CPhysical`, `CVehicle`). Vtable thunks are wrapper functions — they push
the arguments and `call` the real implementation — and typically contain no class-data references
of their own. The thunk for `CPhysical::SetModelIndex(int)`, for example, may just be:

```asm
; thunk for CPhysical::SetModelIndex
push ecx               ; save this
call RealSetModelIndex ; tail call or near call
pop ecx
ret
```

No immediate address to the `CPhysical` class data. The thunk is correctly 🔷 — it does no
independent work and has no independent signature to verify.

**D3: Inlined callers misidentified as callees.** When the compiler inlines a small function,
the caller's body now contains the inlined code — but the function-boundary analysis (from
`gta-reversed`'s hook harvest) may have attributed the combined body's address to the callee's
name. The verification would check the combined body against the callee's class signature, which
may or may not match depending on what the caller does. This is an edge case, not a systematic
failure mode, but it explains a few unexpected mismatches.

---

## 5. The ✅ / 🔷 trajectory over time

| Milestone | ✅ count | % of 492 |
|-----------|---------|---------|
| After C27 (names only, no verification) | 13 | 2.6% |
| After C37 (self-signature, 13 classes) | 132 | 26.8% |
| Projected: C38–C44 land (6 more classes, ~20 promotions/class) | ~252 | ~51% |
| Projected: all C38–C53 land (16 classes, data-flow pass added) | ~350–380 | ~71–77% |
| Hard ceiling without full disassembly | ~380 | ~77% |

The ~23% hard ceiling is the irreducible remainder: thunks, pure-compute helpers with no
class-data references, and the misidentified entries that will only resolve under individual
semantic disassembly.

---

## 6. The C33/C37 complementarity, revisited

[C33](../C33-Attributing-The-Unnamed/C33-Attributing-The-Unnamed.md) takes the 124 **unnamed**
functions and asks: *what class's data do you reference?* → produces class attributions.

C37 takes the 195 **named functions in structured classes** and asks: *does your code confirm
your name?* → promotes confident ones to ✅, leaves the rest at 🔷 with a documented reason.

Both use the same signature map. A function that C33 attributes to `CGarages` as a "data-only
lead" (🟡) could, if later named by external research, be run through C37's check: if it passes,
it promotes to ✅ — the entire evidence chain closes. This is the encyclopedia's intended
long-term arc: each chapter adds to the map, the attribution tool runs, the verification tool
runs, and the confidence tier of every function either rises or its failure mode is catalogued.

---

### Key takeaways

- **~25–30** of the 76 non-promoting methods are utility helpers that receive pre-computed
  pointers — they are correctly-named `CGarages` methods but their code never uses `0x96C048`
  directly. Caller-disassembly provides indirect corroboration.
- **~20–25** use register-mediated loads — the static is loaded into a register in a preamble
  and accessed indirectly thereafter. A data-flow extension to the tool promotes them.
- **~15–20** belong to classes whose chapters (C38–C53) have not yet run — they auto-promote
  when those chapters add their signatures and the tool re-runs.
- **~10–15** are thunks, inlined misidentifications, or genuine naming errors — the irreducible
  remainder for static analysis.
- The **26.8% ✅ rate after C37** is a floor: tool re-runs after C38–C53 land will raise it
  automatically, and a data-flow extension raises it further; full ✅ coverage of all 492 requires
  individual disassembly.

**Previous:** [C37.2 — The 119, and what's left at 🔷](02-the-119.md)
**Up:** [C37 — Verifying the Catalogue hub](C37-Verifying-The-Catalogue.md)
