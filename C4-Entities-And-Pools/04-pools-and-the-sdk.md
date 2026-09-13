# C4.4 — Pools and the SDK

> **The one-sentence version:** fixed pools make enumeration, handles and diagnostics easy — and make
> one thing dangerous, because every limit adjuster rewrites the capacities the encyclopedia just
> documented.

[← C4.3 — The entity→model link](03-entity-model-link.md) · [Chapter 4 hub](C4-Entities-And-Pools.md)

**Confidence:** design, derived from ✅ verified data
**Status:** SDK design — blocked on one open item

---

## 1. What pools give the SDK for free

**Enumeration is a walk, not a traversal.** Pool storage is contiguous, so
`SA::Peds::ForEach(fn)` is a bounded loop over an array. No list chasing, no iterator invalidation
concerns beyond the obvious.

**Handles can be safe.** A pool index plus the pool's own slot-state byte is a validatable handle: given
`{index, generation}` the SDK can answer "is this still the object you were given?" without
dereferencing anything. This is the single most valuable property pools offer a plugin API, because it
removes the dangling-`CPed*` failure mode entirely.

> **Raw engine pointers should never cross the SDK boundary.** A plugin holding a `CPed*` across a
> frame has a use-after-free waiting to happen; a plugin holding a `SA::PedHandle` has a lookup that
> can fail cleanly.

**Diagnostics are already labelled.** The engine stores each pool's name
([C4.1](01-the-pool-table.md)), so `SA::Debug::PoolUsage()` can report all seventeen by their real names
with no table in the SDK at all — read the name pointer out of the pool object and print it.

## 2. The dangerous one: never bake in capacities

The table in [C4.1](01-the-pool-table.md) is the **shipped default**, not an invariant. Open Limit
Adjuster and every comparable tool rewrites these capacities at startup. A plugin compiled against
`Peds = 140` that runs alongside one of them will:

- allocate 140-entry side tables and index them with pool indices up to the raised limit;
- write past the end of its own arrays, not the game's;
- corrupt its own heap in a way that looks like a game bug.

This is the pool-layer restatement of the rule from
[C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md): **measure the process you
are in, do not assume the binary you documented.** There it applied to function addresses; here it
applies to capacities. Same failure mode — trusting static documentation about a mutable runtime.

```cpp
// Wrong — bakes in an encyclopedia fact that mods change.
std::array<MyPedData, 140> data;

// Right — sized from the live pool.
std::vector<MyPedData> data(SA::Pools::Capacity(SA::PoolId::Peds));
```

## 3. The blocker

`SA::Pools::Capacity()` cannot be implemented yet.

Each `0x00B744xx` global holds a **pointer to a 20-byte pool object**
([C4.2 §2](02-construction-idiom.md)), and the capacity lives inside that object — but **the object's
field layout was not recovered** ([C4.1 §4](01-the-pool-table.md)). Until it is, the SDK can locate
every pool and read none of them.

⏳ This is the one open item standing between this chapter and a working API, and it is small: the
`CPool<T>` constructor at `0x00550180` takes `(count, name)` and writes them into the object. Reading
that one function gives the offsets of the capacity, the name pointer, the storage pointer and the slot
array — for all seventeen pools at once, since they share the template.

**Until then the SDK should expose nothing rather than expose the defaults.** Shipping
`Capacity(Peds) { return 140; }` would be an invented constant dressed as an API, which is exactly what
the provenance rule exists to prevent.

## 4. The proposed surface

```cpp
namespace SA::Pools {

enum class PoolId {
    PtrNodeSingle, PtrNodeDouble, EntryInfoNode, Peds, Vehicles,
    Buildings, Objects, Dummys, ColModel, Task, Event,
    PointRoute, PatrolRoute, NodeRoute, TaskAllocator,
    PedIntelligence, PedAttractors
};

Result<std::uint32_t> Capacity(PoolId) noexcept;   // blocked: needs CPool layout
Result<std::uint32_t> Used(PoolId)     noexcept;   // blocked: needs slot-state array
std::string_view      Name(PoolId)     noexcept;   // engine's own string

} // namespace SA::Pools
```

`PoolId` mirrors the engine's own order and spelling — including `Dummys`
([C4.1 §1](01-the-pool-table.md)). The enum is generated from the same JSON as everything else, so the
seventeen globals are never typed by hand.

Two functions return `Result<>` and are currently unimplemented. That is the honest state: the API
shape is known, the data to fill it is one function away, and shipping a stub that returns a plausible
number would be worse than shipping nothing.

## 5. What pools tell the SDK about lifetime

A pool slot is reused. That is the whole point of a pool, and it is why handles need a generation
counter rather than a bare index — otherwise a stale handle silently addresses whatever occupies the
slot now, which is the same class of failure as the unsigned `m_nModelIndex` read in
[C4.3 §2](03-entity-model-link.md): plausible, non-crashing, wrong.

⏳ Whether the engine's own pool objects carry anything usable as a generation counter is part of the
same unrecovered layout. If they do not, the SDK must maintain its own.

---

### Key takeaways

- Pools give the SDK **cheap enumeration**, **validatable handles**, and **self-labelling diagnostics**.
- **Raw engine pointers must not cross the API boundary** — hand out handles.
- **Never compile capacities in.** Limit adjusters change them; a plugin sized to 140 peds corrupts
  *its own* memory when they do.
- Same rule as C0.2, one layer up: **measure the running process, not the documented binary.**
- ⏳ **Blocked on the `CPool` object layout** — one constructor at `0x00550180` yields the offsets for
  all seventeen pools.
- Until then the SDK exposes **nothing** rather than the defaults; a plausible constant dressed as an
  API is the failure this project exists to avoid.
- Slot reuse means handles need a **generation counter**; whether the engine provides one is part of the
  same open item.

**Continue:** [Chapter 4 hub](C4-Entities-And-Pools.md) · [Chapter 5 — CWorld & Spatial Partitioning](../C5-CWorld/C5-CWorld.md)
