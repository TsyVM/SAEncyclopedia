# C14.4 — The `$` Gun-Record Columns

> **The one-sentence version:** the file's own column legend — not the generic header block above it —
> is the authoritative schema, it silently drops two documented fields, and the two record widths C14.2
> left open turn out to be **24 lettered fields + 1 flags byte**, optionally **+ 4 more**, closing exactly
> against every one of the 53 active rows with no residue; §8 now cross-checks the data-file findings
> against `gta_sa.exe` itself.

[← C14.3 — Stats and cross-checks](03-stats-and-crosschecks.md) · [Chapter 14 hub](C14-Peds-And-Weapons.md)

**Confidence:** ✅ Verified over all 58 `$` rows (53 active + 5 disabled), cross-checked against
`gta_sa.exe` (§8)

---

## 1. Two legends, not one

`weapon.dat` documents its own gun-record layout twice. Near the top of the file, a general block lists
letters `A`–`Z` against field names (`weaponType` … `breakoutTime`), plus `a`–`e` for flags and thrown-
weapon physics. Directly above the `$` data rows themselves sits a second, narrower legend:

```
#-------------------------------ranges------modelIds----slot-animgrp----ammo-dam----offset--------------skill---------------loop1--------loop2----break-flags-------old_shot_data-------
#	A				B			C	 D		E	F		I	J			K	 L		M     N     O		P  Q	R	S	 	T  U  V  	 W  X  Y  Z		a			b    c    d      e
```

This second legend **skips `G` and `H` entirely** — it goes straight from `F` (`modelId2`) to `I`
(`weaponslot`). The two blocks disagree, and the second one is the one sitting directly over the data.

✅ *Verified against every row*: no `$` record contains the two extra tokens `G`/`H` would require. The
general block's `G,H: int reloadSampleTime1, reloadSampleTime2` describes a field pair that **is not
present in any shipped `$` record**. 🟡 *Reasoned:* this is vestigial documentation — a field pair that
existed in an earlier version of the loader (or is still read as a compile-time constant elsewhere) and
was dropped from the text format without the top-of-file comment being updated. The same
kind of drift C14.1–C14.3 already found in this data set (orphaned `pedstats.dat` entries, the ped-table
comma) — a schema and its comment fell out of sync, and the byte-level evidence is what settles which one
shipped.

## 2. The field list that actually matches the bytes

Twenty-four lettered fields, in file order, with `G`/`H` removed:

| Letter | Field | Letter | Field | Letter | Field |
|---|---|---|---|---|---|
| A | weaponType (name) | J | assocGroupId | S | moveSpeed |
| B | eFireType | K | ammoClip | T | anim1 start |
| C | targetRange | L | damage | U | anim1 end |
| D | weaponRange | M | fireOffset x | V | anim1 fire |
| E | modelId1 | N | fireOffset y | W | anim2 start |
| F | modelId2 | O | fireOffset z | X | anim2 end |
| I | weaponslot | P | skillLevel | Y | anim2 fire |
| — | | Q | reqStatLevel | Z | breakoutTime |
| — | | R | accuracy | | |

That is **24 fields**, then one more — `a`, the hex flags byte (§4) — for **25 tokens**, and on ten rows
four further floats — `b,c,d,e` — for **29 tokens**.

```
53 active $ rows: 43 at 25 tokens + 10 at 29 tokens = 53.  No third width. No residue.
 5 disabled $ rows: all 25 tokens (none of the cut weapons used b–e).
```

✅ *Verified* — `Counter` over all 58 rows produces exactly `{25: 48, 29: 10}` and nothing else.

> ⚠️ **Correction to [C14.2](02-weapon-dat.md) §1.** That page's table reported the two widths as **26**
> and **30** fields. Counting tokens directly (sigil stripped, whitespace-split) gives **25** and **29** —
> one less in each case. The discrepancy traces to the generic header block's `G,H` pair, which C14.2
> took at face value; §1 above is why it doesn't apply. The record *count* (43/10, 53 total) was already
> right and is unchanged.

