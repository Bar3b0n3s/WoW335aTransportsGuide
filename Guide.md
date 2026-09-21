# Creating Transports in WoW 3.3.5a

**Ships, zeppelins, elevators and moving platforms on a TrinityCore-family server.**

---

## Part 0 — Scope, verification, and which transport you actually want

### 0.1 What this guide covers

Everything needed to put a new moving object into a 3.3.5a world:

- **Moving transports** — ships, zeppelins, boats, gunships. These travel a route, carry
  passengers, and can cross between maps.
- **Animated transports** — elevators, lifts, trapdoors, platforms. These move in place along a
  fixed animation.
- **Cross-map routes** — transports that carry players between continents.
- **Custom models** — putting your own artwork in as a transport.

### 0.2 What it assumes

- A 3.3.5a client (build 12340) and a TrinityCore-family server core.
- You can edit DBC files, rebuild a client patch archive, and run SQL against your world database.
- You can re-run the map/vmap extraction tools that ship with your core.

No specific tool is named anywhere in this guide. Where a file path appears it is a placeholder:

| Placeholder | Meaning |
|---|---|
| `<client>/Data/patch-X.MPQ/DBFilesClient/` | Your custom DBC patch archive — see the naming rule below |
| `<server>/Data/dbc/` | The server's extracted DBC folder, under your core's data directory |
| `<server>/Data/vmaps/` | The server's extracted collision data |

