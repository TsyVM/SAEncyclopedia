# C41.4 — Task Modding and AI Extension

> **The one-sentence version:** the ped AI is modifiable at three depths — data files
> (`pedstats.dat`, `ped.dat`) for stats and acquaintance relationships, SCM task opcodes
> for forcing specific behaviors without exe patching, and ASI hooks into
> `CPedIntelligence::FindTaskToDoNow` or `CPedIntelligence::ProcessDecision` for new task
> types — and the wanted system's spawn table and pursuit thresholds are data-driven at the
> top but call fixed code at the bottom, so wanted-level escalation changes require both a
> data patch and a behavioral hook if non-stock behavior is desired.

**Subsystem category:** AI / gameplay — modding
**Depends on:** [C41.1](01-the-task-tree.md), [C41.2](02-the-wanted-system.md),
[C41.3](03-dispatch-and-roadblocks.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md),
[C26](../C26-Ped-Tables/C26-Ped-Tables.md), [C12](../C12-Path-Network/C12-Path-Network.md)
**RE status:** Documented — all three modding depths confirmed
**Confidence:** ✅ for the SCM task opcodes and data-file relationships · 🟡 for the exact ASI
hook patterns (function addresses confirmed; parameter layout for `ProcessDecision` 🟡)

---

## 1. Layer 1: data file modding (no exe patch required)

The ped AI's behavioral parameters live in three data files:

**`pedstats.dat`** ([C14.3](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)) controls per-ped-type
behavioral stats: flee distance, fight skill, aggression level, accuracy. These map directly to
`CPed::m_pedStats` fields and are read by `CPedIntelligence::ProcessDecision` when choosing
between fight and flee responses. Practical changes:

| Goal | Field to modify | Effect |
|------|----------------|--------|
| More aggressive gang peds | Increase `fight_ability`, decrease `flee_distance` | Peds engage at shorter range, fight longer |
| Coward civilians | Decrease `bravery`, increase `flee_distance` | Peds flee from further away |
| Police are better shots | Increase `shooting_rate`, `hit_level` for COP type | More accurate police fire |
| Invincible enemies | Set `die_health_threshold` = 0 | Peds don't auto-die from low health events |

**`ped.dat`** ([C26.1](../C26-Ped-Tables/C26-Ped-Tables.md)) controls acquaintance relationships —
which ped types attack each other, which flee from each other, which ignore each other. A mod that
adds a new faction with specific hostility relationships (e.g., a mercenary type that attacks all
players but is neutral toward gang peds) modifies this file's relationship matrix.

**`groups.dat` / gang ID data** controls which ped model IDs belong to which gang, feeding
`CTheGangs`' territory system ([C30.4](../C30-Gameplay-Managers/04-cgangwars.md)). New gang
factions require new model IDs in the model range plus entries in this file.

---

## 2. Layer 2: SCM task opcodes

The mission script system ([C18](../C18-SCM-Script/C18-SCM-Script.md)) exposes a large set of
task-forcing opcodes that let a script override any ped's current AI task. Each opcode maps to
one or more internal `CTask` subclass instantiations. The principal ones:

### 2.1 Locomotion tasks

```scm
TASK_GOTO_CHAR_ON_FOOT ped_handle target_handle distance
; → CTaskComplexGoToCharOnFoot: the ped walks/runs toward the target
;   until within distance, then idles

TASK_FLEE_FROM_CHAR_ON_FOOT ped_handle target_handle duration_ms flee_distance
; → CTaskComplexSmartFleeEntity: the ped flees the target for duration_ms
;   then returns to ambient AI

TASK_WANDER_RANGE ped_handle x y z radius
; → CTaskComplexWander with a spatial constraint
;   ped wanders within a sphere of radius around (x,y,z)

TASK_STAND_STILL ped_handle duration_ms
; → CTaskSimpleStandStill: ped stops all movement for duration_ms
```

### 2.2 Combat tasks

```scm
TASK_KILL_CHAR_ON_FOOT ped_handle target_handle
; → CTaskComplexKillCharOnFoot: the ped pursues and shoots the target
;   This is the standard gang-war / wanted-level combat task

TASK_SHOOT_AT_CHAR ped_handle target_handle duration burst_length
; → CTaskSimpleGunControl with explicit burst and duration parameters

TASK_AIM_GUN_AT_CHAR ped_handle target_handle duration_ms
; → CTaskSimpleGunAimAt: ped aims but does not fire automatically
```