## 3. `b, c, d, e` — who gets the extra four fields, and why

The ten rows carrying `speed, radius, lifespan, spread` are exactly the weapons with real flight or
area-effect physics — thrown or launched projectiles, and continuous-area effects:

| Weapon | speed | radius | lifespan | spread |
|---|---:|---:|---:|---:|
| GRENADE | 0.25 | −1.0 | 800.0 | 1.0 |
| TEARGAS | 0.25 | −1.0 | 800.0 | 1.0 |
| MOLOTOV | 0.25 | −1.0 | **2000.0** | **5.0** |
| ROCKET | 0.25 | −1.0 | 800.0 | 1.0 |
| ROCKET_HS | 0.25 | −1.0 | 800.0 | 1.0 |
| FREEFALL_BOMB | 0.25 | −1.0 | 800.0 | 1.0 |
| SATCHEL_CHARGE | 0.25 | −1.0 | 800.0 | 1.0 |
| FTHROWER | 0.5 | 0.0075 | 1000.0 | 2.0 |
| SPRAYCAN | 0.05 | 0.5 | 1000.0 | 0.01 |
| EXTINGUISHER | 0.1 | 0.5 | 1000.0 | 0.01 |

Six of the seven grenade-family weapons share **identical** `(0.25, −1.0, 800.0, 1.0)` values — they are
copies of one template. **MOLOTOV is the exception**: its `lifespan` and `spread` are 2.5× and 5× the
rest of the family, while `speed` and `radius` stay templated. 🟡 *Reasoned:* a burning liquid spreading
over a wider area for longer than a bomb's blast radius is consistent with what a Molotov does in play,
but the field is asserted from the census, not from engine code. `radius = −1.0` on every explosive but
`FTHROWER`/`SPRAYCAN`/`EXTINGUISHER` (which use small positive radii under 1.0) reads as a sentinel for
"no fixed splash radius" on the six templated weapons — ⏳ open, not proven.

