# Class Catalogue
*Source: `class_catalogue.json`*
- **$schema:** class_catalogue.v1
- **Generated:** tools/derive_class_catalogue.py
- **Note:** Regroups C27's function_catalogue by class; every structural fact is asserted against gta_sa.exe's own disassembly.

## Summary
| Key | CTheScripts | CStreaming | CPathFind |
| --- | --- | --- | --- |
| Total named functions | 367 |  |  |
| Distinct classes | 74 |  |  |
| Heavy classes | 34 | 32 | 27 |
| Structural facts checked | 16 |  |  |
| Structural facts passed | 16 |  |  |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x00464C90 | CTheScripts::StartNewScript[indexed] | CRunningScript object stride = 0xE0 (224 B); pool base 0xa8b430 | ✅ true |
| 0x00464D20 | CTheScripts::GetScriptIndexFromPointer | pointer->index divides by the same 224 stride (reciprocal 0x92492493) | ✅ true |
| 0x00464BB0 | CTheScripts::WipeLocalVariableMemoryForMissionScript | mission local-var block = 0x400 dwords (4 KB) zeroed at 0xa48960 | ✅ true |
| 0x00464D50 | CTheScripts::IsPlayerOnAMission | OnAMission flag sits at 0xa49960 = 0xa48960 + 0x1000, right after the block | ✅ true |
| 0x0046A840 | CTheScripts::ClearAllVehicleModelsBlockedByScript | script-blocked vehicle-model list = 0x14 (20) dwords of -1 at 0xa448f0 | ✅ true |
| 0x0046A7C0 | CTheScripts::ClearAllSuppressedCarModels | script-suppressed car-model list = 0x28 (40) dwords of -1 at 0xa44940 | ✅ true |
| 0x0046ABC0 | CTheScripts::RemoveFromWaitingForScriptBrainArray | WaitingForScriptBrain array = 0x96 (150) entries x 8 B at 0xa476b0 | ✅ true |
| 0x00407A40 | CStreaming::ClearFlagForAll | streaming-info record stride 0x14, flags byte at base+6 (0x8e4cc6), array ends 0x9654b6 - independently reproduces C2 | ✅ true |
| 0x00407F80 | CStreaming::WeAreTryingToPhaseVehicleOut | loadState field at record+0x10 (0x8e4cd0), ==1 means loaded | ✅ true |
| 0x00407610 | CStreaming::AddImageToList | IMG archive list = 8 entries x 0x30 (48 B) at 0x8e48d8..0x8e4a58 | ✅ true |
| 0x004096D0 | CStreaming::RenderEntity | render list is a doubly-linked list with head sentinel at 0x8e48a0 | ✅ true |
| 0x0044D230 | CPathFind::These2NodesAreAdjacent | path node stride = 0x1c (28 B); link-count = low nibble of node+0x18 | ✅ true |
| 0x0044D310 | CPathFind::ThisNodeWillLeadIntoADeadEnd | same 0x1c node stride, reached via the link table at 0x96fa94 | ✅ true |
| 0x0044D480 | CPathFind::TestForPedTrafficLight | same 0x1c stride + link-nibble in TestForPedTrafficLight | ✅ true |
| 0x0044D790 | CPathFind::TestCrossesRoad | same 0x1c stride + link-nibble in TestCrossesRoad | ✅ true |
| 0x0044D0F0 | CPathFind::UnLoadPathFindData | six per-area node-resource pointer arrays freed by area*4 index | ✅ true |

## Classes