### 2.3 Vehicle tasks

```scm
TASK_CAR_DRIVE_TO_COORD vehicle_handle x y z speed arrive_mode
; → CTaskComplexCarDriveMission: script-driven vehicle path to coordinates
;   Uses path network (C12) for routing

TASK_PARK_CAR vehicle_handle park_angle
; → CTaskComplexCarPark: parks the vehicle at the nearest valid spot
```

### 2.4 The task interrupt model

Forced tasks are pushed onto the **primary slot** of the ped's task tree (C41.1 §2). Event-response
tasks (explosions, collisions, ambient interrupts) continue to fire in the **event-response slot**
concurrently. When the forced task completes (`TASK_WANDER` radius reached; `TASK_KILL_CHAR` target
dead), the ped falls back to the ambient AI at the bottom of the task tree.

The `CLEAR_TASK` opcode removes the primary task immediately and the ped falls back in the same
frame. This is how mission scripts cleanly end a forced behavior.

---

## 3. Layer 3: ASI hooks for new task types

Deep AI modification requires hooking into the decision engine. The two useful hook points are:

### 3.1 `CPedIntelligence::FindTaskToDoNow` — the ambient task selector

`CPedIntelligence::FindTaskToDoNow` is called when a ped's current primary task has completed
and the AI must choose what to do next. It reads `pedstats`, `ped.dat`, the current threat level,
and the ped's zone to produce a new `CTask*`. Hooking it allows a mod to inject custom ambient
tasks — a ped that occasionally takes a phone call, a guard that patrols a custom path, a vendor
that plays an idle animation at a mod-defined location:

```cpp
typedef CTask* (__thiscall* FindTask_t)(CPedIntelligence*);
FindTask_t OriginalFindTask = (FindTask_t)0x601480;  // approx VA

CTask* __thiscall HookedFindTask(CPedIntelligence* pIntel) {
    CPed* pPed = pIntel->GetPed();

    // Is this ped one of our custom vendor peds?
    if (CustomPedRegistry::IsRegistered(pPed)) {
        int state = CustomPedRegistry::GetState(pPed);
        if (state == STATE_IDLE) {
            return new CTaskSimpleHoldPhone(pPed, DURATION_5000);
        }
        if (state == STATE_SELLING) {
            return new CTaskComplexPatrol(pPed, g_VendorPatrolPath);
        }
    }

    // Fall through to original for all other peds
    return OriginalFindTask(pIntel);
}
```

Custom `CTask` subclasses must implement the full `CTask` interface:

