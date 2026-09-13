# Chapter 4 — Entities & the Pool Allocator

> **Goal of this chapter:** decode where every live object in San Andreas actually lives — the
> seventeen fixed-size pools created at startup, their exact capacities as the engine itself names
> them, and the field that ties an entity back to the streaming system.

**Subsystem category:** Core engine
**Depends on:** [C3 — The Model Stores](../C3-Model-Stores/C3-Model-Stores.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C4.1 — The pool table](01-the-pool-table.md): seventeen pools with the engine's own names.
- [C4.2 — The construction idiom](02-construction-idiom.md): the repeated pattern, and the seventeen
  struct sizes it puts within reach.
- [C4.3 — The entity→model link](03-entity-model-link.md): the signed field at `+0x22`.
- [C4.4 — Pools and the SDK](04-pools-and-the-sdk.md): handles, diagnostics, and why capacities must
  never be compiled in.
- [C4.5 — The `CPool` object](05-the-cpool-object.md): the 20-byte layout, **all seventeen element
  sizes**, and the engine's own 7-bit generation counter — closes four open items.

---

## 4.1 San Andreas does not use the heap for game objects

Every ped, vehicle, building, object and task in the game is allocated from a **fixed-capacity pool**
created once at startup and never grown. `CPools::Initialise` at `0x00550F10` builds seventeen of them
in a single unrolled sequence, and — unusually helpfully — passes each pool its own **name string**,
which the engine keeps for diagnostics.

That means the capacities below are not community measurements. They are the engine's own numbers,
next to the engine's own labels.

## 4.2 The pool table