### CTheScripts
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00464BB0 | 0x01569F10 | WipeLocalVariableMemoryForMissionScript | 32 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00464C20 | 0x0156A2A0 | StartNewScript[last-idle] | 112 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00464C90 | 0x0156E210 | StartNewScript[indexed] | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00464D20 | 0x01562030 | GetScriptIndexFromPointer | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00464D40 | 0x01565610 | StartTestScript | 16 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00464D50 | 0x0156AA10 | IsPlayerOnAMission | 32 | 26 | 🔷 external-derived (unverified by disassembly) |
| 0x0046A7C0 | 0x0156F400 | ClearAllSuppressedCarModels | 32 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0046A840 | 0x01565EA0 | ClearAllVehicleModelsBlockedByScript | 32 | 1 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0046AB60 | 0x01565010 | AddToWaitingForScriptBrainArray | 112 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0046ABC0 | 0x01569CD0 | RemoveFromWaitingForScriptBrainArray | 80 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00470370 | 0x0156E5F0 | ReinitialiseSwitchStatementData | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004703C0 | 0x01561B80 | UseSwitchJumpTable | 208 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00470960 | 0x01562910 | InitialiseAllConnectLodObjects | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00470980 | 0x015661A0 | AddToListOfConnectedLodObjects | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00470A20 | 0x0156D9F0 | ScriptConnectLodsFunction | 112 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00474730 | 0x01562530 | InitialiseSpecialAnimGroupsAttachedToCharModels | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00474750 | 0x01565D70 | AddToListOfSpecialAnimGroupsAttachedToCharModels | 176 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004810C0 | 0x01561250 | GetUniqueScriptThingIndex | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004810E0 | 0x015647C0 | DrawScriptSpheres | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00481140 | 0x01569840 | AddToBuildingSwapArray | 192 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00481200 | 0x0156DF60 | AddToInvisibilitySwapArray | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004812D0 | 0x015652D0 | UndoEntityInvisibilitySettings | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00483B30 | 0x0156E050 | AddScriptSphere | 112 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00483BA0 | 0x01561960 | RemoveScriptSphere | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00486670 | 0x01565CA0 | CleanUpThisVehicle | 112 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x004866C0 | 0x0156B890 | CleanUpThisObject | 96 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00486720 | 0x0156F5A0 | ReadObjectNamesFromScript | 112 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00486780 | 0x01562660 | UpdateObjectIndices | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004867C0 | 0x01565E20 | ReadMultiScriptFileOffsetsFromScript | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00492F90 | 0x0156B9B0 | AddScriptEffectSystem | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00492FD0 | 0x0156F420 | RemoveScriptEffectSystem | 48 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00493000 | 0x015692C0 | AddScriptSearchLight | 352 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004934F0 | 0x01565770 | AttachSearchlightToSearchlightObject | 240 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004936C0 | 0x0156B9F0 | RemoveScriptCheckpoint | 112 | 2 | 🔷 external-derived (unverified by disassembly) |

### CStreaming
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00407610 | 0x01567B90 | AddImageToList | 3392 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004076A0 | 0x0156D7A0 | IsVeryBusy | 32 | 7 | 🔷 external-derived (unverified by disassembly) |
| 0x00407820 | 0x0156A100 | HasVehicleUpgradeLoaded | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00407A40 | 0x01566490 | ClearFlagForAll | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00407C00 | 0x0156C9E0 | GetDefaultCopModel | 80 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00407D10 | 0x015703D0 | DisableCopBikes | 16 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00407D20 | 0x01563A50 | GetDefaultMedicModel | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00407D40 | 0x0156CD70 | GetDefaultFiremanModel | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00407DD0 | 0x015704A0 | IsCarModelNeededInCurrentZone |  | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00407F00 | 0x015642C0 | HasSpecialCharLoaded | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00407F80 | 0x01567B60 | WeAreTryingToPhaseVehicleOut | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00408000 | 0x015662F0 | AddToLoadedVehiclesList | 192 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004080F0 | 0x0156C0A0 | RemoveCarModel | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004084B0 | 0x01563AB0 | Shutdown | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00408C70 | 0x0156C970 | RequestVehicleUpgrade | 64 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x004096D0 | 0x0156D750 | RenderEntity | 80 | 1 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00409710 | 0x01561520 | RemoveEntity | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00409A90 | 0x015664B0 | AreTexturesUsedByRequestedModels | 368 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0040A080 | 0x015663B0 | RequestFile | 176 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0040A120 | 0x01566460 | LoadInitialWeapons | 48 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0040A150 | 0x0156C100 | StreamCopModels | 448 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0040A400 | 0x01570230 | StreamFireEngineAndFireman | 416 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0040B080 | 0x015629D0 | RemoveCurrentZonesModels | 688 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0040B340 | 0x01566EF0 | RemoveLoadedZoneModel | 96 | 3 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0040B3A0 | 0x0156CA30 | RemoveInappropriatePedModels | 176 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0040B4F0 | 0x01563B30 | StreamOneNewCar | 432 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0040BA70 | 0x0156C5B0 | PossiblyStreamCarOutAfterCreation | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0040BDA0 | 0x015703E0 | StreamPedsIntoRandomSlots | 192 | 13 | 🔷 external-derived (unverified by disassembly) |
| 0x0040CF80 | 0x015645F0 | RemoveAllUnusedModels | 80 | 0 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0040D3D0 | 0x015638D0 | LoadInitialPeds | 32 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0040E3A0 | 0x015670A0 | LoadRequestedModels | 208 | 10 | 🔷 external-derived (unverified by disassembly) |
| 0x0040E560 | 0x0156D000 | ReInit | 272 | 1 | 🔷 external-derived (unverified by disassembly) |

