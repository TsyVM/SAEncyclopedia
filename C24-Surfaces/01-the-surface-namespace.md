# C24.1 — The 179-Surface Namespace

> **The one-sentence version:** there are exactly 179 surface types, and three independent parties agree on
> the list — `surfinfo.dat`, `surfaud.dat` and the executable's own hard-coded strings — with the two data
> files naming them in identical order.

[Chapter 24 hub](C24-Surfaces.md) · next: [C24.2 — the physics record](02-surfinfo-the-physics-record.md)

**Confidence:** ✅ Verified (the count, the three-way agreement, the shared order, the procedural subsets)

---

## 1. Two files, one count

The two large surface files each carry one record per surface type, and they carry the same number of them:

```
surfinfo.dat :  179 records
surfaud.dat  :  179 records
```

That alone is only suggestive — two files could hold 179 unrelated rows. The test is whether they hold the
*same* 179, in the *same* order. They do: stripping comments and reading the first token of each record
gives two name lists that are **identical element-for-element**, from `DEFAULT` and `TARMAC` at the top to
the last entry. Because the order matches, the engine can pair a surface's physics row with its audio row
by index alone — no name lookup between the two files is required, and none is done.

✅ *Verified:* `surfinfo.dat` and `surfaud.dat` define the identical 179 surface names in identical order.

## 2. The executable is the third witness

Cross-file agreement between two data files is strong; agreement with the *code* is stronger, because the
executable was built by a different toolchain from the data and has no reason to match unless both describe
the same enumeration. It matches exactly. Every one of the 179 names appears as a NUL-terminated string in
`gta_sa.exe`:

```
all 179 surfinfo names present in gta_sa.exe as strings  →  179 / 179
```

The strings cluster in `.rdata` (file offsets `0x45AE58 … 0x4634F4`), which is where a name→index lookup
table would keep them: the engine reads a surface name from `surfinfo.dat`, matches it against this
compiled list, and stores the resulting index. That is why the data files can be name-keyed while the
runtime is index-keyed — the translation is baked into the executable.

The three filenames themselves are also present, lower-cased, exactly as the loader opens them:
`surfinfo.dat`, `surface.dat` and `surfaud.dat`. (Unlike the audio and gridref paths of
[C20](../C20-Audio/C20-Audio.md)/[C22](../C22-Map-Zones/C22-Map-Zones.md), these are stored lower-case, so a
case-sensitive search finds them directly.)

✅ *Verified:* the surface-type count **179** is stated three times — `surfinfo.dat`, `surfaud.dat` and the
executable — by three independent parties.

## 3. Why this is the strongest kind of evidence

[C22.1 §3](../C22-Map-Zones/01-the-zone-tables.md) set out the principle: a value that reproduces
independently across subsystems cannot be at the wrong offset or the wrong count, because two unrelated
encodings would not share the same mistake. The surface namespace is that principle at its cleanest. There
is no arithmetic to get wrong here and no offset to misjudge — just three lists that either match or do not,
and they match to the entry. When the data and the executable agree on a population this specifically, the
population is real.

It also means the chapter needs **no external list**. A community wiki will happily supply surface-type
names and IDs, but the encyclopedia does not adopt them ([house rules](../handoff.md)); here it does not
have to, because the game's own three sources pin the namespace between them.

## 4. The procedural files key off the same names

Two smaller files describe what the world *grows* on a surface — `procobj.dat` (procedural objects: litter,
seaweed, and such) and `plants.dat` (procedural plants). Both are keyed by surface name, and both stay
inside the namespace:

| File | Distinct surfaces referenced | All defined in `surfinfo`? |
|---|---:|:--:|
| `procobj.dat` | 17 | ✅ |
| `plants.dat` | 42 | ✅ |

Every surface either file names is one of the 179. There is no reference to a surface that does not exist —
the same closed-namespace property the two big files show, extended to the procedural systems. (The reverse
does not hold, and should not: most of the 179 surfaces grow nothing, so they appear in neither procedural
file.)

✅ *Verified:* `procobj.dat` and `plants.dat` reference **only** surfaces defined in `surfinfo.dat`.

---

### Key takeaways

- ✅ There are **179 surface types**; `surfinfo.dat` and `surfaud.dat` define the identical list in
  **identical order**, so physics and audio pair by row index.
- ✅ **Three-way agreement:** all 179 names are hard-coded strings in `gta_sa.exe` (`179 / 179`), clustered
  in `.rdata` as the name→index lookup table; the three filenames are present lower-cased.
- 🧭 This is the project's strongest form of evidence — three independent encodings of one population that
  agree to the entry — and it means **no community list is needed**.
- ✅ `procobj.dat` (17) and `plants.dat` (42) reference **only** defined surfaces — a closed namespace.

**Continue:** [C24.2 — `surfinfo.dat`: the 37-field physics record](02-surfinfo-the-physics-record.md)
