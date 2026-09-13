# Name Verification (Disassembly-Promoted Names)
*Source: `name_verification.json`*
- **$schema:** name_verification.v1
- **Generated:** tools/derive_name_verification.py
- **Note:** Promotes C27 external-only (🔷) names to ✅ where the method's disassembly references its OWN class's signature statics (name + data-flow corroborate).

## Summary
| Key | CIplStore | CStreaming | CColStore | CConversations | CEntryExitManager | CGarages | CPathFind | CPickups | CVehicleRecording | CReplay | CRunningScript | CTheScripts | CShopping |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Considered | 195 |  |  |  |  |  |  |  |  |  |  |  |  |
| Promoted | 119 |  |  |  |  |  |  |  |  |  |  |  |  |
| Prior verified | 13 |  |  |  |  |  |  |  |  |  |  |  |  |
| Total structure or disasm verified | 132 |  |  |  |  |  |  |  |  |  |  |  |  |
| By class | 6 | 15 | 8 | 8 | 10 | 10 | 12 | 6 | 11 | 10 | 5 | 11 | 7 |

## Promotions
| Entry va | Name |
| --- | --- |
| 0x00404C90 | CIplStore::IncludeEntity |
| 0x00405850 | CIplStore::RequestIplAndIgnore |
| 0x00405890 | CIplStore::RemoveIplAndIgnore |
| 0x004058D0 | CIplStore::RemoveIplWhenFarAway |
| 0x00405AC0 | CIplStore::AddIplSlot |
| 0x00405FA0 | CIplStore::Shutdown |
| 0x00407610 | CStreaming::AddImageToList |
| 0x00407820 | CStreaming::HasVehicleUpgradeLoaded |
| 0x00407A40 | CStreaming::ClearFlagForAll |
| 0x00407C00 | CStreaming::GetDefaultCopModel |
| 0x00407F00 | CStreaming::HasSpecialCharLoaded |
| 0x00407F80 | CStreaming::WeAreTryingToPhaseVehicleOut |
| 0x004084B0 | CStreaming::Shutdown |
| 0x00409710 | CStreaming::RemoveEntity |
| 0x00409A90 | CStreaming::AreTexturesUsedByRequestedModels |
| 0x0040A080 | CStreaming::RequestFile |
| 0x0040A150 | CStreaming::StreamCopModels |
| 0x0040A400 | CStreaming::StreamFireEngineAndFireman |
| 0x0040B080 | CStreaming::RemoveCurrentZonesModels |
| 0x0040B4F0 | CStreaming::StreamOneNewCar |
| 0x0040E560 | CStreaming::ReInit |
| 0x004106D0 | CColStore::LoadCol[2] |
| 0x00410730 | CColStore::RemoveCol |
| 0x004107A0 | CColStore::AddRef |
| 0x004107D0 | CColStore::RemoveRef |
| 0x00410800 | CColStore::GetBoundingBox |
| 0x00410820 | CColStore::IncludeModelIndex |
| 0x00411140 | CColStore::AddColSlot |
| 0x00411330 | CColStore::RemoveColSlot |
| 0x0043A7B0 | CConversations::Clear |
| 0x0043A810 | CConversations::AwkwardSay |
| 0x0043A840 | CConversations::StartSettingUpConversation |
| 0x0043A870 | CConversations::SetUpConversationNode |
| 0x0043A960 | CConversations::RemoveConversationForPed |
| 0x0043AAC0 | CConversations::IsConversationGoingOn |
| 0x0043B0B0 | CConversations::IsPlayerInPositionForConversation |
| 0x0043C590 | CConversations::Update |
| 0x0043E410 | CEntryExitManager::AddEntryExitToStack |
| 0x0043ECF0 | CEntryExitManager::SetAreaCodeForVisibleObjects |
| 0x0043ED80 | CEntryExitManager::ResetAreaCodeForVisibleObjects |
| 0x0043F0A0 | CEntryExitManager::PostEntryExitsCreation |
| 0x0043F150 | CEntryExitManager::GetPositionRelativeToOutsideWorld |
| 0x0043F180 | CEntryExitManager::EnableBurglaryHouses |
| 0x0043F880 | CEntryExitManager::Init |
| 0x0043FD50 | CEntryExitManager::DeleteOne |
| 0x00440B90 | CEntryExitManager::Shutdown |
| 0x00440C40 | CEntryExitManager::ShutdownForRestart |
| 0x004471B0 | CGarages::Shutdown |
| 0x00447680 | CGarages::GetGarageNumberByName |
| 0x004476D0 | CGarages::ChangeGarageType |
| 0x00447C40 | CGarages::SetTargetCarForMissionGarage |
| 0x00447CB0 | CGarages::DeActivateGarage |
| 0x00447CD0 | CGarages::ActivateGarage |
| 0x00447D00 | CGarages::IsGarageOpen |
| 0x00447D30 | CGarages::IsGarageClosed |
| 0x00448B30 | CGarages::AllRespraysCloseOrOpen |
| 0x00449E60 | CGarages::PlayerArrestedOrDied |
| 0x0044D230 | CPathFind::These2NodesAreAdjacent |
| 0x0044D310 | CPathFind::ThisNodeWillLeadIntoADeadEnd |
| 0x0044D480 | CPathFind::TestForPedTrafficLight |
| 0x0044D790 | CPathFind::TestCrossesRoad |
| 0x0044E000 | CPathFind::AddDynamicLinkBetween2Nodes_For1Node |
| 0x0044E1A0 | CPathFind::RemoveInterior |
| 0x004504F0 | CPathFind::CountNeighboursToBeSwitchedOff |
| 0x00450560 | CPathFind::MarkRoadNodeAsDontWander |
| 0x00451350 | CPathFind::FindLinkBetweenNodes |
| 0x00452090 | CPathFind::Find2NodesForCarCreation |
| 0x00452160 | CPathFind::SwitchOffNodeAndNeighbours |
| 0x004529F0 | CPathFind::LoadPathFindData[FromStream] |
| 0x00454A70 | CPickups::Init |
| 0x00454B40 | CPickups::IsPickUpPickedUp |
| 0x004551C0 | CPickups::FindPickUpForThisObject |
| 0x00455240 | CPickups::AddToCollectedPickupsArray |
| 0x004552A0 | CPickups::GetActualPickupIndex |
| 0x00455680 | CPickups::UpdateMoneyPerDay |
| 0x00459400 | CVehicleRecording::ShutDown |
| 0x004594C0 | CVehicleRecording::IsPlaybackGoingOnForCar |
| 0x00459740 | CVehicleRecording::PausePlaybackRecordedCar |
| 0x00459850 | CVehicleRecording::UnpausePlaybackRecordedCar |
| 0x00459F80 | CVehicleRecording::RegisterRecordingFile |
| 0x0045A020 | CVehicleRecording::RequestRecordingFile |
| 0x0045A060 | CVehicleRecording::HasRecordingFileBeenLoaded |
| 0x0045A0A0 | CVehicleRecording::RemoveRecordingFile |
| 0x0045A160 | CVehicleRecording::RemoveAllRecordingsThatArentUsed |
| 0x0045A8F0 | CVehicleRecording::Load |
| 0x0045A980 | CVehicleRecording::StartPlaybackRecordedCar |
| 0x0045CEA0 | CReplay::DealWithNewPedPacket |
| 0x0045D4B0 | CReplay::StreamAllNecessaryCarsAndPeds |
| 0x0045D6C0 | CReplay::FindFirstFocusCoordinate |
| 0x0045DDE0 | CReplay::IsThisPedUsedInRecording |
| 0x0045EBB0 | CReplay::RecordVehicleDeleted |
| 0x0045EC20 | CReplay::RecordPedDeleted |
| 0x0045EF20 | CReplay::InitialisePedPoolConversionTable |
| 0x0045F180 | CReplay::StoreStuffInMem |
| 0x004600F0 | CReplay::TriggerPlayback |
| 0x004604A0 | CReplay::PlayBackThisFrame |
| 0x00463CA0 | CRunningScript::GetPointerToLocalVariable |
| 0x00463CF0 | CRunningScript::ReadArrayInformation |
| 0x00464700 | CRunningScript::GetIndexOfGlobalVariable |
| 0x00464BB0 | CTheScripts::WipeLocalVariableMemoryForMissionScript |
| 0x00464D40 | CTheScripts::StartTestScript |
| 0x00464D50 | CTheScripts::IsPlayerOnAMission |
| 0x00464DA0 | CRunningScript::UpdatePC |
| 0x0046A7C0 | CTheScripts::ClearAllSuppressedCarModels |
| 0x0046AB60 | CTheScripts::AddToWaitingForScriptBrainArray |
| 0x0046ABC0 | CTheScripts::RemoveFromWaitingForScriptBrainArray |
| 0x00474750 | CTheScripts::AddToListOfSpecialAnimGroupsAttachedToCharModels |
| 0x00481200 | CTheScripts::AddToInvisibilitySwapArray |
| 0x004812D0 | CTheScripts::UndoEntityInvisibilitySettings |
| 0x00485A50 | CRunningScript::DoDeathArrestCheck |
| 0x00486720 | CTheScripts::ReadObjectNamesFromScript |
| 0x004867C0 | CTheScripts::ReadMultiScriptFileOffsetsFromScript |
| 0x0049AB10 | CShopping::GetItemIndex |
| 0x0049B200 | CShopping::StoreClothesState |
| 0x0049B240 | CShopping::RestoreClothesState |
| 0x0049B5E0 | CShopping::HasPlayerBought |
| 0x0049B610 | CShopping::SetPlayerHasBought |
| 0x0049B640 | CShopping::ShutdownForRestart |
| 0x0049BEF0 | CShopping::UpdateStats |
