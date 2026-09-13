# Chapter 52 — CTimer and the Game Loop

> **Goal of this chapter:** decode San Andreas's time model from first principles. The engine runs on
> a **variable timestep**: a single float, `ms_fTimeStep` at `0xB7CB5C`, that every physics,
> animation, AI, and effect accumulator multiplies by before advancing. The 984 references to that
> address across `.text` are proof — it is the most-referenced single variable in the binary. This
> chapter proves the timer globals cold, characterises the 984-reference propagation across six
> subsystem categories, explains the clamp-at-2.0 anti-tunnel guard, and maps every failure mode
> that produces FPS-dependent behaviour: the places where the 984-reference discipline was not
> followed, and where frame-rate-sensitivity leaks in.

**Subsystem category:** Engine substrate — time, frame rate, and the physics integration clock
**Depends on:** [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) (disassembly method),
[C51](../C51-Executable-Lifecycle/C51-Executable-Lifecycle.md) (the frame loop that calls
`CTimer::Update` each tick)
**Ties:** [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) (vehicle integration multiplies
`ms_fTimeStep`), [C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md) (suspension/drag timestep
coupling), [C54](../C54-Limits-Reference/C54-Limits-Reference.md) (the max-clamp constant is
patchable for slow-motion)
**RE status:** Documented
**Confidence:** ✅ for all CTimer statics (`0xB7CB48`–`0xB7CB84`), the clamp max `0x858C14 = 2.0`,
the 984-reference count, the `bIsFrozen` mechanism, and the sub-step clamp addresses · 🟡 for the
full categorisation of the 984 references (representative sampling, not individually enumerated)
**Data artifact:** [`RE-Data/data/timer_structure.json`](../RE-Data/data/timer_structure.json)

---

## Deep-dive pages

- [C52.1 — CTimer globals and the timestep pipeline](01-timer-and-timestep.md): the six CTimer
  statics (`bIsFrozen`, `bGameStarted`, `game_FPS`, `ms_fTimeStepNonClipped`, `ms_fTimeStep`,
  `ms_nTimeInMilliseconds`) at `0xB7CB48`–`0xB7CB84`; `CTimer::Update` at `0x560D2E` — the
  clamp logic (frozen near-zero fallback, floor, `fcomp [0x858C14]` max = **2.0**); unit is
  **50ms per 1.0** (30fps gives `≈0.667`, 60fps gives `≈0.333`).
- [C52.2 — Frame rate and physics stability](02-frame-rate-and-physics.md): why the variable
  timestep is frame-rate-independent for the 984 correctly-scaled paths; what the 2.0 clamp
  protects (anti-tunnel guard for stall frames); the five FPS-dependent failure paths (ragdolls,
  nitro, swim force, some cutscene deltas) and why FPS-unlock mods fix them by keeping physics at
  30fps while rendering faster; the independent sub-step clamp layers inside vehicle/ped
  integrators.
- [C52.3 — FPS unlock and limit-breaking guide](03-fps-unlock-guide.md): which patches are safe
  (render rate unlocking), which need care (pool-size / density interaction), and what the FPS
  community has correctly and incorrectly attributed to `ms_fTimeStep`; a minimal safe patch set
  for unlocking above 30fps without physics explosion.
- [C52.4 — How ms_fTimeStep propagates through the engine](04-timestep-consumers.md): the
  **984-reference** picture, the canonical `fmul dword ptr [0xB7CB5C]` instruction pattern, the
  six consumer categories (entity velocities, timer countdowns, damage decay, animation blend,
  force accumulation, AI reaction windows) with representative call chains; identifying unscaled
  paths as the diagnostic for FPS-dependent bugs.
- [C52.5 — Pause states and bIsFrozen](05-pause-and-freeze-states.md): all writers of `bIsFrozen`
  (`0xB7CB48`) — loading screen, mission-fail fade, pause menu, script `FREEZE_ONSCREEN_TIMER`;
  the loading-screen state machine; implementing **slow-motion** by writing a fractional
  `ms_fTimeStep`; the `ms_nTimeInMilliseconds` accumulator and why SCM timers that test it are
  immune to slow-motion.

---

## 52.0 The result first