### CPathFind
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0044D080 | 0x01560DC0 | Init | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D0F0 | 0x01563FB0 | UnLoadPathFindData | 192 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D230 | 0x01569710 | These2NodesAreAdjacent | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D310 | 0x01569F30 | ThisNodeWillLeadIntoADeadEnd | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D3B0 | 0x0156DE10 | TidyUpNodeSwitchesAfterMission | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D3E0 | 0x01561710 | ThisNodeHasToBeSwitchedOff | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D400 | 0x01565420 | UnMarkAllRoadNodesAsDontWander | 80 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D480 | 0x0156A400 | TestForPedTrafficLight | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044D790 | 0x01565AF0 | TestCrossesRoad | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044DCD0 | 0x0156DE80 | SetPathsNeededAtPosition | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044DD00 | 0x01561B10 | ReleaseRequestedNodes | 16 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0044DE00 | 0x01565690 | LoadSceneForPathNodes | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044DE80 | 0x0156AD40 | StartNewInterior | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044DED0 | 0x0156EC90 | AddInteriorLink | 96 | 21 | 🔷 external-derived (unverified by disassembly) |
| 0x0044E000 | 0x01560C20 | AddDynamicLinkBetween2Nodes_For1Node | 416 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0044E1A0 | 0x01564320 | RemoveInterior | 720 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044E4E0 | 0x0156A600 | ReInit | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004504F0 | 0x0156A990 | CountNeighboursToBeSwitchedOff | 112 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x00450560 | 0x0156ECF0 | MarkRoadNodeAsDontWander | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00450950 | 0x01562750 | Shutdown | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00450D70 | 0x0156A6B0 | MakeRequestForNodesToBeLoaded | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00451350 | 0x0156A180 | FindLinkBetweenNodes | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00452090 | 0x0156D150 | Find2NodesForCarCreation | 224 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00452160 | 0x015646A0 | SwitchOffNodeAndNeighbours | 272 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00452760 | 0x0156B7D0 | ComputeRoute | 192 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x004529F0 | 0x0156F750 | LoadPathFindData[FromStream] | 720 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00452F00 | 0x01565860 | SwitchPedRoadsOffInArea | 64 | 3 | 🔷 external-derived (unverified by disassembly) |

### CGarages
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004471B0 | 0x01566E00 | Shutdown | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00447680 | 0x0156DCF0 | GetGarageNumberByName | 80 | 9 | 🔷 external-derived (unverified by disassembly) |
| 0x004476D0 | 0x01561690 | ChangeGarageType | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004479A0 | 0x0156B1C0 | IsCarSprayable | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00447B80 | 0x0156CC60 | TriggerMessage | 224 | 17 | 🔷 external-derived (unverified by disassembly) |
| 0x00447C40 | 0x01560BD0 | SetTargetCarForMissionGarage | 80 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00447CB0 | 0x01563E70 | DeActivateGarage | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00447CD0 | 0x015677F0 | ActivateGarage | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00447D00 | 0x0156D2D0 | IsGarageOpen | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00447D30 | 0x01560F70 | IsGarageClosed | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00448B30 | 0x01563A10 | AllRespraysCloseOrOpen | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00449E60 | 0x01569150 | PlayerArrestedOrDied | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044A210 | 0x01567330 | CountCarsInHideoutGarage | 384 | 1 | 🔷 external-derived (unverified by disassembly) |

### CReplay
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0045B150 | 0x01564640 | DisableReplays | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045B160 | 0x01568BC0 | EnableReplays | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045C440 | 0x0156E610 | ShouldStandardCameraBeProcessed | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045CEA0 | 0x0156ED70 | DealWithNewPedPacket | 512 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0045D4B0 | 0x0156A8C0 | StreamAllNecessaryCarsAndPeds | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045D6C0 | 0x0156F180 | FindFirstFocusCoordinate | 160 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x0045DDE0 | 0x01565580 | IsThisPedUsedInRecording | 96 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x0045EBB0 | 0x0156DAB0 | RecordVehicleDeleted | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045EC20 | 0x01561870 | RecordPedDeleted | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045EF20 | 0x0156B3A0 | InitialisePedPoolConversionTable | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045F180 | 0x015689D0 | StoreStuffInMem | 496 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x004600F0 | 0x0156D320 | TriggerPlayback | 672 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004604A0 | 0x01562050 | PlayBackThisFrame | 96 | 1 | 🔷 external-derived (unverified by disassembly) |

### CShopping
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0049AB10 | 0x0156E330 | GetItemIndex | 32 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0049AB30 | 0x01561A10 | GetKey | 128 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0049ADA0 | 0x0156F0D0 | GetNameTag | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049ADE0 | 0x0156F080 | GetExtraInfo | 80 | 1 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0049AE30 | 0x015623F0 | RemoveLoadedShop | 64 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0049AE70 | 0x015659E0 | FindSection | 160 | 7 | 🔷 external-derived (unverified by disassembly) |
| 0x0049AF10 | 0x0156B920 | GetNextSection | 144 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0049B200 | 0x0156E2F0 | StoreClothesState | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049B240 | 0x01565320 | RestoreClothesState | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049B5E0 | 0x0156B770 | HasPlayerBought | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049B610 | 0x0156F3D0 | SetPlayerHasBought | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049B640 | 0x015624D0 | ShutdownForRestart | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049BEF0 | 0x0156B6F0 | UpdateStats | 128 | 2 | 🔷 external-derived (unverified by disassembly) |

