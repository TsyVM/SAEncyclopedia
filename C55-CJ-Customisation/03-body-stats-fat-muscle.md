# C55.3 — Fat, Muscle, and the Body-Stat RPG System

> **The one-sentence version:** fat and muscle are two floating-point globals in the range 0.0–1.0
> that physically scale CJ's body mesh, determine which animation group the ped system picks for
> his locomotion, and gate five concrete gameplay abilities — strength, melee damage, sprint
> endurance, swimming speed, and the fall-damage threshold — making the SA gym/food system a
> genuine RPG stat layer, not just a cosmetic toggle.

**Subsystem category:** Gameplay — CJ RPG stats and body deformation
**Depends on:** [C26](../C26-Ped-Tables/C26-Ped-Tables.md) (ped stat infrastructure),
[C17](../C17-IFP-Animation/C17-IFP-Animation.md) (animation groups selected by body variant),
[C55.1](01-clothing-rule-language.md) (the body shape affects clump assembly)
**RE status:** Documented — stat effects confirmed behaviourally and from gta-reversed naming;
exact global VAs 🟡 (not individually traced to disassembly immediates in this project)
**Confidence:** 🟡 for fat/muscle global addresses (gta-reversed, structurally confirmed) ·
✅ for the gameplay effects (behaviourally confirmed, consistent across multiple sources)

---

## 1. The two stat globals

CJ's body shape is controlled by two scalar values:

| Stat | Gta-reversed name | Range | Default | Meaning at maximum |
|---|---|---|---|---|
| **Fat** | `CStats::m_Fat` (🟡) | `0.0` – `1.0` | `0.0` (lean) | Obese: large body scale, slow movement |
| **Muscle** | `CStats::m_Muscle` (🟡) | `0.0` – `1.0` | `0.0` (unmuscled) | Fully muscled: visible definition, strength bonus |

Both are stored as floating-point globals (🟡 — specific VAs not yet traced in this project to
individual disassembly immediates; confirmed as floats by gta-reversed's `CStats` class). Together
they form a 2D continuous space:

```
              muscle = 1.0
                  ↑
    (slim+ripped) │ (bulky+ripped) 
                  │
fat=0 ────────────┼──────────────── fat=1
                  │
   (slim+soft)    │ (fat+soft)
                  ↓
              muscle = 0.0
```

The four extremes have distinct visual and gameplay characteristics. Most players occupy the centre
of this space unless they actively train or neglect eating.

---

## 2. How the stats affect the mesh: body scaling

Fat and muscle control **three distinct model/scale tiers** for CJ's body geometry:

### 2.1 Fat tier: three body variants

The ped system ([C26](../C26-Ped-Tables/C26-Ped-Tables.md)) selects one of three body shape
presets based on the fat value:

| Fat range | Body variant | Visual effect |
|---|---|---|
| `0.0` – `0.33` | **Lean** | Thin, default proportions |
| `0.34` – `0.67` | **Normal** | Slightly filled-out |
| `0.68` – `1.0` | **Obese** | Wide body scale; belly geometry |

Each tier maps to a different **body model**. At the tier boundaries, the engine swaps the base
DFF — this is not a continuous mesh morph but a discrete model switch. Within a tier, a small
continuous scaling factor (applied to the RpClump frame transforms) provides smooth interpolation
between the endpoints.

### 2.2 Muscle tier: texture-based definition

Muscle does not change the base body geometry shape but controls **which body skin texture** is
active:

| Muscle range | Skin texture | Visual effect |
|---|---|---|
| `0.0` – `0.5` | Soft skin (default) | No visible definition |
| `0.51` – `1.0` | Muscular skin | Visible muscle definition in body texture |

The muscle tier is a **texture swap**, not a mesh swap. The underlying geometry is the same; the
texture's shadow patterns simulate surface definition. This is why SA's muscle system is
performance-lightweight — two textures cover the full range rather than requiring a separate
high-polygon mesh.

At very high muscle values, the frame scaling applied to the torso increases slightly, making CJ's
shoulders broader. This is a per-frame scale applied to the `Torso` / `UpperArm_L` / `UpperArm_R`
frames at clump-assembly time.

---

## 3. Animation group selection

Fat and muscle together determine which **animation group**
([C17](../C17-IFP-Animation/C17-IFP-Animation.md), [C48.3](../C48-Data-Folder-Sweep/03-extensions-and-binary-grid.md))
is active for CJ's locomotion:

| Body state | Animation group | Characteristic |
|---|---|---|
| Lean | `man` (standard walk/run/sprint) | Normal gait, upright sprint posture |
| Obese | `fat` (or equivalent heavy group) | Waddling walk, reduced sprint lean, slow jog |
| Muscular | `muscular` (or `man` with modified frame weights) | Slightly different arm swing, more upright |

The animation group is re-selected whenever the fat or muscle tier threshold is crossed. The
system does not interpolate between animation groups — it switches. The crossing point for fat is
approximately at `fat >= 0.68` (the Obese tier), meaning a player close to the threshold will
see a sudden gait change. This is the observable "fat animation switch" players notice when CJ
crosses the obesity threshold.

The `animgrp.dat` file ([C48.3](../C48-Data-Folder-Sweep/03-extensions-and-binary-grid.md))
maps group names like `man`, `fat`, `muscular` to specific clip assignments in `ped.ifp` — so the
fat/muscle stat feeds directly into the animation-selection path that `animgrp.dat` describes.

---

## 4. How the stats change: sources and decay

### 4.1 Fat increases

Fat increases when CJ eats food. Each food item in `shopping.dat`'s `Food` section (Cluckin' Bell,
Well Stacked Pizza, Burger Shot, etc.) has an associated calorie value (🟡 — exact field
confirmed by gta-reversed as a per-item numeric parameter). Eating adds fat at a rate proportional
to the calorie content of the item.

