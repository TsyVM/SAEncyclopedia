# C43.2 — The menu item

Each of the 44 screens ([C43.1](01-the-screens-table.md)) holds up to twelve options, and every option is a
`tMenuScreenItem` — an **18-byte** record that says what the option does, what it reads, where it goes, and
where it sits on screen. This page lays that record out and shows the stride is proven cold.

## The 18-byte record

`tMenuScreenItem` is `0x12` bytes, and its fields are the complete description of one menu line:

| Offset | Field | Type | Meaning |
|---|---|---|---|
| `+0x00` | `m_nActionType` | `eMenuAction` | what pressing enter *does* (one of 70 actions) |
| `+0x01` | `m_szName` | `char[8]` | the [C19](../C19-GXT-Text/C19-GXT-Text.md) GXT key for the label |
| `+0x09` | `m_nType` | `eMenuEntryType` | `TI_OPTION`, `TI_ENTER`, … (how it's drawn/behaves) |
| `+0x0A` | `m_nTargetMenu` | `eMenuScreen` | which screen to navigate to |
| `+0x0B` | `m_X` | `uint16` | screen X |
| `+0x0D` | `m_Y` | `uint16` | screen Y |
| `+0x0F` | `m_nAlign` | `eMenuAlign` | left / right / center |

The four things a menu line needs are all here: an **action** (toggle music volume, start a new game, go
back), a **label** to display, a **destination** for navigation, and a **position**. There is no code per
menu line — the behaviour is entirely data in this record, interpreted by one dispatch on `m_nActionType`.

## The stride, from the index expression

The item stride is not asserted from the header; it is visible in how the code addresses an item. From the
same instruction sequence that indexes the screen ([C43.1](01-the-screens-table.md)):

```
lea   edx, [eax + eax*8]      ; item * 9
movzx eax, [ecx + edx*2 + …]  ; ... * 2  ->  item * 18 = item * 0x12
```

The compiler multiplied the item index by 9 (`lea [eax+eax*8]`) and then by 2 (`edx*2`) — `9 × 2 = 18 = 0x12`
(`derive_menu.py`: `tmenuitem_stride_0x12`). Building `×0x12` as `×9 ×2` instead of an `imul` is a compiler
optimisation (two fast address arithmetic ops beat a multiply), and it is why the item stride shows up as a
`lea` rather than an `imul` — but it proves the same fact: one menu item is 18 bytes.

## The action type is the dispatch

`m_nActionType` (an `eMenuAction`, 70 values) is what turns a data row into behaviour. The
`ProcessUserInput` code switches on it — a `MENU_ACTION_STAT` line displays a statistic, `MENU_ACTION_MENU`
navigates, `MENU_ACTION_NEW_GAME` starts a game, `MENU_ACTION_BACK` returns to the parent, a
`MENU_ACTION_*` slider toggles a setting. Reading the first screen's items shows the pattern directly:

```
aScreens[0] "FEP_STA" (Stats), 9 items:
  { MENU_ACTION_STAT, "FES_PLA", TI_OPTION, SCREEN_NOP,   57,120, LEFT   }  Player
  { MENU_ACTION_STAT, "FES_MON", TI_OPTION, SCREEN_NOP,    0,  0, LEFT   }  Money
  ...
  { MENU_ACTION_BACK, "FEDS_TB", TI_ENTER,  SCREEN_INITIAL,320,380,CENTER }  Back
```

`derive_menu.py` confirms the first items' GXT labels cold (`item_names_are_gxt`): `aScreens[0].item[0]` is
`FES_PLA` (Player), `aScreens[1].item[0]` is `FES_NGA` (New Game). The `SCREEN_NOP` target on the stat lines
means "this option doesn't navigate"; the `Back` line targets `SCREEN_INITIAL`. So even the navigation graph
of the menu is right here in the item records — no separate menu-flow file.

## Positions are literal screen coordinates

`m_X`/`m_Y` are literal pixel positions (in the menu's virtual 640×480 space); a `0,0` position means "use
the automatic vertical layout" (the items stack under the title), while a non-zero position pins the item
(the `Back` button at `320,380`, bottom-centre). The `m_nAlign` field then left/right/centre-justifies the
text about that point. This is why most menu options flow automatically down the screen but a few — titles,
back buttons, the map legend — sit at fixed spots.

## Key takeaways

- `tMenuScreenItem` is **`0x12`** bytes: action, GXT label, entry type, **target screen**, X/Y, alignment —
  a complete menu line as pure data, 12 per screen.
- The stride is proven cold as `×9 ×2` (`lea [eax+eax*8]` then `×2`), a compiler-optimised `×0x12`.
- `m_nActionType` (one of 70 `eMenuAction`s) is the single dispatch that turns each row into behaviour;
  navigation targets and positions are fields in the record, so there is no separate menu-flow or layout file.

**Continue:** [C43.3 — Text, fonts and navigation →](03-text-fonts-navigation.md)
