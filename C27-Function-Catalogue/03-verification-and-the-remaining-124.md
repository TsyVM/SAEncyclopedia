# C27.3 — Verification, and the Remaining 124

> **The one-sentence version:** twelve matched entries were independently disassembled straight out of
> `gta_sa.exe` — reading `body_va`, not trusting `entry_va`, per C0.2's resolver rule — and in every case
> the actual instructions do what the recovered name says they do; the 124 still-unidentified functions
> are small (median 64 bytes), lightly called (median 2 direct callers, 6 with none visible), and about
> a third sit close enough to a named neighbor to be worth a lead, not a claim.

**Subsystem category:** Binary substrate — verification
**Depends on:** [C27.1](01-methodology-the-hook-harvest.md), [C27.2](02-the-catalogue-and-class-map.md)
**RE status:** Documented
**Confidence:** ✅ for the 12 disassembled here · ⏳ for the 124 open entries — no names asserted

---

## 1. Why disassemble at all

C27.1's address match is real, but it is still a claim from an *external* source, matched by address
alone. The project's own rule, set in [C0.3](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md),
is that an external source is a lead until it's checked against the file. So twelve matches — spread
across different classes, extents, and subsystems — were pulled by `body_va` (not `entry_va`; C0.2 §2
is explicit that `entry_va` on this build is a 5-byte `jmp` into `.HOODLUM`, not code) and disassembled
with Capstone directly against the shipped bytes.

## 2. Two worked examples

**`CGameLogic::IsCoopGameGoingOn`** — `entry_va 0x00441390`, `body_va 0x0156CF30`, extent 32:

```
mov eax, [0xB7CD98]
test eax, eax
je   +0x11          ; -> return 0
mov eax, [0xB7CF28]
test eax, eax
je   +0x11          ; -> return 0
mov eax, 1
ret
```

