# C25.4 — Modding object.dat

## The modder's entry point

`object.dat` is one of SA's most directly moddable physics files. Adding a new entry, changing an existing one, or converting a basic prop to breakable requires understanding the exact format (`sscanf` grammar from C25.1), the field semantics (C25.3), and the named traps this chapter's census uncovered (C25.2).

## The two record templates

### Basic record (17 fields)

```
name  mass  turn_mass  air_resistance  elasticity  percent_submerged  uproot_limit  cdamage_multiplier  cdamage_effect  special_cdr  camera_avoid  causes_explosion  fx_type  fx_offset_x  fx_offset_y  fx_offset_z  fx_name
```

Minimum viable example (a heavy static prop, no FX):
```
MY_PROP  99999.0  99999.0  1.0  0.1  50.0  9999.0  1.0  0  0  0  0  0  0.0  0.0  0.0  none
```

### Breakable record (24 fields)

Append 7 more fields after `fx_name`:
```
break_smash_mult  break_vx  break_vy  break_vz  break_v_rand  break_gun_mode  break_sparks
```

Minimum viable breakable example (a glass object, gun-shatters, no sparks):
```
MY_GLASS  50.0  50.0  0.9  0.3  50.0  100.0  1.0  200  0  0  0  0  0.0  0.0  0.0  none  2.0  0.0  3.0  0.5  0.3  1  0
```

Setting `cdamage_effect = 200` (breakable) and providing all 7 break fields is the correct combination. The C25.1 finding confirms the parser identifies record type by field count — providing 24 fields makes a breakable regardless of `cdamage_effect`, and providing 17 fields makes a basic regardless. The `cdamage_effect = 200` is a hint to the damage handler, not to the parser.

## The breakable-without-params trap

This is the most important trap in the file. If you add:
```
MY_OBJECT  ...  200  ...  (only 17 fields)
```
The parser will register this as a basic 17-field object. The `cdamage_effect = 200` will be stored, but the break fields (velocity, gun mode, sparks) will all be zero. When the object is destroyed by gameplay, the destruction handler will try to read the break velocity fields — which will be zero — and produce no visible destruction effect. The object will simply disappear.

**Fix:** always provide all 24 fields when `cdamage_effect = 200` or `202`. If you are uncertain whether an object should be breakable yet, use `cdamage_effect = 0` with 17 fields until you have designed the break parameters.

## Adding a new entry

### Step 1 — Choose a section

`object.dat` uses asterisk-prefixed section banners (e.g., `**** SECTION NAME ****`). These are cosmetic for organization. Add your entry under an appropriate existing section, or add a new section at the end of the file with an asterisk line.

### Step 2 — Match the model name

The `name` field in `object.dat` must match the model name defined in your IDE entry (the `.ide` file that defines the model). The two files use the same name space — the object name is the IDE's first field, and it is case-insensitive in the loader.

### Step 3 — Set the physical properties

| Goal | Recommended values |
|---|---|
| Immovable prop (lamp post, large static) | `mass=99999, turn_mass=99999, uproot_limit=9999` |
| Heavy movable (dumpster, barrel) | `mass=1000–5000, turn_mass=800–3000, uproot_limit=200–500` |
| Light movable (trash can, newspaper stand) | `mass=10–100, turn_mass=8–80, uproot_limit=50–200` |
| Breakable glass | `mass=50, elasticity=0.3, cdamage_effect=200, break_gun_mode=1` |
| Breakable wood | `mass=200, elasticity=0.1, cdamage_effect=200, break_gun_mode=2` |

### Step 4 — Set air_resistance

For most hard props: `1.0` (minimal drag). For lightweight objects (signs, panels): `0.95–0.98`. For nearly-no-drag props that should sail when hit: do not use air_resistance for gameplay tuning — use mass instead. Low air_resistance makes objects slow down quickly once moving, which usually looks unnatural for hard rigid bodies.

### Step 5 — Set buoyancy

For most land objects: `50.0` (activates at half-submerged). Objects that should sink immediately: `150.0` (requires more-than-fully-submerged — they will sink). Objects that float easily: `10.0–25.0`. Most modded objects should use the default `50.0`.

### Step 6 — Choose cdamage_effect

| Value | Meaning |
|---|---|
| 0 | No special damage behaviour — standard collision |
| 1 | `cdr_smash` — hit response with smash visual |
| 20 | Electric spark on hit |
| 200 | Breakable — shatters when health depletes; provide 24 fields |
| 202 | Breakable and disappears after shattering |

Values 200 and 202 require all 24 fields (the breakable record). All other values work with 17 fields (basic record).

### Step 7 — FX system

If your object has no FX: set `fx_type = 0`, `fx_offset_x/y/z = 0.0 0.0 0.0`, `fx_name = none`. This is correct for the vast majority of props.

To attach an FX system (explosion, tree hit effect): set `fx_type = 1` or `2`, then name a valid FX entry from `effects.fxp`. The 8 distinct FX names already in the file are the reference set. Mispelled or invalid FX names produce no error — the FX simply does not play.

## Changing an existing entry

To change a shipped object's physics, edit its line in `object.dat`. There is no lookup-by-line-number — the loader scans sequentially and uses the first match by name. Duplicate names in the file are resolved in favour of the first occurrence, so adding a second entry for the same model name earlier in the file overrides the original.

This is a useful pattern for mods that use custom `.dat` files via the IPL/IDE replacement flow: replace the entire `object.dat` in your mod's data directory with a modified copy. The loader reads the mod's copy first if it is in the mod's search path.

## Census-informed safe ranges

Based on the C25.2 census of 993 objects, these are the value ranges the shipped game uses:

| Field | Min shipped | Max shipped | Modal value |
|---|---|---|---|
| mass | 0.1 | 99999.0 | 99999.0 |
| turn_mass | 0.1 | 99999.0 | 99999.0 |
| air_resistance | 0.1 | 1.0 | 0.99 |
| elasticity | 0.0 | 0.9 | 0.05 |
| percent_submerged | 10.0 | 150.0 | 50.0 |
| uproot_limit | 0.0 | 99999.0 | varies by category |
| cdamage_multiplier | 0.1 | 4.0 | 1.0 |

Values outside these ranges are accepted by `sscanf` but may produce unexpected physics behaviour because the engine constants were tuned for the shipped range.

**Previous:** [C25.3 — Physics scalars and their runtime roles](03-physics-scalars-and-runtime.md)  
**Up:** [C25 — Object Physics](C25-Object-Physics.md)
