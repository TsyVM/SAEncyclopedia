# CVehicleRecording Table Layout
*Source: `vehicle_recording.json`*
- **$schema:** vehicle_recording.v1
- **Generated:** tools/derive_vehiclerecording.py

## Summary
| Field | Value |
| --- | --- |
| Recording slots | 475 |
| Recording stride | 16 |
| Recording base | 0x97d880 |
| Count global | 0x97f630 |
| Playback slots | 16 |
| Playback flags | 0x97d6f0 |
| Playback carptrs | 0x97d840 |
| Facts checked | 4 |
| Facts passed | 4 |

## Record fields
| Offset | Name |
| --- | --- |
| 0 | recording id |
| 4 | loaded data pointer |
| 12 | status byte |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x0045A060 | CVehicleRecording::HasRecordingFileBeenLoaded | recording table base 0x97D880, stride 0x10 (16 B), live count global @0x97F630 | ✅ true |
| 0x0045A0A0 | CVehicleRecording::RemoveRecordingFile | record layout: id@+0, dataPtr@+4, status byte@+0xC; array walked to 0x97F634 | ✅ true |
| 0x00459400 | CVehicleRecording::ShutDown | ShutDown frees each slot's dataPtr walking the 16-byte records | ✅ true |
| 0x004594C0 | CVehicleRecording::IsPlaybackGoingOnForCar | playback state: flag bytes @0x97D6F0, car-pointer array @0x97D840 (dword-indexed) | ✅ true |
