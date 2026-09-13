# C12.4 — Pathfinding and AI integration

## What the path network is for

The `nodes.dat` binary representation (C12.1–C12.2) is the navigation graph for SA's AI systems. Without it:
- Vehicles would not know where to drive — no lanes, no intersections
- Peds would not know how to walk from A to B
- Police could not chase the player across the city systematically

The path network is the game's map-aware routing infrastructure — a pre-built graph that encodes the legal paths through the city at the topology level (which nodes connect to which) and the semantic level (is this a vehicle lane? a ped sidewalk? which direction?).

## The two path graphs: vehicles and peds

`nodes.dat` encodes two distinct path graphs co-resident in the same file:
- **Vehicle paths:** driveable nodes representing road centerlines, intersections, and parking areas. Each node has an encoded position, connection count, and flags for road type (highway, back alley, etc.).
- **Ped paths:** walkable nodes representing footpaths, crosswalks, and interior paths. Peds use a separate but overlapping network.

The binary format stores both in the same file. `CPathFind::Initialise` loads both graphs simultaneously, populating two node arrays (vehicle nodes and ped nodes).

## How the AI uses the path graph

### Vehicle AI (CCarAI)

When an ambient-traffic vehicle needs a destination, `CCarAI::MakeCarDoAction` calls the path network for routing:
1. `CPathFind::FindNearestNodeToPoint` — find the closest vehicle node to the current position
2. `CPathFind::FindNodeClosestToCoors` — find a target node near the destination
3. `CPathFind::FindShortestRoute` — A* or Dijkstra over the vehicle graph to find a path
4. The route is stored as a sequence of node IDs; the vehicle drives toward the next node each frame

The AI updates its current node target when it gets within a threshold distance of the current one.

### Ped AI (CPedAI)

`CPed::FindNextPointOnRoute` uses the ped path graph similarly — find nearest ped node, pathfind to target, follow nodes. Ped routing is used for:
- Ambient peds walking their normal routes
- Mission peds navigating to objectives
- Police chasing on foot

### Police pursuit routing

The police wanted system (C41) uses the vehicle path graph to route police vehicles ahead of the player: `CPathFind::FindDirectionForVehicleToGoToPoint` computes a heading toward the player's predicted position by projecting onto the nearest vehicle node. This is why police cars can "predict" the player's route and set up roadblocks (C41.3).

## The two representations and their uses

C12.3 documented that the path network exists in both a text source format (`.nod.dat` files) and the binary `nodes.dat`. The runtime uses only the binary. The text source is an authoring artifact — it was used by Rockstar's path-authoring tools to generate the binary. Modders who want to change paths must either:
1. Re-parse `nodes.dat` and patch it (binary editing), or
2. Use a tool that can regenerate `nodes.dat` from an edited text representation

Most SA map mods that add roads use a custom path node editor that exports a patched `nodes.dat`.

## Connection count limits and the graph structure

Each node stores a connection count (how many edges lead to other nodes) directly in the `nodes.dat` binary. The maximum connections per node is encoded in the format (⏳ — exact bit field). High-valency intersections (many-way crossroads) need high connection counts. The format limits this to prevent the node record from growing unboundedly.

## Modding the path network

Adding new roads to a SA mod requires:
1. Adding new vehicle path nodes at the correct world positions
2. Connecting them to existing nodes (bidirectional connections for two-way roads)
3. Adding ped path nodes along the sidewalks of the new road
4. Regenerating `nodes.dat` to include the new graph

Without updated path nodes, AI vehicles will not drive on new roads and police will not be able to route through new areas. This is the most common oversight in large map mods — the visual road exists but the AI ignores it.

**Previous:** [C12.3 — Two representations](03-two-representations.md)  
**Up:** [C12 — Path Network](C12-Path-Network.md)
