# C41.1 — The task tree

Every ped in San Andreas runs its behaviour through a `CTaskManager`. It is a small object — just **48
bytes** — but it is the root of the entire AI system: everything a ped does is a *task* slotted into it. This
page sizes the manager from its constructor and reads out the two slot arrays, whose counts are the whole
structure.

## The constructor writes the layout

`CTaskManager::CTaskManager(CPed*)` at `0x6816A0` is short enough to read whole, and it lays the object out
field by field:

```
0x6816A0: mov  eax, [esp + 4]        ; the CPed* argument
0x6816A6: mov  [edx + 0x2C], eax     ; m_pPed = ped            -> ped at +0x2C
0x6816A9: xor  ecx, ecx              ; ecx = 0 (nullptr)
0x6816AD: mov  [eax], ecx            ; m_aPrimaryTasks[0] = 0   \
0x6816AF: mov  [eax + 4], ecx        ; m_aPrimaryTasks[1] = 0    |  5 primary
0x6816B2: mov  [eax + 8], ecx        ; m_aPrimaryTasks[2] = 0    |  task pointers
0x6816B5: mov  [eax + 0xC], ecx      ; m_aPrimaryTasks[3] = 0    |  at +0x00
0x6816B8: mov  [eax + 0x10], ecx     ; m_aPrimaryTasks[4] = 0   /
0x6816BE: lea  edi, [edx + 0x14]     ; -> m_aSecondaryTasks at +0x14
0x6816C1: mov  ecx, 6                ; 6 of them
0x6816C6: rep stosd                  ; m_aSecondaryTasks[0..5] = 0
0x6816CB: ret  4
```

Three facts fall straight out:

- **`m_aPrimaryTasks`** is **5** `CTask*` pointers at offset **`+0x00`** — the five explicit dword clears.
- **`m_aSecondaryTasks`** is **6** `CTask*` pointers at offset **`+0x14`** — the `mov ecx, 6; rep stosd`.
- **`m_pPed`** is a back-pointer at offset **`+0x2C`**.

## The size tiles

The object closes with no residue:

```
m_aPrimaryTasks   : +0x00,  5 × 4 = 0x14
m_aSecondaryTasks : +0x14,  6 × 4 = 0x18   -> ends at 0x2C
m_pPed            : +0x2C,  4              -> ends at 0x30
sizeof(CTaskManager) = 0x30  (48 bytes)
```

`5×4 + 6×4 + 4 = 0x30`, and `+0x14` (secondary start) and `+0x2C` (ped) are exactly where the constructor
writes them. `derive_ai.py` asserts both the constructor pattern (`taskmanager_layout`) and the tiling
(`taskmanager_tiles`), and agrees with `gta-reversed`'s declaration `std::array<CTask*, 5>` +
`std::array<CTask*, 6>` + `CPed*`.

## The five primary slots are a priority ladder

The primary array is not five interchangeable tasks — its indices are a strict **priority order**, highest
first (`ePrimaryTask`):

| # | Slot | Preempts everything below it because… |
|--:|---|---|
| 0 | `TASK_PRIMARY_PHYSICAL_RESPONSE` | a ragdoll / impact must override intent instantly |
| 1 | `TASK_PRIMARY_EVENT_RESPONSE_TEMP` | a fleeting reaction (flinch, dive) to an event |
| 2 | `TASK_PRIMARY_EVENT_RESPONSE_NONTEMP` | a lasting reaction (flee, fight) to an event |
| 3 | `TASK_PRIMARY_PRIMARY` | the ped's assigned standing task (walk to, guard…) |
| 4 | `TASK_PRIMARY_DEFAULT` | the fallback idle when nothing else applies |

The manager runs the **highest-priority non-null** slot: a physical response at index 0 wins over an event
response, which wins over the standing task, which wins over idle. This is why a pedestrian shoved by a car
(`PHYSICAL_RESPONSE`) drops whatever it was doing, then — once the ragdoll ends and that slot clears — falls
back down the ladder to its event response or standing task. The ladder *is* the AI's arbitration.

## The six secondary slots run alongside

The secondary array holds tasks that execute **concurrently** with the active primary (`eSecondaryTask`):
`ATTACK`, `DUCK`, `SAY`, `FACIAL_COMPLEX`, `PARTIAL_ANIM`, `IK`. Their order carries one deliberate
dependency noted in the source — `DUCK` sits after `ATTACK` because attacking controls the ducking movement.
These are how a ped can walk (primary) while talking (`SAY`) and aiming (`ATTACK`) at once.

## What this gives the rest of the project

The handoff's recurring "unstructured `CTask*`/`CEvent*` tail" (C33/C37) is the hundreds of tiny task
subclasses. This page does not size those — it sizes their **owner**, and in doing so hands the C33
classifier a new signature: any function that indexes `[ped_task_manager + 0x14]` or clears five pointers +
`rep stosd 6` is task-manager code. That is the growth path C33.2 described — each newly structured class
lifts the classifier — applied to the AI cluster for the first time.

## Key takeaways

- `CTaskManager` is **`0x30` bytes**: `m_aPrimaryTasks[5]` @`+0x00`, `m_aSecondaryTasks[6]` @`+0x14`,
  `m_pPed` @`+0x2C` — read directly from the constructor and tiling with no residue.
- The five primary slots are a **priority ladder** (physical → event-temp → event-nontemp → primary →
  default); the manager runs the highest non-null one.
- The six secondary slots run **concurrently** with the primary (attack, duck, say, facial, partial-anim,
  IK), giving the AI cluster its first structured signature.

**Continue:** [C41.2 — The wanted system →](02-the-wanted-system.md)
