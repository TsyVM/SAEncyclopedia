# C28.4 — CPathFind and the 28-byte Node

> **The one-sentence version:** [C12](../C12-Path-Network/C12-Path-Network.md) proved the *on-disk* path
> node is 28 bytes; four `CPathFind` methods show the *in-memory* node is 28 bytes too — the engine loads
> the file record straight into its runtime array with no reshaping — and the same methods expose the
> packed node-id, the 4-byte link record, the low-nibble link count, and the per-area pointer arrays the
> whole network hangs from.

**Subsystem category:** Paths / AI — in-memory structure
**Depends on:** [C28.1](01-the-class-map-by-subsystem.md), [C12](../C12-Path-Network/C12-Path-Network.md)
(the `NODES*.DAT` 28-byte record and the 68,237-node population)
**RE status:** Documented
**Confidence:** ✅ for the 28-byte stride and the arrays (re-checked by the tool) · 🟡 for the individual
node-field *meanings* read from single use sites · 🔷 for methods listed but not disassembled

---

## 1. The node is 28 bytes in memory, same as on disk

[C12.1](../C12-Path-Network/01-nodes-dat.md) decoded the `NODES*.DAT` record as **28 bytes**, proved four
ways from the file. `CPathFind`'s runtime methods index their node array with the identical stride.
`These2NodesAreAdjacent` (`entry_va 0x0044D230`):

```
01569714  movzx ecx, word ptr [esp + 4]     ; node id low word  = area
01569719  shr   eax, 0x10                    ; node id high word = index
0156971C  imul  eax, eax, 0x1C               ; index * 28
0156971F  shl   ecx, 2                        ; area * 4
01569722  add   eax, dword ptr [ecx + 0x96F854]  ; + per-area node-array base
```

`imul …, 0x1C` — **28 bytes** — and it is not a one-off: the same multiply appears in
`ThisNodeWillLeadIntoADeadEnd` (`0x0044D310`), `TestForPedTrafficLight` (`0x0044D480`), and
`TestCrossesRoad` (`0x0044D790`), four independent methods agreeing on the stride. That the runtime stride
equals C12's on-disk record size is the finding: the loader (`LoadPathFindData`, `0x004529F0`) reads the
28-byte file record into a 28-byte runtime slot unchanged, so C12's field layout *is* the memory layout.
This is the `CPathFind` equivalent of C28.2's 224-byte script object and C28.3's 20-byte streaming record —
one class, one stride, read straight from the bytes.

## 2. The packed node id and the per-area arrays

The disassembly above also reveals how a node is *addressed*. A node id is a packed 32-bit value —
**high word = node index, low word = path area** — and the node itself is found by indexing a table of
per-area array pointers, `perAreaNodes[area] + index × 28`, with the pointer table at `0x96F854`. The
adjacent link table `0x96FA94` is the parallel per-area array of *links*.

Those two static addresses are the visible face of a set of parallel per-area pointer arrays that live
inside the `CPathFind` singleton. `UnLoadPathFindData` (`entry_va 0x0044D0F0`) frees a whole area's
resources by walking six of them at once, indexed by area (`this + area×4 + offset`):

```
mov eax, [esi + edi*4 + 0x804]   ; area's node array      -> free
mov eax, [esi + edi*4 + 0x924]   ; area's array #2        -> free
mov eax, [esi + edi*4 + 0xA44]   ; area's link array      -> free
mov eax, [esi + edi*4 + 0xDA4]   ; area's array #4        -> free
mov eax, [esi + edi*4 + 0xB64]   ; area's array #5        -> free
mov eax, [esi + edi*4 + 0xC84]   ; area's array #6        -> free
```

