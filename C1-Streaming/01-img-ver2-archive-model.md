# C1.1 — The IMG VER2 Archive Model

> **The one-sentence version:** an IMG archive is an 8-byte header, a table of 32-byte directory
> records, and 2048-byte-aligned payloads — a format designed so that locating any asset is one
> multiply, verified exhaustively against all six retail archives with zero malformed records.

[← Chapter 1 hub](C1-Streaming.md) · [Next: C1.2 — The CdStream layer →](02-cdstream-layer.md)

**Confidence:** ✅ Verified against retail data
**RE status:** Verified

---

## 1. The container

```c
struct ImgHeader {          // 8 bytes, at offset 0
    char     magic[4];      // 'VER2'
    uint32_t entryCount;
};

struct ImgEntry {           // 32 bytes, entryCount of them, immediately after the header
    uint32_t sectorOffset;  // +0x00  position, in 2048-byte sectors, from file start
    uint16_t sectorCount;   // +0x04  size, in sectors ("streaming size")
    uint16_t sizeInArchive; // +0x06  always 0 in the retail SA data set
    char     name[24];      // +0x08  NUL-padded ASCII, not guaranteed NUL-terminated at 24
};
```

Everything is little-endian. There is **no compression**, **no per-file header**, and **no index other
than the directory** — the whole format exists so that "where is asset N" is `entry[N].sectorOffset * 2048`
and nothing more.

The sector size is 2048 — the CD-ROM sector, inherited from the format's PlayStation-2 ancestry and
still the unit the runtime reads in ([C1.2](02-cdstream-layer.md) works exclusively in sectors).

---

## 2. Verification across the retail data set

Every archive in the shipped game was parsed end to end and every record checked.

| Archive | Entries | Size | Directory ends | First payload | Trailing slack |
|---|---:|---:|---:|---:|---:|
| `models/gta3.img` | 16,297 | 937,680,896 | 521,512 | 522,240 | **0** |
| `models/gta_int.img` | 2,484 | 150,024,192 | 79,496 | 79,872 | **0** |
| `models/player.img` | 542 | 66,738,176 | 17,352 | 18,432 | **0** |
| `models/cutscene.img` | 634 | 26,947,584 | 20,296 | 20,480 | **0** |
| `anim/anim.img` | 132 | 10,688,512 | 4,232 | 6,144 | **0** |
| `data/script/script.img` | 79 | 575,488 | 2,536 | 4,096 | **0** |

**Totals: 20,168 entries.** Checks applied to every record:

| Check | Result |
|---|---|
| Records whose `(sectorOffset + sectorCount) * 2048` exceeds the file | **0 / 20,168** |
| Records with non-ASCII bytes in `name` | **0 / 20,168** |
| Records with `sizeInArchive != 0` | **0 / 20,168** |
| Archives whose last payload ends exactly at EOF | **6 / 6** |

✅ *Verified.* The zero-slack result in all six archives is the strongest single piece of evidence for
the layout: if `sectorOffset` or `sectorCount` were being read at the wrong width or offset, the
computed maximum end would not land precisely on the file size six times out of six.

### 2.1 `sizeInArchive` is dead

The `u16` at `+0x06` is zero in all 20,168 retail records. In the earlier VER1 format (a separate
`.dir` file) the pair distinguished "size on disk" from "size when streamed"; in VER2 the distinction
collapsed and only `sectorCount` at `+0x04` is used.

🟡 *Reasoned:* it is a vestigial field, not a field this build reads. Treat it as reserved: write zero,
and do not assume the runtime ignores it without checking the consumer.

### 2.2 The directory-to-payload gap

The first payload never begins immediately after the directory — the gap is 728 bytes for `gta3.img`,
1,912 for `anim.img`, 1,560 for `script.img`. This is simply the directory being padded up to the next
2048-byte sector boundary, since payload offsets are expressed in sectors and the first one must land
on one.

Consequence for tooling: **the number of directory slots you can add without moving payload data is
`(2048 − (8 + 32·n) mod 2048) / 32`** — for `gta3.img`, 22 free slots. Rebuilders that add more must
relocate every payload.

---

