# Chapter 19 — GXT: the Text & Localization Format

> **Goal of this chapter:** decode `american.gxt` — the container that holds every line of on-screen text
> in the game — with **the whole 738,256-byte file tiled to the byte**, the key hash **recovered and then
> confirmed 99.5 % against the mission scripts of [Chapter 18](../C18-SCM-Script/C18-SCM-Script.md)**, and
> a second language file that agrees with the first on everything it structurally had to.

**Subsystem category:** Localization / text
**Depends on:** [C1.1 — The IMG VER2 archive model](../C1-Streaming/01-img-ver2-archive-model.md) ·
[C18 — SCM script](../C18-SCM-Script/C18-SCM-Script.md) (keys are referenced from script)
**Ties:** [C1](../C1-Streaming/C1-Streaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C17](../C17-IFP-Animation/C17-IFP-Animation.md), [C18](../C18-SCM-Script/C18-SCM-Script.md), [C21](../C21-Particles/C21-Particles.md)
**RE status:** Verified
**Confidence:** ✅ Verified (container, key hash, cross-language) / ⏳ (one header word)

---

## Deep-dive pages

- [C19.1 — The container](01-the-container.md): a four-byte header, a 127-entry table directory, and a
  file that tiles exactly once 4-byte alignment is accounted for.
- [C19.2 — The key hash](02-the-key-hash.md): keys are a 4-byte hash, sorted for binary search — the hash
  is **CRC-32 without its final complement**, proven by hashing the script's own text keys.
- [C19.3 — Two languages](03-two-languages.md): `american` and `spanish` share an identical table and key
  structure, diverge exactly where translation requires, and expose the 8-bit codepage — while the
  executable names **three more** language files that the retail tree does not contain.

---

## 19.1 The result first

| Claim | Evidence |
|---|---|
| **127 text tables**, `MAIN` + 126 named | `TABL` size `1,524 = 127 × 12` ✅ |
| The file **tiles to the byte** | header + `TABL` + Σ(table blocks + 0–3 B align pad) = **738,256** ✅ |
| Keys are **CRC-32 (no final XOR) of the uppercased name** | **1,180 / 1,186** script text-keys hash into the set ✅ |
| Keys are **sorted for binary search** | strictly ascending in **all 127 tables** ✅ |
| Characters are **8-bit** | 16-bit decode yields garbage; 8-bit yields clean strings ✅ |
| `american` and `spanish` share structure | **127 / 127** table names identical ✅ |
| The engine names **five** language files | `AMERICAN` `SPANISH` `ITALIAN` `GERMAN` `FRENCH` in `.rdata` @ `0x870F70`; only two ship ✅ |

✅ **16,588 strings across 127 tables**, every one reachable by a key whose hash the game computes at load
time and looks up by binary search.

## 19.2 The structure

```
american.gxt
├── 04 00 08 00                     header: version word = 4, bitsPerChar = 8
├── 'TABL'  size=1524
│   └── 127 × { char name[8] ; uint32 offset }     directory; entry 0 = MAIN → 1536
├── @1536  MAIN table
│   ├── 'TKEY' size   →  5,428 × { uint32 tdatOffset ; uint32 keyHash }   (sorted by hash)
│   └── 'TDAT' size   →  8-bit, NUL-terminated strings with ~x~ tokens
├── @…     AMBULAE table
│   ├── char name[8]  =  "AMBULAE"          ← non-MAIN tables carry an 8-byte name
│   ├── 'TKEY' … 'TDAT' …
│   └── 0–3 bytes of 00 padding             ← every table starts on a 4-byte boundary
│   ⋮
└── (126 named tables)
```

**A table is found by name; a line is found by hash.** The directory is searched by the 8-byte table name
(`MAIN` for global text, one table per mission for mission text); inside a table, a line is found by the
4-byte hash of its key. Two lookup schemes because two access patterns: tables are opened rarely and by a
human-written name, lines are fetched constantly and by a compile-time key.

## 19.3 Why the keys are hashed and the tables are not

[Chapter 18](../C18-SCM-Script/C18-SCM-Script.md) showed the mission scripts print text by naming a GXT
key — `INTROB`, `HELP21`, `M_FAIL` — as a literal string in the bytecode. The game cannot afford a string
compare against 5,428 keys every time a subtitle appears, so it stores the **hash** of each key and
searches a sorted array. The table name, by contrast, is compared literally — there are only 127 of them
and they are opened once per mission.

✅ **The link is provable from the other side.** Hashing the 1,186 distinct text-keys the scripts actually
reference lands **1,180** of them exactly on a stored GXT key — a 99.5 % hit that both identifies the hash
function and confirms that C18's string arguments and C19's key table describe the same names. Detailed in
[C19.2](02-the-key-hash.md).

## 19.4 What the two languages show

`spanish.gxt` is the same 127 tables with the same names, and **106 of them carry a byte-identical set of
key hashes**; the 21 that differ do so by a handful of keys in both directions, and `spanish` carries three
`MAIN` lines that `american` does not. The text itself diverges where you would expect — **68 % of shared
`MAIN` lines are translated**, the rest are proper nouns and codes left alone — and the Spanish file spends
**12,771 high bytes** on accented characters against American's 7, which is the whole reason the format is
8-bit with a codepage rather than ASCII. See [C19.3](03-two-languages.md).

---

### Key takeaways

- The file is a **directory of 127 tables**; `TABL` size `1,524 = 127 × 12` fixes the count, and the whole
  738,256-byte file **tiles to the byte** once 4-byte table alignment is counted.
- **Tables are found by literal name, lines by a 4-byte key hash** searched in a sorted array — two schemes
  for two access patterns.
- The hash is **CRC-32 with the final complement omitted**, on the uppercased key — recovered from the data
  and confirmed **1,180 / 1,186** against the scripts of [C18](../C18-SCM-Script/C18-SCM-Script.md).
- **Characters are 8-bit**; `american` is essentially ASCII while `spanish` uses a Western codepage for
  accents — the 8-bit design choice made visible.
- `american` and `spanish` agree on **127/127 table names** and **106/127 key sets**, diverging only where
  translation requires — a field-by-field cross-check like the IDE/placement agreement in
  [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md).
- ⏳ The header's **first word (`4`)** is constant and its meaning is not derived; the second (`8`) is proven
  to be the character width.

**Next:** [C19.1 — The container](01-the-container.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C1](../C1-Streaming/C1-Streaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C17](../C17-IFP-Animation/C17-IFP-Animation.md)
- **Known bugs / gotchas:** 6 of 1186 keys miss; hash collisions theoretically possible.
- **Modding:** GXT is the localization-mod file; the CRC-32-no-final-XOR hash is the key spec.
- **Performance:** hash lookup per string draw; cheap.