| Method | Required | Purpose |
|--------|----------|---------|
| `Clone()` | ✅ | Returns a copy — used for task copying during ped serialization |
| `ProcessPed(CPed*)` | ✅ | Per-frame update — returns `true` when task is complete |
| `MakeAbortable(CPed*)` | ✅ | Called when an event forces the task to abort; returns `true` if successful |
| `GetTaskType()` | ✅ | Returns the task type ID (must be unique; pick a value beyond SA's range) |
| `GetSubTask()` | Optional | Returns a sub-task if this is a composite task |

### 3.2 `CPedIntelligence::ProcessDecision` — the threat-response arbiter

`CPedIntelligence::ProcessDecision` fires when the ped receives a threat event (gunshot nearby,
explosion, player aggression). It decides between fight, flee, and ignore by reading the ped's
`pedstats` and the event's threat level. Hooking this allows custom threat responses:

```cpp
typedef void (__thiscall* ProcessDecision_t)(CPedIntelligence*, CEvent*);

void __thiscall HookedProcessDecision(CPedIntelligence* pIntel, CEvent* pEvent) {
    CPed* pPed = pIntel->GetPed();
    if (pEvent->GetEventType() == EVENT_PLAYER_DAMAGED) {
        if (IsMySpecialBoss(pPed)) {
            // Custom behavior: call for reinforcements instead of fleeing
            CallReinforcements(pPed);
            return;  // skip original response
        }
    }
    OriginalProcessDecision(pIntel, pEvent);
}
```

---

## 4. The wanted system's modding surface

### 4.1 Spawn table patching

The wanted system spawns specific ped types at each star level. The spawn table maps
`wanted_level → PEDTYPE + model_id_list`. Patching this table changes which models appear
at which level:

| Wanted level | Default spawn | Patch possibility |
|-------------|--------------|-----------------|
| 1 | Local COP type, on foot | Civilian informants with different model |
| 2 | COP type, police car | Unmarked cars, private security |
| 3 | SWAT by helicopter + ground | Tactical unit with custom weapons |
| 4 | FBI, black cars | Rival gang response instead |
| 5 | ARMY, Rhino tank | PMC with military-grade weapons |
| 6 | ARMY amplified | No change (⏳ spawn table format not fully traced) |

### 4.2 The pursuit intensity model

C41.2 established that pursuit intensity scales with `wanted_level`. The per-cop aggression is
ultimately governed by the assigned task — a cop at wanted level 5 receives `TASK_KILL_CHAR_ON_FOOT`
with aggressive `pedstats` parameters. The "relentlessness" (cop follows player into buildings,
through water, across map) comes from the routing in `CTaskComplexKillCharOnFoot` using
[C12](../C12-Path-Network/C12-Path-Network.md)'s path network for foot pursuit.

Modding the pursuit intensity without exe patching: increase the COP `fight_ability` and
`accuracy` in `pedstats.dat`. For a more dramatic change (cops never break pursuit, cops
teleport to roadblocks instantly), ASI hooks into `CWanted::Update` are required.

### 4.3 Path network coverage and wanted-level-free zones

C41.3 established that roadblocks are placed at path-network nodes. A modded area with no
path nodes is a **wanted-level-free zone** for vehicle roadblocks — police can pursue on foot
(using `CNavMesh` if available), but vehicle dispatch will not route there. This is one of the
most common AI artifacts in large map mods.

Remedy: author path nodes for new roads using path-node authoring tools (OAC, kednative's
path editor). The node format is [C12](../C12-Path-Network/C12-Path-Network.md)'s binary path
network — 28-byte records with position, link count, and link table. A path file added to the
streaming archive is automatically loaded and merged into the path graph at world load.

---

## 5. Performance considerations for AI mods

The ped AI task tree runs once per ped per game-logic frame. At high ped densities (interior
zones, large crowds), the task update can consume a significant fraction of the frame budget.
Performance guidance for AI mods:

| Pattern | Cost | Advice |
|---------|------|--------|
| Custom `ProcessPed` that scans all peds | O(n²) | Cache results; only scan local peds in your range |
| Hooking `FindTaskToDoNow` for every ped | O(peds × hook overhead) | Guard with early return for peds not in your registry |
| Custom task that calls `GetChar*` opcodes | Low (direct pointer access) | Preferred over SCM opcode round-trips |
| `CPedIntelligence::ProcessDecision` hook per event | O(events × peds) | Only for specific event types; don't intercept all events |

The `CPedIntelligence` hook overhead is negligible for a few registered peds but can become
visible at 50+ custom peds if the hook body has non-trivial logic. Keep the hook body to a
fast path (check a registry, return if not found, proceed to original) and move logic into
the custom task's own `ProcessPed` method.

---

### Key takeaways

- **Three modding depths**: data files (no exe patch — stats, acquaintances, group membership),
  SCM task opcodes (script-forced behaviors — all task families available), ASI hooks
  (`FindTaskToDoNow` for ambient tasks, `ProcessDecision` for threat responses, custom `CTask`
  subclasses for new behaviors).
- Custom `CTask` subclasses must implement `Clone`, `ProcessPed`, `MakeAbortable`, and
  `GetTaskType` — without `Clone` the game crashes during ped serialization.
- **Event-response slot** fires concurrently with forced primary tasks — a `TASK_KILL_CHAR`
  ped can still dive from explosions while pursuing; `CLEAR_TASK` removes only the primary slot.
- The wanted system's spawn table and `pedstats` are the non-exe-patch levers; pursuit
  intensity beyond that requires hooks into `CWanted::Update`.
- **Path network coverage is the wanted-level-free zone gate** — new roads without path nodes
  defeat vehicle dispatch; author path nodes for all new roads using C12's binary format.

**Previous:** [C41.3 — Dispatch and roadblocks](03-dispatch-and-roadblocks.md)
**Up:** [C41 — Ped AI, Tasks and Wanted hub](C41-Ped-AI-Tasks-Wanted.md)
