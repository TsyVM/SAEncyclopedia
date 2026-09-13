# C3.3 — Model-Info Polymorphism

> **The one-sentence version:** the model range is the only one that dispatches through a vtable —
> 20,000 slots of `CBaseModelInfo*` covering buildings, vehicles, peds and weapons, distinguished at
> runtime by a type tag returned from vtable slot `+0x10`.

[← C3.2 — The unhandled band](02-the-unhandled-band.md) · [Chapter 3 hub](C3-Model-Stores.md) ·
[Next: C3.4 — Typed IDs for the SDK →](04-typed-ids-for-the-sdk.md)

**Confidence:** ✅ Verified (array, dispatch, tags observed) / 🟡 (tag meanings)

---

## 1. The array

```
004089cf: mov  ebx, dword ptr [esi*4 + 0xa9b0c8]
```

**`CModelInfo::ms_modelInfoPtrs` at `0x00A9B0C8`** — 20,000 pointers, indexed by raw model ID with no
rebasing ([C3.1 §5](01-the-partition.md)). ✅ Verified.

Scale `*4` and no offset: the array base is folded into the displacement, so the whole lookup is one
instruction. Recognising `[reg*4 + 0xa9b0c8]` anywhere in the binary identifies model-info access
immediately, and it is one of the most useful signatures in the game.

## 2. The dispatch

```
004089d6: mov  eax, dword ptr [ebx]        ; vtable
004089d8: mov  ecx, ebx
004089da: call dword ptr [eax + 0x20]      ; virtual #8
004089dd: mov  edx, dword ptr [ebx]
004089df: mov  ecx, ebx
004089e1: call dword ptr [edx + 0x10]      ; virtual #4 -> type tag in al
004089e4: cmp  al, 7
...
00408a0a: mov  eax, dword ptr [ebx]
00408a0c: mov  ecx, ebx
00408a0e: call dword ptr [eax + 0x10]      ; called AGAIN
00408a11: cmp  al, 6
```

✅ *Verified.* Two vtable slots are used:

| Slot | Offset | Behaviour |
|---|---|---|
| #4 | `+0x10` | returns a small integer **type tag** in `al`; no side effects (called twice, result compared twice) |
| #8 | `+0x20` | called first, return value unused here — a **release/delete** of the model's payload |

That slot `+0x10` is called twice with its result compared against different constants, and that the
code does not cache it, is itself evidence: it is a cheap constant-returning accessor, the classic
`virtual int GetModelType()`.

## 3. The tags

Two values are tested: **`7`** and **`6`**.

- `al == 7` → run the 8-slot side-table scrub (§4).
- `al == 6` → call `0x004080F0` with the model ID.

🟡 *Reasoned:* these are two of the model-info subclasses. Which two was not established — doing so
means finding the concrete `GetModelType` overrides and reading their immediate returns, which this
pass did not do. The community mapping assigns low tags to simple/timed/weapon/clump types and higher
ones to ped and vehicle info, but that is not evidence and is not adopted here.

⏳ **Open:** the full tag → subclass map. What *is* verified is that a tag exists, that it is returned
from `+0x10`, and that at least tags 6 and 7 receive special unload handling.

## 4. The 8-slot side table

Inside the tag-7 path:

```
004089e8: mov  ecx, dword ptr [0x8e4bb0]   ; count
004089ee: mov  eax, 0x8e4c00               ; table base
004089f3: cmp  dword ptr [eax], esi        ; == this model id?
004089f5: jne  0x4089fa
004089f7: mov  dword ptr [eax], ebp        ; ebp = -1  -> clear the slot
004089f9: dec  ecx
004089fa: add  eax, 4
004089fd: cmp  eax, 0x8e4c20               ; end
00408a02: jl   0x4089f3
00408a04: mov  dword ptr [0x8e4bb0], ecx
```

✅ **`0x008E4C00 .. 0x008E4C20`** — 32 bytes, **8 entries of 4**, holding model IDs, with a live count at
`0x008E4BB0` (33 references). Cleared slots are set to `-1`.

The scan is linear and unconditional — it checks all 8 slots even after a match, and decrements the
count once per match. A small fixed registry of models that some other subsystem is holding by ID and
must be told to forget.

🟡 *Reasoned:* 8 slots of tag-7 models, scrubbed on unload, reads like a cache of currently-relevant
special models. What holds it was not identified.

## 5. Why only this range is polymorphic

Every other range is flat because its assets are homogeneous — a TXD is a TXD, a collision archive is a
collision archive. The model range covers fundamentally different things that happen to share an ID
space: static buildings, animated peds, drivable vehicles, held weapons.

The engine's answer is to make the *store* uniform (a flat pointer array) and the *element*
polymorphic. That is why range 1 needs no rebasing — its "store index" is the array index — and why it
is the only range whose unload path cannot be a single function call.

**Consequence for the SDK:** `SA::Model::Info(id)` must return something type-safe. Handing back a raw
`CBaseModelInfo*` invites callers to cast to `CVehicleModelInfo*` based on an assumption. The type tag
exists and is cheap to call; the SDK should query it and return a discriminated handle, so the cast is
checked once at the boundary rather than guessed at every call site.

---

### Key takeaways

- **`ms_modelInfoPtrs` at `0x00A9B0C8`**, 20,000 entries, indexed by raw model ID — the
  `[reg*4 + 0xa9b0c8]` idiom identifies model-info access anywhere.
- Two vtable slots used: **`+0x10` returns a type tag**, **`+0x20` releases the payload**.
- Slot `+0x10` is called twice without caching — evidence it is a cheap constant accessor.
- Tags **6** and **7** get special unload handling; the full tag map is **open**.
- An **8-entry side table** at `0x008E4C00` (count at `0x008E4BB0`) is scrubbed when a tag-7 model
  unloads; its owner is unidentified.
- The model range is polymorphic because it is the only *heterogeneous* range — which is also why it
  alone needs no rebasing.

**Continue:** [C3.4 — Typed IDs for the SDK](04-typed-ids-for-the-sdk.md)