**[VERIFIED] The archive name matters.** Put every custom DBC in **one** archive, under
`DBFilesClient\` at its root, named `Data/patch-<single character>.MPQ` — for example
`patch-4.MPQ`. The extraction tools only open archives matching `patch-?.MPQ`, exactly one
character, so `patch-custom.MPQ` or `patch-4b.MPQ` is never opened by them even though the client
may load it. They read the base `Data/patch-?.MPQ` group *before* the locale group and take the
first copy of a file they find, which is the opposite of the client's locale-first preference — so
never keep a second copy of the same DBC in a locale patch. And do not ship the DBCs as a
folder-style archive: a *directory* named `patch-4.MPQ` makes the extractor fail to open a required
archive and skip that entire locale without extracting anything.

### 0.3 How the claims in this guide were verified — and how to re-check them

Almost everything here was read directly out of the core's C++ source rather than taken from
existing tutorials, because much of the folklore around transports is wrong for 3.3.5. The files
that matter, by repo-relative path:

| Path | What lives there |
|---|---|
| `src/server/game/Entities/Transport/Transport.cpp` / `.h` | The `Transport` class: movement tick, passengers, map changes |
| `src/server/game/Maps/TransportMgr.cpp` / `.h` | Template loading, path generation, spawning |
| `src/server/game/Entities/GameObject/GameObject.cpp` | Type-11 behaviour, model/collision, event dispatch |
| `src/server/game/Entities/GameObject/GameObjectData.h` | The `DataN` field names for every gameobject type |
| `src/server/game/Globals/ObjectMgr.cpp` | Template and spawn loading, and every validation message |
| `src/server/game/DataStores/DBCStores.cpp` | How the taxi path tables are indexed |
| `src/server/shared/DataStores/DBCStructure.h`, `DBCfmt.h` | DBC record layouts and column formats |
| `src/tools/vmap4_extractor/gameobject_extract.cpp` | How gameobject models get collision |

**File locations differ between cores and revisions.** `TransportMgr` in particular lives under
`Maps/` in some trees and `Entities/Transport/` in others. If a path above does not exist in your
tree, search for the class name rather than assuming this guide is wrong.

Claims are marked throughout:

- **[VERIFIED]** — read directly from core source. Should hold on any TrinityCore 3.3.5 tree.
- **[CONVENTION]** — not enforced by code. It is what Blizzard's data does, or what avoids
  trouble. Safe to deviate from if you know why.
- **[FORK]** — known to differ between cores. Check your own source before relying on it.

### 0.4 Differences between cores — read this before you start

3.3.5 emulation is not one codebase, and transports are one of the areas that drifted most.

**[FORK] Where animation data comes from.** This guide documents cores that read elevator
animation from `TransportAnimation.dbc` and `TransportRotation.dbc`. Some cores and older
revisions read the same data from world-database tables named `transport_animation` and
`transport_rotation` instead. To check yours, find the transport manager's animation-loading
function: if it iterates a DBC store you are on the DBC path; if it runs a `SELECT` you are on the
SQL path, and your animation frames belong in those tables instead of the DBC.

**[FORK] `transport_template` / `transport_spawn` tables.** Later cores moved transport definitions
into dedicated tables. On 3.3.5 TrinityCore these **do not exist** — and confusingly, the core
still emits error text naming `transport_template`. That message refers to an in-memory structure,
not a table. Do not go looking for it. If your core genuinely has those tables, its own schema
supersedes Part 1.7 here.

**[FORK] Column sets.** `gameobject_template` gained and lost columns over the years (`StringId`,
`VerifiedBuild`, and on later cores extra addon fields). Always write explicit column lists in your
`INSERT` statements rather than relying on positional order.

### 0.5 Which one do you want?

| What you want | Type | Read |
|---|---|---|
| A boat or zeppelin that travels a route | 15 | Part 1 |
| Something that carries NPCs or props with it | 15 | Part 1, especially 1.8 |
| An elevator, lift, trapdoor or platform that moves in place | 11 | Part 2 |
| A vehicle that crosses between continents | 15 | Part 1, then Part 3 |
| Your own model as any of the above | either | Part 5 |

### 0.6 The two systems, side by side

Both are called transports and both are gameobjects, but they share almost nothing.

| | Type 11 `TRANSPORT` | Type 15 `MAP_OBJ_TRANSPORT` |
|---|---|---|
| Typical use | Elevator, lift, platform | Ship, zeppelin, gunship |
| C++ class | plain `GameObject` | dedicated `Transport` class |
| Spawned from | the `gameobject` table, like any object | the `transports` table — never `gameobject`. Continent routes only; a route on an instanceable map is created by an instance script instead (Part 3.2) |
| Can you put it in `gameobject`? | Yes | **No** — the core refuses it outright |
| Path data | `TransportAnimation.dbc`, keyed by **gameobject entry** | `TaxiPath.dbc` + `TaxiPathNode.dbc` |
| Does the server move it? | **No** — that code is disabled | Yes, every 200 ms |
| Can NPCs ride it? | **No** | Yes |
| Can players ride it? | Yes (client-side only) | Yes |
| Lives in the world grid? | Yes | No — tracked separately per map |

**[VERIFIED] The most consequential difference:** the server does not move a type-11 elevator at
all. The movement code exists but is commented out, and the server only broadcasts a progress
counter. The client animates the platform locally. That is why an elevator cannot carry an NPC, a
chest, or anything else the server needs to track — the server genuinely does not know where the
platform is.

---

## Part 1 — Moving transports (type 15): ships, zeppelins, boats

### 1.1 How it actually works — and the mistake almost every guide makes

Here is the part that trips people up:

> **[VERIFIED] The server does not stream the transport's position to clients. Each client
> calculates it independently, from its own copy of the taxi path.**

The server ticks the transport five times a second, advances it along a spline, and moves its
passengers. But trace what goes out on the wire during that tick and the answer is: **nothing about
the transport itself**. Its position is updated with a plain member assignment — no packet is
built, no update block is queued, no visibility update is triggered.

What the client gets instead:

1. **Its position at that instant**, once, in the create block built when the object first
   becomes visible to that client. This is the only position on the wire — a type-15 transport
   sends a "stationary position" block and no movement block — and that accessor is overridden for
   this type to return the live simulated position, so it is where the ship is *now*, not where the
   route starts.
2. **A `PathProgress` counter** — milliseconds elapsed into the route's cycle.
3. **All 24 `DataN` fields**, via the gameobject query the client sends when it first meets an
   unknown entry. That includes the taxi path id, the movement speed and the acceleration rate.

From those three things plus its own `TaxiPath.dbc` and `TaxiPathNode.dbc`, the client reproduces
the same spline the server computed and draws the ship where it should be. The server's copy of
the position answers *server-side* questions — collision, passenger offsets, whether a grid is
loaded — not "where should this be drawn".

The core comments on this directly: divergent `PathProgress` between clients is described as
causing players to see the object in a different position, which only makes sense if each client
derives position from that value.

**The consequence you must plan around:**

> Your new `TaxiPath` and `TaxiPathNode` rows must exist in **both** the client's DBC files and the
> server's extracted DBC folder, and they must contain **identical** data.

If the two disagree, server and client each simulate a different route from the same clock. The
ship is drawn somewhere other than where the server thinks it is. Players who board it are
attached server-side to a vessel that is not visually there — they slide, fall through the world,
or appear to stand on open water. Nothing logs an error, because from each side's own perspective
everything is consistent.

This is also the most likely reason an existing server has subtly broken boats. If the client's
DBCs were ever replaced — a content patch, a port from a newer client, anything — and the server's
extracted copies were not regenerated to match, **every** transport is desynced, not just new ones.

**[CONVENTION] Practical rule:** edit the taxi tables once, in one place, and deploy the *same
bytes* to both sides. Either extract the server's DBCs from the patched client, or copy the edited
files into both locations. Never hand-edit them twice.

**[VERIFIED] If you re-extract rather than hand-copy, empty the extractor's output `dbc/` folder
first.** The DBC extractor skips any file that already exists at the destination — no size or
timestamp check — and reports that it extracted zero DBC files while silently handing you your old
ones back.

### 1.2 Choosing IDs that will not collide

You need free IDs in several tables. These are the **stock 3.3.5a maxima** — a useful floor, but
not the whole answer:

| Table | Stock max ID | [CONVENTION] Safe starting point |
|---|---|---|
| `TaxiPath.dbc` | 1978 | 2000+ |
| `TaxiPathNode.dbc` | 46874 | 50000+ |
| `TaxiNodes.dbc` | 440 | 500+ |
| `GameObjectDisplayInfo.dbc` | 9624 | 10000+ |
| `Map.dbc` | 724 | 800+ |
| `gameobject_template.entry` | ~245000 in current data | 900000+ |

**Do not simply trust that table.** Any custom content already installed — ported assets, another
patch, a previous project — may have claimed IDs far above the stock maximum. Before picking:

1. Check the **client's** copy of each DBC for its highest used ID.
2. Check the **server's** copy. These can differ, and often do.
3. Take the larger of the two and leave a gap.

**[VERIFIED] A new taxi path ID does not need to be contiguous.** Gaps are free; the core sizes its
lookup array to the highest path ID and leaves unused slots empty.

**[VERIFIED] But a hard constraint hides here.** That array is sized from the highest ID in
`TaxiPath.dbc` and is then indexed by each node's `PathID` **with no bounds check**. A
`TaxiPathNode` row whose `PathID` exceeds every `TaxiPath.ID` writes past the end of a heap buffer
during startup. That is memory corruption, not an error message.

> **Always add the `TaxiPath.dbc` row before, or together with, the `TaxiPathNode.dbc` rows.** A
> path row without nodes is survivable. Node rows without a path row are not.

### 1.3 The endpoint trap in `TaxiPath.dbc`

`TaxiPath` has two endpoint columns, `FromTaxiNode` and `ToTaxiNode`. For a transport they do
nothing useful — a transport is not a flight path and is never routed through the taxi network.

**[VERIFIED] But they are not ignored.** The core builds a lookup keyed on the pair
`(FromTaxiNode, ToTaxiNode)`, and that lookup is a plain assignment. **Two paths sharing an
endpoint pair silently overwrite each other**, and the loser stops existing as far as the flight
network is concerned.

In stock data four pairs are already duplicated this way, so this is not hypothetical.

**[CONVENTION] Three safe options, best first:**

1. **Add two new `TaxiNodes.dbc` rows** for your route's endpoints and use those IDs. This is what
   Blizzard *usually* does — most transport routes use dedicated nodes named `Transport, <place>`.
   **Set both `MountCreatureID` columns to `0`.** That is the property that keeps a node out of the
   faction taxi masks and out of the nearest-taxi-node lookup; the check is per faction, so even one
   non-zero entry makes the node a live flight-master position for that side with no NPC there — and
   because that lookup runs on the flight master's own coordinates, a stray node at a dock can
   hijack a nearby real flight master. Never clone a flight-master row as a template without zeroing
   these. Blizzard is not consistent here, so do not verify the pattern by sampling: several shipped
   routes end at genuine flight-master nodes, and the gunship paths use `-1`. Copy the
   dedicated-node pattern, not those.
2. **Use a pair nothing else uses.** In stock data `(0, 0)` is free — but **verify that in your own
   copy first**, because it is an attractive default that custom content often grabs.
3. **Reuse the endpoints of the route you are mirroring** — only after confirming no other path
   already uses that exact pair.

**[VERIFIED] Endpoints that do not exist in `TaxiNodes.dbc` are harmless.** The flight network is
built by iterating real taxi nodes and looking paths up, so a path whose endpoints are not real
nodes is simply never visited.

### 1.4 Authoring the route

A route is an ordered list of points. Three things make it more than that: stops, the closing leg,
and padding.

#### Capturing coordinates

Fly or swim the route in game and record your position at each point the vessel should pass
through. Any GM command that reports your current coordinates works.

**[VERIFIED] These are plain world coordinates on the map named by the node's `ContinentID`** — not
offsets, not relative to anything.

**[CONVENTION] Height, by vehicle type.** Very consistent in Blizzard's data, and worth copying:

- **Boats use `Z = 0.0` on every node** on every open-sea route Blizzard ships. Sea level is zero
  on these maps and the hull sits correctly with the transport origin at zero, so you do not need to
  sample water height. The one exception is the Sister Mercy, which manoeuvres inside a harbour and
  uses real heights.
- **Zeppelins and gunships use real `Z` values** — roughly 50–280 for the open-world zeppelins,
  580–790 for the two Icecrown patrol gunships. The instanced raid gunship paths sit much lower and
  span a far wider range, so do not copy their numbers. Fly the route and record actual heights.

#### Spacing

**[CONVENTION]** Blizzard's routes place nodes roughly 100–300 yards apart, tightening to 50–100
around docks and turns. The path is interpolated as a smooth curve through the nodes, so widely
spaced points give lazy sweeping arcs and closely spaced points give tight control. Two nodes at
nearly the same position are dangerous — see Part 6.

#### Stops

A node is a stop when its `Flags` value is exactly `2`.

**[VERIFIED] This is an exact equality test, not a bitmask test.** `Flags = 3` is *not* a stop. If
you want a node that is both a stop and a teleport, you cannot express it — pick one.

A stop holds the vessel in place for `Delay` **seconds**, then departs. Blizzard's values: 60
seconds at long-haul docks, 30 on the short ferry run, 5–10 at patrol waypoints.

#### The closing leg, and why Blizzard pads its paths

Two behaviours combine here, and together they explain something that looks like a mistake in the
stock data.

**[VERIFIED] First: the core discards your first and last node.** After building the keyframe list
it removes the first and last entry — *unless* that entry is a stop frame or carries an
arrival/departure event. They are treated as spline control points that exist only to give the
curve a well-defined direction at each end.

**[VERIFIED] Second: the leg from the last keyframe back to the first is always a teleport**, even
for a route that returns to where it started. There is no such thing as a seamlessly looping path.

Together: whatever your last surviving keyframe is, the vessel **snaps instantly** from there back
to the first surviving keyframe, once per cycle, forever.

**[CONVENTION] The fix Blizzard uses is padding.** Continue the route past its start point by
repeating the first few nodes at the end. After the core trims one node off each end, the last
remaining keyframe sits on top of the first — so the mandatory teleport covers zero distance and is
invisible.

Measuring the closing jump across the real single-map routes shows both the technique and what
happens without it:

| Route | Closing jump | Result |
|---|---|---|
| Feathermoon Ferry | 0.3 yd | Padded — seamless |
| Green Island | 0.5 yd | Padded — seamless |
| Walker of Waves | 0.8 yd | Padded — seamless |
| The Skybreaker (patrol) | 2.0 yd | Padded — seamless |
| The Zephyr | 5.3 yd | Close enough to pass |
| Sister Mercy | 38 yd | Noticeable |
| Moonspray | **600 yd** | Visibly snaps back every loop |

The Moonspray really does jump 600 yards on every circuit. That is a flaw in Blizzard's data, not a
mystery — and it is exactly what your route will do if you skip the padding.

> **[CONVENTION] Recipe for a seamless loop:** lay the circuit out so it comes back round to its
> start, then continue past it by repeating your first nodes — **three** copies for a route that
> stops just short of its start point. The test is the invariant, not the count: after dropping the
> first and last node, the new first and last nodes must be at the same position. Blizzard's
> Feathermoon Ferry does exactly this — 19 nodes, indices 16–18 repeating 0–2, closing gap 0.3 yd.
>
> Node 0 and all three pad nodes must be **plain** — `Flags = 0`, no arrival or departure event. The
> core only trims an end node that is neither a stop frame nor an event node, so a node 0 that is a
> dock stop survives, the padding is wasted, and the loop closes on a full leg. Put your first dock
> stop at node 1 or later.

A route that is not a loop — a patrol that reverses, or a shuttle between two docks — should be
authored as a there-and-back circuit, since every path cycles forever.

### 1.5 `TaxiPath.dbc` and `TaxiPathNode.dbc` field reference

#### `TaxiPath.dbc` — one row per route

| # | Field | Meaning |
|---|---|---|
| 0 | `ID` | Path ID. Referenced by `gameobject_template.Data0`. |
| 1 | `FromTaxiNode` | Endpoint node. See the collision warning in 1.3. |
| 2 | `ToTaxiNode` | Endpoint node. Same warning. |
| 3 | `Cost` | Flight cost in copper. **[CONVENTION]** `0` on every transport route. |

#### `TaxiPathNode.dbc` — one row per point

| # | Field | Meaning |
|---|---|---|
| 0 | `ID` | Unique row ID across the whole file. |
| 1 | `PathID` | The `TaxiPath.ID` this node belongs to. |
| 2 | `NodeIndex` | Order along the route, starting at `0`. |
| 3 | `ContinentID` | Map ID this node is on. |
| 4–6 | `X`, `Y`, `Z` | World coordinates on that map. |
| 7 | `Flags` | `0` normal, bit 0 (`& 1`) teleport, `2` stop. The two tests differ: stop is exact equality, teleport is a bitmask — so `3` teleports but is **not** a stop. |
| 8 | `Delay` | **Seconds** to wait. Only meaningful on a stop node. |
| 9 | `ArrivalEventID` | Event fired on arrival at this node. `0` = none. |
| 10 | `DepartureEventID` | Event fired on departure. `0` = none. |

**[VERIFIED] `NodeIndex` must be dense and zero-based — `0, 1, 2 … N-1`, no gaps.** Nothing
validates this. A gap leaves a null entry that the path generator dereferences unconditionally,
crashing the server during startup. If you delete a node, renumber everything after it.

**[VERIFIED] A plain path needs at least four nodes.** The core trims one node off each end and
then asserts that at least one keyframe survives — an assertion that is live in release builds — so
a two-node plain path trims to nothing and **aborts the server at startup**. A three-node plain path
leaves a single keyframe, and the movement tick returns immediately whenever there is one or fewer,
so the vessel spawns and never moves. Four plain nodes is the first count that leaves two usable
keyframes. Zero or one node is worse still: nothing validates the count, and the spline builder
reads past the end of the point array. (Nodes that are stops or carry events are not trimmed, so a
route whose ends are stops can survive on fewer — do not rely on it.)

**[VERIFIED] `Flags` bit 0 (value `1`) marks a teleport — and costs you two nodes.** The flagged
node *and the node immediately after it* are both dropped from the keyframe list: the flag latches a
skip that swallows the following iteration. The vessel therefore jumps from the node *before* the
flagged one to the node *two after* it. Budget two sacrificial nodes for every jump. The same
two-node loss happens **automatically** wherever `ContinentID` changes between consecutive nodes —
you do not need to set the flag for a map change, and Blizzard generally does not. See Part 3.

**[VERIFIED] `Delay` is in seconds.** Everything else time-related in this system is milliseconds,
which makes this an easy mistake: `Delay = 60000` is not a one-minute stop, it is a sixteen-hour
one.

### 1.6 `gameobject_template` — the transport definition

Create one row with `type = 15`.

**[VERIFIED] Field meanings**, from the core's own field-name struct:

| Column | Name | Meaning |
|---|---|---|
| `Data0` | `taxiPathId` | The `TaxiPath.ID` to follow. **Required.** |
| `Data1` | `moveSpeed` | Cruise speed, yards/second. Integer only. |
| `Data2` | `accelRate` | Acceleration, yards/second². Integer only. |
| `Data3` | `startEventID` | Not read by the server. Client-side. |
| `Data4` | `stopEventID` | Not read by the server. Client-side. |
| `Data5` | `transportPhysics` | `TransportPhysics.dbc` row — client-side buoyancy and sway. |
| `Data6` | `mapID` | Passenger pseudo-map. `0` = no static passengers. See 1.8. |
| `Data7` | `worldState1` | Not read by the server. |
| `Data8` | `canBeStopped` | `1` allows the vessel to be halted at stops. |

**Real values from the shipped transports**, which bound what is known to work:

| | Ships | Zeppelins | Turtles | Gunships (patrol) | Gunships (raid) |
|---|---|---|---|---|---|
| `moveSpeed` (`Data1`) | 30 (15–21 on some) | 30 (10 and 40 on two) | 30 | 2 | 20 |
| `accelRate` (`Data2`) | 1 | 1 | 1 | 1 | 10 |
| `transportPhysics` (`Data5`) | 1 | 0 | 0 | 61 | 0 or 61 |

**[VERIFIED] `accelRate` is `1` on every shipped ship, zeppelin and turtle, and on the two Icecrown
patrol airships.** The eight raid gunships use `10` and the raid zeppelin uses `5` — so higher values
are known-good, they are just rare. Only the patrol airships appear in the `transports` table; the
raid ones are created by instance scripts.

**[VERIFIED] `TransportPhysics.dbc` contains exactly three rows** — `1`, `21` and `61`. Any other
value in `Data5` points at a row that does not exist.

**[VERIFIED] Neither `moveSpeed` nor `accelRate` is validated, and `0` in either is fatal.**
`accelRate = 0` divides by zero while computing the timing model; `moveSpeed = 0` produces an
infinite travel time. Both corrupt the route's total duration, and a duration of zero makes the
movement tick divide by zero on its very first run. Nothing warns you.

**[VERIFIED] `Data0 = 0` is silently skipped.** No template is built and nothing is logged. The
transport simply never exists.

Other columns worth noting:

- `displayId` — the model. See Part 5.
- `size` — scale multiplier. **[VERIFIED] Never validated anywhere**; `0` gives a zero-scale
  object. Use `1`.
- `AIName` — set to the smart-AI gameobject handler if you want SQL scripting. See 1.9.
- `ScriptName` — a C++ script name, if you have one.

#### How speed and acceleration become a timetable

**[VERIFIED]** The core precomputes an arrival and departure time for every keyframe, using a
simple accelerate–cruise–decelerate model: the vessel accelerates from each stop at `accelRate`
until it reaches `moveSpeed`, cruises, then decelerates into the next stop. Where two stops are
close enough together that cruise speed is never reached, it accelerates to the midpoint and
decelerates from there.

Two practical consequences:

- **Stops, not nodes, define the speed profile.** The vessel does not slow down for ordinary
  nodes, only for `Flags = 2` ones. But a route with **no** `Flags = 2` node anywhere does *not*
  cruise at constant speed: with no stop to anchor the timetable the core falls back to keyframe 0,
  so the vessel decelerates to a dead stop at the end of every lap and accelerates away again. It
  simply never pauses, because no `Delay` applies. Genuinely constant motion is not achievable — put
  the unavoidable slow-down somewhere the player will not be looking.
- **The total cycle time is derived, not configured.** There is no "period" column anywhere. Change
  the speed and the whole timetable shifts.

At `moveSpeed = 30` and `accelRate = 1`, a vessel takes 30 seconds and 450 yards to reach cruise
speed — which is why Blizzard's dock-to-dock legs are kilometres long and the stops are a minute.

### 1.7 `gameobject_template_addon` and the `transports` row

#### The addon row

**[CONVENTION] Every shipped transport but one uses identical values:** `faction = 0`,
`flags = 40` — a combination of "is a transport" and "does not despawn". The exception is the
Icecrown raid zeppelin, which ships with `flags = 0`.

**[VERIFIED] The row is not required by any check — and omitting it is a silent mistake.** With no
addon row the core skips applying faction and flags entirely, leaving both zero, so your transport
never receives its transport flag. Add the row.

#### The `transports` row

This is what actually spawns the vessel.

| Column | Meaning |
|---|---|
| `guid` | Unique spawn ID. Any free value. |
| `entry` | Your `gameobject_template.entry`. |
| `name` | **[VERIFIED] Documentation only — never read by the core.** |
| `ScriptName` | **[VERIFIED] Never applied.** C++ script names go in `gameobject_template.ScriptName`. |

**[VERIFIED] `entry` carries a unique index — one spawn per template.** You cannot run two vessels
from a single template row. A second ship needs a second `gameobject_template` entry, which may
reuse the same `Data0` path if you want them on the same route.

**[VERIFIED] The spawn position and map are not in this table.** They come from the **first
surviving keyframe** of the taxi path. Because the core trims the first node unless it is a stop or
carries an event (see 1.4), that is normally **node index 1**, not node 0 — on one shipped route
that is 165 yards along. Only two shipped paths keep their node 0. There is nowhere to specify where
a transport starts.

**[VERIFIED] The worst failure mode in the whole system lives here.** If the entry named by a
`transports` row is not `type = 15` — or is missing from `gameobject_template` entirely — the row
is skipped **completely silently**. No error, no warning. The only symptom is that the count in the
"spawned N continent transports" startup line is lower than you expected. If your transport does
not appear and nothing is logged, check this first.

### 1.8 Putting NPCs, props and doodads on the deck

Passengers are ordinary `creature` and `gameobject` rows with two differences: their `map` is the
transport's **pseudo-map**, and their coordinates are **offsets from the transport's origin**
rather than world positions.

#### The pseudo-map

`gameobject_template.Data6` names a map ID that exists only as a filing key for passenger spawns.
Blizzard gives every transport its own, with names like `Transport: Menethil to Theramore`.

**[VERIFIED] What a new pseudo-map actually requires:**

- **A `Map.dbc` row on the server.** Without it, every passenger row on that map is rejected at
  startup with a "spawned at nonexistent map" error and your deck comes up empty.
- **`InstanceType = 0` on that row.** If the map is flagged as a dungeon or raid, an extra
  validation demands an instance template, and **every gameobject passenger is silently dropped**
  with a misleading "invalid coordinates" message. Creatures survive; objects do not.
- **Nothing else.** No terrain tiles, no collision data, no navmesh. The server never creates a
  real map object for it — passengers are created on the transport's *actual* map.
- **`Data6` must match.** The core learns which maps are transport maps by reading `Data6` from
  every type-15 template. If it does not match, the passenger special-casing never engages.

**[VERIFIED] The client does not need this map row** for passengers, since passengers exist on the
real world map. **[CONVENTION]** Add it to both copies anyway, so the two never drift.

**[CONVENTION] Do not share a pseudo-map between two transports.** Both will load the same
passenger set, and you will get two copies of every crewman.

#### Offsets, and the two limits that constrain your deck

Passenger coordinates are relative to the transport origin, with `orientation` relative to its
facing. They are small signed numbers — a large ship's hull spans roughly −58 to +44 along its
long axis.

**[VERIFIED] Two different limits exist, at two different layers, and the tighter one binds
first:**

| Limit | Where | What happens |
|---|---|---|
| **±75** on each axis | Player movement packets | The packet is **silently dropped**. A player past this point stops updating. |
| **±250** on each axis | Player login | The player is relocated to their homebind with an error logged. |

**[VERIFIED] Neither applies to `creature` or `gameobject` spawn rows.** Those are not
range-checked at all — an out-of-range passenger simply floats in space somewhere, with no error.

> **[CONVENTION] Design any walkable deck within ±75 on every axis.** That is the real constraint.
> It fits every Blizzard ship and zeppelin hull, but **not** Orgrim's Hammer, whose upper-deck crew
> sit at Z offsets of 84–90 — outside the limit. Watch the Z axis in particular: the check is
> applied per axis, so on a tall vessel the deck can sit more than 75 above the transport origin
> even when X and Y are comfortable.

#### Writing the spawn rows

**[CONVENTION] Values the shipped data uses**, consistent across hundreds of rows:

| Column | Value | Why |
|---|---|---|
| `map` | your `Data6` | Not the world map |
| `position_x/y/z`, `orientation` | transport-local offsets | Not world coordinates |
| `zoneId`, `areaId` | `0` | Resolved at runtime from the transport's live position |
| `phaseMask` | `1` | No shipped transport uses phasing |
| `spawnMask` | `1` | See below |

**[VERIFIED] `spawnMask` matters functionally even though it is not validated on a transport map.**
Passengers are filed by difficulty index and loaded using the transport's map spawn mode, which is
`0` for a continent transport. **Bit 0 must be set, so `spawnMask = 1`.** An instanced transport
needs the bit matching its difficulty instead.

#### The GM commands — one helps, one hurts

**[VERIFIED] The NPC-add command is fully transport-aware.** Stand on the deck where you want the
NPC, add it, and the core stores transport-local offsets and saves the row with the pseudo-map ID
automatically. This is by far the easiest way to author passengers — let the server write the SQL,
then read it back out.

**[VERIFIED] The gameobject-add command is not.** It has no transport branch at all. Used while
standing on a deck it writes a world-space row on the *real* map, which then sits motionless in the
ocean. **Gameobject passengers must be written by hand.**

To get offsets by hand, the position-reporting GM command prints your transport-local coordinates
and the pseudo-map ID whenever you are standing on a transport. Those values can be pasted straight
into a spawn row.

#### Things that do not work with passengers

- **[VERIFIED] Spawn groups.** Transport spawns are forced into the legacy compatibility group at
  load. Listing one in a spawn group is rejected with "already a member of spawn group 1".
- **[VERIFIED] Game events and spawn pools.** Both work by suppressing the normal grid registration
  — which is the exact store the transport reads to find its passengers. A pooled or event-gated
  passenger is invisible to the transport. Additionally, when a game event fires it tries to
  instantiate a real map for the pseudo-map, which is not something you want.
- **[VERIFIED] Surviving a map change.** Only players cross with the vessel. Creatures and
  gameobjects are dropped and re-created from the destination map's copy of the spawn list. Any
  runtime state they held is lost. See Part 3.

### 1.9 Scripting a transport without writing C++

This is the most useful technique in the guide, and it is not documented anywhere else: **Blizzard's
zeppelins are scripted entirely in SQL.** No C++ script is involved in any shipped transport — the
`ScriptName` column is empty on all twenty of them.

#### How it works

**[VERIFIED] The chain, end to end:**

1. A taxi path node carries an `ArrivalEventID` or `DepartureEventID`.
2. When the vessel reaches that node, the core fires the event two ways at once: it starts any
   matching `event_scripts` rows, **and** it informs the gameobject's AI.
3. If the template's `AIName` is set to the smart-AI gameobject handler, that inform is translated
   into a smart-script event.
4. Any `smart_scripts` row listening for that event id runs.

#### The two ingredients

**1. Set the AI on the template:**

```sql
UPDATE `gameobject_template` SET `AIName` = 'SmartGameObjectAI' WHERE `entry` = @ENTRY;
```

**2. Write smart-script rows** with:

| Column | Value |
|---|---|
| `entryorguid` | your `gameobject_template.entry` |
| `source_type` | `1` (gameobject) |
| `event_type` | `71` — the "gameobject event inform" event |
| `event_param1` | the event id from the taxi node |

That is the whole mechanism. Every one of the fourteen shipped transport script rows uses exactly
this shape.

#### The two patterns Blizzard uses

**Pattern A — a dockmaster announces the arrival.** Action type `1` (talk), targeting type `19`
(nearest creature of a given entry, within a radius):

```sql
-- On arrival event 15318, make the nearest Frezza (entry 9564) within 100 yards say her line 0
INSERT INTO `smart_scripts`
 (`entryorguid`,`source_type`,`id`,`link`,`event_type`,`event_phase_mask`,`event_chance`,`event_flags`,
  `event_param1`,`event_param2`,`event_param3`,`event_param4`,`event_param5`,
  `action_type`,`action_param1`,`action_param2`,`action_param3`,`action_param4`,`action_param5`,`action_param6`,
  `target_type`,`target_param1`,`target_param2`,`target_param3`,`target_param4`,
  `target_x`,`target_y`,`target_z`,`target_o`,`comment`) VALUES
 (@ENTRY,1,0,0,71,0,100,0, 15318,0,0,0,0, 1,0,0,0,0,0,0, 19,9564,100,0,0, 0,0,0,0,
  'Transport - On arrival event - dockmaster says line 0');
