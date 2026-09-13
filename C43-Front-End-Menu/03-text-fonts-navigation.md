# C43.3 — Text, fonts and navigation

The screen table ([C43.1](01-the-screens-table.md)) and the item record ([C43.2](02-the-menu-item.md)) store
*keys and indices*, not text and not pixels-of-glyphs. This page follows those keys out to the systems that
resolve them — [C19](../C19-GXT-Text/C19-GXT-Text.md) for text, [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) for
fonts and sprites — and reads the navigation graph the indices form.

## Every string is a GXT key — the C19 tie

Neither a screen title nor a menu label is stored as display text. `m_szTitleName` and `m_szName` are both
**8-byte GXT keys** — `FEP_STA`, `FES_PLA`, `FEA_MUS`, `FET_PAU` — the exact key format
[C19](../C19-GXT-Text/C19-GXT-Text.md) decoded. At draw time the menu hashes each key
([C19](../C19-GXT-Text/C19-GXT-Text.md)'s CRC-32-without-final-XOR) and looks up the localized string in the
loaded GXT table, so the same `aScreens` table renders in any of the shipped languages without change. This
is why the menu is a *thin* system: it owns the layout and the flow, but not a single word of text — that all
lives in `american.gxt` / `spanish.gxt` ([C19](../C19-GXT-Text/C19-GXT-Text.md)). The `FE*` prefix
(`FrontEnd`) on every key is the naming convention that marks a string as menu text.

That the titles read back as valid GXT keys at stride `0xE2` ([C43.1](01-the-screens-table.md)) is
simultaneously the proof of the table layout *and* the proof of the C19 tie: two subsystems, the menu table
and the GXT key namespace, agree on the same 44+ strings.

## Fonts and sprites — the C23 tie

The glyphs those strings draw as come from [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md): the menu calls the same
`CFont` path, using the two proportional fonts in `fonts.txd` whose 210-byte width records
[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) decoded, and the `m_nAlign` field ([C43.2](02-the-menu-item.md))
feeds `CFont`'s justification. The menu's *pictures* — the radio-station icons on the audio screen, the
background art, the selector arrows — are sprites in the front-end texture dictionaries `fronten1.txd`,
`fronten2.txd`, `fronten3.txd` and (PC) `fronten_pc.txd`, loaded through the same TXD format
[C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)/[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)
document. The engine indexes them by a `FRONTEND_SPRITE_COUNT = 25` sprite enum split across the three base
TXDs (`FRONTEND1_START = 0`, `FRONTEND2_START = 13`, `FRONTEND3_START = 21`, PC extras from 23) — so the menu
art is 25 named sprites, the front-end analogue of [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)'s 69-sprite
`hud.txd`.

## The navigation graph

The two `eMenuScreen` fields — `m_nParentMenu` on the screen and `m_nTargetMenu` on each item — are the menu
tree, stored as indices into `aScreens` itself:

- **Forward** (enter): a `MENU_ACTION_MENU` item sets `m_nCurrentScreen = item.m_nTargetMenu`. From the main
  screen, the `Options` item targets the settings screen; a settings item targets `SCREEN_AUDIO_SETTINGS`
  (`FEH_AUD`), and so on.
- **Back**: `m_nParentMenu` is the return target. `FEP_STA` (Stats), `FEH_LOA` (Game) and `FEH_MAP` all carry
  parent `42` = `SCREEN_INITIAL`, so backing out of them returns to the pause root; the audio and display
  screens carry parent `33`, so they return to the settings screen that owns them.

Because both directions are just indices into the same 44-entry array, the entire menu flow is a graph
embedded in the table — no state machine file, no scripted transitions. Walking `m_nTargetMenu`/`m_nParentMenu`
reconstructs the full navigation map of the pause menu directly from `aScreens`.

## What this closes, and what's left

**Closes** the front-end as a structured system: it is a hardcoded 44-screen table
([C43.1](01-the-screens-table.md)) of 12-item records ([C43.2](02-the-menu-item.md)) whose text is
[C19](../C19-GXT-Text/C19-GXT-Text.md) GXT, whose glyphs and art are [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)
fonts/sprites, and whose flow is the parent/target indices above. It is the UI counterpart to the data-file
chapters — a table, decoded to the byte.

**Opens:**

- ⏳ The full `eMenuAction` dispatch (all 70 actions and what each does to game state).
- ⏳ The `CMenuManager` field map (`0x1B78` bytes of settings state — brightness, volumes, control bindings,
  the `m_abPrefs*` toggles).
- ⏳ The map screen's rendering (the pause-menu map is a distinct draw path over the radar tiles of
  [C22](../C22-Map-Zones/C22-Map-Zones.md)).
- ⏳ **Input** — how keyboard/pad events drive `m_nCurrentScreenItem`, a sibling system (the MW gap this
  chapter borders).

## Key takeaways

- Every menu string is a [C19](../C19-GXT-Text/C19-GXT-Text.md) GXT key (`FE*`), so the one `aScreens` table
  renders in every shipped language; the titles reading back as valid keys proves both the table layout and
  the C19 tie.
- Glyphs come from [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) fonts and the art from the `fronten*.txd`
  dictionaries (`FRONTEND_SPRITE_COUNT = 25` named sprites) — the menu owns layout and flow, not text or
  textures.
- Navigation is a graph embedded in the table: `m_nTargetMenu` (forward) and `m_nParentMenu` (back) are
  indices into `aScreens`, so the whole pause-menu flow reconstructs from the 44 records.

**Continue:** [back to the C43 hub →](C43-Front-End-Menu.md) · or [C19 — GXT Text](../C19-GXT-Text/C19-GXT-Text.md) · [C23 — Fonts & HUD](../C23-Fonts-HUD/C23-Fonts-HUD.md).
