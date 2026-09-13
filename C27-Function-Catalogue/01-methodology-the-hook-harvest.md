# C27.1 — Methodology: Harvesting the Hook Addresses

> **The one-sentence version:** `gta-reversed` reimplements the game by hooking the original functions
> at their real addresses, and every hook install site is written as
> `RH_ScopedInstall(MethodName, 0xADDRESS)` inside a scope opened by `RH_ScopedClass(ClassName)` — which
> means the project's 2,232-file source tree contains, incidentally, a 6,988-entry address database with
> full class-qualified names, free for the extracting.

**Subsystem category:** Binary substrate — methodology
**Depends on:** [C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md) (`entry_va`
is the key we match on)
**RE status:** Documented
**Confidence:** ✅ Verified (the parser's output was checked against source by hand; see §3)

---

## 1. What `RH_ScopedInstall` actually is

`gta-reversed` is not a disassembly or a decompiler output — it is a from-scratch C++ reimplementation
of GTA:SA's engine that installs itself over the original code function-by-function, so that individual
systems can be toggled between "real game code" and "reimplemented code" at runtime for testing. To do
that safely, every hook needs to know exactly where the original function lives. The project's own
header lays the pattern out plainly:

```cpp
#define RH_ScopedClass(cls) \
    using RHCurrentNS = cls; \
    ReversibleHooks::ScopeName RHCurrentScopeName {#cls};

#define RH_ScopedInstall(fn, fnAddr, ...) \
    ReversibleHooks::Install(RHCurrentCat.name + "/" + RHCurrentScopeName.name, #fn, fnAddr, \
                              &RHCurrentNS::fn __VA_OPT__(,) __VA_ARGS__)
```

A typical call site (`source/game_sa/Pools/VehiclePool.h`):

```cpp
RH_ScopedClass(CVehiclePool);
RH_ScopedInstall(New, 0x006E2A50);
```

Read together, that's a direct, load-bearing claim: **`CVehiclePool::New` lives at `0x006E2A50`.** If
the address were wrong, the hook would install over the wrong bytes and the reimplementation would
crash or corrupt state the first time that path ran — which happens constantly during normal play and
during the project's own CI. Unlike a symbol name in a comment, a wrong address here is not silently
wrong; it is loudly, immediately wrong. That is a materially stronger evidence tier than the templated
COM-guess names C0.3 found in the automated dump, and it is why this source is treated as a genuine lead
rather than another "external, unaudited" cite.

There are five install macro variants encountered in the tree; all follow the same address-carrying
shape:

| Macro | Used for |
|---|---|
| `RH_ScopedInstall(fn, addr)` | Ordinary member function of the current scoped class |
| `RH_ScopedGlobalInstall(fn, addr)` | Free function, no class scope |
| `RH_ScopedOverloadedInstall(fn, suffix, addr, cast)` | One of several overloads of `fn`; `suffix` disambiguates |
| `RH_ScopedVMTInstall(fn, addr)` / `RH_ScopedVMTOverloadedInstall(...)` | Virtual function, installed via vtable slot rather than a direct jump |
| `RH_ScopedNamedInstall(fn, name, addr)` / `RH_ScopedNamedGlobalInstall(...)` | Explicit name override |

---

## 2. The scope-tracking parser

Addresses alone aren't useful — `0x6E2A50` needs to be `CVehiclePool::New`, not just `New`. But the
class name isn't repeated at every install site; it's set once by `RH_ScopedClass` (or
`RH_ScopedVirtualClass`, `RH_ScopedNamespace`, `RH_ScopedNamespaceName`) and stays in effect for every
`RH_Scoped*Install` that follows in the same file, until the next scope macro changes it. So recovering
full names means a small stateful line-by-line scan per file, not a single regex pass:

```python
for line in file:
    if RH_ScopedClass / RH_ScopedNamespace / ... matches:
        current_class = <captured name>
    elif RH_ScopedInstall / RH_ScopedOverloadedInstall / RH_ScopedVMTInstall matches:
        record(address, f"{current_class}::{fn}")
```

Run across `gta-reversed/source/**/*.{h,cpp}` (2,232 files), this produces **6,988 hook install sites**
resolving to **8,340 distinct addresses** — more addresses than install sites because a handful of
addresses are hooked more than once under different names in different contexts (overload
disambiguation, or the same code hooked at both a wrapper and its target). Only **4** addresses in the
whole tree resolve to genuinely conflicting names on inspection; that low a collision rate over 6,988
call sites is itself a sign the scope-tracking is working correctly, not silently misattributing class
context across file boundaries.

## 3. Matching against `entry_va`

C0.2 established that `entry_va` — the original call-site address, still the correct address to *call*
even on this HOODLUM build — is exactly the address any external, non-crack-aware source (including
`gta-reversed`, which targets the uncracked Compact-exe address layout) will report for a function. So
matching is a direct address lookup, no fuzzing or heuristics needed:

```python
for r in hoodlum_relocation_map["relocations"]:          # 492 entries
    if hex(int(r["entry_va"], 16)) in gta_reversed_addr_map:
        r["name"] = gta_reversed_addr_map[...][0]
```

**367 of 492** relocation entries match this way. One further entry (`0x00406360`) is already named
`CdStreamShutdown` by this encyclopedia's own C1 chapter and is preferred over any external source at
✅ rather than 🔷, per the tiering this project has used since C0.3. That's **368 named, 124 open** —
see [C27.2](02-the-catalogue-and-class-map.md) for the result and
[C27.3](03-verification-and-the-remaining-124.md) for verification and what the remaining 124 look like.

---

### Key takeaways

- `gta-reversed`'s `RH_ScopedInstall` call sites are a real, self-checking address database: a wrong
  address there breaks the project's own hooking at runtime, which is a much stronger correctness
  signal than an unaudited comment or an automated scanner's guess.
- Recovering full class-qualified names requires tracking scope state (`RH_ScopedClass` /
  `RH_ScopedNamespace`) per file, not a single-pass regex.
- The harvest yields 8,340 distinct addresses from 6,988 install sites across 2,232 files; only 4
  addresses show name conflicts.
- Matching against the 492 relocated `entry_va` values is a direct lookup — C0.2's resolver model is
  exactly what makes this trivial rather than heuristic.

**Next:** [C27.2 — The catalogue and the class map](02-the-catalogue-and-class-map.md).