**Weapons without `b–e`** are everything else: hitscan guns (`INSTANT_HIT`), the two goggle items, the
camera, the detonator, and `MINIGUN` (whose `flame`-typed rounds do not carry independent projectile
physics the way `FTHROWER`'s stream does). 🟡 *Reasoned* correlation, not derived from engine code:
`eFireType` alone does not fully predict the split — `RLAUNCHER`/`RLAUNCHER_HS` are `PROJECTILE`-typed
like the grenade family but carry **no** `b–e` fields, so rockets fired from a launcher apparently reuse
some other (undiscovered) physics path. Recorded as ⏳, not rounded into the 🟡 claim above.

## 4. The flags byte, decoded against every populated row

`a` packs five 4-bit nibbles, most-significant first, against the bit table the file itself documents at
the top:

| Nibble | Bits | Meaning |
|---:|---|---|
| 1st | 1/2/4/8 | CANAIM / AIMWITHARM / 1STPERSON / ONLYFREEAIM |
| 2nd | 1/2 | MOVEAIM / MOVEFIRE |
| 3rd | 1/2/4/8 | THROW / HEAVY / CONTINUOUSFIRE / TWIN_PISTOL |
| 4th | 1/2/4/8 | RELOAD / CROUCHFIRE / RELOAD2START / LONG_RELOAD |
| 5th | 1/2/4 | SLOWSDWN / RANDSPEED / EXPANDS |

Decoded over the **full** 53-row census (a first pass over six sample rows undercounted this — see the
correction below), 27 of the 53 active rows carry a bit the documented table does not name. Two nibbles
account for all of them, and both resolve into a coherent pattern rather than noise:

**2nd nibble, bit `8`** (table defines only `1`/`2`, MOVEAIM/MOVEFIRE) is set on exactly four weapons —
`COUNTRYRIFLE`, `SNIPERRIFLE`, `RLAUNCHER`, `RLAUNCHER_HS` — and no others. 🟡 *Reasoned:* these are the
game's four long-range, zoom-aimed weapons; the bit reads as a scope/zoom-capable flag distinct from the
1st nibble's `1STPERSON`, though no name is asserted.

**2nd nibble, bit `4`** (same undocumented-bit family) splits the multi-tier hitscan guns into two
groups with no exceptions:

| nibble = 3 (MOVEAIM+MOVEFIRE only) | nibble = 7 (+ bit 4) |
|---|---|
| PISTOL (rows 1–3, `colt45`/`colt45pro`), SAWNOFF, MICRO_UZI, TEC9 | PISTOL row 4 (`colt_cop`), PISTOL_SILENCED, DESERT_EAGLE, SPAS12, MP5, AK47, M4 |

🟡 *Reasoned:* the right-hand column is exactly the heavier/two-handed sidearms and rifles, and the one-
handed `PISTOL`'s **fourth, `P=3` row** (§5) crosses over into it — the same `colt_cop` row that already
broke the `skillLevel` enum picks up this second undocumented bit too, reinforcing that it is a distinct
weapon variant, not a data-entry accident. `SHOTGUN` sits alone at nibble `2` (`MOVEFIRE` only, no
`MOVEAIM`) — the one weapon in the file with that exact value.

**5th nibble, bit `8`** (table defines only `1`/`2`/`4`, `SLOWSDWN`/`RANDSPEED`/`EXPANDS`) is set on
`COUNTRYRIFLE`, `FTHROWER`, `MINIGUN` and `EXTINGUISHER`. This is the more interesting of the two finds,
because it **cross-checks against the first one**: `SNIPERRIFLE`, `RLAUNCHER` and `RLAUNCHER_HS` — three
of the four scoped weapons identified above — carry the *documented* `EXPANDS` bit (`4`) here instead,
consistent with a zoomed scope view. `COUNTRYRIFLE` is the fourth scoped-nibble weapon but does **not**
get `EXPANDS` — it gets this second undocumented bit instead, joining three unrelated continuous-effect
weapons (`FTHROWER`, `MINIGUN`, `EXTINGUISHER`). 🟡 *Reasoned:* in the shipped game, `COUNTRYRIFLE` is the
*unscoped* hunting rifle (`SNIPERRIFLE` is its scoped sibling using the same model band, [C14.1](01-the-ped-table.md)-adjacent) —
so a weapon that sets the "long-range aim" bit (nibble 2) but explicitly **not** `EXPANDS` (nibble 5) is
exactly what "aims like a rifle, has no zoom scope" should look like. That two independently-parsed
nibbles agree on the same four-weapon split, in the direction the shipped game's own weapon behaviour
predicts, is the strongest evidence this page has for an unnamed bit's meaning — the same cross-
subsystem agreement standard the rest of the encyclopedia uses — while still stopping short of naming it,
since no string or table in this session's inputs confirms it.

⏳ **Open, now checked against the executable (§8.3):** neither undocumented bit (2nd-nibble `8`,
5th-nibble `8`) has a name in the file itself. With `gta_sa.exe` in hand this update searched it for
every one of the 15 documented flags-bit names (`CANAIM` … `EXPANDS`) as a standalone string and found
**none of them** — not the two undocumented bits, and not even the thirteen the file's own header
already names. The bit table lives only in `weapon.dat`'s comments; nothing in the compiled binary
carries these names as text. That rules out "the exe has a debug string that names them" as a way to
close this, so the tier stays 🟡/⏳ on the cross-subsystem reasoning above — but it is now a checked
absence, not an unexamined one.

> ⚠️ **Self-correction.** An earlier pass through this section decoded only six sample rows and reported
> the 5th-nibble bit as occurring on just `FTHROWER` and `MINIGUN`. Running the same decoder over all 53
> active rows (now persisted as `tools/derive_weapon.py`) found two more occurrences (`COUNTRYRIFLE`,
> `EXTINGUISHER`) and a second, more common undocumented bit in the 2nd nibble that the sample had missed
> entirely. Recorded here rather than silently fixed, per house style: **a six-row sample is not a
> census**, and the two extra hits changed the finding from "two anomalous weapons" to "a coherent
> four-weapon scope-flag pattern."

`CAMERA`'s lone `EXPANDS` bit and `DETONATOR`'s all-zero flags (`0`) are consistent with what those two
items do — a still camera has no aim/fire/reload behaviour to flag, and `EXPANDS` is the only bit that
plausibly applies to a zoom lens. 🟡 *Reasoned*, not confirmed against engine code.

## 5. `P` (skillLevel): the enum the file mostly obeys, and the one row that doesn't

The header declares `P: int skillLevel  0:POOR  1:STD  2:PRO`. Census over every active `$` name:

| Pattern | Weapons | Count |
|---|---|---:|
| Full triple, `P = 0,1,2` | PISTOL_SILENCED, DESERT_EAGLE, SHOTGUN, SAWNOFF, SPAS12, MICRO_UZI, TEC9, MP5, AK47, M4 | 10 names, 30 rows |
| Single row, `P = 1` only | GRENADE, TEARGAS, MOLOTOV, ROCKET, ROCKET_HS, FREEFALL_BOMB, COUNTRYRIFLE, SNIPERRIFLE, RLAUNCHER, RLAUNCHER_HS, FTHROWER, MINIGUN, SATCHEL_CHARGE, DETONATOR, SPRAYCAN, EXTINGUISHER, CAMERA, NIGHTVISION, INFRARED | 19 names, 19 rows |
| **Four rows, `P = 0,1,2,3`** | **PISTOL** | **1 name, 4 rows** |

✅ *Verified*: 10×3 + 19×1 + 4 = **53**, matching the active-row count exactly, no residue.

**`PISTOL` is the one named exception.** Its fourth row uses `P = 3` — a value the header's enum does not
define — paired with `Q = 5000` (every other weapon's highest `reqStat` is 999) and the model name
`colt_cop`, distinct from the `colt45` / `colt45pro` used by the other three rows. 🟡 *Reasoned:* `colt_cop`
is the police sidearm skin, and `P = 3` reads as a fourth, above-PRO tier reserved for law-enforcement
NPCs rather than a player-reachable skill rank — but the header names no such tier, so it is recorded as
an undocumented value, not renamed into the enum. This is the same discipline as C20's unnamed audio
event IDs and C25's out-of-range collision code: the enum says three, the data says four, and the fourth
is described rather than smoothed away.

## 6. Five rows cut, two different ways

Five `$` records are commented out — sigil present, line disabled with a leading `#`:

| Weapon | Cut row(s) | What survives active |
|---|---|---|
| COUNTRYRIFLE | `P=0` (POOR), `P=2` (PRO) | `P=1` (STD) only |
| SNIPERRIFLE | `P=0` (POOR), `P=2` (PRO) | `P=1` (STD) only |
| JETPACK | `P=1` (its only row) | **nothing — fully cut** |

✅ *Verified* over all 5 disabled rows; all are 25-token (base-width) records.

The two sniper-family weapons were **partially** cut — POOR and PRO variants disabled, STD left live —
which is why they show up above as single-row weapons indistinguishable, by row count alone, from
weapons that only ever had one skill tier. Only the presence of the commented rows in the file
distinguishes "designed with one tier" from "trimmed down to one tier." `JETPACK` was cut **completely**:
🟡 *Reasoned* the same fate as the `£`-sigil `SKATEBOARD` melee weapon C14.2 found — a fully-written,
fully-disabled record, content residue from something removed late rather than never finished. Between
the two chapters, that makes **three** completely-disabled weapons in this one file (`SKATEBOARD`,
`JETPACK`) plus **two** partially-disabled ones (`COUNTRYRIFLE`, `SNIPERRIFLE`) — five cuts of two
different shapes, both consistent with the late-cut pattern C13's handling entries and C14.2's
`pedstats.dat` orphans already established for this game.

## 7. `G`/`H` are not gone — they moved

The `%` aim-offset block (C14.2 §3) uses a **different** nine-letter legend, `A`–`I`, and its `G`/`H`
**are populated**:

```
% python   0.1  0.5  0.0  0.0  254  633  254  633
#  A       B    C    D    E    F    G    H    I
```

Here `F,G` is a reload-position pixel pair and `H,I` is its crouching counterpart — plainly not
`reloadSampleTime`. 🟡 *Reasoned:* the letter labels are **local to each record type**, reused with
unrelated meanings across the file's three sigils, not a single global column index. This resolves the
apparent contradiction in §1 the other direction: `G`/`H` were not deleted from the format, they were
never a `$`-record field to begin with once the second legend (not the generic top-of-file block) is
taken as authoritative — the top block's `G,H: reloadSampleTime1/2` describes a field that belongs to
neither the `$` nor the `%` schema as shipped, and its origin is unresolved. ⏳ **Open, checked
against the executable (§8.4):** a case-insensitive search of all of `gta_sa.exe` for
`reloadSampleTime` finds **zero occurrences** — no string anywhere in the binary carries that name.
That does not prove the field is unused (a compiled build carries no field-name strings for
plain numeric constants either way), but it rules out finding a hard-coded value *by name*; the
field pair remains vestigial documentation with an unresolved origin.

## 8. Cross-checked against `gta_sa.exe`

This update's input adds the game executable itself — `references/GTASA/gta_sa.exe`, MD5
`170b3a9108687b26da2d8901c6948a18`, matching the binary this whole encyclopedia is built against
(C0). Four checks below draw directly on it; `tools/derive_weapon.py` now re-runs all of them
(29/29 pass) when given both `weapon.dat` and the executable.

### 8.1 The compiled `WEAPONTYPE` name table

`.rdata` contains a 49-entry array of pointers to weapon-type name strings — found by locating the
`BRASSKNUCKLE` string's own pointer and walking outward while the neighbouring 4-byte values keep
resolving into `.rdata` (no offsets hard-coded; the walk is self-bounding). The array runs, in
order, `UNARMED` (index 0) through `ARMOUR` (index 48), and it is the same order for every entry
this page has already named from the data file:

```
0 UNARMED … 22 PISTOL … 31 M4 … 43 CAMERA … 46 PARACHUTE, 47 <no name>, 48 ARMOUR
```

✅ *Verified*: this is a genuine pointer array, not adjacent strings read as one — each entry is a
real 4-byte pointer whose target resolves to a printable, correctly-terminated string, and the
walk stops cleanly on both sides when a slot is no longer a valid `.rdata` address.

### 8.2 `JETPACK` and `SKATEBOARD` — the code-side confirmation

C14.4 §6 (data-file evidence only) found `JETPACK` and the melee `SKATEBOARD` fully commented out —
every row present but disabled, nothing live. Diffing every `$` and `£` name in `weapon.dat` (49
distinct names) against the 48 named slots in §8.1's table gives:

```
In weapon.dat but not in the compiled WEAPONTYPE table: JETPACK, SKATEBOARD
In the table but not in weapon.dat: ARMOUR (a pickup, not a weapon — expected)
```

✅ *Verified* — this is independent, code-side confirmation of the data-file finding: the two
weapons whose *only* rows are commented out are exactly the two names the compiled enum never
mentions at all. A weapon that shipped live, even in a single disabled-by-default tier, kept its
enum slot; `JETPACK` and `SKATEBOARD` did not, consistent with being cut before the type table was
finalised rather than merely disabled at the data layer.

### 8.3 The undocumented flag bits still have no name

§4 found two undocumented bits (2nd-nibble `8`, 5th-nibble `8`) with only 🟡 reasoned meanings. This
update searched the full executable for all 15 documented bit names as standalone strings
(word-bounded, so `THROW` inside the weapon name `FTHROWER` doesn't count) — see §4's revision
above. Result: **zero matches**, for the documented names as much as the undocumented ones. The
`WEAPONTYPE` table (§8.1) shows the engine *does* carry runtime string identity for weapon types;
the flags byte evidently does not — it's read straight into a bitfield with no name lookup, which
is itself a small architectural finding: not every enum in this engine gets a string table, and
flags is one of the ones that doesn't.

### 8.4 `reloadSampleTime` is not in the binary either

§7's `G`/`H` question — whether `reloadSampleTime1/2` is read from the executable as a hard-coded
constant — gets the same negative result: zero occurrences of the string anywhere in
`gta_sa.exe`, case-insensitive. See §7's revision above for what this does and doesn't prove.

### 8.5 A loose end this update found but did not chase

Index **47** of the `WEAPONTYPE` table — between `PARACHUTE` (46) and `ARMOUR` (48) — resolves to a
zero-length string: a real slot in the array with no name behind it. ⏳ *Open*: not investigated
further this session (no weapon in `weapon.dat` maps to it, and nothing else in this session's
inputs bears on it); noted here rather than silently skipped, per house style.

---

### Key takeaways

- The **column legend printed directly above the `$` rows** — not the generic top-of-file block — is
  the one the data obeys: it omits `G`/`H`, and no active or disabled `$` record contains them.
- The two record widths are **25 tokens** (24 lettered fields + flags) and **29** (+ `speed, radius,
  lifespan, spread`), not 26/30 as [C14.2](02-weapon-dat.md) reported — a one-token miscount now
  corrected there and here. **43 + 10 = 53**, no residue, no third width.
- The ten `b–e` rows are exactly the thrown/area-effect weapons; six of seven grenade-family rows are
  template copies, and **MOLOTOV is the one outlier** (2.5× lifespan, 5× spread).
- The five-nibble flags byte decodes cleanly against the file's own bit table, **except two undocumented
  bits** — 2nd-nibble `8` and 5th-nibble `8` — that together isolate the game's four scoped/long-range
  weapons and, on three of the four, line up with the documented `EXPANDS` bit; `COUNTRYRIFLE` (the
  unscoped rifle) is the one that diverges, in the direction the shipped weapon roster predicts.
- `skillLevel` obeys its declared `{0,1,2}` enum on 29 of 30 multi-tier rows — **`PISTOL`'s `colt_cop`
  row uses `P=3`**, an undocumented fourth tier paired with the highest `reqStat` in the file (5000).
- **Five `$` rows are disabled**, two shapes: `COUNTRYRIFLE`/`SNIPERRIFLE` each lost their POOR/PRO
  variants (STD ships alone), while `JETPACK` was cut **entirely** — the same complete-cut pattern as
  the `£`-sigil `SKATEBOARD` in C14.2.
- `G`/`H` are not missing from the game — they're a **different pair of fields, reused under the same
  letters, in the `%` aim-offset record**. Column letters are local to each sigil's schema, not global.
- **`gta_sa.exe` cross-check (§8, this update):** the compiled `WEAPONTYPE` name table (49 entries,
  `UNARMED`→`ARMOUR`, extracted from a self-bounding pointer-array walk in `.rdata`) independently
  confirms the two fully-cut weapons — `JETPACK` and `SKATEBOARD` are the only `weapon.dat` names
  absent from it. Neither the documented nor undocumented flags-bit names, nor `reloadSampleTime`,
  appear anywhere in the binary as strings — checked and absent, not merely unexamined.
- ⏳ One new loose end: `WEAPONTYPE` index 47, between `PARACHUTE` and `ARMOUR`, is a real slot with
  no name behind it.

**Continue:** [Chapter 14 hub](C14-Peds-And-Weapons.md)
