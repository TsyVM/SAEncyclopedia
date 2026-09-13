# C13.4 — Handling.cfg at runtime and vehicle modding

## How handling.cfg fields become runtime behavior

C13.1 decoded the `handling.cfg` grammar and C13.2 repaired the CARS section. This page traces how the parsed values flow into the physics engine (C42/C47) and what a modder can change by editing the file.

## The runtime flow: disk → CHandlingData → CVehicle

At startup, `CFileLoader::LoadHandling` parses each `handling.cfg` line into a `CHandlingData` record. The record is stored in the handling data table indexed by handling ID (the 3-char code in the final column of `vehicles.ide`).

When a vehicle is created from `CModelInfo` (`CModelInfo::CreateInstance`), its handling ID is looked up to retrieve the `CHandlingData` pointer. The vehicle stores this pointer for its entire lifetime. All physics calculations that need handling parameters read from this shared record.

**Key property:** the `CHandlingData` record is shared across all instances of the same vehicle model. Changing a value in the handling table affects all active instances simultaneously — there is no per-instance copy of handling data.

## Which fields map to which physics

| handling.cfg field | Physics effect (chapter) |
|---|---|
| `fMass` | Inertia in all force calculations | C42.2 |
| `fTurnMass` | Rotational inertia (yaw/pitch/roll response) | C42.2 |
| `fDragMult` | Air drag coefficient, fed by `ms_fTimeStep` | C42.2, C52 |
| `fBrakeDecel` | Braking force applied to all four wheel torques | C47.2 |
| `fTractionMult` | Combined traction scalar applied to slip force | C47.2 |
| `fSuspensionForceLevel` | Spring constant in Hooke's law integration | C47.1 |
| `fSuspensionDampingLevel` | Damper coefficient (velocity-proportional force) | C47.1 |
| `fSuspensionHighSpdCompress` | Additional compression damping at speed | C47.1 |
| `nMaxVelocity` | Hard cap on forward velocity applied in physics | C42.3 |

The full 92 unmapped bytes in the current schema (C13 confidence note) represent additional fields not yet traced — many are likely steering, engine torque curve, gear ratios, and wheel-offset positions.

## Editing handling.cfg: effects and limits

### Safe changes

- Adjusting `fMass`: affects crash behavior and how much vehicles bounce off each other. Higher mass = less bounce, more momentum. Does not affect visual.
- `fSuspensionForceLevel` / `fSuspensionDampingLevel`: changes how the car sits and bounces. Lower spring = more squat, lower damping = more oscillation.
- `fBrakeDecel` / `fTractionMult`: affects how quickly the car stops and how it handles on wet vs dry surfaces.
- `nMaxVelocity`: the top-speed cap. Raising it requires ensuring the engine torque is high enough to reach the new cap.

### Things that can break

- Setting `fMass` to zero or near-zero: physics becomes numerically unstable (division by mass in force calculations produces infinity).
- Inconsistent suspension geometry (high spring + low damping): oscillation that compounds each frame until the vehicle launches off the ground.
- Very high `fDragMult` without increased engine force: vehicle can only reach very low speeds.

## CarcOls and visual customization (C13.3 extended)

`carcols.dat` assigns color pairs to each vehicle model. At vehicle creation, the game picks a random entry from the model's color pair list. The `CVehicleModelInfo::ChooseVehicleColour` method selects the pair. This is why the same vehicle model appears in different colors in ambient traffic.

The `carmods.dat` file links vehicle models to available vehicle modification parts (tuning shop upgrades). Each modification part has its own model in `gta3.img` and its own handling contribution. `CCarModifierManager::ApplyMods` reads this table and attaches the correct sub-model to the vehicle. Some mods (spoilers, skirts) have no physics effect; others (nitrous, turbo) modify handling data fields directly.

## Adding new vehicles: the full workflow

To add a new vehicle model to SA:
1. **Model:** add `.dff` + `.txd` to `gta3.img`
2. **Collision:** add `.col` to `gta3.img`
3. **IDE:** add a row to `vehicles.ide` with the new model ID, model name, txd name, handling ID, and type
4. **Handling:** add a row to `handling.cfg` with a new 3-char handling ID
5. **CarcOls:** add color pairs to `carcols.dat`
6. **Spawning:** use a script opcode (`CREATE_CAR`) or place a spawn entry in an IPL to make it appear

The handling ID in `vehicles.ide` column 5 must exactly match the identifier in `handling.cfg` column 1 (the 3-char code). Mismatched IDs cause the vehicle to use the default handling data.

**Previous:** [C13.3 — Carcols and carmods](03-carcols-and-carmods.md)  
**Up:** [C13 — Vehicle Data](C13-Vehicle-Data.md)