Six parallel arrays, one pointer per path area, each freed and then null-cleared — the node array
(`+0x804`), the link/navi array (`+0xA44`), and four more (naviNodes, links-to-navi, and the
address/connection tables C12 enumerates). `CPathFind::Init` (`0x0044D080`) is the mirror image: it zeroes
the same block of arrays in a loop bounded at `0x48` (72) slots — so the network is partitioned into up to
**72 per-area buckets**, each a set of independently heap-allocated node/link arrays. (The offsets sit
`0x120` = 72 × 4 apart, which is why the same loop stride reaches every array.)

## 3. The link record, the link count, and the node flags

Inside a node, the connectivity fields decode from the loop in `These2NodesAreAdjacent`:

- **Link count** is the low nibble of the byte at **node +0x18**: `mov dl, byte ptr [eax + 0x18]` then
  `and edx, 0xF` — so a node has at most 15 links (the high nibble carries other bits). All four adjacency
  methods read it the same way.
- **First-link index** is the signed word at **node +0x10** (`movsx …, word ptr [eax + 0x10]`), used to
  index the per-area link array.
- A **link record is 4 bytes** — the loop advances `add eax, 4` per link and compares the target's packed
  id as two words (`cmp word ptr [eax], cx` against area, `cmp word ptr [eax + 2], di` against index). So
  each link is `{uint16 area, uint16 nodeIndex}`: a packed reference to the neighbour, in the same
  area/index form as the node id itself.

A second byte, **node +0x1A**, carries node *flags* in its high nibble: `ThisNodeHasToBeSwitchedOff`
(`0x0044D3E0`) does `mov al, [eax + 0x1A]; shr al, 4; cmp al, 1 … cmp al, 2` (node types 1 and 2 are the
switchable ones), and `ThisNodeWillLeadIntoADeadEnd` tests the same nibble in the range `1…10`. These are
the traffic/behaviour bits C12's file fields describe, read here from their runtime use — hence 🟡 on the
exact per-bit meanings, ✅ on where they live.

## 4. The rest of the class

The other 21 methods (🔷, C27-inherited) are the network's runtime services and read coherently against the
structure above: streaming path data in and out (`LoadPathFindData`, `MakeRequestForNodesToBeLoaded`,
`ReleaseRequestedNodes`, `SetPathsNeededAtPosition`, `LoadSceneForPathNodes`), routing
(`ComputeRoute`, `FindLinkBetweenNodes`, `Find2NodesForCarCreation`), the interior/dynamic-link system
(`StartNewInterior`, `AddInteriorLink`, `RemoveInterior`, `AddDynamicLinkBetween2Nodes_For1Node`), and the
mission-driven node switching (`SwitchOffNodeAndNeighbours`, `SwitchPedRoadsOffInArea`,
`MarkRoadNodeAsDontWander`, `UnMarkAllRoadNodesAsDontWander`, `TidyUpNodeSwitchesAfterMission`). Full list
in [`class_catalogue.json`](../RE-Data/data/class_catalogue.json).

---

### Key takeaways

- The **in-memory path node is 28 bytes (`0x1C`)** — proven by four independent `CPathFind` methods and
  equal to [C12](../C12-Path-Network/C12-Path-Network.md)'s on-disk record, so the file layout is the
  memory layout.
- A node id packs **{high word = index, low word = area}**; nodes are found via per-area pointer tables at
  `0x96F854` (nodes) and `0x96FA94` (links).
- A **link is 4 bytes** `{uint16 area, uint16 nodeIndex}`; a node's **link count is the low nibble of
  +0x18**, its first-link index the word at +0x10, and its behaviour flags the high nibble of +0x1A.
- The network is partitioned into up to **72 per-area buckets**, each a set of six independently allocated
  arrays, freed by `UnLoadPathFindData` and zeroed by `Init`.

**Next:** back to the [C28 hub](C28-Class-Catalogue.md), or on to the undocumented gameplay-manager classes
([C28.1 §3](01-the-class-map-by-subsystem.md#3-which-classes-already-have-a-home-and-which-are-candidates))
as the seed for a future chapter.