```

The talker is a normal NPC standing on the dock. The radius works because the vessel is physically
alongside when the arrival event fires.

**Pattern B — a timed sequence.** Action type `80` calls a timed action list (`source_type = 9`),
whose rows run in order with delays. Inside an action list, `event_param1` and `event_param2` are
reused as the **minimum and maximum delay in milliseconds** before that step runs:

```sql
-- Arrival event 21868 triggers action list @ENTRY*100
 (@ENTRY,1,0,0,71,0,100,0, 21868,0,0,0,0, 80,@ENTRY*100,0,0,0,0,0, 1,0,0,0,0, 0,0,0,0,
  'Transport - On dock - run script'),

-- ...and the list itself: two lines, then an emote three seconds later
 (@ENTRY*100,9,0,0,0,0,100,0, 0,0,0,0,0, 1,0,0,0,0,0,0, 19,34715,100,0,0, 0,0,0,0,
  'Captain says line 0'),
 (@ENTRY*100,9,1,0,0,0,100,0, 0,0,0,0,0, 1,1,0,0,0,0,0, 19,34721,100,0,0, 0,0,0,0,
  'First mate says line 1'),
 (@ENTRY*100,9,2,0,0,0,100,0, 3000,3000,0,0,0, 5,5,0,0,0,0,0, 19,34715,100,0,0, 0,0,0,0,
  'Captain plays emote 5 (exclamation) after 3s');
