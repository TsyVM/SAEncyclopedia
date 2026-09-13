# C5.2 — Sector Index Arithmetic

> **The one-sentence version:** three floating-point instructions turn a world coordinate into a sector
> index, and the two constants they use — `0.02` and `60.0` — *define* the world rather than describe
> it.

[← C5.1 — The two grids](01-the-two-grids.md) · [Chapter 5 hub](C5-CWorld.md) ·
[Next: C5.3 — Repeat sectors →](03-repeat-sectors.md)

**Confidence:** ✅ Verified

---

## 1. The instructions

From `0x004090A0`, applied identically to X and then to Y:

```
004090c2: fld   dword ptr [esp + 8]
004090c6: fmul  dword ptr [0x858b38]     ; × 0.02
004090cc: sub   esp, 8
004090d7: fadd  dword ptr [0x858b34]     ; + 60.0
004090dd: fstp  qword ptr [esp]
004090e0: call  0x8219f0                 ; floor
...
004090e7: fld   dword ptr [esp + 0x14]   ; the Y coordinate
004090eb: fmul  dword ptr [0x858b38]
004090f1: fadd  dword ptr [0x858b34]
004090f7: fstp  qword ptr [esp]
004090fa: call  0x8219f0
```

✅ *Verified.*

```
sectorIndex = floor(coord × 0.02 + 60.0)
```

## 2. The constants

Read directly from `.rdata`:

| Address | Exact value | Meaning |
|---|---|---|
| `0x00858B38` | `0.019999999552965164` | float `1/50` — the reciprocal of the cell size |
| `0x00858B34` | `60.0` | half the grid width — the origin shift |
| `0x00858B40` | `50.0` | the cell size itself, stored separately |

The multiply is by the **reciprocal**, not a divide — the standard 2004 optimisation, and the reason
`0.02` appears as a slightly-off float. `0.019999999552965164` is the nearest `float` to `1/50`; the
imprecision is inherent to the representation and has no practical effect at these magnitudes.

That `50.0` also exists as its own constant at `0x00858B40` means the engine keeps both forms — the
reciprocal for the hot index path, the true value for arithmetic that needs it.

## 3. The endpoints prove the geometry

```
−3000 × 0.02 + 60 =   0
+3000 × 0.02 + 60 = 120
```

**Exactly 0 and exactly 120.** The constants are not approximations fitted to a world of some other
size — they define a grid of 120 cells of 50 units spanning −3000 … +3000, and the endpoints land on
integers with no rounding.

This is why the geometry is ✅ rather than 🟡. A formula that produced `0.03` and `119.97` at the
extremes would suggest the real bounds lay elsewhere; landing precisely on the boundaries means the
constants and the intended extent are the same thing.

## 4. Flattening the 2-D index

The row stride appears as a literal multiply:

```
imul reg, reg, 0x78          ; × 120
```

found in **five** distinct functions: `0x004090A0`, `0x00409210`, `0x0041A820`, `0x00546670`,
`0x0054BA60`. ✅ Verified.

```
flatIndex = sectorY × 120 + sectorX
```

Five independent sites using the same stride is strong evidence that 120 is the row width rather than a
coincidence in one function. The compiler chose `imul` with an immediate rather than a `lea` chain
because 120 does not decompose as cheaply as, say, 20 did in
[C2.1 §2](../C2-CStreaming/01-the-id-space.md) — a small detail, but it means sector indexing is
recognisable by a *different* signature than streaming-record indexing.

## 5. The clamp

`0x00546670` contains:

```
005466f4: cmp  esi, 0x77           ; 119
005466f7: jl   0x5466fe
005466f9: mov  esi, 0x77           ; clamp to 119
```

preceded by a clamp of the same value against zero:

```
005466e8: xor  eax, eax
005466ea: test edx, edx
005466ec: setle al
005466ef: dec  eax
005466f0: and  eax, edx            ; max(edx, 0) without a branch
```

✅ Verified. The pair clamps a sector index into `0 … 119` — the branchless `max(x,0)` idiom followed by
an explicit `min(x,119)`.

This confirms 120 cells a third time, independently of both the `+60.0` arithmetic and the `× 120`
stride: the code's own idea of the maximum valid index is 119.

⏳ **Open:** whether every path clamps. This one does; the index computation in `0x004090A0` does not
visibly clamp before use. Unclamped paths are the likely mechanism behind out-of-bounds behaviour for
entities placed far outside the map.

## 6. Using this

To find the sector containing a point, in any tool:

```python
def sector_index(x, y):
    sx = math.floor(x * 0.02 + 60.0)
    sy = math.floor(y * 0.02 + 60.0)
    return sy * 120 + sx          # caller must clamp to 0..119 per axis
```

The `floor` matters: the engine calls it explicitly (`0x008219F0`) rather than truncating, so negative
coordinates round the correct way. Truncation toward zero would misplace everything with a negative
sector index — which, given the origin shift, is nothing inside the map, but is everything outside it.

---

### Key takeaways

- **`sectorIndex = floor(coord × 0.02 + 60.0)`**, verified from three instructions.
- Constants at `0x00858B38` (`1/50` as a float) and `0x00858B34` (`60.0`); `50.0` also stored at
  `0x00858B40`. The multiply is by the **reciprocal**.
- **The endpoints land exactly on 0 and 120** — the constants define the extent rather than approximate
  it.
- Row stride `× 120` via `imul reg, reg, 0x78` in **five** functions.
- A `0 … 119` clamp at `0x00546670` confirms the cell count a **third** independent time.
- ⏳ Not all paths visibly clamp — the likely source of far-from-map anomalies.
- Use `floor`, not truncation, or negative coordinates land in the wrong cell.

**Continue:** [C5.3 — Repeat sectors](03-repeat-sectors.md)