### CIplStore
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004045B0 | 0x01567820 | AddIplsNeededAtPosn | 48 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x004045E0 | 0x0156CFF0 | ClearIplsNeededAtPosn | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00404780 | 0x015649E0 | GetNewIplEntityIndexArray | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004047B0 | 0x01569770 | GetIplEntityIndexArray | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00404C90 | 0x01563730 | IncludeEntity | 176 | 5 | 🔷 external-derived (unverified by disassembly) |
| 0x00405850 | 0x01565190 | RequestIplAndIgnore | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00405890 | 0x0156A140 | RemoveIplAndIgnore | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004058D0 | 0x0156DE30 | RemoveIplWhenFarAway | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00405AC0 | 0x0156C470 | AddIplSlot | 144 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00405B60 | 0x0156C5F0 | RemoveIplSlot | 160 | 0 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00405FA0 | 0x015610B0 | Shutdown | 256 | 1 | 🔷 external-derived (unverified by disassembly) |

### CRunningScript
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00463CA0 | 0x0156E2B0 | GetPointerToLocalVariable | 32 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00463CF0 | 0x0156E350 | ReadArrayInformation | 96 | 6 | 🔷 external-derived (unverified by disassembly) |
| 0x00464700 | 0x0156F340 | GetIndexOfGlobalVariable | 144 | 12 | 🔷 external-derived (unverified by disassembly) |
| 0x004648E0 | 0x015626B0 | Init | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00464BD0 | 0x0156DD60 | RemoveScriptFromList | 48 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00464C00 | 0x01561990 | AddScriptToList | 32 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00464D70 | 0x0156E7B0 | IsPedDead | 48 | 6 | 🔷 external-derived (unverified by disassembly) |
| 0x00464DA0 | 0x015620B0 | UpdatePC | 32 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0046AF50 | 0x0156B440 | ScriptTaskPickUpObject | 688 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00470150 | 0x01569460 | PlayAnimScriptCommand | 688 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00485A50 | 0x01564F00 | DoDeathArrestCheck | 208 | 1 | 🔷 external-derived (unverified by disassembly) |

### CVehicleRecording
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00459400 | 0x0156E770 | ShutDown | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004594C0 | 0x01561F20 | IsPlaybackGoingOnForCar | 224 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00459740 | 0x01565B90 | PausePlaybackRecordedCar | 272 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00459850 | 0x0156BA60 | UnpausePlaybackRecordedCar | 272 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00459F80 | 0x0156F110 | RegisterRecordingFile | 112 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045A020 | 0x0156D110 | RequestRecordingFile | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045A060 | 0x01560E90 | HasRecordingFileBeenLoaded | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045A0A0 | 0x015640F0 | RemoveRecordingFile | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045A160 | 0x015688F0 | RemoveAllRecordingsThatArentUsed | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045A8F0 | 0x0156F510 | Load | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0045A980 | 0x01565F10 | StartPlaybackRecordedCar | 432 | 3 | 🔷 external-derived (unverified by disassembly) |

### CColStore
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004103A0 | 0x015672B0 | AddCollisionNeededAtPosn | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004104E0 | 0x0156CF50 | SetCollisionRequired | 128 | 4 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x004106D0 | 0x015613A0 | LoadCol[2] | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00410730 | 0x01564E90 | RemoveCol | 112 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004107A0 | 0x01569960 | AddRef | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004107D0 | 0x0156DBE0 | RemoveRef | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00410800 | 0x01565240 | GetBoundingBox | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00410820 | 0x0156A260 | IncludeModelIndex | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00411140 | 0x01563290 | AddColSlot | 720 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00411330 | 0x01567210 | RemoveColSlot | 160 | 1 | 🔷 external-derived (unverified by disassembly) |

### CEntryExitManager
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043E410 | 0x01564090 | AddEntryExitToStack | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043ECF0 | 0x01560930 | SetAreaCodeForVisibleObjects | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043ED80 | 0x015678A0 | ResetAreaCodeForVisibleObjects | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043F0A0 | 0x015631C0 | PostEntryExitsCreation | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043F150 | 0x01566880 | GetPositionRelativeToOutsideWorld | 48 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0043F180 | 0x0156C6D0 | EnableBurglaryHouses | 112 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043F880 | 0x0156A6F0 | Init | 208 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043FD50 | 0x01560ED0 | DeleteOne | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00440B90 | 0x01563940 | Shutdown | 208 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00440C40 | 0x01567550 | ShutdownForRestart | 208 | 1 | 🔷 external-derived (unverified by disassembly) |