```

Action list steps can target on-deck crew or dockside NPCs interchangeably.

#### Free hooks in the stock data

**[VERIFIED]** Of the 51 arrival/departure event ids in the shipped transport paths, **37 fire into
nothing at all** — no script row, no C++ handler. Among them:

- **Every departure event on the three classic zeppelins.** They fire on schedule and do nothing.
- **All eight events on the Sister Mercy** patrol route.

These are ready-made hooks: you can attach behaviour to an existing transport with nothing but a
few `smart_scripts` rows and no DBC edits at all. Conversely, two shipped routes (the two turtle
ferries) have **no event ids anywhere**, so scripting those does require a DBC edit.

#### The other routes, and what does not work

- **`event_scripts`** — **[VERIFIED]** fully supported and automatically permitted: the core adds
  every taxi node event id to its list of legal script ids, so your rows will not be rejected. No
  shipped transport uses it. Note that both the source and the target passed to the script are the
  **transport itself**, so a summon command places the creature at the literal world coordinates in
  the row, not at a deck offset.
- **C++ transport scripts** — a dedicated script type exists with hooks for boarding, relocation
  and per-tick updates. **[VERIFIED]** Bind it through `gameobject_template.ScriptName`, never
  through the `transports` table. No shipped transport uses one.
- **[VERIFIED] Smart-script `source_type = 7` ("transport") does not work.** It is marked
  not-yet-implemented in the source, and there is not a single row of it in the shipped data. Use
  `source_type = 1` as above.

### 1.10 NPCs that walk around on a moving deck

**[VERIFIED] This works**, and the shipped data does it — nine crewmen patrol moving vessels.

The trick is that **waypoint coordinates are transport-local offsets too**, exactly like the spawn
position. Set the creature's movement type to waypoint movement, give it a waypoint path, and write
the path points as offsets.

**[CONVENTION] How the shipped data does it:**

- The creature's spawn row uses waypoint movement type.
- A creature-addon row links the spawn to a waypoint path id.
- The first waypoint sits exactly on the spawn offset.
- `orientation` is set only on points where the NPC pauses, and left null elsewhere. **[VERIFIED]**
  this one is enforced, not stylistic: the core applies a waypoint's facing only where `delay > 0`
  and ignores it everywhere else, with nothing logged.
- Waypoint `delay` is in **milliseconds** — unlike taxi node `Delay`, which is seconds.

The core keeps each passenger's "home position" updated as the vessel moves, so an NPC that is
pulled away and returns walks back to the right place on deck rather than to a fixed spot in the
ocean.

**[VERIFIED] One caveat:** every NPC created on a transport is flagged to ignore pathfinding,
because transport geometry is not part of the server's navigation data. Deck NPCs move in straight
lines between waypoints. Keep paths simple and clear of railings.

### 1.11 Removing or replacing a transport

Tearing one down has its own ordering trap, which is the trap in 1.2 running backwards.

1. **`DELETE` the `transports` row first.** Leaving it behind after the template goes is exactly the
   silent failure described in 1.7.
2. **Delete the passenger rows** on the pseudo-map, or they log "spawned at nonexistent map" on
   every boot once the `Map.dbc` row goes.
3. **Delete the script, addon and template rows.**
4. **In the DBCs, delete the `TaxiPathNode.dbc` rows *before* the `TaxiPath.dbc` row.** This is the
   1.2 trap in reverse: the node lookup array is sized from the highest `TaxiPath.ID` and indexed by
   each node's `PathID` with no bounds check, so removing the highest path row while its nodes
   survive corrupts the heap at startup. It bites hardest when tearing down the Part 4 example,
   whose path id is the highest in the file.
5. **Redeploy both DBC copies** (clearing the extractor output first, per 1.1) and bump the client
   cache version (Part 4.5).
6. **Expect one casualty:** any character who logged out on the deck is relocated to their homebind
   with an error on next login. That is the documented behaviour, not a new fault.

**[CONVENTION]** To reroute an existing transport, edit its path's coordinates in place rather than
renumbering. Keeping the same `TaxiPath.ID` and `NodeIndex` range avoids both ordering traps.

---

## Part 2 — Animated transports (type 11): elevators, lifts, platforms

### 2.1 Start here: the server does not move it

**[VERIFIED]** The server-side movement code for type 11 exists and is **commented out**. What the
server actually does each tick is advance a counter and broadcast it as a fraction of the
animation's total length. The client receives that fraction and animates the platform locally from
its own copy of `TransportAnimation.dbc`.

Design consequences, all of them unavoidable:

- **Nothing can ride an elevator except a player.** No NPCs, no gameobjects, no summons. The server
  cannot place them, because it does not know where the platform is.
- **You cannot script it from its position.** There is no "arrived at top" event.
- **[VERIFIED] The pause event fields are dead.** `Data3` and `Data4` are named `pause1EventID` and
  `pause2EventID`, and nothing in the core ever reads them. The shipped values point at event
  script ids that do not exist. Elevators cannot be scripted from SQL through their own motion at
  all.
- **Collision does not change with state.** **[VERIFIED]** The core explicitly skips collision
  toggling for anything flagged as a transport, so the platform's collision stays put.

If you need something that carries NPCs or fires events as it moves, you want a **type 15**
transport with a short path, not an elevator. A type-15 can follow a purely vertical route.

### 2.2 `TransportAnimation.dbc` — the motion itself

**[FORK] Check your core first** — some read this from world-DB tables instead. See 0.4.

**[VERIFIED] Keyed by gameobject entry, not display id.** This surprises people: the animation is
attached to your `gameobject_template.entry`, so two elevators sharing a model can move
differently, and a new elevator entry needs its own animation rows even if the model already has
some.

| # | Field | Meaning |
|---|---|---|
| 0 | `ID` | Unique row id. |
| 1 | `TransportID` | **The `gameobject_template.entry`.** |
| 2 | `TimeIndex` | Milliseconds into the cycle. |
| 3–5 | `X`, `Y`, `Z` | Offset from the object's spawn position, in model space. |
| 6 | `SequenceID` | Not used by the server. |

**[VERIFIED] The cycle length is the largest `TimeIndex` for that entry.** There is no separate
duration field — add a frame at a later time and the animation gets longer.

**[VERIFIED] Frames are looked up by "first frame at or after this time".** Between frames the
client interpolates. You therefore do not need many frames: a lift that rises, waits, and descends
needs only the handful of moments where its motion changes.

**[VERIFIED] If an entry has no animation rows at all**, the cycle length is zero, the server
broadcasts an invalid progress value, and the platform does not animate. This is the usual cause of
"my elevator just sits there".

#### `TransportRotation.dbc`

Same idea for orientation: keyed by gameobject entry, a `TimeIndex`, and a quaternion (`X`, `Y`,
`Z`, `W`). Only needed for platforms that turn or tilt as they travel — most lifts do not have any
rotation rows.

### 2.3 Authoring a lift cycle

Work in offsets from the spawn position. A simple lift that rises 30 yards, waits at the top,
comes back down and waits at the bottom:

| `TimeIndex` (ms) | X | Y | Z | What it represents |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | At the bottom |
| 5000 | 0 | 0 | 0 | Still at the bottom — holds for 5 s |
| 13000 | 0 | 0 | 30 | Arrives at the top after an 8 s climb |
| 18000 | 0 | 0 | 30 | Holds at the top for 5 s |
| 26000 | 0 | 0 | 0 | Back at the bottom |

Total cycle: 26 seconds, looping forever.

**[CONVENTION] Points worth copying from the shipped data:**

- **Repeat a position to create a pause.** Motion between two frames is continuous, so a hold is
  just two frames with the same coordinates.
- **Frame counts vary enormously** — from about 5 for a simple trapdoor to several hundred for a
  long, curved travel path. Use as many as the shape needs.
- **The first frame should be at `TimeIndex = 0`** and at the object's resting offset.
- **Make the last frame match the first**, or the platform will jump when the cycle restarts —
  the same closing-loop problem as taxi paths, for the same reason.

### 2.4 The spawn row

An elevator is an ordinary `gameobject` row. Fields that behave unusually:

**[VERIFIED] `state` is overwritten at spawn.** The core sets the object's state from the
template's `Data1` (`startOpen`) regardless of what the spawn row says. **[CONVENTION]** Keep them
consistent anyway — `state = 1` when `Data1 = 0`. A value of 3 or higher is rejected and the spawn
is skipped.

**[VERIFIED] `Data1` must be `0` for a self-cycling lift.** The animation counter only advances
while the object is in the *ready* state. A non-zero `startOpen` puts it in the *active* state at
creation, the counter never increments, the broadcast phase never changes, and **the platform never
moves — with nothing logged**. The three shipped templates that set it are all deliberately parked
until an instance script releases them.

**[CONVENTION] An elevator needs a `gameobject_template_addon` row too** — `faction = 0`,
`flags = 40`, exactly as for a type 15 (Part 1.7). Every shipped elevator template that has a spawn
carries the transport bit; without it the client does not treat the platform as something you ride.

**[VERIFIED] `Data0` (`pause`) is written straight to the object's level field and is otherwise
unused server-side.** The one function that classifies on it is never called, so a non-zero value
does *not* stop the server animating the platform — but the value is published to clients, and that
is how the client knows where to park it. **[CONVENTION]** Leave it at `0` for a continuously
cycling lift: 70 of the 89 shipped type-11 templates do. It is not universal — 19 set a non-zero
pause, including several named elevators, all of them deliberately parked until a script releases
them by clearing the value and setting the ready state.

**[VERIFIED] `Data2` (`autoCloseTime`) is in milliseconds** — the `/65536` conversion the field
name implies was removed in 3.0.3 and is not applied. Shipped elevators use `0` or `3000`, which
only makes sense read as 3 seconds. **In practice it does nothing for type 11**: the elevator tick
never consults it, and the only readers are the door/button/goober use paths, so an elevator's
auto-close timer never fires. Leave it at `0`.

**[VERIFIED] `animprogress`** is passed through untouched. **[CONVENTION]** `100` on the classic
elevators, `255` on some later ones. It does not drive the animation.

#### Rotation: you do not need to compute a quaternion

**[VERIFIED]** `orientation` and `rotation0`–`rotation3` are separate columns and both are read.
But if the four rotation values do not form a unit quaternion — and all-zeros does not — the core
**derives the quaternion from `orientation`** automatically, as a rotation about the vertical axis.

So `rotation0 = rotation1 = rotation2 = rotation3 = 0` plus a correct `orientation` works fine. The
only cost is one error line per spawn at startup.

**[CONVENTION] To avoid the log noise**, fill them the way the shipped data does — `rotation0` and
`rotation1` stay `0`, and:

```
rotation2 = SIN(orientation / 2)
rotation3 = COS(orientation / 2)
```

A hand-built quaternion is only needed for a platform tilted off the vertical axis, which
`orientation` alone cannot express.

**[VERIFIED] `gameobject_addon.parent_rotation0-3`** is a separate thing again: the rotation of an
animated sub-model's pivot, written straight through to the client. Identity (`0,0,0,1`) is correct
for almost everything. A non-unit value there is discarded and reset to identity, with an error —
it is **not** renormalised for you.

---

## Part 3 — Cross-map and multi-continent routes

A transport that carries players between continents is a normal type-15 with nodes on more than one
map. There is no separate mechanism, but several rules become load-bearing.

### 3.1 Map changes are automatic

**[VERIFIED] You do not set a flag for a map change.** Whenever `ContinentID` differs between two
consecutive nodes, the core inserts a teleport automatically.

**[VERIFIED] The two nodes at the boundary are both discarded.** The node whose `ContinentID`
differs from the next one is never turned into a keyframe — it only sets the *previous* keyframe's
teleport flag — and the node immediately after it is skipped as well. So the vessel sails to the
**second-to-last** node on the first map, vanishes, and reappears at the **second** node on the
next one. On the shipped Menethil-to-Theramore route, nodes 0–9 sit on the Eastern Kingdoms and
10–18 on Kalimdor; the jump actually runs node 8 → node 11, and nodes 9 and 10 are never visited.
This is on top of the first/last-node trim in 1.4 — a separate mechanism.

**[CONVENTION]** Put **four** nodes out at sea around the transition — two on each side, well away
from anywhere a player can see shore. The inner pair is consumed by the mechanism above, so it is
the *outer* pair that becomes the visible departure and arrival point. Author exactly one node per
side and you lose both, moving the jump somewhere you did not choose. It is instant and obvious if
it lands in view of land.

**[VERIFIED] The closing leg is a teleport anyway**, so a two-continent round trip closes naturally
— the padding advice in 1.4 only matters for single-map loops.

### 3.2 The rules that will stop your server booting

**[VERIFIED] Every map on the route must exist in the server's `Map.dbc`.** A node naming a map
that does not exist is dereferenced without a null check while the path is built — an immediate
crash at startup, not an error message.

**[VERIFIED] A multi-map route may not touch an instanceable map.** The core asserts this while
generating the path, and a failed assertion aborts the server. Dungeon, raid, battleground and
arena maps are all instanceable. Cross-map transports are for the open world only.

**[VERIFIED] Single-map transports inherit their instancing from their map.** A route entirely
inside an instanceable map is treated as an instanced transport, and instanced transports are
**not** spawned from the `transports` table at all — they have to be created by an instance script.
If you put a `transports` row in for one, it will be silently ignored.

### 3.3 What happens to everyone on board

**[VERIFIED] The hop happens in two phases of the same world tick**: the transport marks itself for
a delayed teleport and unloads its static passengers, then, in the delayed-update phase that runs
immediately after every map's normal update, it switches maps, moves itself, and re-adds itself to
the new map.

Who survives:

| On board | Outcome |
|---|---|
| **Players** | **Carried across.** Teleported to the destination with their transport attachment preserved, keeping their deck position. |
| **Creatures** | **Dropped.** Removed as passengers, then re-created from the destination map's copy of the spawn list. |
| **Gameobjects** | **Dropped**, and re-created the same way. |
| **Dynamic objects** | Removed outright. |

**The practical implication:** a crewman is not the same creature after the crossing. Anything
tracked at runtime — combat state, a script variable, a temporary summon, an in-progress
conversation — is gone. **[CONVENTION]** Script arrival behaviour off the arrival *event*, not off
state you expect an NPC to have carried across.

**[VERIFIED] Same-map jumps still teleport players.** A `Flags = 1` node within one map forces the
same player-teleport path, so the client resynchronises. This is how the classic
Undercity-to-Grom'gol zeppelin covers the length of the Eastern Kingdoms: a single mid-route jump.

### 3.4 Pseudo-maps on a cross-map route

The pseudo-map in `Data6` is independent of the maps the route visits — it is just a filing key.
**[VERIFIED]** Static passengers are unloaded before the hop and re-created afterwards from the same
spawn list, so one pseudo-map serves the whole journey. You do not need one per continent.

---

## Part 4 — Worked example: a new ship, end to end

This builds a complete, working ship called **The Seafarer**: a ferry with two docks, a crew on
deck, and a dockmaster who announces arrivals — all from SQL plus three DBC files
(`TaxiPath.dbc`, `TaxiPathNode.dbc`, and one copied `Map.dbc` row).

### 4.1 What this example does, and one deliberate choice

**The route reuses the geometry of an existing, known-good ferry run** — a circuit of open sea off
the west coast of Kalimdor. That is intentional. Your first custom transport should not be able to
fail because a node landed on a reef, so the example starts from coordinates already proven to be
navigable, already padded for a seamless loop, and already at the correct height for a boat.

You will therefore see **two vessels sharing the route** once it runs. That is the point: the
existing one is your control. Once yours sails correctly, replace the coordinates with your own
(Part 1.4) and the rest of the setup does not change.

Everything below uses stock 3.3.5a data only, so it reproduces on any unmodified setup.

### 4.2 Pick your IDs

**Check these are free in your own client and server copies before continuing** (Part 1.2). The
values here are clear of stock 3.3.5a, but not necessarily clear of whatever else you have
installed.

| Thing | Value here |
|---|---|
| `gameobject_template.entry` | `900100` |
| `TaxiPath.ID` | `2000` |
| `TaxiPathNode.ID` range | `50000`–`50018` |
| Pseudo-map (`Map.dbc` ID) | `800` |
| `transports.guid` | `100` |
| `displayId` | `3015` — the standard ship model, already present |
| Arrival/departure event ids | `900001`–`900004` |

### 4.3 DBC work

Remember: **every DBC change goes into both the client patch and the server's DBC folder**, byte
for byte (Part 1.1).

#### `TaxiPath.dbc` — one new row

| ID | FromTaxiNode | ToTaxiNode | Cost |
|---|---|---|---|
| 2000 | 0 | 0 | 0 |

`(0, 0)` is unused in stock data — **verify it is unused in yours** (Part 1.3), and if not, add two
`TaxiNodes.dbc` rows and use their IDs instead.

#### `TaxiPathNode.dbc` — 19 new rows

All on map `1`, all at `Z = 0` because it is a boat. Nodes 16–18 repeat nodes 0–2: that is the
padding that makes the loop close seamlessly (Part 1.4).

| ID | PathID | NodeIndex | ContinentID | X | Y | Z | Flags | Delay | Arrival | Departure |
|---|---|---|---|---|---|---|---|---|---|---|
| 50000 | 2000 | 0 | 1 | -4198.493 | 2783.839 | 0 | 0 | 0 | 0 | 0 |
| 50001 | 2000 | 1 | 1 | -4235.141 | 2589.259 | 0 | 0 | 0 | 0 | 0 |
| 50002 | 2000 | 2 | 1 | -4257.384 | 2479.412 | 0 | 0 | 0 | 0 | 0 |
| 50003 | 2000 | 3 | 1 | -4352.362 | 2441.197 | 0 | **2** | **30** | **900001** | **900002** |
| 50004 | 2000 | 4 | 1 | -4617.479 | 2461.326 | 0 | 0 | 0 | 0 | 0 |
| 50005 | 2000 | 5 | 1 | -4776.117 | 2630.971 | 0 | 0 | 0 | 0 | 0 |
| 50006 | 2000 | 6 | 1 | -5070.033 | 3060.514 | 0 | 0 | 0 | 0 | 0 |
| 50007 | 2000 | 7 | 1 | -5125.462 | 3334.675 | 0 | 0 | 0 | 0 | 0 |
| 50008 | 2000 | 8 | 1 | -5125.955 | 3813.640 | 0 | 0 | 0 | 0 | 0 |
| 50009 | 2000 | 9 | 1 | -4838.082 | 3964.543 | 0 | 0 | 0 | 0 | 0 |
| 50010 | 2000 | 10 | 1 | -4384.695 | 3932.678 | 0 | 0 | 0 | 0 | 0 |
| 50011 | 2000 | 11 | 1 | -4220.626 | 3763.554 | 0 | 0 | 0 | 0 | 0 |
| 50012 | 2000 | 12 | 1 | -4201.353 | 3475.157 | 0 | 0 | 0 | 0 | 0 |
| 50013 | 2000 | 13 | 1 | -4200.269 | 3281.396 | 0 | **2** | **30** | **900003** | **900004** |
| 50014 | 2000 | 14 | 1 | -4206.146 | 3030.528 | 0 | 0 | 0 | 0 | 0 |
| 50015 | 2000 | 15 | 1 | -4201.542 | 2907.536 | 0 | 0 | 0 | 0 | 0 |
| 50016 | 2000 | 16 | 1 | -4198.430 | 2783.702 | 0 | 0 | 0 | 0 | 0 |
| 50017 | 2000 | 17 | 1 | -4234.843 | 2589.283 | 0 | 0 | 0 | 0 | 0 |
| 50018 | 2000 | 18 | 1 | -4257.580 | 2479.442 | 0 | 0 | 0 | 0 | 0 |

Two stops, 30 seconds each, each with an arrival and a departure event.

#### `Map.dbc` — one new row for the pseudo-map

This file has around 66 columns, most of which are localisation and flags. **[CONVENTION] Do not
build a row from scratch** — copy an existing transport map row and change three things:

| Field | Value |
|---|---|
| `ID` | `800` |
| `Directory` | `Transport900100` (any unique string) |
| `MapName` (your locale's column) | `Transport: The Seafarer` |

**Leave `InstanceType` at `0`.** Part 1.8 explains why — a non-zero value silently deletes every
gameobject you place on the deck.

### 4.4 The SQL

Written with explicit column lists so it survives schema differences between cores, and with
deletes in front so it can be re-run safely.

```sql
-- ===========================================================
--  The Seafarer - a custom transport
-- ===========================================================
SET @ENTRY   := 900100;   -- gameobject_template.entry
SET @PATH    := 2000;     -- TaxiPath.ID
SET @PMAP    := 800;      -- passenger pseudo-map (Map.dbc row)
SET @GUID    := 100;      -- transports.guid
SET @DISPLAY := 3015;     -- ship model
SET @CGUID   := 9001000;  -- first creature spawn guid (must be free)
SET @CREW    := 3084;     -- any creature_template entry you want on deck

