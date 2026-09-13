# C6.3 — CColStore & the Model Binding

> **The one-sentence version:** collision attaches to geometry by model ID with a name-hash guard, so
> the binding survives the `.ide` reordering that mods cause — and the store it lives in is a `CPool`
> of 44-byte records outside the seventeen built at startup.

[← C6.2 — The header and bounds](02-header-and-bounds.md) · [Chapter 6 hub](C6-Collision.md)

**Confidence:** ✅ Verified (binding, store shape) / 🟡 (record fields)

---

## 1. The binding

```
005384f4: cmp   eax, 0x4e20                       ; modelId < 20000  (C3.1 boundary)
005384fd: jge   0x538506                          ; out of range -> skip lookup
005384ff: mov   ebx, dword ptr [eax*4 + 0xa9b0c8] ; ms_modelInfoPtrs[modelId]
00538506: test  ebx, ebx
00538508: je    0x53851e                          ; no such model -> fallback
0053850a: mov   esi, dword ptr [ebx + 4]          ; modelInfo->nameHash
0053850d: lea   edx, [esp + 0x20]                 ; the COL header's name
00538512: call  0x53cf30                          ; hash it
0053851a: cmp   esi, eax
0053851c: je    0x538569                          ; match -> bind
                                                  ; mismatch -> fall through
```

✅ *Verified.* Three checks in sequence: the ID is in the model range, the slot is non-null, and the
name hashes to the same value the model info holds at `+0x04`.

**`CBaseModelInfo` carries its name hash at `+0x04`** — a new field, recovered here, adding to the
`+0x22` model index from [C4.3](../C4-Entities-And-Pools/03-entity-model-link.md) and the vtable
observations in [C3.3](../C3-Model-Stores/03-model-info-polymorphism.md).

## 2. Why the guard exists

Model IDs in San Andreas are assigned by position in the `.ide` files. Add or remove a line and every
ID after it shifts. Mods do this constantly.

Without the guard, a shifted ID would attach a building's collision to whatever model now occupies that
slot — geometry and collision silently disagreeing, which is among the hardest classes of modding bug
to diagnose because nothing errors and the visual is correct.

With the guard, a shifted ID fails the hash comparison and control falls to `0x0053851E`, which is a
search path. The design is **ID as a fast path, name as the authority**.

🟡 *Reasoned:* the fallback is a name-based lookup rather than an abort — consistent with the branch
target continuing into more work rather than returning, and with the game tolerating reordered `.ide`
files in practice. The exact fallback was not fully traced.

## 3. The store is a `CPool` of 44-byte records

The fallback path immediately works with an object at `0x00965560`:

```
00538523: mov   esi, dword ptr [0x965560]
00538529: mov   edx, dword ptr [esi + 4]      ; +0x04 -> byte map
0053852c: cmp   byte ptr [edx + eax], 0
00538530: jns   0x538536                      ; sign bit = 0x80 = free
00538536: mov   edi, dword ptr [esi]          ; +0x00 -> objects
0053853a: imul  ecx, ecx, 0x2c                ; × 44
0053853d: add   ecx, edi
```

✅ *Verified.* That is the `CPool` layout from
[C4.5 §2](../C4-Entities-And-Pools/05-the-cpool-object.md) used verbatim — `+0x00` objects, `+0x04`
byte map — including the **`0x80` free-bit test via `jns`**, which is the same encoding the pool
constructor establishes.

So `0x00965560` holds a **pointer to a `CPool` whose element size is 44 bytes**.

It is **not** one of the seventeen pools from `CPools::Initialise`
([C4.1](../C4-Entities-And-Pools/01-the-pool-table.md)) — those globals are contiguous at
`0x00B744xx`. This pool is created elsewhere and belongs to the collision store.

🟡 *Reasoned:* 44 bytes, indexed alongside collision loading, is the per-archive COL-store record — one
entry per streamed `.col`, i.e. per COL-range streaming ID. 255 × 44 = 11,220 bytes, which is a
plausible size for a store that tracks 255 archives.

✅🔷 **Closed** in [X1 §3.4](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md): the record is
`ColDef`, 44 bytes, and the two signed fields are `m_nModelIdStart` / `m_nModelIdEnd` — a COL archive
covers a *model-ID range*, which is why it needs both. The offsets observed here:

```
00538550: movsx ecx, word ptr [ecx + 0x24]
00538554: movsx edx, word ptr [eax + 0x22]
```

`+0x22` and `+0x24` of the 44-byte record, both `movsx` — so both are **signed** `int16`, and per the
lesson in [C4.3 §2](../C4-Entities-And-Pools/03-entity-model-link.md), `-1` is very likely a sentinel.
Their meaning was not determined.

## 4. The budget, from three directions

The collision system's sizing is unusually well cross-checked, because three independently recovered
numbers constrain each other:

| Quantity | Value | Source |
|---|---:|---|
| COL streaming slots | 255 | [C3.1](../C3-Model-Stores/01-the-partition.md) — `cmp` boundaries |
| Shipped `.col` archives | 251 | archive census, [C6 hub](C6-Collision.md) |
| `ColModel` pool capacity | 10,150 | [C4.5](../C4-Entities-And-Pools/05-the-cpool-object.md) — ctor |
| Shipped collision models | 10,155 | archive walk |

**251 archives in 255 slots. 10,155 models against 10,150 pool entries.**

The first is a four-slot margin. The second is a five-slot *deficit* — and it is fine only because
archives stream: never all resident, so the peak live model count stays under the pool capacity.

🟡 *Reasoned:* both numbers look chosen against the shipped content census rather than picked as round
figures, which is consistent with `ColModel` being 10,150 (not 10,000 or 10,240) and with the COL range
being 255 (not 256 — one slot presumably reserved, or simply the largest value a `uint8` index can
address before the sentinel).

The practical consequence for modders: **the COL store has four spare slots.** Adding a fifth new
collision archive to a stock install requires raising the limit, and that limit is a `cmp` immediate in
the dispatcher rather than a data-driven value.

---

### Key takeaways

- Collision binds **by model ID with a name-hash guard** — ID is the fast path, name is the authority.
- **`CBaseModelInfo` holds its name hash at `+0x04`** — recovered here.
- The guard exists because `.ide` reordering shifts model IDs; without it, collision would silently
  attach to the wrong geometry.
- The COL store is a **`CPool` at `0x00965560` with 44-byte records** — the same pool layout as C4.5,
  including the `0x80` free-bit `jns` test — but **not** one of the seventeen startup pools.
- Two signed `int16` fields at record `+0x22` and `+0x24`; meanings **open**.
- The budget cross-checks three ways: **251 archives / 255 slots** (four spare) and **10,155 models /
  10,150 pool entries** (a deficit that only streaming makes viable).
- Adding a fifth collision archive to stock requires patching a **`cmp` immediate**, not a data file.

**Continue:** [Chapter 6 hub](C6-Collision.md) · next chapter: `C7 — RenderWare: DFF, TXD & the Model Payload`