### CGameLogic
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00441210 | 0x01563910 | InitAtStartOfGame | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00441240 | 0x015672E0 | ForceDeathRestart | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00441390 | 0x0156CF30 | IsCoopGameGoingOn | 32 | 32 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00441560 | 0x01561270 | ClearSkip | 112 | 6 | 🔷 external-derived (unverified by disassembly) |
| 0x004415C0 | 0x01564860 | SkipCanBeActivated | 320 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004416C0 | 0x01569C30 | IsSkipWaitingForScriptToFadeIn | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00441D30 | 0x01560F90 | RestorePedsWeapons | 128 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00442020 | 0x0156FF70 | IsPlayerUse2PlayerControls | 64 | 5 | 🔷 external-derived (unverified by disassembly) |
| 0x004423C0 | 0x01563E90 | SetUpSkip | 192 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x004428B0 | 0x0156AA30 | DoWeaponStuffAtStartOf2PlayerGame | 272 | 2 | 🔷 external-derived (unverified by disassembly) |

### CConversations
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043A7B0 | 0x0156DFF0 | Clear | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043A810 | 0x01561CC0 | AwkwardSay | 48 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x0043A840 | 0x01565550 | StartSettingUpConversation | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043A870 | 0x0156A7C0 | SetUpConversationNode | 256 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0043A960 | 0x0156ADC0 | RemoveConversationForPed | 80 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043AAC0 | 0x01570080 | IsConversationGoingOn | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043B000 | 0x0156C500 | IsConversationAtNode | 176 | 1 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0043B0B0 | 0x01563250 | IsPlayerInPositionForConversation | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043C590 | 0x0156D7C0 | Update | 144 | 1 | 🔷 external-derived (unverified by disassembly) |

### CGangWars
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00443920 | 0x0156B150 | InitAtStartOfGame | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004439C0 | 0x0156EFF0 | DontCreateCivilians | 16 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00443AC0 | 0x01563620 | GangWarFightingGoingOn | 176 | 9 | 🔷 external-derived (unverified by disassembly) |
| 0x00443F80 | 0x015699C0 | CanPlayerStartAGangWarHere | 112 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00443FF0 | 0x0156DD40 | ClearSpecificZonesToTriggerGangWar | 32 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x004444B0 | 0x01564200 | ClearTheStreets | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00444530 | 0x01568E00 | TellGangMembersTo | 768 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00445C30 | 0x015609C0 | ReleasePedsInAttackWave | 528 | 8 | 🔷 external-derived (unverified by disassembly) |
| 0x00445F50 | 0x01565090 | StrengthenPlayerInfluenceInZone | 128 | 1 | 🔷 external-derived (unverified by disassembly) |

### CPickups
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00454A70 | 0x01564650 | Init | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00454AC0 | 0x01568BD0 | ModelForWeapon | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00454B40 | 0x0156DA60 | IsPickUpPickedUp | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004551C0 | 0x0156D870 | FindPickUpForThisObject | 64 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00455240 | 0x01561580 | AddToCollectedPickupsArray | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004552A0 | 0x01564BB0 | GetActualPickupIndex | 48 | 11 | 🔷 external-derived (unverified by disassembly) |
| 0x004554C0 | 0x0156A950 | PlayerCanPickUpThisWeaponTypeAtThisMoment | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00455680 | 0x0156B360 | UpdateMoneyPerDay | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00457380 | 0x0156DF10 | GenerateNewOne_WeaponType | 80 | 2 | 🔷 external-derived (unverified by disassembly) |

### CDarkel
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043CEB0 | 0x015647B0 | Init | 16 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043D1F0 | 0x015635C0 | FrenzyOnGoing | 32 | 8 | 🔷 external-derived (unverified by disassembly) |
| 0x0043D2F0 | 0x01567170 | ThisPedShouldBeKilledForFrenzy | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043D350 | 0x0156CED0 | ThisVehicleShouldBeKilledForFrenzy | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043D6A0 | 0x01561630 | ResetModelsKilledByPlayer | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043D6C0 | 0x01564C20 | QueryModelsKilledByPlayer | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0043D6E0 | 0x01569900 | FindTotalPedsKilledByPlayer | 96 | 1 | 🔷 external-derived (unverified by disassembly) |

### CScriptsForBrains
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0046A8C0 | 0x0156BB70 | Init | 64 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0046A900 | 0x0156FA50 | SwitchAllObjectBrainsWithThisID | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0046A9C0 | 0x015628A0 | AddNewStreamedScriptBrainForCodeUse | 112 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0046AA30 | 0x01569100 | GetIndexOfScriptBrainWithThisName | 80 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x0046AB20 | 0x01561650 | HasAttractorScriptBrainWithThisNameLoaded | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0046B390 | 0x0156A340 | StartAttractorScriptBrainWithThisName | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0046CED0 | 0x01562360 | StartOrRequestNewStreamedScriptBrainWithThisName | 48 | 1 | 🔷 external-derived (unverified by disassembly) |

### CStreamedScripts
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00470660 | 0x01562430 | Initialise | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004706A0 | 0x01565AD0 | ReInitialise | 32 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x004706C0 | 0x0156B7A0 | RegisterScript | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00470750 | 0x0156F450 | ReadStreamedScriptData | 192 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00470840 | 0x01565EC0 | LoadStreamedScript | 80 | 1 | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00470890 | 0x0156BFB0 | StartNewStreamedScript | 80 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x004708E0 | 0x0156F710 | RemoveStreamedScriptFromMemory | 32 | 2 | 🔷 external-derived (unverified by disassembly) |