Two independent globals are read; the function returns `1` only if *both* are non-zero, `0` otherwise.
That is exactly the shape of a co-op-mode readiness check gating on two separate flags (e.g. "co-op
enabled" and "co-op session active") — not a guess this chapter is making, but the direct reading of
what the six instructions do, which happens to match what the recovered name claims.

**`CTheScripts::ClearAllVehicleModelsBlockedByScript`** — `entry_va 0x0046A840`,
`body_va 0x01565EA0`, extent 32:

```
push edi
mov  ecx, 0x14
or   eax, 0xFFFFFFFF
mov  edi, 0xA448F0
rep stosd es:[edi], eax     ; write -1, 20 times, starting at 0xA448F0
pop  edi
ret
```

A fixed-size table of 20 dwords at a single static address gets filled with `-1` (a `rep stosd` loop —
compiler-idiomatic for "clear this array" with a sentinel of `-1` rather than `0`, i.e. "no model id
blocked" rather than "model id 0 blocked"). That is precisely a "clear all \[script-imposed
vehicle-model block\] entries" operation — again read from the bytes, not asserted from the name.

## 3. All twelve, summarized

| `entry_va` | Name | Extent | Disassembly consistent with name? |
|---|---|---|---|
| `0x0048DFE0` | `CTaskSimpleUseAtm::Constructor` | 48 B | ✅ field-init pushes into a fixed-size object |
| `0x0040CF80` | `CStreaming::RemoveAllUnusedModels` | 80 B | ✅ fixed-count (`0x32`) loop calling a per-model remover |
| `0x00405B60` | `CIplStore::RemoveIplSlot` | 160 B | ✅ index-bounds-checked slot clear, same accessor idiom as §4 below |
| `0x00441390` | `CGameLogic::IsCoopGameGoingOn` | 32 B | ✅ shown in full above |
| `0x0043E1C0` | `CEventLeaderEntryExit::Constructor` | 32 B | ✅ single-field init + vtable-pointer store |
| `0x0043B000` | `CConversations::IsConversationAtNode` | 176 B | ✅ linear scan of a fixed-stride node table |
| `0x004104E0` | `CColStore::SetCollisionRequired` | 128 B | ✅ sentinel check (`-1`) then dispatch |
| `0x0040B340` | `CStreaming::RemoveLoadedZoneModel` | 96 B | ✅ guarded pointer-list unlink |
| `0x0049ADE0` | `CShopping::GetExtraInfo` | 80 B | ✅ bounds-checked array lookup |
| `0x0046A840` | `CTheScripts::ClearAllVehicleModelsBlockedByScript` | 32 B | ✅ shown in full above |
| `0x004096D0` | `CStreaming::RenderEntity` | 80 B | ✅ null/self-check guard before a render dispatch |
| `0x00470840` | `CStreamedScripts::LoadStreamedScript` | 80 B | ✅ string-tagged call into a loader |

12 / 12 disassemble to instruction sequences consistent with the claimed name and no cases where the
code contradicts it. That is a small sample against 356 remaining unverified 🔷 matches, not a proof
they're all correct — but it is a real result, not a coin flip: a wrong class/method match would need to
coincidentally produce plausible-looking code for its specific claimed behavior, which is a much higher
bar than a templated automated tool clearing (see [C0.3 §6](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md)
for what that failure mode actually looks like — this is its opposite).

A thirteenth address, `CObjectPool::GetAt` (`entry_va 0x00404870`, `body_va 0x0156DE60`), was
disassembled during pipeline development rather than as part of the fixed sample and is worth including
for the same reason: `mov edx,[ecx+4]` / flag-byte check / `mov edx,[ecx]` / `imul eax,eax,0x19C` /
`add eax,edx` — read a validity byte, then return `base + index*412`. That is the textbook shape of a
templated pool's `GetAt(index)` — check the slot's live flag, index into the flat array by the pool's
element stride — independently confirming both the name and, incidentally, a 412-byte `CObject` pool
element stride, a fact this chapter does not otherwise need but is free once the bytes are already open.

## 4. The remaining 124

No names are assigned here — per this project's standing rule, a lead is recorded as a lead. Structural
profile of the 124:

| Property | Value |
|---|---|
| Body extent | 16–768 B, median 64 B |
| Direct call sites (linear-scan count) | 0–46, median 2 |
| Functions with **zero** visible direct callers | 6 |
| Have a same-class named neighbor within `0x100` bytes of `entry_va` | 47 / 124 |

The last row is the most actionable lead. `hoodlum_relocation_map.json`'s entries are sorted by
`entry_va`, and functions from the same source file tend to land near each other in `.text` — so an
unmatched function sandwiched tightly between two `CStreaming::` entries is more likely to also be
`CStreaming` than an unmatched function with no named neighbor for a wide stretch. The strongest
per-class clusters at that tight (`< 0x100` byte) threshold:

| Nearby class | Unmatched functions this close |
|---|---|
| `CStreaming` | 5 |
| `CEntity` | 4 |
| `CCarCtrl` | 3 |
| `CTheScripts` | 3 |
| `CRunningScript` | 3 |

This is deliberately kept at 🟡-tier "lead" and not promoted into the catalogue: proximity in `.text` is
a real signal (compilers and linkers do generally keep a translation unit's functions contiguous) but it
is not an address match, and C0.3's whole finding about the automated dump was that plausible-looking
signals which aren't independently checked produce confident wrong names. The other 77 unmatched
functions have no close named neighbor at all and are, honestly, unidentified — small, thinly-called
functions in classes `gta-reversed` either hasn't reconstructed yet or reached through a path this
harvest doesn't see (a pure `RH_ScopedVMTInstall` virtual slot with no separate direct-call hook, for
instance, or a function `gta-reversed` calls without ever needing to hook it directly).

## 5. Open items

- ⏳ **The 124 unnamed functions.** Closing more of them needs either a newer/different `gta-reversed`
  revision, a second independent source (plugin-sdk ships an `address_translator.exe` +
  `translate_gtasa10us_hoodlum.bat` pairing that explicitly names this build's HOODLUM address set as a
  distinct target — strong corroboration that this problem is well-known — but its address database is
  an external download not present in this checkout, so it could not be used here), or manual
  disassembly against call-site context.
- 🟡 **The 47 near-neighbor leads** in §4 — worth checking first if closing more of the 124 becomes a
  priority, precisely because they're cheap to check (a handful of instructions each) and the payoff
  (confirm or reject) is high per function.
- ⏳ **Why the 6 zero-caller unmatched functions exist at all.** C27.2 §3 already showed a *named*
  function can have zero visible direct callers (vtable/pointer dispatch); whether the same applies here
  or whether these are dead code / rare edge-case handlers is unknown.

---

### Key takeaways

- 12 / 12 disassembled spot-checks confirm their recovered name against real instruction semantics, not
  just an address match — the strongest evidence tier this chapter uses.
- `CObjectPool::GetAt`'s disassembly, checked independently during development, both confirms the match
  and yields a 412-byte `CObject` pool stride as a free byproduct.
- The 124 unidentified functions are structurally unremarkable (small, lightly called) rather than
  outliers — nothing here suggests they're a special category, just gaps in the external source's
  coverage.
- Proximity-based leads (47 of 124, same class within `0x100` bytes) are recorded but deliberately not
  promoted to named entries, per this project's standing evidence-tier discipline.

**Next:** a class-catalogue chapter turning the 75 classes surfaced here into subsystem entries, starting
from the heaviest (`CTheScripts`, `CStreaming`, `CPathFind`) — see
[C27.0](C27-Function-Catalogue.md#why-this-chapter-and-why-now).
