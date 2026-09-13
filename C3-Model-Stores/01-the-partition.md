# C3.1 — The Partition

> **The one-sentence version:** eight ranges, seven `cmp` immediates, one descending chain — and the
> counts sum to exactly 26,316, which is what turns a list of boundaries into a verified partition.

[← Chapter 3 hub](C3-Model-Stores.md) · [Next: C3.2 — The unhandled band →](02-the-unhandled-band.md)

**Confidence:** ✅ Verified (boundaries, arithmetic, handlers) / 🟡 (type labels)
**Dispatcher:** `0x004089A0`

---

## 1. The chain

The dispatcher decides what an ID *is* by falling through a descending sequence of comparisons. Each
arm rebases the ID to a store-local index and calls that store.

```
004089c6: cmp  esi, 0x4e20      ; 20000   -> below: models
00408a28: cmp  esi, 0x61a8      ; 25000   -> below: textures
00408a44: cmp  esi, 0x62a7      ; 25255   -> below: collision
00408a5d: cmp  esi, 0x63a7      ; 25511   -> below: map sections
00408a76: cmp  esi, 0x63e7      ; 25575   -> below: path/node data
00408a91: cmp  esi, 0x649b      ; 25755   -> below: animations
00408aaa: cmp  esi, 0x6676      ; 26230   -> below: NOTHING (C3.2)
                                ;         -> at/above: streamed scripts
```

✅ *Verified* — each immediate read directly from its instruction.

## 2. The table

| # | Range | Count | Rebase | Handler | Call form |
|---|---|---:|---|---|---|
| 1 | `0 – 19999` | 20,000 | none | `ms_modelInfoPtrs[id]` | virtual ([C3.3](03-model-info-polymorphism.md)) |
| 2 | `20000 – 24999` | 5,000 | `− 0x4E20` | `0x00731E90` | `__cdecl` |
| 3 | `25000 – 25254` | 255 | `− 0x61A8` | `0x00410730` | `__cdecl` |
| 4 | `25255 – 25510` | 256 | `− 0x62A7` | ⏳ not captured | — |
| 5 | `25511 – 25574` | 64 | `− 0x63A7` | `0x0044D0F0` | `__thiscall`, `this = 0x0096F050` |
| 6 | `25575 – 25754` | 180 | `− 0x63E7` | `0x004D3F40` | `__cdecl` |
| 7 | `25755 – 26229` | **475** | — | **none** | — |
| 8 | `26230 – 26315` | 86 | `− 0x6676` | `0x004708E0` | `__thiscall`, `this = 0x00A47B60` |

```
20000 + 5000 + 255 + 256 + 64 + 180 + 475 + 86 = 26,316
```

## 3. Why the sum is the proof

Seven boundary values read from seven separate instructions have no obvious reason to add up to
anything in particular. That they reproduce **26,316** — the array capacity established by a completely
different method in [C2.1 §3](../C2-CStreaming/01-the-id-space.md), where `base + n × 20` had to land
on the next global — is the check that validates both results at once.

Either result alone would be 🟡. Together they are ✅, because two unrelated derivations agreeing by
coincidence is not plausible.

This is worth stating as a general method: **a partition is verified when its parts sum to an
independently established whole.** Reading boundaries off `cmp` instructions is easy; knowing you found
*all* of them is the hard part, and the sum is what tells you.

## 4. Two call conventions

Ranges 5 and 8 are `__thiscall` against a fixed global object:

```
00408a85: mov  ecx, 0x96f050
00408a8a: call 0x44d0f0

00408ab9: mov  ecx, 0xa47b60
00408abe: call 0x4708e0
```

Ranges 2, 3 and 6 are `__cdecl` free functions with the rebased index pushed and `add esp, 4` after:

```
00408a30: lea  ecx, [esi - 0x4e20]
00408a36: push ecx
00408a37: call 0x731e90
00408a3c: add  esp, 4
```

🟡 *Reasoned:* the split reflects two eras of store design — some stores are singleton *objects*
(`0x0096F050`, `0x00A47B60`), others are namespaces of free functions over file-static arrays. Nothing
here depends on which; it matters only to the SDK's binding, which must record the convention per
range.

## 5. The rebasing asymmetry

Range 1 does **not** rebase — model IDs are already the model store's index. Every other range
subtracts its own base.

This is the single most error-prone thing in the whole ID space, because it makes the flat ID and the
store index coincide for exactly one range. Code that works on models and is then generalised to
textures will be wrong by 20,000 and will index a valid-looking slot rather than crashing.

The SDK's answer is typed IDs ([C3.4](04-typed-ids-for-the-sdk.md)) — make the conversion impossible to
forget by making the types incompatible.

## 6. What was not captured

⏳ Range 4's handler. The disassembly shows the boundary test and the rebase:

```
00408a5d: cmp  esi, 0x63a7
00408a63: jge  0x408a76
00408a65: lea  eax, [esi - 0x62a7]
```

but the following `push`/`call` was not read in this pass. The *range* is verified; the *handler
address* is not, and is recorded as unknown rather than guessed from the pattern of its neighbours.

---

### Key takeaways

- Eight ranges, seven boundaries, all read directly from `cmp` immediates.
- **The counts sum to exactly 26,316**, independently confirming
  [C2.1](../C2-CStreaming/01-the-id-space.md) — and confirmed *by* it.
- General method: **a partition is verified when its parts sum to an independently established whole.**
- Two call conventions in use — `__thiscall` on singleton objects for ranges 5 and 8, `__cdecl` free
  functions for 2, 3 and 6.
- **Only range 1 does not rebase**, which is the classic off-by-20000 trap.
- Range 4's handler address was not captured and is left open rather than inferred.

**Continue:** [C3.2 — The unhandled band](02-the-unhandled-band.md)
