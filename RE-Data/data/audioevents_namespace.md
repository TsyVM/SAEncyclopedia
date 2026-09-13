# Audio-Event Namespace (recovered from AudioEvents.txt)
*Source: `audioevents_namespace.json`*
- **$schema:** audioevents_namespace.v1
- **Generated:** tools/derive_audioevents.py

## Correction to C20.5
C20.5 conflated 'not in the binary' (true - gta_sa.exe has bare integer immediates) with 'not in the shipped files' (FALSE). data/AudioEvents.txt names 7071 audio events; it names 447 of the 597 configured EventVol trim entries. Only the 149 low-block (<1000) engine-internal events remain unnamed by shipped files.

## Summary
| Field | Value |
| --- | --- |
| AudioEvents.txt names | 7071 |
| ID range | 1000..45400 |
| Configured EventVol entries | 597 |
| **Named by shipped file** | **447/597 (74.9%)** |
| Low-block unnamed (<1000) | 149 |
| Checks passed | 5/5 |

## Examples (named + trim)
| ID | Name | Trim |
| --- | --- | --- |
| 1002 | SOUND_CEILING_VENT_LAND | -8 |
| 1005 | SOUND_CLAXON_START | -2 |
| 1006 | SOUND_CLAXON_STOP | -2 |
| 1007 | SOUND_BLAST_DOOR_SLIDE_START | -2 |
| 1008 | SOUND_BLAST_DOOR_SLIDE_STOP | -2 |
| 1009 | SOUND_BONNET_DENT | -6 |
| 1010 | SOUND_BASKETBALL_BOUNCE | -6 |
| 1011 | SOUND_BASKETBALL_HIT_HOOP | -2 |
| 1012 | SOUND_BASKETBALL_SCORE | -5 |
| 1013 | SOUND_POOL_BREAK | -2 |
| 1014 | SOUND_POOL_HIT_WHITE | -6 |
| 1015 | SOUND_POOL_BALL_HIT_BALL | -6 |
| 1016 | SOUND_POOL_HIT_CUSHION | -10 |
| 1017 | SOUND_POOL_BALL_POT | -6 |
| 1018 | SOUND_POOL_CHALK_CUE | -6 |
| 1019 | SOUND_CRANE_ENTER | 0 |
| 1020 | SOUND_CRANE_MOVE_START | 14 |
| 1021 | SOUND_CRANE_MOVE_STOP | 2 |
| 1022 | SOUND_CRANE_EXIT | 9 |
| 1023 | SOUND_CRANE_SMASH_PORTACABIN | -2 |

## Checks
| Check | Result |
| --- | --- |
| audioevents_is_name_map | PASS |
| eventvol_matches_c20 | PASS |
| configured_entries_named | PASS |
| remainder_is_low_block | PASS |
| names_spot_check | PASS |