-- --- the transport template -------------------------------
DELETE FROM `gameobject_template` WHERE `entry` = @ENTRY;
INSERT INTO `gameobject_template`
  (`entry`,`type`,`displayId`,`name`,`IconName`,`castBarCaption`,`unk1`,`size`,
   `Data0`,`Data1`,`Data2`,`Data3`,`Data4`,`Data5`,`Data6`,`Data7`,`Data8`,
   `AIName`,`ScriptName`) VALUES
  (@ENTRY, 15, @DISPLAY, 'The Seafarer', '', '', '', 1,
   @PATH,   -- Data0 taxiPathId
   30,      -- Data1 moveSpeed      (yards/sec)
   1,       -- Data2 accelRate      (never 0)
   0,       -- Data3 startEventID   (client-side)
   0,       -- Data4 stopEventID    (client-side)
   1,       -- Data5 transportPhysics (1 = ship)
   @PMAP,   -- Data6 passenger pseudo-map
   0,       -- Data7 worldState1
   0,       -- Data8 canBeStopped
   'SmartGameObjectAI', '');

-- --- faction / flags (omit this and the transport flag is lost)
DELETE FROM `gameobject_template_addon` WHERE `entry` = @ENTRY;
INSERT INTO `gameobject_template_addon`
  (`entry`,`faction`,`flags`,`mingold`,`maxgold`) VALUES
  (@ENTRY, 0, 40, 0, 0);

