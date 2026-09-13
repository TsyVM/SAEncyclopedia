# SA:MP Structure
*Source: `samp_structure.json`*
- **$schema:** samp_structure.v1
- **Generated:** tools/derive_samp.py

## Summary
| Field | Value |
| --- | --- |
| Client | samp.dll v0.3.7-R5 |
| Network | RakNet (UDP, reliability layer) |
| Sync packets | 7 |
| SAMP.img | 1673 entries (VER2) |
| SAMP.ide | 1447 objs (1363 added at 18631+) |
| Checks | 6/6 |

## Sync packet taxonomy
ID_PLAYER_SYNC, ID_VEHICLE_SYNC, ID_AIM_SYNC, ID_PASSENGER_SYNC, ID_TRAILER_SYNC, ID_UNOCCUPIED_SYNC, ID_SPECTATOR_SYNC

## Checks
| Check | Result |
| --- | --- |
| samp_version | PASS |
| uses_raknet | PASS |
| sync_packet_taxonomy | PASS |
| client_hooks | PASS |
| sampimg_is_ver2 | PASS |
| sampide_adds_high_ids | PASS |