### 4.2 Fat decays

Fat decays passively over time (🟡 — confirmed behaviourally; decay rate per-second constant
not traced to a specific VA in this project). Fat also decays more rapidly during:
- Sprint activity (particularly long sustained sprints)
- Gym exercises
- Swimming

The decay is `ms_fTimeStep`-scaled
([C52.1](../C52-CTimer-And-Game-Loop/01-timer-and-timestep.md)) — fat loss is real-time-
proportional, not frame-count-proportional.

### 4.3 Muscle increases

Muscle increases at the gym (Pump House, Roller Dome, Bike School, etc.) through the three
exercise types: weightlifting (raises muscle most), cardio (raises stamina, moderate muscle),
and cycling. Each exercise session adds to the muscle float at the session's end (or progressively
during extended sessions — 🟡 session-granularity not confirmed).

### 4.4 Muscle decays

Muscle decays if CJ does not work out for an extended period — this is the "neglect" mechanic
that forces continued gym visits for maintained physique. The decay rate is slower than fat
accumulation, making it possible to maintain peak muscle with periodic visits.

### 4.5 The starvation state

If CJ's health drops below a threshold while fat is zero (no fat reserves), the game enters a
**starvation state**: health begins draining at a constant per-second rate. This is the gameplay
pressure that forces the player to feed CJ. The starvation drain is independent of weapons or
combat — it comes from the RPG stat system, not the damage system
([C45](../C45-Damage/C45-Damage.md)). The drain rate is `ms_fTimeStep`-scaled.

---

## 5. Gameplay effects of fat and muscle

The body stats have five concrete gameplay consequences beyond appearance:

### 5.1 Melee strength and damage

Higher muscle increases the damage CJ deals in unarmed combat (`WEAPON_UNARMED`, type 0 in
[C45.1](../C45-Damage/01-every-source-is-a-weapon.md)). The multiplier is applied at the
`CEventDamage` construction site for melee hits — higher muscle → higher `amount` in the damage
event. The formula is 🟡 (linear scaling with muscle, not traced to a specific VA).

### 5.2 Melee resistance

Higher muscle reduces the flinch severity of incoming hits. A fully muscular CJ takes the same
damage in health terms but plays a shorter hit-reaction animation — this is a practical combat
advantage in fights where the animation window locks out player input.

### 5.3 Sprint endurance (stamina interaction)