-- --- the spawn --------------------------------------------
-- No position here: the vessel starts at the first *surviving* node of @PATH.
-- Node 0 is trimmed (Part 1.4), so that is node 1.
DELETE FROM `transports` WHERE `entry` = @ENTRY OR `guid` = @GUID;
INSERT INTO `transports` (`guid`,`entry`,`name`,`ScriptName`) VALUES
  (@GUID, @ENTRY, 'The Seafarer - custom ferry', '');

-- --- crew on deck -----------------------------------------
-- map = the pseudo-map; positions are OFFSETS from the ship origin.
-- These three offsets are real main-deck positions for this hull model.
DELETE FROM `creature` WHERE `guid` BETWEEN @CGUID AND @CGUID+2;
INSERT INTO `creature`
  (`guid`,`id`,`map`,`zoneId`,`areaId`,`spawnMask`,`phaseMask`,`modelid`,`equipment_id`,
   `position_x`,`position_y`,`position_z`,`orientation`,
   `spawntimesecs`,`wander_distance`,`currentwaypoint`,`curhealth`,`curmana`,`MovementType`) VALUES
  (@CGUID,   @CREW, @PMAP, 0, 0, 1, 1, 0, 0,  -2.2334,  2.55383, 6.09902, 1.57667,  300, 0, 0, 1, 0, 0),
  (@CGUID+1, @CREW, @PMAP, 0, 0, 1, 1, 0, 0,  -9.323,   -1.66992, 6.09808, 0.0174532, 300, 0, 0, 1, 0, 0),
  (@CGUID+2, @CREW, @PMAP, 0, 0, 1, 1, 0, 0,  21.2882, -6.49847, 6.34678, 3.66717,  300, 0, 0, 1, 0, 0);

-- --- dock announcements, no C++ required ------------------
-- event_type 71 = "gameobject event inform"; event_param1 = the taxi node's event id.
DELETE FROM `smart_scripts` WHERE `entryorguid` = @ENTRY AND `source_type` = 1;
INSERT INTO `smart_scripts`
  (`entryorguid`,`source_type`,`id`,`link`,`event_type`,`event_phase_mask`,`event_chance`,`event_flags`,
   `event_param1`,`event_param2`,`event_param3`,`event_param4`,`event_param5`,
   `action_type`,`action_param1`,`action_param2`,`action_param3`,`action_param4`,`action_param5`,`action_param6`,
   `target_type`,`target_param1`,`target_param2`,`target_param3`,`target_param4`,
   `target_x`,`target_y`,`target_z`,`target_o`,`comment`) VALUES
  (@ENTRY,1,0,0,71,0,100,0, 900001,0,0,0,0,  1,0,0,0,0,0,0,  19,@CREW,100,0,0, 0,0,0,0,
   'The Seafarer - arrive dock A - crew says line 0'),
  (@ENTRY,1,1,0,71,0,100,0, 900004,0,0,0,0,  1,1,0,0,0,0,0,  19,@CREW,100,0,0, 0,0,0,0,
   'The Seafarer - depart dock B - crew says line 1');