### CCarAI
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0041BFA0 | 0x015612E0 | BackToCruisingIfNoWantedLevel | 192 | 11 | 🔷 external-derived (unverified by disassembly) |
| 0x0041C050 | 0x0156FB90 | CarHasReasonToStop | 32 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0041C4A0 | 0x01563CE0 | AddAmbulanceOccupants | 400 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0041C600 | 0x01568C10 | AddFiretruckOccupants | 384 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0041C760 | 0x01569A30 | TellOccupantsToLeaveCar | 352 | 9 | 🔷 external-derived (unverified by disassembly) |
| 0x0041CA50 | 0x0156FFB0 | FindPoliceBoatMissionForWantedLevel | 96 | 2 | 🔷 external-derived (unverified by disassembly) |

### COnscreenTimer
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0044CD50 | 0x0156E3E0 | AddClock | 464 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0044CDA0 | 0x01561C50 | AddCounter | 112 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0044CE60 | 0x01565730 | ClearClock | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044CE80 | 0x0156AD90 | ClearCounter | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0044CEB0 | 0x0156EC20 | SetCounterFlashWhenFirstDisplayed | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0044CEE0 | 0x01562130 | SetClockBeepCountdownSecs | 48 | 1 | 🔷 external-derived (unverified by disassembly) |

### CCollision
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00411E30 | 0x015678D0 | SortOutCollisionAfterLoad | 64 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x004152C0 | 0x01566DA0 | IsThisVehicleSittingOnMe | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004154A0 | 0x01567650 | CheckPeds | 144 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00415540 | 0x0156D280 | ResetMadeInvisibleObjects | 80 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00417730 | 0x01564C40 | TestLineOfSight | 592 | 2 | 🔷 external-derived (unverified by disassembly) |

### CStreamingInfo
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00407460 | 0x01567520 | Init | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00407480 | 0x015674C0 | AddToList | 96 | 12 | 🔷 external-derived (unverified by disassembly) |
| 0x004074E0 | 0x0156CE90 | RemoveFromList | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004075A0 | 0x01560E50 | GetCdPosnAndSize | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004075E0 | 0x01564070 | SetCdPosnAndSize | 32 | 1 | 🔷 external-derived (unverified by disassembly) |

### CStuckCarCheck
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00463B80 | 0x0156DC70 | RemoveCarFromCheck | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00463C00 | 0x015619B0 | HasCarBeenStuckForAWhile | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00463C40 | 0x01565300 | ClearStuckFlagForCar | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00463C70 | 0x0156A310 | IsCarInStuckCarArray | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00465970 | 0x015627A0 | AddCarToCheck | 208 | 2 | 🔷 external-derived (unverified by disassembly) |

### CTagManager
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0049CC50 | 0x0156AC60 | Init | 16 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CC60 | 0x0156E900 | ShutdownForRestart | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CC90 | 0x01562110 | AddTag | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CDE0 | 0x0156B330 | UpdateNumTagged | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CE10 | 0x0156F270 | SetupAtomic | 48 | 1 | 🔷 external-derived (unverified by disassembly) |

### CCarCtrl
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00421740 | 0x015616E0 | InitSequence | 48 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x00423ED0 | 0x01561040 | RemoveFromInterestingVehicleList | 48 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x0042C250 | 0x015636D0 | IsAnyoneParking | 96 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0042FBC0 | 0x01570100 | ScriptGenerateOneEmergencyServicesCar | 144 | 1 | 🔷 external-derived (unverified by disassembly) |

### CCollisionData
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0040F070 | 0x0156FAE0 | RemoveCollisionVolumes | 176 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0040F120 | 0x01562C90 | Copy | 1328 | 0 | 🔷 external-derived (unverified by disassembly) |
| 0x0040F6C0 | 0x01568990 | SetLinkPtr | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0040F6E0 | 0x0156D850 | GetLinkPtr | 32 | 3 | 🔷 external-derived (unverified by disassembly) |

### CPed
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043ABA0 | 0x0156CAE0 | PedIsReadyForConversation | 192 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x004590F0 | 0x01564140 | CreateDeadPedMoney | 192 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00459180 | 0x01568940 | CreateDeadPedPickupCoors | 80 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00481090 | 0x0156D8B0 | SetStayInSamePlace | 32 | 2 | 🔷 external-derived (unverified by disassembly) |

### CPickup
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00454BE0 | 0x0156DB40 | ExtractAmmoFromPickup | 160 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00455500 | 0x0156EC50 | FindTextIndexForString | 64 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00455540 | 0x01565750 | FindStringForTextIndex | 32 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x004588B0 | 0x01562550 | ProcessGunShot | 224 | 1 | 🔷 external-derived (unverified by disassembly) |