## 3. Content census

By extension, across all six archives:

| Extension | Count | What it is |
|---|---:|---|
| `.dff` | 15,325 | RenderWare clumps — models |
| `.txd` | 3,974 | RenderWare texture dictionaries |
| `.ifp` | 285 | animation packages |
| `.col` | 251 | collision archives |
| `.ipl` | 190 | binary item-placement (streamed map sections) |
| `.scm` | 79 | compiled mission scripts |
| `.dat` | 64 | miscellaneous data |

✅ *Verified.* Two observations that matter later:

- **`.dff` and `.txd` are 95.7 % of all entries.** The streaming system is, statistically, a model and
  texture pump; everything else is a rounding error by count. Any performance work on streaming is
  work on those two paths.
- **`.ipl` inside the archives** is the *binary* streamed form, distinct from the text `.ipl` files in
  `data/maps/`. The world's placement data is itself streamed. This is the hook into the IPL/world
  chapters and the reason `CIplStore` sits alongside `CModelInfo` as a streamer client.

`models/player.img` inverts the usual ratio — 390 `.txd` to 152 `.dff` — because player clothing is
overwhelmingly texture variation over a small set of meshes.

---

## 4. Reading an entry

```python
import struct

SECTOR = 2048

def read_directory(path):
    with open(path, 'rb') as f:
        magic, count = struct.unpack('<4sI', f.read(8))
        assert magic == b'VER2', magic
        blob = f.read(32 * count)
    for i in range(count):
        off, sectors, _reserved = struct.unpack_from('<IHH', blob, 32 * i)
        name = blob[32*i + 8 : 32*i + 32].split(b'\0')[0].decode('ascii')
        yield name, off * SECTOR, sectors * SECTOR
```

The third field is deliberately named `_reserved` rather than `sizeInArchive`: §2.1 established it
carries no information in this data set, and a name that implies otherwise invites misuse.

### 4.1 The size a file *is* versus the size it *occupies*

`sectorCount * 2048` is the space reserved, not the asset's true length. A 3,100-byte `.dff` occupies
2 sectors and the trailing 996 bytes are undefined padding. Every consumer must therefore get the real
length from the payload's own format — the RenderWare section header for `.dff`/`.txd`, the `COL`
header for collision — and never from the directory.

⚠️ This is the most common source of corruption in third-party IMG tools: treating `sectorCount * 2048`
as the file length round-trips fine through a rebuild but silently appends garbage to every extracted
asset.

---

## 5. Limits

| Limit | Value | Origin |
|---|---|---|
| Files per archive | 4,294,967,295 | `u32` entry count — never the binding constraint |
| **Largest single file** | **128 MiB** | `u16 sectorCount` × 2048 |
| **Addressable archive size** | **32 GiB** | not this format — the 24-bit offset field in [C1.2](02-cdstream-layer.md) |

The interesting limit is the one the *format* does not impose: `sectorOffset` is a full `u32` here, so
the directory could describe an 8 TiB archive. It is the runtime's packed handle that narrows it to 24
bits. **The archive format is more permissive than the code that reads it** — a distinction that only
appears when both are documented against the same build.

---

### Key takeaways

- IMG VER2 = `'VER2'` + `u32 count` + `count × 32-byte` records + sector-aligned payloads. No
  compression, no per-file headers.
- The record is `{u32 sectorOffset, u16 sectorCount, u16 reserved, char name[24]}`; the reserved field
  is **zero in all 20,168 retail entries**.
- Verified exhaustively: **0 malformed records, 0 non-ASCII names, and all 6 archives end exactly on
  their last payload byte.**
- `.dff` + `.txd` are **95.7 %** of entries — the streamer is a model/texture pump.
- `sectorCount * 2048` is **reserved space, not file length**; real length comes from the payload's own
  header. Getting this wrong is the classic IMG-tool corruption bug.
- The format allows an 8 TiB archive; the **runtime's 24-bit offset field** cuts it to 32 GiB — the
  binding limit lives in the code, not the container.

**Continue:** [C1.2 — The CdStream layer](02-cdstream-layer.md) · [Chapter 1 hub](C1-Streaming.md)