```

The two script rows need matching `creature_text` entries for `@CREW` (groups `0` and `1`) to
produce speech. Without them the event still fires and simply says nothing — harmless.

### 4.5 Bring it up

1. **Deploy the DBC changes to both sides.** Client patch archive *and* server DBC folder. If you
   re-extract rather than hand-copy, **empty the extractor's output `dbc/` folder first** — it skips
   files that already exist and will hand you your old DBCs back while reporting success.
2. **If you changed `GameObjectDisplayInfo.dbc`, re-run the vmap extraction** (Part 5), clearing its
   working directory first. This example reuses an existing model, so you can skip it.
3. **Run the SQL.**
4. **Restart worldserver.** **[VERIFIED] There is no reload command** for `gameobject_template`,
   `gameobject`, `creature`, `transports`, or any DBC. A restart is mandatory.
5. **Bump the client cache version — if you edited an entry a client has already seen.** The client
   caches gameobject query responses on disk, and that cache holds the `DataN` values: the taxi path
   id, the speed and the acceleration. Re-point `Data0`, retune `Data1`/`Data2`, or edit a shipped
   boat, and returning clients keep simulating the old route while the server runs the new one —
   with nothing logged on either side. Raise the cache version your core exposes for this (a
   `ClientCacheVersion` setting, backed by a value in the world database), which is sent at login and
   makes the client discard the cached data. For local testing you can instead delete the client's
   `Cache/WDB/` folder. **A brand-new entry, like this example, has never been queried, so this does
   not apply on first deployment.**
6. **Ship the patch archive to every player.** Each client that needs to see or board the vessel
   must hold the same `TaxiPath.dbc` and `TaxiPathNode.dbc` bytes. The server never sends the
   transport's position (Part 1.1), so a client without them is told the object exists but cannot
   draw it, and its player cannot board it. Nothing is logged on either side. This is easy to miss
   because it works perfectly on the machine you built it on.

### 4.6 Verify it, in order

Work down this list. Each step isolates a different failure.

**1 — Did the template load?**
Watch the startup log for the transport template count. If it did not increase, your row is not
`type = 15`, or `Data0` is `0`, or the path id is invalid.

**2 — Did it spawn?**
Look for the "spawned N continent transports" line. **[VERIFIED] If the count did not go up and
nothing was logged, the `transports` row references an entry that is not `type = 15`** — the single
silent failure in the system.

**3 — Check for load errors.** Search the SQL error log for your entry, your pseudo-map, and your
spawn guids. Missing-map and invalid-display errors both appear here.

**4 — Find it in the world.** Fly to the first *surviving* keyframe — node 1,
`(-4235.141, 2589.259)`, **not** node 0, which is trimmed (Part 1.4). The vessel starts there at
server boot and moves continuously, so allow for it being partway round.

**5 — Stand on it.** The deck should be solid. **If you fall through, the model has no collision
entry** — see Part 5. This failure is silent.

**6 — Check your offsets.** Use the position-reporting GM command while aboard. It should print
your pseudo-map and small offset coordinates. If it does not mention a transport at all, the server
does not think you are aboard — the usual cause is a client/server path mismatch (Part 1.1).

**7 — Ride a full circuit.** Confirm it stops at both docks for 30 seconds, that the crew is
standing on deck rather than hovering or sunk into it, and — critically — that **the vessel does
not jump at the end of the loop**. A snap-back means your padding is wrong (Part 1.4).

**8 — Watch the dock events.** Your script rows should fire on arrival and departure.

**9 — Log out on board, then back in.** You should return to the deck, not your homebind. Being
sent home means your offsets exceeded the limits in Part 1.8.

### 4.7 Second example: a custom elevator

Much shorter, because an elevator is a template plus its addon row, one `gameobject` row, and
animation frames.

```sql
SET @EENTRY := 900200;
SET @EGUID  := 9002000;

DELETE FROM `gameobject_template` WHERE `entry` = @EENTRY;
INSERT INTO `gameobject_template`
  (`entry`,`type`,`displayId`,`name`,`IconName`,`castBarCaption`,`unk1`,`size`,
   `Data0`,`Data1`,`Data2`,`AIName`,`ScriptName`) VALUES
  (@EENTRY, 11, 455, 'Custom Lift', '', '', '', 1,
   0,     -- Data0 pause      (0 = continuously cycling)
   0,     -- Data1 startOpen
   0,     -- Data2 autoCloseTime
   '', '');

-- --- faction / flags (same as every shipped elevator)
DELETE FROM `gameobject_template_addon` WHERE `entry` = @EENTRY;
INSERT INTO `gameobject_template_addon`
  (`entry`,`faction`,`flags`,`mingold`,`maxgold`) VALUES
  (@EENTRY, 0, 40, 0, 0);

DELETE FROM `gameobject` WHERE `guid` = @EGUID;
INSERT INTO `gameobject`
  (`guid`,`id`,`map`,`zoneId`,`areaId`,`spawnMask`,`phaseMask`,
   `position_x`,`position_y`,`position_z`,`orientation`,
   `rotation0`,`rotation1`,`rotation2`,`rotation3`,
   `spawntimesecs`,`animprogress`,`state`) VALUES
  (@EGUID, @EENTRY, 0, 0, 0, 1, 1,
   0, 0, 0, 0,                       -- put your own position here
   0, 0, 0, 0,                       -- derived from orientation automatically
   300, 100, 1);