| Pool object | Capacity | Name (engine's own string) |
|---|---:|---|
| `0x00B74484` | **70,000** | `PtrNode Single` |
| `0x00B74488` | **3,200** | `PtrNode Double` |
| `0x00B7448C` | **500** | `EntryInfoNode` |
| `0x00B74490` | **140** | `Peds` |
| `0x00B74494` | **110** | `Vehicles` |
| `0x00B74498` | **13,000** | `Buildings` |
| `0x00B7449C` | **350** | `Objects` |
| `0x00B744A0` | **2,500** | `Dummys` |
| `0x00B744A4` | **10,150** | `ColModel` |
| `0x00B744A8` | **500** | `Task` |
| `0x00B744AC` | **200** | `Event` |
| `0x00B744B0` | **64** | `PointRoute` |
| `0x00B744B4` | **32** | `PatrolRoute` |
| `0x00B744B8` | **64** | `NodeRoute` |
| `0x00B744BC` | **16** | `TaskAllocator` |
| `0x00B744C0` | **140** | `PedIntelligence` |
| `0x00B744C4` | **64** | `PedAttractors` |

✅ *Verified* — every row read from the `push <count>` / `push <name>` / `mov [global], eax` triple in
`CPools::Initialise`. The globals are contiguous at 4-byte spacing from `0x00B74484` to `0x00B744C4`,
which is 17 slots exactly.

### The construction idiom

```
00550f26: push 0x14                  ; sizeof(CPool) = 20 bytes
00550f28: call 0x82119a              ; operator new
00550f40: push 0x863d10              ; -> "PtrNode Single"
00550f45: push 0x11170               ; 70000
00550f4a: mov  ecx, eax
00550f4c: call 0x550180              ; CPool<T>::ctor(count, name)
00550f5f: mov  dword ptr [0xb74484], eax
```

Each pool *object* is **20 bytes** (`operator new(0x14)`); the pool's storage is allocated separately by
the constructor. Each pool gets its own constructor address (`0x550180`, `0x550250`, `0x550320`, …) —
the template instantiated per element type.

## 4.3 Reading the table

Three observations that are not obvious from the numbers alone:

**The two `PtrNode` pools dwarf everything.** 70,000 single-link and 3,200 double-link nodes exist
because every entity is threaded into *every world sector it overlaps*
([C5](../C5-CWorld/C5-CWorld.md)). A building spanning four sectors consumes four pointer nodes. The
pool is not sized for the number of objects; it is sized for the number of object-sector memberships.

**`Peds` is 140 and `Vehicles` is 110.** These are the two most famous limits in San Andreas modding,
and they are strikingly small next to `Buildings` at 13,000. The asymmetry is the design: buildings are
static and cheap, peds and vehicles carry AI, physics and tasks.

**`PedIntelligence` is exactly 140** — one per ped slot, so it can never be the binding constraint.
`Task` at 500 and `Event` at 200 *can* be: they are shared across all peds, so roughly 3.5 tasks per
ped before exhaustion.

## 4.4 The entity → model link

`CEntity` carries its streaming ID in a 16-bit field at **`+0x22`**:

```
0054667c: movsx eax, word ptr [edi + 0x22]        ; entity->m_nModelIndex
00546680: mov   ecx, dword ptr [eax*4 + 0xa9b0c8] ; ms_modelInfoPtrs[modelIndex]
0054669b: mov   edx, dword ptr [ecx + 0x14]
0054669e: mov   eax, dword ptr [edx + 0x24]
```

✅ *Verified.* This four-instruction sequence is the join between this chapter and
[C3](../C3-Model-Stores/C3-Model-Stores.md): an entity holds an ID, the ID indexes
`ms_modelInfoPtrs` at `0x00A9B0C8`, and the model info holds the RenderWare object at `+0x14 → +0x24`.

`movsx` on a `word` means the field is **signed** — `-1` is a valid "no model" sentinel, and code that
treats it as unsigned will index the pointer array with 65535 and read 60 KB past the end.

⏳ **Open:** the rest of the `CEntity` layout — the RenderWare object pointer, the flags word, the
placement matrix — was not recovered in this pass. Only `+0x22` is established.

## 4.5 Why pools matter to the SDK

Fixed pools with known capacities and known base pointers make three SDK features straightforward, and
one dangerous:

- **Enumeration** — `SA::Peds::ForEach(...)` is a walk over a contiguous array, not a linked-list
  traversal.
- **Handle validation** — a pool index plus the pool's own slot state is a cheap, safe handle. Raw
  `CPed*` pointers should never cross the SDK boundary.
- **Diagnostics** — `SA::Debug::PoolUsage()` can report all seventeen by name, because the engine
  already stores the names.
- **The dangerous one: resizing.** Every limit-adjuster patches these capacities. The SDK must **read**
  them from the live pool objects rather than compiling the table above in as constants — a plugin
  built against 140 peds that runs alongside a limit adjuster will otherwise corrupt memory. The table
  in §4.2 is the *shipped default*, not an invariant.

That last point is the pool-layer equivalent of C0.2's rule about build identity: **measure the
process you are in, do not assume the binary you documented.**

---

### Key takeaways

- Game objects live in **17 fixed-capacity pools** built at `0x00550F10`, with globals contiguous from
  `0x00B74484` to `0x00B744C4`.
- The capacities come with the engine's **own name strings** — `Peds` 140, `Vehicles` 110, `Buildings`
  13,000, `ColModel` 10,150, `Task` 500.
- **`PtrNode Single` is 70,000** because entities are threaded into every sector they overlap — the
  pool is sized for memberships, not objects.
- Each `CPool` object is **20 bytes**; storage is allocated by a per-type constructor.
- **`CEntity::m_nModelIndex` is a *signed* 16-bit field at `+0x22`**, indexing `ms_modelInfoPtrs` at
  `0x00A9B0C8` — the join to C3, and a sentinel trap for unsigned readers.
- The SDK must **read pool capacities at runtime**, never bake them in: limit adjusters change them.

**Next:** [Chapter 5 — CWorld & Spatial Partitioning](../C5-CWorld/C5-CWorld.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md)
- **Known bugs / gotchas:** pool exhaustion is the classic 'too many objects' crash; each pool is a hard cap.
- **Modding:** CPool bases are the universal entity-hook target (peds/vehicles/objects).
- **Performance:** flat pools, generational handles; fast but capped.