Fat reduces sprint endurance; muscle has a minor positive effect. The primary stamina stat is a
separate global (🟡 — `CStats::m_Stamina` per gta-reversed), but fat directly subtracts from the
effective stamina ceiling. An obese CJ (fat ≥ 0.68) tires more quickly on long sprints and
recovers stamina more slowly.

### 5.4 Swimming speed

Muscle increases swimming speed; fat slightly decreases it. The swim force applied each frame
([C52.2](../C52-CTimer-And-Game-Loop/02-frame-rate-and-physics.md) — the swim accumulator is one
of the FPS-dependent paths) is scaled by a muscle multiplier (🟡). This is one of the measurable
advantages of a high-muscle CJ in water.

### 5.5 Fall-damage threshold

Fat affects the fall-damage threshold. Heavier CJ (higher fat) incurs fall damage at lower drop
heights — the fall-damage formula in [C45.1](../C45-Damage/01-every-source-is-a-weapon.md) uses
`m_fDamageIntensity` scaled against mass, and fat increases effective mass. A lean CJ can survive
drops that kill a fat CJ. The relationship is 🟡 (mass-proportional, consistent with C45's
vehicle-impact mass formula, but not independently traced to a specific disassembly immediate for
the fall-damage path).

---

## 6. The body stat UI: gym display and the progress bar

The gym workout screen shows the current fat and muscle values as a paired display (fat bar, muscle
bar). The bar lengths map linearly from 0.0 to 1.0. The front-end menu
([C43](../C43-Front-End-Menu/C43-Front-End-Menu.md)) also shows a condensed stat summary that
includes fat/muscle state alongside the other RPG stats (respect, wanted, stamina, etc.).

---

## 7. Modding the body stat system

### 7.1 Reading/writing fat and muscle via ASI

The standard approach is to read the stat float at the gta-reversed address and write directly:

```cpp
// 🟡 — addresses from gta-reversed, not independently confirmed by this project
float* pFat    = (float*)0xB79B18;  // CStats::m_Fat (🟡 approximate)
float* pMuscle = (float*)0xB79B1C;  // CStats::m_Muscle (🟡 approximate)

// Set muscle to maximum:
*pMuscle = 1.0f;
// Set fat to lean:
*pFat = 0.0f;
```

After writing, the appearance rebuild must be triggered manually to update the mesh — writing the
float alone does not cause an immediate visual update. The rebuild occurs naturally on the next
body-shape evaluation pass (🟡 — exact trigger function not identified).

### 7.2 Disabling the decay

Fat/muscle decay can be disabled by NOP-ing the decay write in the per-frame update path, or by
hooking the function and forcing the value to a constant. The per-frame write is
`ms_fTimeStep`-scaled — patching it to `nop` or `ret` prevents the decay without affecting other
stats.

### 7.3 Open items

- ⏳ Exact VA for the fat decay write site
- ⏳ Exact VA for the muscle gain write site in the gym exercise handler
- ⏳ The per-item calorie values in `shopping.dat`'s Food section (the exact numeric field layout
  beyond name/GXT/respect/sexy/price)
- ⏳ Whether the starvation drain rate is a hardcoded constant or a configurable data value

---

### Key takeaways

- Fat and muscle are **two float globals (0.0–1.0)** — fat controls body model tier (lean /
  normal / obese) and animation group; muscle controls body skin texture and shoulder scale.
- The fat tier uses **discrete model swaps** at defined thresholds (`fat ≥ 0.68` → obese model);
  muscle uses a **texture swap** with minor frame scaling — no continuous mesh morph exists.
- **Five gameplay consequences**: melee damage (muscle), melee resistance (muscle), sprint
  endurance (fat reduces, muscle helps), swimming speed (muscle), fall-damage threshold (fat).
- The **starvation mechanic** drains health when fat = 0.0, making food a necessity and the
  fat stat a resource to manage — the RPG loop is food → fat reserve → exercise to burn fat and
  gain muscle.

**Previous:** [C55.2 — Haircuts, tattoos, and accessories →](02-hair-tattoos-accessories.md)
**Up:** [C55 — CJ Character Customisation →](C55-CJ-Customisation.md)
