# C25.1 — One `sscanf`, Two Record Variants

> **The one-sentence version:** every object record is parsed by a single 24-conversion `sscanf` format
> compiled into the executable, and the two variants — 17-field basic and 24-field breakable — are told
> apart by how many fields that one call matches, not by any value inside the record.

[Chapter 25 hub](C25-Object-Physics.md) · next: [C25.2 — The census](02-census-and-exceptions.md)

**Confidence:** ✅ Verified (the format string and its 24 fields, the two variants, the skip rules, the
count-not-value variant rule)

---

## 1. The format string is the schema

`object.dat` is loaded by a routine at VA `0x5B5360`, and the record itself is read by one `sscanf` against
a format string at VA `0x868DC8`:

```
%s %f %f %f %f %f %f %f %d %d %d %d %d %f %f %f %s %f %f %f %f %f %d %d
```

Counting the conversions gives the record's exact shape — `1 %s`, then `7 %f`, then `5 %d`, then `3 %f`,
then `1 %s`, then `5 %f`, then `2 %d`: **24 fields**. Laid against the header's field list, they are:

| # | Field | Type | Meaning |
|---:|---|---|---|
| 1 | `name` | `%s` | object model name |
| 2–8 | `mass`, `turn_mass`, `air_resistance`, `elasticity`, `percent_submerged`, `uproot_limit`, `cdamage_multiplier` | `%f` | the seven physics scalars |
| 9–13 | `cdamage_effect`, `special_cdr`, `camera_avoid`, `causes_explosion`, `fx_type` | `%d` | five integer codes |
| 14–16 | `fx_offset_x/y/z` | `%f` | FX system offset from the pivot |
| 17 | `fx_name` | `%s` | FX system name (`"none"` if `fx_type = 0`) |
| 18–22 | `break_smash_mult`, `break_vx`, `break_vy`, `break_vz`, `break_v_rand` | `%f` | break velocities |
| 23–24 | `break_gun_mode`, `break_sparks` | `%d` | gun-break mode, spark flag |

The field **order and types come from the executable**, not from reading the columns and guessing — the
same standard the rest of the encyclopedia holds itself to. A reordering of the data columns would be read
wrong silently; the format string is the authority that fixes them.

## 2. Basic vs breakable: 17 or 24 fields

A line that describes an ordinary object supplies the first **17** fields and ends at the FX name. A line
for a breakable object supplies all **24**, adding the seven break-info fields. Across the file:

```
basic (17 fields)      :  758 objects
breakable (24 fields)  :  235 objects
                          ─────
well-formed            :  993
```

Both variants are parsed by the *same* 24-conversion format. `sscanf` simply returns the number of fields it
managed to fill — 17 for a basic line, 24 for a breakable one — and the loader keeps that count. There is no
second format string and no schema flag in the data; the record's length is its type.

✅ *Verified:* one format, two variants, distinguished by field count.

## 3. The skip rules, read from the loader

Before it parses, the loader decides whether a line is data at all. Two character tests do the filtering,
visible at the top of the parse loop:

```asm
005B5476  cmp  al, 0x3B      ; ';'  → skip (comment)
005B5486  cmp  al, 0x2A      ; '*'  → skip (section marker)
```

Semicolon lines are the file's comments; asterisk lines are its section banners (`*********melee
weapons*************` and a lone `*` at the end). Both are skipped outright. Notably `#` is **not** in this
list — and `object.dat` contains exactly one `#` line, the pool-balls warning `# DONT USE THESE NUMBERS…`.
It is not skipped by the character test; instead it reaches the `sscanf`, which finds no leading float, fills
zero fields, and the loader drops the line because the count is too low. A comment survives by *failing to
parse* rather than by being recognised — untidy, but harmless, and worth recording because a reader who
assumes `#` is a comment character here would be wrong about *why* the line is ignored.

✅ *Verified:* `';'` and `'*'` lines are skipped by the loader; the stray `#` line is rejected by `sscanf`
returning too few fields.

## 4. The variant is the count, not the collision code

The header attaches the break fields to a value: they "need set if I ≥ 200", where `I` (collision-damage
effect) takes `200` = breakable and `202` = breakable-then-removed. That reads like the parser's rule, but
the parser has no such rule — it reads whatever fields are present. The data settles it by contradicting the
hint in both directions:

| Situation | Count | Example |
|---|---:|---|
| `I ≥ 200` **and** has break fields | 222 | most breakables |
| `I < 200` but **has** break fields | 13 | `petrolpump` (breaks by gunfire/explosion) |
| `I ≥ 200` but **no** break fields | 1 | `sec_keypad` (declared breakable, none supplied) |

If the collision code drove the parse, the second and third rows could not exist — a sub-200 object could
not carry break data, and a 200 object could not lack it. Both exist. So the break fields are attached
per-line at the author's discretion, and `sec_keypad` is simply a mistake the tolerant parser accepts
([C25.2](02-census-and-exceptions.md) lists it among the named defects). The "if I ≥ 200" note is guidance
to whoever edits the file, not a rule the engine enforces.

✅ *Verified:* the record variant is a property of the line's field count, independent of field `I`.

---

### Key takeaways

- ✅ One `sscanf` format at VA `0x868DC8` — `%s %f×7 %d×5 %f×3 %s %f×5 %d×2`, **24 fields** — is the whole
  schema; field order and types are read from the executable.
- ✅ **Basic = 17 fields** (`758`), **breakable = 24** (`235`); the two share the one format and are told
  apart by how many fields `sscanf` matched.
- ✅ The loader **skips** `';'` and `'*'` lines; the single `#` line is dropped because `sscanf` fails on it,
  not because `#` is recognised.
- ✅ The variant is decided by **field count, not the collision code** — 13 breakables have `I < 200`, and
  `sec_keypad` has `I ≥ 200` with no break fields.

**Continue:** [C25.2 — The census and its named exceptions](02-census-and-exceptions.md)