### CRestart
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00460630 | 0x015658E0 | Initialise | 256 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x004607D0 | 0x0156B8F0 | OverrideNextRestart | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00460810 | 0x0156F610 | SetRespawnPointForDurationOfMission | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00460840 | 0x015626A0 | ClearRespawnPointForDurationOfMission | 16 | 1 | 🔷 external-derived (unverified by disassembly) |

### CScriptResourceManager
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00470480 | 0x01565710 | Initialise | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004704B0 | 0x0156ACE0 | AddToResourceManager | 96 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00470510 | 0x0156B210 | RemoveFromResourceManager | 288 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00470620 | 0x0156F2A0 | HasResourceBeenRequested | 64 | 1 | 🔷 external-derived (unverified by disassembly) |

### CStuntJumpManager
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0049CA50 | 0x0156DDB0 | Init | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CB10 | 0x0156E3B0 | ShutdownForRestart | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CB40 | 0x01561A90 | AddOne | 128 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0049CBC0 | 0x01565470 | Shutdown | 112 | 1 | 🔷 external-derived (unverified by disassembly) |

### CdStream
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004063E0 | 0x01563850 | CdStreamGetStatus | 128 | 7 | 🔷 external-derived (unverified by disassembly) |
| 0x00406460 | 0x0156CD80 | CdStreamSync | 160 | 5 | 🔷 external-derived (unverified by disassembly) |
| 0x004067B0 | 0x01564A90 | CdStreamOpen | 288 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00406A20 | 0x0156C2C0 | CdStreamRead | 432 | 6 | 🔷 external-derived (unverified by disassembly) |

### CCheat
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00407410 | 0x01563A60 | IsZoneStreamingAllowed | 80 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00438450 | 0x01560F40 | ResetCheats | 48 | 4 | 🔷 external-derived (unverified by disassembly) |
| 0x00439AF0 | 0x01566D70 | DoCheats | 48 | 1 | 🔷 external-derived (unverified by disassembly) |

### CColModel
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0040F740 | 0x01564A10 | MakeMultipleAlloc | 128 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0040F810 | 0x0156DEB0 | AllocateData[void] | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x0040F870 | 0x01561730 | AllocateData[params] | 320 | 17 | 🔷 external-derived (unverified by disassembly) |

### CEntity
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00446F90 | 0x01569BF0 | UpdateRwMatrix | 64 | 20 | 🔷 external-derived (unverified by disassembly) |
| 0x004633E0 | 0x015619F0 | GetIsStatic | 32 | 18 | 🔷 external-derived (unverified by disassembly) |
| 0x0046A2D0 | 0x01565110 | GetModellingMatrix | 32 | 8 | 🔷 external-derived (unverified by disassembly) |

### CRoadBlocks
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00460DF0 | 0x0156AB70 | RegisterScriptRoadBlock | 240 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00460EC0 | 0x0156EF70 | ClearScriptRoadBlocks | 32 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x00461100 | 0x01568D90 | Init | 112 | 2 | 🔷 external-derived (unverified by disassembly) |

### CUpsideDownCarCheck
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004638D0 | 0x0156BBB0 | AddCarToCheck | 1024 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00463910 | 0x0156FA20 | RemoveCarFromCheck | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00463940 | 0x01562870 | HasCarBeenUpsideDownForAWhile | 48 | 1 | 🔷 external-derived (unverified by disassembly) |

### CGarage
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00447D50 | 0x01567910 | OpenThisGarage | 32 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00447D70 | 0x0156D5C0 | CloseThisGarage | 16 | 1 | 🔷 external-derived (unverified by disassembly) |

### CIplDefPool
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00405900 | 0x01561B20 | Constructor | 96 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x004059B0 | 0x015654E0 | New | 112 | 0 | 🔷 external-derived (unverified by disassembly) |

### CPedToPlayerConversations
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043AAE0 | 0x015635E0 | Clear | 48 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0043AB10 | 0x01566F50 | EndConversation | 304 | 1 | 🔷 external-derived (unverified by disassembly) |

### CSetPieces
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004994F0 | 0x01562350 | Init | 16 | 2 | 🔷 external-derived (unverified by disassembly) |
| 0x0049AA00 | 0x01569C50 | Update | 128 | 1 | 🔷 external-derived (unverified by disassembly) |

### CTaskComplexSeekEntityRadiusAngleOffset
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00493730 | 0x0156F640 | Constructor | 208 | 5 | 🔷 external-derived (unverified by disassembly) |
| 0x00493890 | 0x01566130 | Destructor | 112 | 1 | 🔷 external-derived (unverified by disassembly) |

### CTrafficLights
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0049D2D0 | 0x0156E5B0 | LightForCars1 | 64 | 3 | 🔷 external-derived (unverified by disassembly) |
| 0x0049D310 | 0x015620D0 | LightForCars2 | 64 | 3 | 🔷 external-derived (unverified by disassembly) |

