# C39.1 — TheCamera singleton and the three-cam array

The camera manager is a single global object. Every function that wants to know where the view is, or to
take control of it, reaches the same address: **`0xB6F028`**. `gta-reversed` names it `TheCamera`, a
`CCamera`; this page recovers its two load-bearing internals — the array of cameras it owns, and the byte
that selects the active one — directly from the indexing code, and lets the arithmetic settle the layout.

## Reading `GetActiveCam` out of the instructions

The most-executed thing the manager does is "give me the active camera." In C++ that is one line,
`m_aCams[m_nActiveCam]`. In the shipped machine code it is a three-instruction idiom that appears all over
the camera region:

```
0x50A6ED: movzx ecx, byte ptr [0xB6F081]   ; ecx = m_nActiveCam
0x50A6F4: imul  ecx, ecx, 0x238            ; * sizeof(CCam)
0x50A6FA: add   ecx, 0xB6F19C              ; + &m_aCams[0]
```

Three constants fall out of this, and each one is a structural fact:

- `0xB6F081` is **`m_nActiveCam`**, read as a single byte — the index of the live camera.
- `0x238` is **`sizeof(CCam)`** — the stride between camera objects.
- `0xB6F19C` is **`&m_aCams[0]`** — the base of the camera array.

`derive_camera.py` counts these across the camera code and requires the pattern to recur: the `imul …,0x238`
appears **27** times, the `add …,0xB6F19C` **5** times, the `m_nActiveCam` byte read **13** times. A stride
seen 27 ways and a base seen 5 ways in independent methods is the C28/C31 standard — no single disassembly
could be a coincidence at that multiplicity.

## The offsets close inside TheCamera

Both globals are fields of the one object at `0xB6F028`, so their offsets are just subtraction:

```
m_nActiveCam  offset = 0xB6F081 − 0xB6F028 = 0x59
m_aCams       offset = 0xB6F19C − 0xB6F028 = 0x174
```

The camera array is three objects (`gta-reversed`: `std::array<CCam, 3>`), so it spans:

```
0x174  +  3 × 0x238  =  0x174 + 0x6A8  =  0x81C
```

The array therefore occupies `+0x174 … +0x81C` inside the manager. That is comfortably within the object —
which is the first half of the size proof.

## Bracketing the manager's size — `CCamera` = `0xD78` (🟡)

`gta-reversed` asserts `VALIDATE_SIZE(CCamera, 0xD78)`. Rather than adopt that number outright, the exe
brackets it from both sides:

- **Lower bound.** The `m_aCams` array proven above ends at `+0x81C`, so `CCamera` is at least `0x81C` bytes.
- **Upper bound.** The next *named* global after `TheCamera` sits at `0xB6FE40`; nothing is referenced as a
  standalone global between `0xB6F028` and there, because everything in that span is one of `TheCamera`'s own
  fields (accessed as `base + displacement`). That puts a ceiling of `0xB6FE40 − 0xB6F028 = 0xE18` on the
  object.

So `0x81C ≤ sizeof(CCamera) ≤ 0xE18`, and `gta-reversed`'s `0xD78` sits inside that window. The size is
therefore recorded **🟡 reasoned (bracketed)** — the exe constrains it to a 1.5 KB window and the external
static-assert pins the exact value, but no single exe instruction proves `0xD78` outright the way `0x238`
is proven. `derive_camera.py` asserts the bracket (`ccamera_size_bracketed`), not the bare number.

## What the three cameras are for

The array size of three is small on purpose. Reading the accessors, the roles are:

| Slot | Role |
|---|---|
| the active cam (`m_aCams[m_nActiveCam]`) | the camera actually driving the view this frame |
| the previous cam | the camera being interpolated *from* during a mode switch (transitions blend `(active + 1) % 2`) |
| the debug / free cam | the developer fly-cam (`MODE_DEBUG` / `MODE_EDITOR`) |

The interpolation-from slot is why a change of camera mode glides rather than snapping: the manager keeps the
old `CCam` alive in the array and lerps between the two until the switch completes. The count itself is held
🟡 (it comes from `gta-reversed`'s array declaration; the active index is a byte and the transition logic
blends two of them), but the two live slots are visible directly in the transition code.

## Key takeaways

- `GetActiveCam` is a three-instruction idiom — `movzx [0xB6F081]; imul 0x238; add 0xB6F19C` — and it hands
  up all three structural constants: `m_nActiveCam` (+0x59), the `CCam` stride (`0x238`), and `m_aCams`
  (+0x174).
- The camera array spans `+0x174 … +0x81C` (three `0x238`-byte cameras); the manager as a whole is
  bracketed to `0x81C … 0xE18`, with `gta-reversed` fixing `CCamera` at `0xD78` (🟡).
- Three cameras cover active view, the interpolate-from camera during a cut, and the debug cam.

**Continue:** [C39.2 — The CCam object →](02-the-ccam-object.md)
