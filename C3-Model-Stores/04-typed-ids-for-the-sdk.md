# C3.4 — Typed IDs for the SDK

> **The one-sentence version:** the partition is exactly the kind of knowledge that should stop being
> knowledge and start being a type — generated from the verified boundaries, so the off-by-20000 bug
> becomes a compile error.

[← C3.3 — Model-info polymorphism](03-model-info-polymorphism.md) · [Chapter 3 hub](C3-Model-Stores.md)

**Confidence:** design, derived from ✅ verified data
**Status:** SDK design — not yet implemented

---

## 1. The problem the partition creates

Three properties of the ID space, all verified in [C3.1](01-the-partition.md), combine badly:

1. **An ID does not carry its type.** `25100` is a valid collision ID and a valid nothing-else, but
   nothing in the integer says so.
2. **Only range 1 does not rebase.** For models, flat ID and store index coincide; for everything else
   they differ by the range base.
3. **Wrong answers are in-range.** Passing a texture's flat ID (`20500`) to a store expecting a local
   index yields `20500` — a plausible-looking index into a 5,000-entry store. It does not crash. It
   reads something else.

Property 3 is what makes this worth designing around. A bug class that crashes gets fixed; a bug class
that silently returns the wrong asset does not.

## 2. The design

Distinct types per range, with no implicit conversion between them and no implicit conversion to `int`:

```cpp
namespace SA {

enum class AssetKind { Model, Txd, Col, Ipl, PathNode, Anim, Recording, Script };

template <AssetKind K>
class AssetId {
public:
    // Local index -> typed id. The only public constructor.
    static constexpr AssetId FromLocal(std::uint32_t local) noexcept;

    // Flat streaming id -> typed id. Fails if the id is outside K's range.
    static Result<AssetId> FromStreaming(std::uint32_t flat) noexcept;

    constexpr std::uint32_t local()     const noexcept;   // store-local index
    constexpr std::uint32_t streaming() const noexcept;   // flat id

private:
    std::uint32_t local_;
};

using ModelId     = AssetId<AssetKind::Model>;
using TxdId       = AssetId<AssetKind::Txd>;
using ColId       = AssetId<AssetKind::Col>;
// ...

} // namespace SA
```

Both directions are explicit and named. `ModelId` and `TxdId` are unrelated types, so the compiler
rejects the mistake that §1.3 describes.

## 3. Generated, not written

The range table is data recovered from the binary, so per the rule established in
[C0.2 §4](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md) it lives in JSON and the
header is a build artifact:

```jsonc
// RE-Data/data/streaming_partition.json
{
  "build": "3fe96d15",
  "total_ids": 26316,
  "ranges": [
    { "kind": "Model",     "first": 0,     "count": 20000, "rebase": false, "handler": null },
    { "kind": "Txd",       "first": 20000, "count": 5000,  "rebase": true,  "handler": "0x00731E90" },
    { "kind": "Col",       "first": 25000, "count": 255,   "rebase": true,  "handler": "0x00410730" },
    { "kind": "Ipl",       "first": 25255, "count": 256,   "rebase": true,  "handler": null,
      "note": "handler not captured — C3.1 §6" },
    { "kind": "PathNode",  "first": 25511, "count": 64,    "rebase": true,  "handler": "0x0044D0F0",
      "this": "0x0096F050" },
    { "kind": "Anim",      "first": 25575, "count": 180,   "rebase": true,  "handler": "0x004D3F40" },
    { "kind": "Recording", "first": 25755, "count": 475,   "rebase": true,  "handler": null,
      "note": "no dispatch arm — C3.2" },
    { "kind": "Script",    "first": 26230, "count": 86,    "rebase": true,  "handler": "0x004708E0",
      "this": "0x00A47B60" }
  ]
}
```

The generator emits the `constexpr` bounds and a `static_assert` that the counts sum to `total_ids` —
so the invariant that *validates* the partition in C3.1 §3 becomes an invariant the build *enforces*:

```cpp
static_assert(SA::detail::kRangeSum == 26316,
              "streaming partition does not cover the id space");
```

If a future build changes a boundary and the JSON is updated inconsistently, compilation fails.

## 4. Carrying the open items into the type system

Two ranges have gaps ([C3.1 §6](01-the-partition.md), [C3.2](02-the-unhandled-band.md)). The SDK should
not paper over them:

- `Ipl` has no known unload handler → `SA::Ipl::Release()` should not exist yet. Generating a stub that
  calls a guessed address would be exactly the invented-address failure MWSDK's provenance rule exists
  to prevent.
- `Recording` has no dispatch arm at all → the generated header should carry the note, and any
  `Release` for it should return a "not supported on this build" error rather than silently doing
  nothing.

**The encyclopedia's confidence markers should survive into the SDK.** A range whose handler is ⏳ open
produces an API that reports the gap; it does not produce a function that appears to work.

## 5. What this does not solve

Typed IDs prevent *category* errors. They do not prevent an out-of-range index within a category, which
is why `FromStreaming` returns `Result<>` and `FromLocal` is `constexpr` (checkable at compile time for
literals, and the natural place for a debug-build bounds assert).

They also do not help with the model range's *internal* heterogeneity —
[C3.3](03-model-info-polymorphism.md) — where a valid `ModelId` may refer to a building or a vehicle.
That needs the type tag, which is a separate discriminated-handle problem at the model-info layer.

---

### Key takeaways

- The dangerous property is that **wrong IDs are in-range**: passing a flat texture ID where a local
  index belongs returns the wrong asset instead of crashing.
- The fix is **distinct types per range** with explicit `FromStreaming` / `FromLocal` conversions and no
  implicit decay to `int`.
- The table is **generated from JSON**, not hand-written — and the generator emits a `static_assert`
  that the counts sum to 26,316, turning C3.1's validation into a build-time invariant.
- **Open items propagate into the API**: ranges with unknown or absent handlers produce errors or no
  function at all, never a guessed address.
- Typed IDs solve category confusion only; range checking and model-info heterogeneity remain separate
  problems.

**Continue:** [Chapter 3 hub](C3-Model-Stores.md) · [Chapter 4 — Entities & the Pool Allocator](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)