```

Then add `TransportAnimation.dbc` rows **keyed on `900200`**, using the cycle table in Part 2.3.
Deploy to both client and server, and restart.

**[VERIFIED] Without animation rows the lift will not move**, and nothing is logged. That is the
first thing to check if it sits still.

---

## Part 5 — Using your own model

Reusing a Blizzard hull needs no art work at all: pick an existing `displayId` and you are done.
This part is for putting your own model in.

### 5.1 `GameObjectDisplayInfo.dbc`

| # | Field | Meaning |
|---|---|---|
| 0 | `ID` | The `displayId` your `gameobject_template` points at. |
| 1 | `ModelName` | Path to the model **inside the client archive**. |
| 2–11 | `Sound[10]` | Sound slots. Not used by the server. |
| 12–14 | `GeoBoxMin` | Bounding box minimum, model space. |
| 15–17 | `GeoBoxMax` | Bounding box maximum, model space. |
| 18 | `ObjectEffectPackageID` | Not used by the server. |

**[VERIFIED] The file extension in `ModelName` decides how the model is treated:**

| Extension | Treatment |
|---|---|
| `.wmo` | Extracted as a world model. This is what every Blizzard transport uses. |
| `.mdx`, `.m2` | Extracted as a doodad model. Used by most elevators. |
| `.mdl` | **Skipped entirely.** No collision will ever be generated. |

**[CONVENTION] Use a WMO for anything walkable.** Every shipped ship and zeppelin is a WMO; the
`.m2` display ids are elevator platforms.

**[VERIFIED] `GeoBoxMin`/`GeoBoxMax` are used for interaction range.** They are swapped
automatically if inverted. A box of all zeros is legal — some shipped transport models have one
— but interaction then degrades to a simple distance test from the object's origin, which on a
hundred-yard hull is not what you want. Fill it in for anything clickable.

**[CONVENTION] Model origin matters more than anything else here.** Passenger offsets, the ±75
movement limit and the path itself are all relative to the model's origin. Put it at the waterline
in the middle of the deck, not at a corner of the bounding box.

### 5.2 Collision — the step that is easy to skip and silent when missed

Server-side collision for gameobjects does **not** come from the DBC at runtime. It comes from a
compiled model index built by the extraction tools, keyed by `displayId`.

**[VERIFIED] The extractor reads `GameObjectDisplayInfo.dbc` from the client archives, not from the
server's extracted DBC folder.** This catches people out constantly. Dropping an edited DBC into
the server folder gives the server the row but produces no collision, because the extractor never
saw it.

**The order is therefore fixed:**

1. Put the model files into the client patch archive.
2. Put the new `GameObjectDisplayInfo` row into the **client's** DBC, inside that archive.
3. **Empty the extractor's output first.** Delete its working directory and the old `vmaps/` output.
   **[VERIFIED]** The extractor refuses to start at all if it finds its own previous working
   directory — it prints that the output directory is polluted and exits *before extracting
   anything*, which on a previously-extracted server is what will happen every time. It also skips
   any model whose converted file is already present, so a model re-exported under its old filename
   is silently kept at the old geometry.
4. Run the **vmap extractor** against the client. It walks the display info table and extracts
   every referenced model.
5. Run the **vmap assembler**. This produces the compiled gameobject model index.
6. Copy the resulting vmap data — including that index — to the server's data folder.
7. Copy the DBC to the server's DBC folder too, so the server has the row.
8. Restart.

**[VERIFIED] If the display id is missing from the compiled index, nothing is logged at all.** The
object loads, renders on the client, and has no collision. Players walk through the hull and fall
into the sea. There is no error, no warning, and no clue in any log.

**[VERIFIED] The one case that does log** is a model with degenerate bounds, reported as having
zero bounds and skipped. If you see that, your export is broken.

**[VERIFIED] A transport with an invalid `displayId` is never validated.** The check that rejects
bad display ids only runs for rows in the `gameobject` table — and a type-15 transport has no such
row. It will spawn happily with a nonexistent model.

**[CONVENTION] Quick way to check the server knows your display id:** temporarily create a normal
gameobject template using it and try to spawn it with the GM object-add command. That path *does*
validate the display id and will tell you if it is missing. Delete it afterwards.

This only proves the **DBC row** exists. It says nothing about collision — that check never consults
the compiled model index and never complains when a display id is missing from it. To test
collision, spawn the temporary object and try to stand on it. If you fall through, the extraction
failed, not the DBC.

### 5.3 Navigation

**[VERIFIED] Navmesh data is not generated for transport interiors**, and every NPC created on a
transport is flagged to ignore pathfinding. Deck NPCs move in straight lines. Nothing you can do to
the model changes this — design decks so that straight-line movement between waypoints does not
pass through railings or masts.

---

## Part 6 — Troubleshooting

### 6.1 Server will not start after adding a transport

All of these are crashes rather than error messages, because the relevant data is never validated.

| Symptom | Cause | Fix |
|---|---|---|
| Crash during startup, shortly after DBC load | A `TaxiPathNode.PathID` higher than any `TaxiPath.ID` | Add the `TaxiPath.dbc` row (Part 1.2) |
| Crash while loading transport templates | `NodeIndex` not dense from `0` | Renumber with no gaps (Part 1.5) |
| Crash while loading transport templates | Path has fewer than two nodes | Add nodes |
| Crash while loading transport templates | A node's `ContinentID` names a map with no `Map.dbc` row | Fix the map id |
| Abort during startup | Multi-map route touching an instanceable map | Cross-map routes are open-world only (Part 3.2) |
| Crash on the first movement tick | Total route duration computed as zero | Check `accelRate` and `moveSpeed` are non-zero, and that no two nodes share a position |

An error line reading roughly *"have data0=N but TaxiPath (Id: N) not exist"* is a **warning that a
crash is coming** — the path id is in range but has no usable nodes.

### 6.2 Transport does not appear

| Symptom | Cause |
|---|---|
| **Nothing logged at all**, spawn count unchanged | **[VERIFIED]** The `transports` entry is not `type = 15`, or has no `gameobject_template` row. The silent failure — check this first. |
| Nothing logged, template count unchanged | `Data0` is `0` — silently skipped |
| *"invalid path specified in Data0"* | Path id above the highest `TaxiPath.ID` |
| Template loads, never spawns, route is inside an instance map | Instanced transports are not spawned from the `transports` table (Part 3.2) |

### 6.3 It spawns but behaves wrongly

| Symptom | Likely cause |
|---|---|
| **Visible to you in the wrong place**, or players cannot board something they can see | **Client and server taxi tables disagree** (Part 1.1). The most common cause of "boats are broken". |
| Wrong position, but the client and server DBCs are byte-identical | Stale client-side gameobject query cache, after editing `DataN` on an entry a client had already seen. Bump the cache version or clear the client's cache folder (Part 4.5) |
| **Works on your client, invisible or un-boardable for everyone else** | Those clients do not have your patch archive (Part 4.5, step 6) |
| **Players fall through the deck** | Display id missing from the compiled model index (Part 5.2). Silent. |
| **Jumps position once per loop** | Closing leg not padded (Part 1.4) |
| Never moves at all | Only one keyframe survived path generation — the movement tick returns immediately at one or fewer. Usually a three-node path whose ends were trimmed (Part 1.4), or a map change that consumed one. Add nodes, or make an end node a stop so it is not trimmed. A zero route duration does **not** stall it — that crashes the server; see 6.1 |
| Moves but never stops | No node has `Flags` exactly `2` — remember it is not a bitmask |
| Stops for far too long | `Delay` entered in milliseconds; it is **seconds** |
| Crosses the map in view of shore | Put the transition nodes further out (Part 3.1) |

### 6.4 Passengers

| Symptom | Likely cause |
|---|---|
| Deck completely empty | `Data6` does not match the spawn rows' `map`, or `spawnMask` has bit 0 unset |
| *"spawned at nonexistent map"* in the SQL log | The pseudo-map has no `Map.dbc` row on the server (Part 1.8) |
| NPCs appear but objects do not | Pseudo-map's `InstanceType` is `1` or `2` (dungeon/raid) and it has no instance template row — every gameobject passenger is dropped with a misleading "invalid coordinates" message. Creatures are unaffected |
| **Two of every crewman** | Two transports sharing one pseudo-map in `Data6` |
| NPCs hover above or sink into the deck | Offsets wrong — stand where you want them and read your own offsets |
| A placed object sits motionless in the ocean | Written with the GM object-add command, which is not transport-aware (Part 1.8) |
| **Player teleported home on login** | Offsets exceed ±250 |
| **Player stops moving / rubber-bands on deck** | Player is beyond ±75 from the origin; movement packets are being dropped |
| Passengers vanish after a continent crossing | Expected — only players cross (Part 3.3) |
| Spawn rejected: *"already a member of spawn group 1"* | Transport spawns cannot be in spawn groups |
| Passenger never appears and is in a pool or game event | Both suppress the registration the transport reads from |

### 6.5 Scripts

| Symptom | Likely cause |
|---|---|
| Script never fires | `AIName` not set to the smart-AI gameobject handler |
| Script never fires | `source_type` is `7` — **[VERIFIED]** not implemented; use `1` |
| Script never fires | `event_param1` does not match the node's event id |
| Script never fires | The node has no event id at all — two shipped routes have none |
| Event fires, nothing visible happens | The talk target has no matching text rows, or no creature of that entry is within range |
| C++ script never runs | Bound via `transports.ScriptName`, which **[VERIFIED]** is never read — use `gameobject_template.ScriptName` |

### 6.6 Elevators

| Symptom | Likely cause |
|---|---|
| Does not move | No `TransportAnimation` rows for that **gameobject entry**. Silent. |
| Does not move | `Data1` (`startOpen`) is non-zero, so the object spawns in the active state and its animation counter never advances. Silent |
| Does not move | `Data0` (`pause`) is non-zero. This does **not** stop the server animating it, but the value is published to clients and parks the platform — set it to `0` |
| Moves for some players only | Client DBC copies differ between them |
| Jumps at the end of its cycle | First and last animation frames are not at the same position |
| NPCs will not ride it | **[VERIFIED]** Not possible — use a type-15 with a vertical path instead |

### 6.7 General

| Symptom | Cause |
|---|---|
| Changes have no effect | **[VERIFIED]** There is no reload command for `gameobject_template`, `gameobject`, `creature`, `transports` or any DBC — restart worldserver. The script tables are the exception: `.reload smart_scripts` and `.reload event_scripts` both work, so you can iterate on Part 1.9 scripting without restarting |
| Cannot teleport to the pseudo-map | By design — the GM teleport commands explicitly refuse transport maps. Go to the vessel's world position instead. |
| A second ship on the same template | Not possible — `transports.entry` is unique. Add a second template. |

---

## Part 7 — Reference

### 7.1 `gameobject_template`, type 15 (`MAP_OBJ_TRANSPORT`)

| Column | Name | Read by server? | Notes |
|---|---|---|---|
| `Data0` | `taxiPathId` | Yes | Required. `0` = silently skipped |
| `Data1` | `moveSpeed` | Yes | Integer yards/sec. Never `0` |
| `Data2` | `accelRate` | Yes | Integer yards/sec². Never `0`. `1` on 21 of the 30 shipped type-15 templates; the raid gunships use `10` and the raid zeppelin `5` |
| `Data3` | `startEventID` | No | Client-side |
| `Data4` | `stopEventID` | No | Client-side |
| `Data5` | `transportPhysics` | No | Only `1`, `21`, `61` exist |
| `Data6` | `mapID` | Yes | Passenger pseudo-map; `0` = none |
| `Data7` | `worldState1` | No | |
| `Data8` | `canBeStopped` | Yes | `1` allows halting at stops |

### 7.2 `gameobject_template`, type 11 (`TRANSPORT`)

| Column | Name | Read by server? | Notes |
|---|---|---|---|
| `Data0` | `pause` | Yes | Non-zero stops it counting as a moving transport |
| `Data1` | `startOpen` | Yes | Overrides the spawn row's `state` |
| `Data2` | `autoCloseTime` | Effectively no | Milliseconds on 3.3.5 (the `/65536` the name implies was removed in 3.0.3), but never read for type 11 |
| `Data3` | `pause1EventID` | **No** | Dead field |
| `Data4` | `pause2EventID` | **No** | Dead field |
| `Data5` | `mapID` | No | Not used for type 11 |

### 7.3 `transports`

| Column | Notes |
|---|---|
| `guid` | Unique spawn id |
| `entry` | `gameobject_template.entry`, `type = 15`. **Unique** — one vessel per template |
| `name` | Never read |
| `ScriptName` | Never applied |

### 7.4 `TaxiPath.dbc`

| # | Field | Notes |
|---|---|---|
| 0 | `ID` | Referenced by `Data0` |
| 1 | `FromTaxiNode` | Duplicate `(From, To)` pairs silently clobber each other |
| 2 | `ToTaxiNode` | As above |
| 3 | `Cost` | `0` on transports |

### 7.5 `TaxiPathNode.dbc`

| # | Field | Notes |
|---|---|---|
| 0 | `ID` | Unique across the file |
| 1 | `PathID` | Must not exceed the highest `TaxiPath.ID` |
| 2 | `NodeIndex` | Dense, zero-based, no gaps |
| 3 | `ContinentID` | Must be a real map |
| 4–6 | `X`, `Y`, `Z` | World coordinates. Boats use `Z = 0` |
| 7 | `Flags` | `0` normal, `1` teleport, `2` stop. Teleport is tested as a bitmask (`& 1`), stop as exact equality — so `3` teleports but is not a stop |
| 8 | `Delay` | **Seconds** |
| 9 | `ArrivalEventID` | Fires `event_scripts` and the gameobject AI |
| 10 | `DepartureEventID` | As above |

### 7.6 `TransportAnimation.dbc` / `TransportRotation.dbc`

| # | `TransportAnimation` | `TransportRotation` |
|---|---|---|
| 0 | `ID` | `ID` |
| 1 | `TransportID` — **gameobject entry** | `GameObjectsID` — **gameobject entry** |
| 2 | `TimeIndex` (ms) | `TimeIndex` (ms) |
| 3–5 | `X`, `Y`, `Z` offsets | `X`, `Y`, `Z` |
| 6 | `SequenceID` | `W` |

Cycle length = the largest `TimeIndex` present for that entry.

### 7.7 Passenger spawn rows

| Column | Value |
|---|---|
| `map` | The pseudo-map from `Data6` |
| `position_x/y/z`, `orientation` | Offsets from the transport origin |
| `zoneId`, `areaId` | `0` |
| `phaseMask` | `1` |
| `spawnMask` | `1` for continent transports (bit 0 must be set) |

Keep offsets within **±75** on every axis.

### 7.8 Which tables you actually need

| Table | Status |
|---|---|
| `gameobject_template` | **Required** |
| `transports` | **Required** for a continent transport |
| `gameobject_template_addon` | Optional, but omitting it loses faction and flags |
| `creature` / `gameobject` | Optional — passengers |
| `creature_addon` + waypoint data | Optional — walking crew |
| `smart_scripts` | Optional — the SQL scripting route |
| `event_scripts` | Optional — alternative scripting route |
| `gameobject_addon` | Rarely — parent rotation for animated sub-models |
| `spawn_group`, pools, game events | **Do not use** with transport spawns |
| `transport_template`, `transport_animation`, `transport_rotation` | **[FORK]** Do not exist on 3.3.5 TrinityCore |

### 7.9 Quick checklist for a new type-15 transport

1. Free IDs confirmed in **both** client and server DBC copies.
2. `TaxiPath.dbc` row added.
3. `TaxiPathNode.dbc` rows added — dense indices, ≥2 nodes, padded for a seamless loop.
4. Both DBCs deployed to **client and server**, identical.
5. Pseudo-map added to `Map.dbc` with `InstanceType = 0`, if using passengers.
6. `gameobject_template` row, `type = 15`, non-zero `Data0`/`Data1`/`Data2`.
7. `gameobject_template_addon` row with `faction = 0`, `flags = 40`.
8. `transports` row.
9. Passenger rows on the pseudo-map with offset coordinates, if any.
10. vmap extraction re-run, if a new `displayId` was added — with its working directory cleared
    first, or it will silently keep the old output.
11. worldserver restarted.
12. Client cache version bumped, if you edited an entry clients had already seen (Part 4.5).
13. Patch archive shipped to every player who needs to see or board it (Part 4.5).
14. Verified against the checklist in Part 4.6.