### CTxdStore
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00408340 | 0x0156C9B0 | GetTxd | 48 | 1 | 🔷 external-derived (unverified by disassembly) |
| 0x00408370 | 0x015700A0 | GetParentTxdSlot | 48 | 1 | 🔷 external-derived (unverified by disassembly) |

### CWorld
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00407260 | 0x01566820 | GetSector | 64 | 12 | 🔷 external-derived (unverified by disassembly) |
| 0x004072A0 | 0x0156C6B0 | GetRepeatSector | 32 | 1 | 🔷 external-derived (unverified by disassembly) |

### CBaseModelInfo
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004212C0 | 0x0156C780 | SwaysInWind | 496 | 3 | 🔷 external-derived (unverified by disassembly) |

### CBox
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0040EDE0 | 0x01563AF0 | Set | 64 | 26 | 🔷 external-derived (unverified by disassembly) |

### CColLine
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0040EF10 | 0x0156D710 | Set | 64 | 4 | 🔷 external-derived (unverified by disassembly) |

### CCollisionPlugin
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0041B310 | 0x015671D0 | PluginAttach | 64 | 1 | 🔷 external-derived (unverified by disassembly) |

### CConversationForPed
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043C190 | 0x015668B0 | Update | 1184 | 0 | 🔷 external-derived (unverified by disassembly) |

### CEntryExit
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043E650 | 0x01569780 | GetEntryExitToDisplayNameOf | 80 | 1 | 🔷 external-derived (unverified by disassembly) |

### CEventAcquaintancePedHate
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00420E70 | 0x01561070 | Constructor2 | 32 | 8 | 🔷 external-derived (unverified by disassembly) |

### CEventLeaderEnteredCarAsDriver
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048E1C0 | 0x0156DC10 | Constructor | 96 | 5 | 🔷 external-derived (unverified by disassembly) |

### CEventLeaderEntryExit
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043E1C0 | 0x01566D50 | Constructor | 32 | 1 | ✅ verified (disassembly spot-check confirms plausible semantics) |

### CObjectPool
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00404870 | 0x0156DE60 | GetAt | 32 | 1 | 🔷 external-derived (unverified by disassembly) |

### CPedIntelligence
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x004893E0 | 0x0156AA00 | GetPedEntities | 16 | 2 | 🔷 external-derived (unverified by disassembly) |

### CPlaceable
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00420B80 | 0x01566E30 | SetPosn[xyz] | 64 | 7 | 🔷 external-derived (unverified by disassembly) |

### CPlayerPed
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0041BE60 | 0x0156D300 | GetWantedLevel | 32 | 26 | 🔷 external-derived (unverified by disassembly) |

### CPopulation
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00406F50 | 0x015642E0 | DoesCarGroupHaveModelId | 64 | 0 | 🔷 external-derived (unverified by disassembly) |

### CTaskComplexDiveFromAttachedEntityAndGetUp
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00492E20 | 0x01565A80 | Constructor | 80 | 4 | 🔷 external-derived (unverified by disassembly) |

### CTaskComplexLeaveAnyCar
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00421150 | 0x015667C0 | Constructor | 48 | 11 | 🔷 external-derived (unverified by disassembly) |

### CTaskComplexSeekEntityStandard
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0046AC10 | 0x0156E100 | Constructor | 208 | 22 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleCower
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048DE70 | 0x0156F220 | Constructor | 48 | 4 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleHandsUp
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048E970 | 0x01566240 | Constructor | 64 | 5 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleLeaveGroup
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00463540 | 0x0156AE30 | Constructor | 32 | 5 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleLookAbout
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048E0A0 | 0x01564820 | Constructor | 64 | 6 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimplePause
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048E750 | 0x01562630 | Constructor | 48 | 27 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleSay
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048E360 | 0x015655E0 | Constructor | 48 | 20 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleSetCharDecisionMaker
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0046A470 | 0x01565670 | Constructor | 32 | 1 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleSetCharIgnoreWeaponRangeFlag
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00474620 | 0x0156F2E0 | Constructor | 32 | 1 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleUninterruptable
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0043E2E0 | 0x0156CBA0 | Constructor | 32 | 1 | 🔷 external-derived (unverified by disassembly) |

### CTaskSimpleUseAtm
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0048DFE0 | 0x01565D10 | Constructor | 48 | 2 | ✅ verified (disassembly spot-check confirms plausible semantics) |

### CTaskTimer
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x00420E30 | 0x0156D5D0 | IsOutOfTime | 64 | 27 | 🔷 external-derived (unverified by disassembly) |

### CTempColModels
| Entry va | Body va | Method | Body extent | Direct call sites | Confidence |
| --- | --- | --- | --- | --- | --- |
| 0x0041B360 | 0x0156CBC0 | Shutdown | 160 | 1 | 🔷 external-derived (unverified by disassembly) |
