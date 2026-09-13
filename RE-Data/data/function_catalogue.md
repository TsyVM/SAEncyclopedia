# Function Catalogue
*Source: `function_catalogue.json`*
- **$schema:** function_catalogue.v1
- **Generated:** tools/derive_function_catalogue.py

## Summary
| Field | Value |
| --- | --- |
| Total relocated functions | 492 |
| Named | 368 |
| Unidentified | 124 |
| Named pct | 74.8 |

## Functions
| Entry va | Body va | Body extent | Direct call sites | Entry bytes | Name | Name alternates | Source | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0x00401000 | 0x01562980 | 80 | 1 | e97b191601 |  |  |  | ⏳ unidentified |
| 0x00403C70 | 0x01566E70 | 128 | 1 | e9fb311601 |  |  |  | ⏳ unidentified |
| 0x00403E00 | 0x0156CFD0 | 32 | 4 | e9cb911601 |  |  |  | ⏳ unidentified |
| 0x004040A0 | 0x0156C060 | 64 | 5 | e9bb7f1601 |  |  |  | ⏳ unidentified |
| 0x004043E0 | 0x015700F0 | 16 | 2 | e90bbd1601 |  |  |  | ⏳ unidentified |
| 0x00404420 | 0x01563A40 | 16 | 1 | e91bf61501 |  |  |  | ⏳ unidentified |
| 0x004045B0 | 0x01567820 | 48 | 4 | e96b321601 | CIplStore::AddIplsNeededAtPosn |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004045E0 | 0x0156CFF0 | 16 | 1 | e90b8a1601 | CIplStore::ClearIplsNeededAtPosn |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00404780 | 0x015649E0 | 48 | 1 | e95b021601 | CIplStore::GetNewIplEntityIndexArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004047B0 | 0x01569770 | 16 | 1 | e9bb4f1601 | CIplStore::GetIplEntityIndexArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00404870 | 0x0156DE60 | 32 | 1 | e9eb951601 | CObjectPool::GetAt |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00404C90 | 0x01563730 | 176 | 5 | e99bea1501 | CIplStore::IncludeEntity |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00405850 | 0x01565190 | 64 | 2 | e93bf91501 | CIplStore::RequestIplAndIgnore |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00405890 | 0x0156A140 | 64 | 2 | e9ab481601 | CIplStore::RemoveIplAndIgnore |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004058D0 | 0x0156DE30 | 48 | 1 | e95b851601 | CIplStore::RemoveIplWhenFarAway |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00405900 | 0x01561B20 | 96 | 1 | e91bc21501 | CIplDefPool::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004059B0 | 0x015654E0 | 112 | 0 | e92bfb1501 | CIplDefPool::New |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00405AC0 | 0x0156C470 | 144 | 2 | e9ab691601 | CIplStore::AddIplSlot |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00405B60 | 0x0156C5F0 | 160 | 0 | e98b6a1601 | CIplStore::RemoveIplSlot |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00405FA0 | 0x015610B0 | 256 | 1 | e90bb11501 | CIplStore::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00406360 | 0x015700D0 | 32 | 1 | e96b9d1601 | CdStreamShutdown |  | SAEncyclopedia:C1-Streaming/02-cdstream-layer.md | ✅ verified (internal RE) |
| 0x004063E0 | 0x01563850 | 128 | 7 | e96bd41501 | CdStream::CdStreamGetStatus |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00406450 | 0x01567540 | 16 | 1 | e9eb101601 |  |  |  | ⏳ unidentified |
| 0x00406460 | 0x0156CD80 | 160 | 5 | e91b691601 | CdStream::CdStreamSync |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004067B0 | 0x01564A90 | 288 | 4 | e9dbe21501 | CdStream::CdStreamOpen |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00406A20 | 0x0156C2C0 | 432 | 6 | e99b581601 | CdStream::CdStreamRead |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00406E80 | 0x01560E20 | 48 | 1 | e99b9f1501 |  |  |  | ⏳ unidentified |
| 0x00406F50 | 0x015642E0 | 64 | 0 | e98bd31501 | CPopulation::DoesCarGroupHaveModelId |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407000 | 0x01562960 | 32 | 1 | e95bb91501 |  |  |  | ⏳ unidentified |
| 0x00407260 | 0x01566820 | 64 | 12 | e9bbf51501 | CWorld::GetSector |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004072A0 | 0x0156C6B0 | 32 | 1 | e90b541601 | CWorld::GetRepeatSector |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004072E0 | 0x01570050 | 48 | 2 | e96b8d1601 |  |  |  | ⏳ unidentified |
| 0x00407410 | 0x01563A60 | 80 | 2 | e94bc61501 | CCheat::IsZoneStreamingAllowed |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407460 | 0x01567520 | 32 | 1 | e9bb001601 | CStreamingInfo::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407480 | 0x015674C0 | 96 | 12 | e93b001601 | CStreamingInfo::AddToList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004074E0 | 0x0156CE90 | 64 | 1 | e9ab591601 | CStreamingInfo::RemoveFromList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004075A0 | 0x01560E50 | 64 | 1 | e9ab981501 | CStreamingInfo::GetCdPosnAndSize |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004075E0 | 0x01564070 | 32 | 1 | e98bca1501 | CStreamingInfo::SetCdPosnAndSize |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407610 | 0x01567B90 | 3392 | 2 | e97b051601 | CStreaming::AddImageToList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004076A0 | 0x0156D7A0 | 32 | 7 | e9fb601601 | CStreaming::IsVeryBusy |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407800 | 0x01565170 | 32 | 1 | e96bd91501 |  |  |  | ⏳ unidentified |
| 0x00407820 | 0x0156A100 | 64 | 2 | e9db281601 | CStreaming::HasVehicleUpgradeLoaded |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407A10 | 0x0156FEE0 | 32 | 1 | e9cb841601 |  |  |  | ⏳ unidentified |
| 0x00407A30 | 0x01562C80 | 16 | 4 | e94bb21501 |  |  |  | ⏳ unidentified |
| 0x00407A40 | 0x01566490 | 32 | 1 | e94bea1501 | CStreaming::ClearFlagForAll |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407C00 | 0x0156C9E0 | 80 | 3 | e9db4d1601 | CStreaming::GetDefaultCopModel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407D10 | 0x015703D0 | 16 | 4 | e9bb861601 | CStreaming::DisableCopBikes |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407D20 | 0x01563A50 | 16 | 1 | e92bbd1501 | CStreaming::GetDefaultMedicModel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407D30 | 0x015674B0 | 16 | 1 | e97bf71501 |  |  |  | ⏳ unidentified |
| 0x00407D40 | 0x0156CD70 | 16 | 1 | e92b501601 | CStreaming::GetDefaultFiremanModel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407DC0 | 0x0156CE80 | 16 | 1 | e9bb501601 |  |  |  | ⏳ unidentified |
| 0x00407DD0 | 0x015704A0 |  | 2 | e9cb861601 | CStreaming::IsCarModelNeededInCurrentZone |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407F00 | 0x015642C0 | 32 | 1 | e9bbc31501 | CStreaming::HasSpecialCharLoaded |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00407F80 | 0x01567B60 | 48 | 1 | e9dbfb1501 | CStreaming::WeAreTryingToPhaseVehicleOut |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00408000 | 0x015662F0 | 192 | 2 | e9ebe21501 | CStreaming::AddToLoadedVehiclesList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004080F0 | 0x0156C0A0 | 96 | 1 | e9ab3f1601 | CStreaming::RemoveCarModel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00408230 | 0x01563580 | 64 | 3 | e94bb31501 |  |  |  | ⏳ unidentified |
| 0x004082A0 | 0x01566860 | 32 | 6 | e9bbe51501 |  |  |  | ⏳ unidentified |
| 0x00408340 | 0x0156C9B0 | 48 | 1 | e96b461601 | CTxdStore::GetTxd |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00408370 | 0x015700A0 | 48 | 1 | e92b7d1601 | CTxdStore::GetParentTxdSlot |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004084B0 | 0x01563AB0 | 64 | 1 | e9fbb51501 | CStreaming::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00408C70 | 0x0156C970 | 64 | 3 | e9fb3c1601 | CStreaming::RequestVehicleUpgrade |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004096D0 | 0x0156D750 | 80 | 1 | e97b401601 | CStreaming::RenderEntity |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00409710 | 0x01561520 | 96 | 2 | e90b7e1501 | CStreaming::RemoveEntity |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00409A90 | 0x015664B0 | 368 | 4 | e91bca1501 | CStreaming::AreTexturesUsedByRequestedModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040A080 | 0x015663B0 | 176 | 3 | e92bc31501 | CStreaming::RequestFile |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040A120 | 0x01566460 | 48 | 3 | e93bc31501 | CStreaming::LoadInitialWeapons |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040A150 | 0x0156C100 | 448 | 2 | e9ab1f1601 | CStreaming::StreamCopModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040A400 | 0x01570230 | 416 | 2 | e92b5e1601 | CStreaming::StreamFireEngineAndFireman |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040B080 | 0x015629D0 | 688 | 2 | e94b791501 | CStreaming::RemoveCurrentZonesModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040B340 | 0x01566EF0 | 96 | 3 | e9abbb1501 | CStreaming::RemoveLoadedZoneModel |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0040B3A0 | 0x0156CA30 | 176 | 2 | e98b161601 | CStreaming::RemoveInappropriatePedModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040B4F0 | 0x01563B30 | 432 | 2 | e93b861501 | CStreaming::StreamOneNewCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040BA70 | 0x0156C5B0 | 64 | 1 | e93b0b1601 | CStreaming::PossiblyStreamCarOutAfterCreation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040BDA0 | 0x015703E0 | 192 | 13 | e93b461601 | CStreaming::StreamPedsIntoRandomSlots |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040CF80 | 0x015645F0 | 80 | 0 | e96b761501 | CStreaming::RemoveAllUnusedModels |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0040D3D0 | 0x015638D0 | 32 | 3 | e9fb641501 | CStreaming::LoadInitialPeds |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040E3A0 | 0x015670A0 | 208 | 10 | e9fb8c1501 | CStreaming::LoadRequestedModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040E560 | 0x0156D000 | 272 | 1 | e99bea1501 | CStreaming::ReInit |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040EDE0 | 0x01563AF0 | 64 | 26 | e90b4d1501 | CBox::Set |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040EE70 | 0x01567850 | 80 | 4 | e9db891501 |  |  |  | ⏳ unidentified |
| 0x0040EEC0 | 0x0156D230 | 80 | 0 | e96be31501 |  |  |  | ⏳ unidentified |
| 0x0040EF10 | 0x0156D710 | 64 | 4 | e9fbe71501 | CColLine::Set |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040EF50 | 0x015611B0 | 64 | 46 | e95b221501 |  |  |  | ⏳ unidentified |
| 0x0040F070 | 0x0156FAE0 | 176 | 2 | e96b0a1601 | CCollisionData::RemoveCollisionVolumes |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040F120 | 0x01562C90 | 1328 | 0 | e96b3b1501 | CCollisionData::Copy |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040F6C0 | 0x01568990 | 32 | 1 | e9cb921501 | CCollisionData::SetLinkPtr |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040F6E0 | 0x0156D850 | 32 | 3 | e96be11501 | CCollisionData::GetLinkPtr |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040F740 | 0x01564A10 | 128 | 1 | e9cb521501 | CColModel::MakeMultipleAlloc |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040F7C0 | 0x015697F0 | 80 | 1 | e92ba01501 |  |  |  | ⏳ unidentified |
| 0x0040F810 | 0x0156DEB0 | 96 | 1 | e99be61501 | CColModel::AllocateData[void] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040F870 | 0x01561730 | 320 | 17 | e9bb1e1501 | CColModel::AllocateData[params] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0040FAF0 | 0x01566620 | 96 | 7 | e92b6b1501 |  |  |  | ⏳ unidentified |
| 0x0040FB60 | 0x0156C690 | 32 | 27 | e92bcb1501 |  |  |  | ⏳ unidentified |
| 0x0040FC40 | 0x01570190 | 64 | 11 | e94b051601 |  |  |  | ⏳ unidentified |
| 0x0040FC80 | 0x015637E0 | 112 | 16 | e95b3b1501 |  |  |  | ⏳ unidentified |
| 0x0040FCF0 | 0x01567080 | 32 | 25 | e98b731501 |  |  |  | ⏳ unidentified |
| 0x0040FD50 | 0x0156CE20 | 96 | 1 | e9cbd01501 |  |  |  | ⏳ unidentified |
| 0x004103A0 | 0x015672B0 | 48 | 2 | e90b6f1501 | CColStore::AddCollisionNeededAtPosn |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004104E0 | 0x0156CF50 | 128 | 4 | e96bca1501 | CColStore::SetCollisionRequired |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x004106D0 | 0x015613A0 | 96 | 1 | e9cb0c1501 | CColStore::LoadCol[2] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00410730 | 0x01564E90 | 112 | 2 | e95b471501 | CColStore::RemoveCol |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004107A0 | 0x01569960 | 48 | 1 | e9bb911501 | CColStore::AddRef |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004107D0 | 0x0156DBE0 | 48 | 1 | e90bd41501 | CColStore::RemoveRef |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00410800 | 0x01565240 | 32 | 1 | e93b4a1501 | CColStore::GetBoundingBox |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00410820 | 0x0156A260 | 64 | 2 | e93b9a1501 | CColStore::IncludeModelIndex |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00410F80 | 0x015611F0 | 96 | 1 | e96b021501 |  |  |  | ⏳ unidentified |
| 0x00411030 | 0x0156FBB0 | 112 | 0 | e97beb1501 |  |  |  | ⏳ unidentified |
| 0x00411140 | 0x01563290 | 720 | 2 | e94b211501 | CColStore::AddColSlot |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00411330 | 0x01567210 | 160 | 1 | e9db5e1501 | CColStore::RemoveColSlot |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00411A80 | 0x0156FF00 | 112 | 18 | e97be41501 |  |  |  | ⏳ unidentified |
| 0x00411BF0 | 0x01563610 | 16 | 2 | e91b1a1501 |  |  |  | ⏳ unidentified |
| 0x00411E30 | 0x015678D0 | 64 | 3 | e99b5a1501 | CCollision::SortOutCollisionAfterLoad |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004152C0 | 0x01566DA0 | 96 | 1 | e9db1a1501 | CCollision::IsThisVehicleSittingOnMe |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004154A0 | 0x01567650 | 144 | 1 | e9ab211501 | CCollision::CheckPeds |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00415540 | 0x0156D280 | 80 | 1 | e93b7d1501 | CCollision::ResetMadeInvisibleObjects |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004161F0 | 0x01570010 | 64 | 2 | e91b9e1501 |  |  |  | ⏳ unidentified |
| 0x00417730 | 0x01564C40 | 592 | 2 | e90bd51401 | CCollision::TestLineOfSight |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041B310 | 0x015671D0 | 64 | 1 | e9bbbe1401 | CCollisionPlugin::PluginAttach |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041B360 | 0x0156CBC0 | 160 | 1 | e95b181501 | CTempColModels::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041B7D0 | 0x01564FD0 | 64 | 10 | e9fb971401 |  |  |  | ⏳ unidentified |
| 0x0041B820 | 0x0156A220 | 64 | 3 | e9fbe91401 |  |  |  | ⏳ unidentified |
| 0x0041B870 | 0x0156E0C0 | 64 | 2 | e94b281501 |  |  |  | ⏳ unidentified |
| 0x0041B950 | 0x01562000 | 48 | 7 | e9ab661401 |  |  |  | ⏳ unidentified |
| 0x0041BDD0 | 0x01567620 | 48 | 10 | e94bb81401 |  |  |  | ⏳ unidentified |
| 0x0041BE60 | 0x0156D300 | 32 | 26 | e99b141501 | CPlayerPed::GetWantedLevel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041BFA0 | 0x015612E0 | 192 | 11 | e93b531401 | CCarAI::BackToCruisingIfNoWantedLevel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041C050 | 0x0156FB90 | 32 | 4 | e93b3b1501 | CCarAI::CarHasReasonToStop |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041C4A0 | 0x01563CE0 | 400 | 1 | e93b781401 | CCarAI::AddAmbulanceOccupants |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041C600 | 0x01568C10 | 384 | 1 | e90bc61401 | CCarAI::AddFiretruckOccupants |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041C760 | 0x01569A30 | 352 | 9 | e9cbd21401 | CCarAI::TellOccupantsToLeaveCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041CA50 | 0x0156FFB0 | 96 | 2 | e95b351501 | CCarAI::FindPoliceBoatMissionForWantedLevel |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0041CC10 | 0x015638F0 | 32 | 6 | e9db6c1401 |  |  |  | ⏳ unidentified |
| 0x00420920 | 0x01565640 | 48 | 14 | e91b4d1401 |  |  |  | ⏳ unidentified |
| 0x00420950 | 0x0156AB40 | 48 | 4 | e9eba11401 |  |  |  | ⏳ unidentified |
| 0x00420980 | 0x0156E7E0 | 48 | 4 | e95bde1401 |  |  |  | ⏳ unidentified |
| 0x00420B80 | 0x01566E30 | 64 | 7 | e9ab621401 | CPlaceable::SetPosn[xyz] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00420E30 | 0x0156D5D0 | 64 | 27 | e99bc71401 | CTaskTimer::IsOutOfTime |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00420E70 | 0x01561070 | 32 | 8 | e9fb011401 | CEventAcquaintancePedHate::Constructor2 |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00421050 | 0x0156FC20 | 704 | 5 | e9cbeb1401 |  |  |  | ⏳ unidentified |
| 0x00421120 | 0x01563560 | 32 | 1 | e93b241401 |  |  |  | ⏳ unidentified |
| 0x00421150 | 0x015667C0 | 48 | 11 | e96b561401 | CTaskComplexLeaveAnyCar::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004212C0 | 0x0156C780 | 496 | 3 | e9bbb41401 | CBaseModelInfo::SwaysInWind |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00421740 | 0x015616E0 | 48 | 3 | e99bff1301 | CCarCtrl::InitSequence |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00421770 | 0x01565130 | 64 | 2 | e9bb391401 |  |  |  | ⏳ unidentified |
| 0x00422590 | 0x01567AD0 | 144 | 4 | e93b551401 |  |  |  | ⏳ unidentified |
| 0x00423880 | 0x0156A3A0 | 96 | 1 | e91b6b1401 |  |  |  | ⏳ unidentified |
| 0x00423ED0 | 0x01561040 | 48 | 4 | e96bd11301 | CCarCtrl::RemoveFromInterestingVehicleList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00423F00 | 0x015649A0 | 16 | 5 | e99b0a1401 |  |  |  | ⏳ unidentified |
| 0x00423FC0 | 0x01569420 | 64 | 12 | e95b541401 |  |  |  | ⏳ unidentified |
| 0x00424160 | 0x015667F0 | 48 | 16 | e98b261401 |  |  |  | ⏳ unidentified |
| 0x0042A730 | 0x0156DD90 | 32 | 1 | e95b361401 |  |  |  | ⏳ unidentified |
| 0x0042C250 | 0x015636D0 | 96 | 2 | e97b741301 | CCarCtrl::IsAnyoneParking |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0042F870 | 0x0156A4A0 | 352 | 16 | e92bac1301 |  |  |  | ⏳ unidentified |
| 0x0042FBC0 | 0x01570100 | 144 | 1 | e93b051401 | CCarCtrl::ScriptGenerateOneEmergencyServicesCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00438450 | 0x01560F40 | 48 | 4 | e9eb8a1201 | CCheat::ResetCheats |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00439AF0 | 0x01566D70 | 48 | 1 | e97bd21201 | CCheat::DoCheats |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043A6A0 | 0x015697D0 | 32 | 4 | e92bf11201 |  |  |  | ⏳ unidentified |
| 0x0043A7B0 | 0x0156DFF0 | 96 | 2 | e93b381301 | CConversations::Clear |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043A810 | 0x01561CC0 | 48 | 0 | e9ab741201 | CConversations::AwkwardSay |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043A840 | 0x01565550 | 48 | 1 | e90bad1201 | CConversations::StartSettingUpConversation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043A870 | 0x0156A7C0 | 256 | 4 | e94bff1201 | CConversations::SetUpConversationNode |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043A960 | 0x0156ADC0 | 80 | 2 | e95b041301 | CConversations::RemoveConversationForPed |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043AAC0 | 0x01570080 | 32 | 1 | e9bb551301 | CConversations::IsConversationGoingOn |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043AAE0 | 0x015635E0 | 48 | 2 | e9fb8a1201 | CPedToPlayerConversations::Clear |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043AB10 | 0x01566F50 | 304 | 1 | e93bc41201 | CPedToPlayerConversations::EndConversation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043ABA0 | 0x0156CAE0 | 192 | 4 | e93b1f1301 | CPed::PedIsReadyForConversation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043B000 | 0x0156C500 | 176 | 1 | e9fb141301 | CConversations::IsConversationAtNode |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0043B0B0 | 0x01563250 | 64 | 1 | e99b811201 | CConversations::IsPlayerInPositionForConversation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043C190 | 0x015668B0 | 1184 | 0 | e91ba71201 | CConversationForPed::Update |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043C590 | 0x0156D7C0 | 144 | 1 | e92b121301 | CConversations::Update |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043CEB0 | 0x015647B0 | 16 | 2 | e9fb781201 | CDarkel::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043D1F0 | 0x015635C0 | 32 | 8 | e9cb631201 | CDarkel::FrenzyOnGoing |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043D2F0 | 0x01567170 | 96 | 2 | e97b9e1201 | CDarkel::ThisPedShouldBeKilledForFrenzy |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043D350 | 0x0156CED0 | 96 | 2 | e97bfb1201 | CDarkel::ThisVehicleShouldBeKilledForFrenzy |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043D6A0 | 0x01561630 | 32 | 1 | e98b3f1201 | CDarkel::ResetModelsKilledByPlayer |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043D6C0 | 0x01564C20 | 32 | 1 | e95b751201 | CDarkel::QueryModelsKilledByPlayer |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043D6E0 | 0x01569900 | 96 | 1 | e91bc21201 | CDarkel::FindTotalPedsKilledByPlayer |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043E1C0 | 0x01566D50 | 32 | 1 | e98b8b1201 | CEventLeaderEntryExit::Constructor |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0043E2E0 | 0x0156CBA0 | 32 | 1 | e9bbe81201 | CTaskSimpleUninterruptable::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043E400 | 0x01560F30 | 16 | 1 | e92b2b1201 |  |  |  | ⏳ unidentified |
| 0x0043E410 | 0x01564090 | 96 | 2 | e97b5c1201 | CEntryExitManager::AddEntryExitToStack |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043E650 | 0x01569780 | 80 | 1 | e92bb11201 | CEntryExit::GetEntryExitToDisplayNameOf |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043ECF0 | 0x01560930 | 144 | 1 | e93b1c1201 | CEntryExitManager::SetAreaCodeForVisibleObjects |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043ED80 | 0x015678A0 | 48 | 1 | e91b8b1201 | CEntryExitManager::ResetAreaCodeForVisibleObjects |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043EF00 | 0x0156DA90 | 32 | 3 | e98beb1201 |  |  |  | ⏳ unidentified |
| 0x0043EF20 | 0x015615C0 | 112 | 1 | e99b261201 |  |  |  | ⏳ unidentified |
| 0x0043EF90 | 0x01564BE0 | 64 | 1 | e94b5c1201 |  |  |  | ⏳ unidentified |
| 0x0043F0A0 | 0x015631C0 | 144 | 1 | e91b411201 | CEntryExitManager::PostEntryExitsCreation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043F150 | 0x01566880 | 48 | 3 | e92b771201 | CEntryExitManager::GetPositionRelativeToOutsideWorld |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043F180 | 0x0156C6D0 | 112 | 1 | e94bd51201 | CEntryExitManager::EnableBurglaryHouses |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043F720 | 0x01561900 | 96 | 0 | e9db211201 |  |  |  | ⏳ unidentified |
| 0x0043F7D0 | 0x015651D0 | 112 | 1 | e9fb591201 |  |  |  | ⏳ unidentified |
| 0x0043F880 | 0x0156A6F0 | 208 | 1 | e96bae1201 | CEntryExitManager::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0043F9B0 | 0x0156AE10 | 32 | 2 | e95bb41201 |  |  |  | ⏳ unidentified |
| 0x0043FD50 | 0x01560ED0 | 96 | 2 | e97b111201 | CEntryExitManager::DeleteOne |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00440B90 | 0x01563940 | 208 | 1 | e9ab2d1201 | CEntryExitManager::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00440C40 | 0x01567550 | 208 | 1 | e90b691201 | CEntryExitManager::ShutdownForRestart |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004410E0 | 0x015667A0 | 32 | 1 | e9bb561201 |  |  |  | ⏳ unidentified |
| 0x00441130 | 0x0156C740 | 64 | 8 | e90bb61201 |  |  |  | ⏳ unidentified |
| 0x00441210 | 0x01563910 | 48 | 2 | e9fb261201 | CGameLogic::InitAtStartOfGame |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00441240 | 0x015672E0 | 80 | 1 | e99b601201 | CGameLogic::ForceDeathRestart |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00441390 | 0x0156CF30 | 32 | 32 | e99bbb1201 | CGameLogic::IsCoopGameGoingOn |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00441560 | 0x01561270 | 112 | 6 | e90bfd1101 | CGameLogic::ClearSkip |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004415C0 | 0x01564860 | 320 | 2 | e99b321201 | CGameLogic::SkipCanBeActivated |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004416C0 | 0x01569C30 | 32 | 1 | e96b851201 | CGameLogic::IsSkipWaitingForScriptToFadeIn |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00441D30 | 0x01560F90 | 128 | 3 | e95bf21101 | CGameLogic::RestorePedsWeapons |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00442020 | 0x0156FF70 | 64 | 5 | e94bdf1201 | CGameLogic::IsPlayerUse2PlayerControls |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004423C0 | 0x01563E90 | 192 | 4 | e9cb1a1201 | CGameLogic::SetUpSkip |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004428B0 | 0x0156AA30 | 272 | 2 | e97b811201 | CGameLogic::DoWeaponStuffAtStartOf2PlayerGame |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00443920 | 0x0156B150 | 48 | 2 | e92b781201 | CGangWars::InitAtStartOfGame |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004439C0 | 0x0156EFF0 | 16 | 2 | e92bb61201 | CGangWars::DontCreateCivilians |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00443AC0 | 0x01563620 | 176 | 9 | e95bfb1101 | CGangWars::GangWarFightingGoingOn |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00443F80 | 0x015699C0 | 112 | 3 | e93b5a1201 | CGangWars::CanPlayerStartAGangWarHere |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00443FF0 | 0x0156DD40 | 32 | 3 | e94b9d1201 | CGangWars::ClearSpecificZonesToTriggerGangWar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004442D0 | 0x0156CD40 | 48 | 24 | e96b8a1201 |  |  |  | ⏳ unidentified |
| 0x004444B0 | 0x01564200 | 128 | 1 | e94bfd1101 | CGangWars::ClearTheStreets |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00444530 | 0x01568E00 | 768 | 2 | e9cb481201 | CGangWars::TellGangMembersTo |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00445C30 | 0x015609C0 | 528 | 8 | e98bad1101 | CGangWars::ReleasePedsInAttackWave |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00445F50 | 0x01565090 | 128 | 1 | e93bf11101 | CGangWars::StrengthenPlayerInfluenceInZone |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00446F90 | 0x01569BF0 | 64 | 20 | e95b2c1201 | CEntity::UpdateRwMatrix |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004471B0 | 0x01566E00 | 48 | 1 | e94bfc1101 | CGarages::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447680 | 0x0156DCF0 | 80 | 9 | e96b661201 | CGarages::GetGarageNumberByName |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004476D0 | 0x01561690 | 80 | 1 | e9bb9f1101 | CGarages::ChangeGarageType |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447720 | 0x01565260 | 112 | 2 | e93bdb1101 |  |  |  | ⏳ unidentified |
| 0x004479A0 | 0x0156B1C0 | 80 | 1 | e91b381201 | CGarages::IsCarSprayable |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447B80 | 0x0156CC60 | 224 | 17 | e9db501201 | CGarages::TriggerMessage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447C40 | 0x01560BD0 | 80 | 2 | e98b8f1101 | CGarages::SetTargetCarForMissionGarage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447CB0 | 0x01563E70 | 32 | 1 | e9bbc11101 | CGarages::DeActivateGarage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447CD0 | 0x015677F0 | 48 | 1 | e91bfb1101 | CGarages::ActivateGarage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447D00 | 0x0156D2D0 | 48 | 1 | e9cb551201 | CGarages::IsGarageOpen |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447D30 | 0x01560F70 | 32 | 1 | e93b921101 | CGarages::IsGarageClosed |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447D50 | 0x01567910 | 32 | 1 | e9bbfb1101 | CGarage::OpenThisGarage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00447D70 | 0x0156D5C0 | 16 | 1 | e94b581201 | CGarage::CloseThisGarage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00448550 | 0x0156D8D0 | 112 | 1 | e97b531201 |  |  |  | ⏳ unidentified |
| 0x004485C0 | 0x01561490 | 144 | 1 | e9cb8e1101 |  |  |  | ⏳ unidentified |
| 0x00448650 | 0x01565080 | 16 | 2 | e92bca1101 |  |  |  | ⏳ unidentified |
| 0x00448660 | 0x01569B90 | 96 | 1 | e92b151201 |  |  |  | ⏳ unidentified |
| 0x00448990 | 0x0156EF90 | 96 | 1 | e9fb651201 |  |  |  | ⏳ unidentified |
| 0x00448B30 | 0x01563A10 | 48 | 2 | e9dbae1101 | CGarages::AllRespraysCloseOrOpen |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00448E50 | 0x01569180 | 80 | 5 | e92b031201 |  |  |  | ⏳ unidentified |
| 0x00449100 | 0x01566680 | 288 | 3 | e97bd51101 |  |  |  | ⏳ unidentified |
| 0x004493E0 | 0x015676E0 | 272 | 1 | e9fbe21101 |  |  |  | ⏳ unidentified |
| 0x004494F0 | 0x0156D610 | 256 | 3 | e91b411201 |  |  |  | ⏳ unidentified |
| 0x004495F0 | 0x01561400 | 144 | 1 | e90b7e1101 |  |  |  | ⏳ unidentified |
| 0x00449E60 | 0x01569150 | 48 | 1 | e9ebf21101 | CGarages::PlayerArrestedOrDied |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044A210 | 0x01567330 | 384 | 1 | e91bd11101 | CGarages::CountCarsInHideoutGarage |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044CBC0 | 0x015701D0 | 96 | 1 | e90b361201 |  |  |  | ⏳ unidentified |
| 0x0044CC20 | 0x01569FD0 | 304 | 1 | e9abd31101 |  |  |  | ⏳ unidentified |
| 0x0044CD50 | 0x0156E3E0 | 464 | 3 | e98b161201 | COnscreenTimer::AddClock |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044CDA0 | 0x01561C50 | 112 | 2 | e9ab4e1101 | COnscreenTimer::AddCounter |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044CE60 | 0x01565730 | 32 | 1 | e9cb881101 | COnscreenTimer::ClearClock |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044CE80 | 0x0156AD90 | 48 | 1 | e90bdf1101 | COnscreenTimer::ClearCounter |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044CEB0 | 0x0156EC20 | 48 | 2 | e96b1d1201 | COnscreenTimer::SetCounterFlashWhenFirstDisplayed |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044CEE0 | 0x01562130 | 48 | 1 | e94b521101 | COnscreenTimer::SetClockBeepCountdownSecs |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D080 | 0x01560DC0 | 96 | 1 | e93b3d1101 | CPathFind::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D0F0 | 0x01563FB0 | 192 | 1 | e9bb6e1101 | CPathFind::UnLoadPathFindData |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D230 | 0x01569710 | 96 | 2 | e9dbc41101 | CPathFind::These2NodesAreAdjacent |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D310 | 0x01569F30 | 160 | 1 | e91bcc1101 | CPathFind::ThisNodeWillLeadIntoADeadEnd |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D3B0 | 0x0156DE10 | 32 | 1 | e95b0a1201 | CPathFind::TidyUpNodeSwitchesAfterMission |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D3E0 | 0x01561710 | 32 | 1 | e92b431101 | CPathFind::ThisNodeHasToBeSwitchedOff |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D400 | 0x01565420 | 80 | 2 | e91b801101 | CPathFind::UnMarkAllRoadNodesAsDontWander |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D480 | 0x0156A400 | 160 | 1 | e97bcf1101 | CPathFind::TestForPedTrafficLight |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044D520 | 0x0156E9C0 | 608 | 1 | e99b141201 |  |  |  | ⏳ unidentified |
| 0x0044D790 | 0x01565AF0 | 160 | 1 | e95b831101 | CPathFind::TestCrossesRoad |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044DCD0 | 0x0156DE80 | 48 | 1 | e9ab011201 | CPathFind::SetPathsNeededAtPosition |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044DD00 | 0x01561B10 | 16 | 2 | e90b3e1101 | CPathFind::ReleaseRequestedNodes |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044DE00 | 0x01565690 | 128 | 1 | e98b781101 | CPathFind::LoadSceneForPathNodes |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044DE80 | 0x0156AD40 | 80 | 1 | e9bbce1101 | CPathFind::StartNewInterior |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044DED0 | 0x0156EC90 | 96 | 21 | e9bb0d1201 | CPathFind::AddInteriorLink |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044DF60 | 0x01562390 | 96 | 1 | e92b441101 |  |  |  | ⏳ unidentified |
| 0x0044E000 | 0x01560C20 | 416 | 4 | e91b2c1101 | CPathFind::AddDynamicLinkBetween2Nodes_For1Node |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044E1A0 | 0x01564320 | 720 | 1 | e97b611101 | CPathFind::RemoveInterior |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0044E4E0 | 0x0156A600 | 16 | 1 | e91bc11101 | CPathFind::ReInit |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004504F0 | 0x0156A990 | 112 | 0 | e99ba41101 | CPathFind::CountNeighboursToBeSwitchedOff |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00450560 | 0x0156ECF0 | 128 | 1 | e98be71101 | CPathFind::MarkRoadNodeAsDontWander |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00450950 | 0x01562750 | 80 | 1 | e9fb1d1101 | CPathFind::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00450D70 | 0x0156A6B0 | 64 | 1 | e93b991101 | CPathFind::MakeRequestForNodesToBeLoaded |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00451350 | 0x0156A180 | 160 | 1 | e92b8e1101 | CPathFind::FindLinkBetweenNodes |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00452090 | 0x0156D150 | 224 | 2 | e9bbb01101 | CPathFind::Find2NodesForCarCreation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00452160 | 0x015646A0 | 272 | 3 | e93b251101 | CPathFind::SwitchOffNodeAndNeighbours |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00452760 | 0x0156B7D0 | 192 | 3 | e96b901101 | CPathFind::ComputeRoute |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004529F0 | 0x0156F750 | 720 | 2 | e95bcd1101 | CPathFind::LoadPathFindData[FromStream] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00452F00 | 0x01565860 | 64 | 3 | e95b291101 | CPathFind::SwitchPedRoadsOffInArea |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00453000 | 0x01563F50 | 96 | 8 | e94b0f1101 |  |  |  | ⏳ unidentified |
| 0x004541E0 | 0x01568BF0 | 32 | 1 | e90b4a1101 |  |  |  | ⏳ unidentified |
| 0x00454920 | 0x0156F730 | 32 | 2 | e90bae1101 |  |  |  | ⏳ unidentified |
| 0x00454A70 | 0x01564650 | 80 | 1 | e9dbfb1001 | CPickups::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00454AC0 | 0x01568BD0 | 32 | 1 | e90b411101 | CPickups::ModelForWeapon |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00454B40 | 0x0156DA60 | 48 | 1 | e91b8f1101 | CPickups::IsPickUpPickedUp |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00454BE0 | 0x0156DB40 | 160 | 2 | e95b8f1101 | CPickup::ExtractAmmoFromPickup |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004551C0 | 0x0156D870 | 64 | 3 | e9ab861101 | CPickups::FindPickUpForThisObject |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00455240 | 0x01561580 | 64 | 1 | e93bc31001 | CPickups::AddToCollectedPickupsArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004552A0 | 0x01564BB0 | 48 | 11 | e90bf91001 | CPickups::GetActualPickupIndex |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004554C0 | 0x0156A950 | 64 | 2 | e98b541101 | CPickups::PlayerCanPickUpThisWeaponTypeAtThisMoment |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00455500 | 0x0156EC50 | 64 | 1 | e94b971101 | CPickup::FindTextIndexForString |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00455540 | 0x01565750 | 32 | 4 | e90b021101 | CPickup::FindStringForTextIndex |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00455680 | 0x0156B360 | 64 | 1 | e9db5c1101 | CPickups::UpdateMoneyPerDay |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00457380 | 0x0156DF10 | 80 | 2 | e98b6b1101 | CPickups::GenerateNewOne_WeaponType |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004588B0 | 0x01562550 | 224 | 1 | e99b9c1001 | CPickup::ProcessGunShot |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004590F0 | 0x01564140 | 192 | 1 | e94bb01001 | CPed::CreateDeadPedMoney |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00459180 | 0x01568940 | 80 | 2 | e9bbf71001 | CPed::CreateDeadPedPickupCoors |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00459400 | 0x0156E770 | 64 | 2 | e96b531101 | CVehicleRecording::ShutDown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004594C0 | 0x01561F20 | 224 | 2 | e95b8a1001 | CVehicleRecording::IsPlaybackGoingOnForCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00459740 | 0x01565B90 | 272 | 1 | e94bc41001 | CVehicleRecording::PausePlaybackRecordedCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00459850 | 0x0156BA60 | 272 | 1 | e90b221101 | CVehicleRecording::UnpausePlaybackRecordedCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00459F80 | 0x0156F110 | 112 | 1 | e98b511101 | CVehicleRecording::RegisterRecordingFile |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045A020 | 0x0156D110 | 64 | 1 | e9eb301101 | CVehicleRecording::RequestRecordingFile |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045A060 | 0x01560E90 | 64 | 1 | e92b6e1001 | CVehicleRecording::HasRecordingFileBeenLoaded |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045A0A0 | 0x015640F0 | 80 | 1 | e94ba01001 | CVehicleRecording::RemoveRecordingFile |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045A160 | 0x015688F0 | 80 | 1 | e98be71001 | CVehicleRecording::RemoveAllRecordingsThatArentUsed |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045A8F0 | 0x0156F510 | 144 | 1 | e91b4c1101 | CVehicleRecording::Load |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045A980 | 0x01565F10 | 432 | 3 | e98bb51001 | CVehicleRecording::StartPlaybackRecordedCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045AD40 | 0x0156E620 | 336 | 1 | e9db381101 |  |  |  | ⏳ unidentified |
| 0x0045AE80 | 0x01562160 | 192 | 3 | e9db721001 |  |  |  | ⏳ unidentified |
| 0x0045AFB0 | 0x015658A0 | 64 | 5 | e9eba81001 |  |  |  | ⏳ unidentified |
| 0x0045B150 | 0x01564640 | 16 | 1 | e9eb941001 | CReplay::DisableReplays |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045B160 | 0x01568BC0 | 16 | 1 | e95bda1001 | CReplay::EnableReplays |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045C440 | 0x0156E610 | 16 | 1 | e9cb211101 | CReplay::ShouldStandardCameraBeProcessed |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045C4B0 | 0x01561CF0 | 560 | 0 | e93b581001 |  |  |  | ⏳ unidentified |
| 0x0045CD10 | 0x0156A610 | 160 | 1 | e9fbd81001 |  |  |  | ⏳ unidentified |
| 0x0045CEA0 | 0x0156ED70 | 512 | 2 | e9cb1e1101 | CReplay::DealWithNewPedPacket |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045D4B0 | 0x0156A8C0 | 144 | 1 | e90bd41001 | CReplay::StreamAllNecessaryCarsAndPeds |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045D6C0 | 0x0156F180 | 160 | 0 | e9bb1a1101 | CReplay::FindFirstFocusCoordinate |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045DDE0 | 0x01565580 | 96 | 0 | e99b771001 | CReplay::IsThisPedUsedInRecording |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045DEF0 | 0x0156AE50 | 768 | 3 | e95bcf1001 |  |  |  | ⏳ unidentified |
| 0x0045EBB0 | 0x0156DAB0 | 144 | 1 | e9fbee1001 | CReplay::RecordVehicleDeleted |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045EC20 | 0x01561870 | 144 | 1 | e94b2c1001 | CReplay::RecordPedDeleted |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045EF20 | 0x0156B3A0 | 128 | 1 | e97bc41001 | CReplay::InitialisePedPoolConversionTable |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0045F180 | 0x015689D0 | 496 | 0 | e94b981001 | CReplay::StoreStuffInMem |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004600F0 | 0x0156D320 | 672 | 2 | e92bd21001 | CReplay::TriggerPlayback |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004604A0 | 0x01562050 | 96 | 1 | e9ab1b1001 | CReplay::PlayBackThisFrame |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00460630 | 0x015658E0 | 256 | 3 | e9ab521001 | CRestart::Initialise |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004607D0 | 0x0156B8F0 | 48 | 1 | e91bb11001 | CRestart::OverrideNextRestart |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00460810 | 0x0156F610 | 48 | 1 | e9fbed1001 | CRestart::SetRespawnPointForDurationOfMission |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00460840 | 0x015626A0 | 16 | 1 | e95b1e1001 | CRestart::ClearRespawnPointForDurationOfMission |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00460DF0 | 0x0156AB70 | 240 | 1 | e97b9d1001 | CRoadBlocks::RegisterScriptRoadBlock |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00460EC0 | 0x0156EF70 | 32 | 2 | e9abe01001 | CRoadBlocks::ClearScriptRoadBlocks |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00461100 | 0x01568D90 | 112 | 2 | e98b7c1001 | CRoadBlocks::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463370 | 0x0156E2D0 | 32 | 5 | e95baf1001 |  |  |  | ⏳ unidentified |
| 0x004633E0 | 0x015619F0 | 32 | 18 | e90be60f01 | CEntity::GetIsStatic |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463490 | 0x01565620 | 32 | 1 | e98b211001 |  |  |  | ⏳ unidentified |
| 0x00463540 | 0x0156AE30 | 32 | 5 | e9eb781001 | CTaskSimpleLeaveGroup::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463640 | 0x0156F250 | 32 | 1 | e90bbc1001 |  |  |  | ⏳ unidentified |
| 0x004637E0 | 0x01565D40 | 48 | 31 | e95b251001 |  |  |  | ⏳ unidentified |
| 0x004638D0 | 0x0156BBB0 | 1024 | 1 | e9db821001 | CUpsideDownCarCheck::AddCarToCheck |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463910 | 0x0156FA20 | 48 | 1 | e90bc11001 | CUpsideDownCarCheck::RemoveCarFromCheck |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463940 | 0x01562870 | 48 | 1 | e92bef0f01 | CUpsideDownCarCheck::HasCarBeenUpsideDownForAWhile |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463970 | 0x015660C0 | 112 | 1 | e94b271001 |  |  |  | ⏳ unidentified |
| 0x00463B80 | 0x0156DC70 | 128 | 1 | e9eba01001 | CStuckCarCheck::RemoveCarFromCheck |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463C00 | 0x015619B0 | 64 | 1 | e9abdd0f01 | CStuckCarCheck::HasCarBeenStuckForAWhile |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463C40 | 0x01565300 | 32 | 1 | e9bb161001 | CStuckCarCheck::ClearStuckFlagForCar |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463C70 | 0x0156A310 | 48 | 1 | e99b661001 | CStuckCarCheck::IsCarInStuckCarArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463CA0 | 0x0156E2B0 | 32 | 3 | e90ba61001 | CRunningScript::GetPointerToLocalVariable |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00463CF0 | 0x0156E350 | 96 | 6 | e95ba61001 | CRunningScript::ReadArrayInformation |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464700 | 0x0156F340 | 144 | 12 | e93bac1001 | CRunningScript::GetIndexOfGlobalVariable |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004648E0 | 0x015626B0 | 160 | 1 | e9cbdd0f01 | CRunningScript::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464BB0 | 0x01569F10 | 32 | 2 | e95b531001 | CTheScripts::WipeLocalVariableMemoryForMissionScript |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464BD0 | 0x0156DD60 | 48 | 3 | e98b911001 | CRunningScript::RemoveScriptFromList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464C00 | 0x01561990 | 32 | 3 | e98bcd0f01 | CRunningScript::AddScriptToList |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464C20 | 0x0156A2A0 | 112 | 4 | e97b561001 | CTheScripts::StartNewScript[last-idle] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464C90 | 0x0156E210 | 160 | 1 | e97b951001 | CTheScripts::StartNewScript[indexed] |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464D20 | 0x01562030 | 32 | 1 | e90bd30f01 | CTheScripts::GetScriptIndexFromPointer |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464D40 | 0x01565610 | 16 | 2 | e9cb081001 | CTheScripts::StartTestScript |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464D50 | 0x0156AA10 | 32 | 26 | e9bb5c1001 | CTheScripts::IsPlayerOnAMission |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464D70 | 0x0156E7B0 | 48 | 6 | e93b9a1001 | CRunningScript::IsPedDead |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00464DA0 | 0x015620B0 | 32 | 4 | e90bd30f01 | CRunningScript::UpdatePC |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00465070 | 0x01561010 | 48 | 5 | e99bbf0f01 |  |  |  | ⏳ unidentified |
| 0x004650A0 | 0x01564280 | 64 | 1 | e9dbf10f01 |  |  |  | ⏳ unidentified |
| 0x004650F0 | 0x01567930 | 416 | 1 | e93b281001 |  |  |  | ⏳ unidentified |
| 0x004652D0 | 0x01569D40 | 464 | 1 | e96b4a1001 |  |  |  | ⏳ unidentified |
| 0x004654B0 | 0x0156E810 | 240 | 21 | e95b931001 |  |  |  | ⏳ unidentified |
| 0x00465970 | 0x015627A0 | 208 | 2 | e92bce0f01 | CStuckCarCheck::AddCarToCheck |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046A2D0 | 0x01565110 | 32 | 8 | e93bae0f01 | CEntity::GetModellingMatrix |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046A2F0 | 0x01569D20 | 32 | 2 | e92bfa0f01 |  |  |  | ⏳ unidentified |
| 0x0046A340 | 0x0156E1D0 | 64 | 1 | e98b3e1001 |  |  |  | ⏳ unidentified |
| 0x0046A470 | 0x01565670 | 32 | 1 | e9fbb10f01 | CTaskSimpleSetCharDecisionMaker::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046A630 | 0x0156B420 | 32 | 1 | e9eb0d1001 |  |  |  | ⏳ unidentified |
| 0x0046A7C0 | 0x0156F400 | 32 | 2 | e93b4c1001 | CTheScripts::ClearAllSuppressedCarModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046A7E0 | 0x015624A0 | 48 | 1 | e9bb7c0f01 |  |  |  | ⏳ unidentified |
| 0x0046A840 | 0x01565EA0 | 32 | 1 | e95bb60f01 | CTheScripts::ClearAllVehicleModelsBlockedByScript |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0046A8C0 | 0x0156BB70 | 64 | 2 | e9ab121001 | CScriptsForBrains::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046A900 | 0x0156FA50 | 48 | 2 | e94b511001 | CScriptsForBrains::SwitchAllObjectBrainsWithThisID |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046A9C0 | 0x015628A0 | 112 | 2 | e9db7e0f01 | CScriptsForBrains::AddNewStreamedScriptBrainForCodeUse |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046AA30 | 0x01569100 | 80 | 0 | e9cbe60f01 | CScriptsForBrains::GetIndexOfScriptBrainWithThisName |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046AA80 | 0x0156D980 | 112 | 1 | e9fb2e1001 |  |  |  | ⏳ unidentified |
| 0x0046AAE0 | 0x0156D940 | 64 | 1 | e95b2e1001 |  |  |  | ⏳ unidentified |
| 0x0046AB20 | 0x01561650 | 64 | 1 | e92b6b0f01 | CScriptsForBrains::HasAttractorScriptBrainWithThisNameLoaded |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046AB60 | 0x01565010 | 112 | 2 | e9aba40f01 | CTheScripts::AddToWaitingForScriptBrainArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046ABC0 | 0x01569CD0 | 80 | 3 | e90bf10f01 | CTheScripts::RemoveFromWaitingForScriptBrainArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046AC10 | 0x0156E100 | 208 | 22 | e9eb341001 | CTaskComplexSeekEntityStandard::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046AE80 | 0x015622E0 | 112 | 2 | e95b740f01 |  |  |  | ⏳ unidentified |
| 0x0046AF50 | 0x0156B440 | 688 | 2 | e9eb041001 | CRunningScript::ScriptTaskPickUpObject |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046B390 | 0x0156A340 | 96 | 1 | e9abef0f01 | CScriptsForBrains::StartAttractorScriptBrainWithThisName |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0046CED0 | 0x01562360 | 48 | 1 | e98b540f01 | CScriptsForBrains::StartOrRequestNewStreamedScriptBrainWithThisName |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470070 | 0x01561090 | 32 | 2 | e91b100f01 |  |  |  | ⏳ unidentified |
| 0x00470100 | 0x015649B0 | 48 | 2 | e9ab480f01 |  |  |  | ⏳ unidentified |
| 0x00470150 | 0x01569460 | 688 | 4 | e90b930f01 | CRunningScript::PlayAnimScriptCommand |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470370 | 0x0156E5F0 | 32 | 1 | e97be20f01 | CTheScripts::ReinitialiseSwitchStatementData |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004703C0 | 0x01561B80 | 208 | 1 | e9bb170f01 | CTheScripts::UseSwitchJumpTable |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470480 | 0x01565710 | 32 | 1 | e98b520f01 | CScriptResourceManager::Initialise |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004704B0 | 0x0156ACE0 | 96 | 4 | e92ba80f01 | CScriptResourceManager::AddToResourceManager |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470510 | 0x0156B210 | 288 | 4 | e9fbac0f01 | CScriptResourceManager::RemoveFromResourceManager |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470620 | 0x0156F2A0 | 64 | 1 | e97bec0f01 | CScriptResourceManager::HasResourceBeenRequested |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470660 | 0x01562430 | 64 | 1 | e9cb1d0f01 | CStreamedScripts::Initialise |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004706A0 | 0x01565AD0 | 32 | 0 | e92b540f01 | CStreamedScripts::ReInitialise |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004706C0 | 0x0156B7A0 | 48 | 1 | e9dbb00f01 | CStreamedScripts::RegisterScript |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470750 | 0x0156F450 | 192 | 1 | e9fbec0f01 | CStreamedScripts::ReadStreamedScriptData |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470840 | 0x01565EC0 | 80 | 1 | e97b560f01 | CStreamedScripts::LoadStreamedScript |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x00470890 | 0x0156BFB0 | 80 | 2 | e91bb70f01 | CStreamedScripts::StartNewStreamedScript |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004708E0 | 0x0156F710 | 32 | 2 | e92bee0f01 | CStreamedScripts::RemoveStreamedScriptFromMemory |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470960 | 0x01562910 | 32 | 1 | e9ab1f0f01 | CTheScripts::InitialiseAllConnectLodObjects |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470980 | 0x015661A0 | 160 | 1 | e91b580f01 | CTheScripts::AddToListOfConnectedLodObjects |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00470A20 | 0x0156D9F0 | 112 | 2 | e9cbcf0f01 | CTheScripts::ScriptConnectLodsFunction |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00474620 | 0x0156F2E0 | 32 | 1 | e9bbac0f01 | CTaskSimpleSetCharIgnoreWeaponRangeFlag::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00474730 | 0x01562530 | 32 | 1 | e9fbdd0e01 | CTheScripts::InitialiseSpecialAnimGroupsAttachedToCharModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00474750 | 0x01565D70 | 176 | 1 | e91b160f01 | CTheScripts::AddToListOfSpecialAnimGroupsAttachedToCharModels |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0047C0E0 | 0x015689B0 | 32 | 1 | e9cbc80e01 |  |  |  | ⏳ unidentified |
| 0x0047E070 | 0x015688D0 | 32 | 1 | e95ba80e01 |  |  |  | ⏳ unidentified |
| 0x00481090 | 0x0156D8B0 | 32 | 2 | e91bc80e01 | CPed::SetStayInSamePlace |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004810C0 | 0x01561250 | 32 | 1 | e98b010e01 | CTheScripts::GetUniqueScriptThingIndex |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004810E0 | 0x015647C0 | 96 | 1 | e9db360e01 | CTheScripts::DrawScriptSpheres |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00481140 | 0x01569840 | 192 | 1 | e9fb860e01 | CTheScripts::AddToBuildingSwapArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00481200 | 0x0156DF60 | 144 | 1 | e95bcd0e01 | CTheScripts::AddToInvisibilitySwapArray |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004812D0 | 0x015652D0 | 48 | 1 | e9fb3f0e01 | CTheScripts::UndoEntityInvisibilitySettings |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00483B30 | 0x0156E050 | 112 | 1 | e91ba50e01 | CTheScripts::AddScriptSphere |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00483BA0 | 0x01561960 | 48 | 1 | e9bbdd0d01 | CTheScripts::RemoveScriptSphere |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00485A50 | 0x01564F00 | 208 | 1 | e9abf40d01 | CRunningScript::DoDeathArrestCheck |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00486670 | 0x01565CA0 | 112 | 4 | e92bf60d01 | CTheScripts::CleanUpThisVehicle |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004866C0 | 0x0156B890 | 96 | 3 | e9cb510e01 | CTheScripts::CleanUpThisObject |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00486720 | 0x0156F5A0 | 112 | 1 | e97b8e0e01 | CTheScripts::ReadObjectNamesFromScript |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00486780 | 0x01562660 | 64 | 1 | e9dbbe0d01 | CTheScripts::UpdateObjectIndices |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004867C0 | 0x01565E20 | 128 | 1 | e95bf60d01 | CTheScripts::ReadMultiScriptFileOffsetsFromScript |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004893E0 | 0x0156AA00 | 16 | 2 | e91b160e01 | CPedIntelligence::GetPedEntities |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0048DE70 | 0x0156F220 | 48 | 4 | e9ab130e01 | CTaskSimpleCower::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0048DF30 | 0x01562470 | 48 | 13 | e93b450d01 |  |  |  | ⏳ unidentified |
| 0x0048DFE0 | 0x01565D10 | 48 | 2 | e92b7d0d01 | CTaskSimpleUseAtm::Constructor |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0048E0A0 | 0x01564820 | 64 | 6 | e97b670d01 | CTaskSimpleLookAbout::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0048E180 | 0x01569990 | 48 | 5 | e90bb80d01 |  |  |  | ⏳ unidentified |
| 0x0048E1C0 | 0x0156DC10 | 96 | 5 | e94bfa0d01 | CEventLeaderEnteredCarAsDriver::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0048E360 | 0x015655E0 | 48 | 20 | e97b720d01 | CTaskSimpleSay::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0048E4F0 | 0x0156B180 | 64 | 9 | e98bcc0d01 |  |  |  | ⏳ unidentified |
| 0x0048E610 | 0x0156F300 | 64 | 4 | e9eb0c0e01 |  |  |  | ⏳ unidentified |
| 0x0048E750 | 0x01562630 | 48 | 27 | e9db3e0d01 | CTaskSimplePause::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0048E970 | 0x01566240 | 64 | 5 | e9cb780d01 | CTaskSimpleHandsUp::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00492CB0 | 0x0156E990 | 48 | 1 | e9dbbc0d01 |  |  |  | ⏳ unidentified |
| 0x00492D10 | 0x015622B0 | 48 | 2 | e99bf50c01 |  |  |  | ⏳ unidentified |
| 0x00492E20 | 0x01565A80 | 80 | 4 | e95b2c0d01 | CTaskComplexDiveFromAttachedEntityAndGetUp::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00492F90 | 0x0156B9B0 | 64 | 2 | e91b8a0d01 | CTheScripts::AddScriptEffectSystem |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00492FD0 | 0x0156F420 | 48 | 3 | e94bc40d01 | CTheScripts::RemoveScriptEffectSystem |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00493000 | 0x015692C0 | 352 | 2 | e9bb620d01 | CTheScripts::AddScriptSearchLight |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00493420 | 0x0156F000 | 128 | 2 | e9dbbb0d01 |  |  |  | ⏳ unidentified |
| 0x00493480 | 0x01562220 | 144 | 1 | e99bed0c01 |  |  |  | ⏳ unidentified |
| 0x004934F0 | 0x01565770 | 240 | 1 | e97b220d01 | CTheScripts::AttachSearchlightToSearchlightObject |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x004936C0 | 0x0156B9F0 | 112 | 2 | e92b830d01 | CTheScripts::RemoveScriptCheckpoint |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00493730 | 0x0156F640 | 208 | 5 | e90bbf0d01 | CTaskComplexSeekEntityRadiusAngleOffset::Constructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00493890 | 0x01566130 | 112 | 1 | e99b280d01 | CTaskComplexSeekEntityRadiusAngleOffset::Destructor |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x00493900 | 0x0156C000 | 96 | 2 | e9fb860d01 |  |  |  | ⏳ unidentified |
| 0x004993D0 | 0x0156E930 | 96 | 4 | e95b550d01 |  |  |  | ⏳ unidentified |
| 0x004994F0 | 0x01562350 | 16 | 2 | e95b8e0c01 | CSetPieces::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049AA00 | 0x01569C50 | 128 | 1 | e94bf20c01 | CSetPieces::Update |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049AB10 | 0x0156E330 | 32 | 2 | e91b380d01 | CShopping::GetItemIndex |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049AB30 | 0x01561A10 | 128 | 4 | e9db6e0c01 | CShopping::GetKey |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049ABD0 | 0x01565360 | 192 | 2 | e98ba70c01 |  |  |  | ⏳ unidentified |
| 0x0049ACD0 | 0x0156AC70 | 80 | 2 | e99bff0c01 |  |  |  | ⏳ unidentified |
| 0x0049ADA0 | 0x0156F0D0 | 64 | 1 | e92b430d01 | CShopping::GetNameTag |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049ADE0 | 0x0156F080 | 80 | 1 | e99b420d01 | CShopping::GetExtraInfo |  | disassembly | ✅ verified (disassembly spot-check confirms plausible semantics) |
| 0x0049AE30 | 0x015623F0 | 64 | 3 | e9bb750c01 | CShopping::RemoveLoadedShop |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049AE70 | 0x015659E0 | 160 | 7 | e96bab0c01 | CShopping::FindSection |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049AF10 | 0x0156B920 | 144 | 2 | e90b0a0d01 | CShopping::GetNextSection |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049B010 | 0x015691D0 | 240 | 1 | e9bbe10c01 |  |  |  | ⏳ unidentified |
| 0x0049B200 | 0x0156E2F0 | 64 | 1 | e9eb300d01 | CShopping::StoreClothesState |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049B240 | 0x01565320 | 64 | 1 | e9dba00c01 | CShopping::RestoreClothesState |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049B5E0 | 0x0156B770 | 48 | 1 | e98b010d01 | CShopping::HasPlayerBought |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049B610 | 0x0156F3D0 | 48 | 1 | e9bb3d0d01 | CShopping::SetPlayerHasBought |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049B640 | 0x015624D0 | 96 | 1 | e98b6e0c01 | CShopping::ShutdownForRestart |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049BEF0 | 0x0156B6F0 | 128 | 2 | e9fbf70c01 | CShopping::UpdateStats |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049C3B0 | 0x0156ACC0 | 32 | 1 | e90be90c01 |  |  |  | ⏳ unidentified |
| 0x0049C8F0 | 0x0156FA80 | 96 | 0 | e98b310d01 |  |  |  | ⏳ unidentified |
| 0x0049C9A0 | 0x01562930 | 48 | 0 | e98b5f0c01 |  |  |  | ⏳ unidentified |
| 0x0049C9E0 | 0x01566280 | 112 | 1 | e99b980c01 |  |  |  | ⏳ unidentified |
| 0x0049CA50 | 0x0156DDB0 | 96 | 1 | e95b130d01 | CStuntJumpManager::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CB10 | 0x0156E3B0 | 48 | 1 | e99b180d01 | CStuntJumpManager::ShutdownForRestart |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CB40 | 0x01561A90 | 128 | 2 | e94b4f0c01 | CStuntJumpManager::AddOne |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CBC0 | 0x01565470 | 112 | 1 | e9ab880c01 | CStuntJumpManager::Shutdown |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CC50 | 0x0156AC60 | 16 | 1 | e90be00c01 | CTagManager::Init |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CC60 | 0x0156E900 | 48 | 1 | e99b1c0d01 | CTagManager::ShutdownForRestart |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CC90 | 0x01562110 | 32 | 1 | e97b540c01 | CTagManager::AddTag |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CDE0 | 0x0156B330 | 48 | 1 | e94be50c01 | CTagManager::UpdateNumTagged |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049CE10 | 0x0156F270 | 48 | 1 | e95b240d01 | CTagManager::SetupAtomic |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049D2D0 | 0x0156E5B0 | 64 | 3 | e9db120d01 | CTrafficLights::LightForCars1 |  | disassembly | 🔷 external-derived (unverified by disassembly) |
| 0x0049D310 | 0x015620D0 | 64 | 3 | e9bb4d0c01 | CTrafficLights::LightForCars2 |  | disassembly | 🔷 external-derived (unverified by disassembly) |