| Claim | Value | Tier | Evidence |
|---|---|:--:|---|
| Integration scalar | `ms_fTimeStep` @`0xB7CB5C` | ✅ | 984 `.text` reads; `CTimer::Update` writes |
| References to `ms_fTimeStep` | **984** | ✅ | full `.text` scan |
| Update function | `CTimer::Update` @`0x560D2E` | ✅ | writes all 6 statics |
| Max clamp | **2.0** @rdata `0x858C14` | ✅ | `fcomp dword ptr [0x858C14]` at `0x560DA2` |
| Frozen fallback value | `0x3C23D70A ≈ 0.008` | ✅ | push-immediate before frozen `fstp` |
| `bIsFrozen` | byte @`0xB7CB48` | ✅ | `cmp byte [0xB7CB48], 0` at 12+ sites |
| `ms_nTimeInMilliseconds` | uint32 @`0xB7CB84` | ✅ | `add dword [0xB7CB84], ecx` in `Update` |
| Unit | **50ms = 1.0** | ✅ | division constant in `Update`; freeze value confirms |
| Sub-step clamp | additional clamp per-integrator at `0x54D8EA`, `0x54D96E` | ✅ | direct write to `0xB7CB5C` from inside vehicle/ped code |

## 52.1 Why `ms_fTimeStep` is the whole story

Most game engines hide their time model behind several layers of abstraction. SA does not: there is
one variable, one update call per frame, and 984 uses. The design is explicit and visible to anyone
who runs a disassembly cross-reference query. That single `fmul dword ptr [0xB7CB5C]` pattern is
the proof test for whether any given rate is frame-rate-independent. If you find a rate accumulator
that lacks this multiply in its call chain, you have found an FPS-dependent code path — the
diagnosis and the fix are the same exercise.

The unit choice (50ms = 1.0) means that at the PS2's target of ~20fps, `ms_fTimeStep ≈ 1.0` — the
game ran at nominal speed on its primary development platform. The 30fps PC rate runs at `≈0.667`,
"faster" than nominal, which compensates for the fact that the physical constants (spring stiffness,
force magnitudes) were tuned at 20–25fps. The 60fps rate at `≈0.333` runs physics at half the
nominal rate — twice as many shorter steps accumulating the same result, which is correct for all
984 uses and wrong for the handful that are not.

## 52.2 The clamp, the frozen state, and the pause architecture

The two paths through `CTimer::Update`:

```
if bIsFrozen || !bGameStarted:
    ms_fTimeStep = 0.008   ← frozen near-zero; world renders but does not advance
else:
    raw = measure_elapsed() / 50.0
    raw = max(raw, floor)
    ms_fTimeStep = min(raw, 2.0)   ← clamped at 2.0 (= 100ms, anti-tunnel)
```

The **frozen state** is how the loading screen and cutscene transitions work: the renderer keeps
drawing frames, animations play, but every physics and gameplay accumulator sees ≈0.008 instead of
a real step — the world effectively stands still. The **anti-tunnel clamp** at 2.0 is the safety
valve: a one-second stall produces at most 100ms of physics advance, not a second of physics
(which would teleport objects through walls).

---

## Key takeaways

- `CTimer::Update` at `0x560D2E` is called once per `CGame::Process` tick and writes all six
  CTimer statics; `ms_fTimeStep` at `0xB7CB5C` is the **only integration variable** — 984 references.
- The clamp ceiling is **2.0 ≡ 100ms** (rdata `0x858C14`), proven from the `fcomp` write site at
  `0x560DA2`; the unit is 50ms = 1.0.
- The variable timestep is frame-rate-independent for all correctly-scaled paths. FPS-dependent bugs
  come from missing `fmul [0xB7CB5C]` multiplies in a handful of non-scaled code paths (ragdolls,
  nitro, swim force) — not from the timer system itself.
- `bIsFrozen` at `0xB7CB48` is the loading-screen / pause gate; writing a fractional value to
  `ms_fTimeStep` directly implements slow-motion for any correctly-scaled path.

**Continue:** [C52.1 — CTimer globals and the timestep pipeline →](01-timer-and-timestep.md)


## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `CTimer::Update` (0x560D2E)
- **References to `ms_fTimeStep`:** **984** across all `.text` subsystems
- **Known bugs / gotchas:** ragdoll impulse, nitro force, swim-force accumulator not timestep-scaled — FPS-dependent at 60fps+.
- **Modding:** write fractional `ms_fTimeStep` for slow-motion; patch rdata `0x858C14` to change the clamp ceiling.
- **Performance:** `CTimer::Update` is a trivial function; the 984 multiplies are already in the hot code path.
