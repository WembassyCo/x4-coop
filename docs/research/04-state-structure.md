# X4: Foundations — Game State Structure for Co-op Sync

**Research Ticket #4** | X4-Coop Wayfinder Map  
**Date:** 2026-10-10  
**Status:** Complete

---

## Executive Summary

X4: Foundations stores all game state in a single massive XML file (gzipped, typically 50–200 MB compressed, 500 MB – 2+ GB uncompressed for late-game saves). The save format is well-documented by the modding community and directly accessible via both the Lua/Mission Director (MD) scripting API and external named pipe IPC. This report maps the full state tree, estimates data volumes, change frequencies, and delta-compression potential for each category.

### Key Architectural Notes

1. **High-Attention vs Low-Attention Mode (OOS):** X4 uses a two-tier simulation. Entities within ~200km of the player ("high attention") get full physics/animation/AI. Everything else runs in "low attention" (abstracted stat-based simulation). This is critical for co-op: only one player can be "high attention" on a given entity at a time, and the simulation fidelity differs by location.

2. **NPCs are ship equipment, not persistent agents.** The game simulates zero NPCs at start. NPCs are spawned as needed (pilots, crew, marines, bar patrons) and tied to ships/stations. They have skills, roles, and assignments but no independent world simulation.

3. **Economy is a stock-level simulation.** Stations have cargo bays with ware quantities. Production modules consume inputs and produce outputs on cycles. Prices are determined by supply/demand relative to station stock levels. The simulation is global but abstracted OOS.

4. **Named Pipes API exists for real-time IPC.** The `sn_mod_support_apis` mod provides OS-level named pipe communication between the game's MD/Lua runtime and external processes. This is the most viable path for real-time state synchronization without modifying the game engine.

5. **Save file is XML with a well-known structure.** The root `<savegame>` contains `<info>`, `<stats>`, `<player>`, `<universe>`, `<factions>`, `<logbook>`, and `<missions>` sections. The `<universe>` section contains the `<component>` tree with all ships, stations, sectors, and zones.

---

## 1. Entity State

### 1.1 Ships

**Save XML Structure:** `<component class="ship_s|ship_m|ship_l|ship_xl">` with child elements for `weapons`, `shields`, `engines`, `cargo`, `units`, `conections` (docking), `order` (current orders), `blackboard` (AI state), `pilot` (NPC reference).

**Per-ship data fields:**
| Field | Type | Source | Notes |
|-------|------|--------|-------|
| `code` | string | XML attr | Unique ID (e.g., "ZVF-031") |
| `class` | enum | XML attr | ship_s, ship_m, ship_l, ship_xl |
| `macro` | string | XML attr | Hull type macro reference |
| `name` | string | XML attr | Display name |
| `owner` | faction ID | XML attr | Owning faction |
| `commander` | entity ID | XML attr | Commander reference (if subordinate) |
| `amount` | int | cargo tag | Hull points (current/max) |
| `shield` | array | shields tags | Per-shield-module state |
| `position` | vec3 | XML attr | x, y, z coordinates |
| `rotation` | quat | XML attr | Quaternion rotation |
| `cargo` | ware list | cargo tags | Ware type + quantity pairs |
| `orders` | order list | order tags | Queued order list with params |
| `weapons` | weapon list | weapon tags | Turret/weapon loadout + ammo |
| `pilot` | NPC ref | pilot tag | Assigned pilot NPC ID |
| `marines` | NPC list | marine tags | Assigned marines |
| `blackboard` | key-value | blackboard tags | AI behavior state variables |

**Data size estimate:**
- Per ship: ~2–8 KB XML (uncompressed), ~500–2000 bytes binary
- Typical save: 200–2,000 ships (NPC factions have hundreds); player empire 20–200 ships
- Total ship data: ~400 KB – 4 MB (uncompressed XML), ~100 KB – 1 MB binary

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Position/rotation | Per tick (high attention) / per second (OOS) | Continuous; most volatile |
| Hull/Shield | Per second (combat) / rare (idle) | Spike during combat |
| Cargo | Per trade completion (~seconds–minutes) | Stepwise changes |
| Orders | Per order completion (seconds–minutes) | Queue-based, changes are discrete |
| Weapons | Rare (refit events) | Only at equipment docks |
| Pilot/Marines | Rare (assignment changes) | Manual or AI reassignment |
| Blackboard | Per tick (AI decisions) | Internal AI state, high frequency |

**Delta compression: Excellent.** Position changes are continuous and compress well with delta encoding (typically 2–6 bytes per axis per tick). Cargo/orders change discretely and infrequently. Hull/shield are slow-changing except in combat. Blackboard state is the hardest to delta-compress (semi-random key-value updates).

### 1.2 Stations

**Save XML Structure:** `<component class="station">` with child `<production>` modules, `<cargo>` bays, `<build>` queue, `<unit>` assignments (mining ships, traders), `< defence>` settings.

**Per-station data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `code` | string | Unique ID |
| `owner` | faction ID | Owning faction |
| `position` | vec3 | Fixed (stations don't move) |
| `rotation` | quat | Fixed |
| `production modules` | list | Each has: module type, production cycle state, current inputs/outputs |
| `cargo bays` | ware list | Ware type + quantity per bay |
| `build queue` | list | Ships/stations under construction with progress |
| `assigned ships` | list | Mining/trader/military ships assigned to station |
| `defence` | settings | Turret configurations, patrol settings |
| `price overrides` | list | Manual price overrides (player stations only) |
| `stock limits` | list | Manual stock limit overrides (player stations only) |

**Data size estimate:**
- Per station: ~5–20 KB XML (uncompressed), ~1–5 KB binary
- Typical save: 200–800 stations (all factions); player empire 5–50 stations
- Total station data: ~1–8 MB (uncompressed XML), ~200 KB – 2 MB binary

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Position/rotation | Never (fixed) | No sync needed after initial load |
| Production cycle | Per cycle (seconds–minutes) | Discrete state transitions |
| Cargo | Per trade (seconds–minutes) | Continuous flux in active economies |
| Build queue | Per build completion (minutes–hours) | Progress updates per cycle |
| Assigned ships | Rare | Assignment changes are manual |
| Price overrides | Rare (player input) | Only on manual adjustment |
| Defence settings | Rare | Only on manual change |

**Delta compression: Very Good.** Position is static (one-time sync). Production state changes at discrete intervals. Cargo fluctuates but in quantized steps (ware units). Build queue progress is monotonic (increases only).

### 1.3 NPCs (Personnel)

**Save XML Structure:** NPCs are embedded within their parent component (ship/station) or in a global pool. Each NPC has: `name`, `skill` levels (piloting, morale, engineering, management, boarding, combat), `role`, `assignment`, `location` (parent entity reference).

**Per-NPC data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `id` | int | Unique NPC ID |
| `name` | string | First + last name |
| `skills` | 6× int | piloting, morale, engineering, management, boarding, combat (0–5 stars) |
| `role` | enum | captain, pilot, servicecrew, marine, manager, engineer, etc. |
| `assignment` | entity ref | Ship/station they're assigned to |
| `location` | entity ref | Where they currently are (may differ from assignment) |
| `blackboard` | key-value | AI behavior state |
| `conversation` | state | Dialog state if in conversation |

**Data size estimate:**
- Per NPC: ~500 bytes XML, ~100 bytes binary
- Typical save: 500–5,000 NPCs (most are spawned on demand)
- Total NPC data: ~250 KB – 2.5 MB XML, ~50 KB – 500 KB binary

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Skills | Rare (training events) | Only on skill improvement |
| Role/Assignment | Rare | Manual or AI reassignment |
| Location | Per docking/undocking | When parent ship moves between stations |
| Blackboard | Per tick (if active) | Only for NPCs in active conversations |
| Conversation | Per dialog step | Only during player interactions |

**Delta compression: Excellent.** NPC state is almost entirely static. Skills and assignments change rarely. Location is derived from parent entity. Only blackboard and conversation state need frequent updates, and only for NPCs currently interacting with a player.

---

## 2. Sector State

**Save XML Structure:** The universe is a hierarchy: `<galaxy>` → `<cluster>` → `<sector>` → `<zone>` → `<connection>` (content placement). Gates are `<connection>` elements linking sectors.

**Sector data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `cluster/sector/zone IDs` | int | Hierarchical identifiers |
| `sector name` | string | Display name |
| `owner` | faction ID | Controlling faction |
| `gates` | list | Gate connections to other sectors (target sector IDs + position) |
| `stations` | list | Station placements (IDs + positions) |
| `asteroid fields` | list | Field positions, asteroid count, resource types, yields |
| `celestial bodies` | list | Stars, planets, nebulae (visual only) |
| `zones` | list | Sub-regions within sector (docking zones, hazard zones) |
| `region` | ref | Economic/military region assignment |
| `security` | enum | Police presence level |

**Data size estimate:**
- Per sector: ~10–50 KB XML (uncompressed), ~2–10 KB binary
- Typical save: ~40–60 sectors (base game + all DLCs)
- Total sector data: ~400 KB – 3 MB XML, ~80 KB – 600 KB binary

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Sector ownership | Rare (war conquest) | Only on faction territory changes |
| Gate connections | Very rare (Xenon anomalies) | Almost never changes |
| Station placements | Rare (new construction/destruction) | When stations are built or destroyed |
| Asteroid fields | Rare (mining depletion) | Resource nodes deplete over time |
| Celestial bodies | Never | Static visual elements |
| Zones | Rare | Zone definitions are mostly static |
| Security level | Per minute (patrol changes) | Fluctuates with faction military activity |

**Delta compression: Excellent.** Sector structure is almost entirely static. The only dynamic elements are ownership (rare), station existence (rare), and asteroid yields (slow depletion). Initial sync at game load covers 99% of sector state; thereafter only ownership changes and station additions/removals need syncing.

---

## 3. Economy State

**Save XML Structure:** Economy data is distributed across station components. Each station's `<cargo>` tags hold ware quantities. `<production>` modules hold cycle state. Price calculations are dynamic (computed from stock levels relative to capacity, not stored as absolute values). Trade offers are stored as `<trade>` elements on stations.

**Economy data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `ware stock` | ware→quantity map | Per station cargo bay |
| `ware capacity` | ware→max int | Per station (determined by module design) |
| `production cycle` | state + progress | Per production module: current cycle, inputs consumed, outputs pending |
| `trade offers` | list | Active trade offers (buy/sell, ware, price, quantity, station) |
| `price overrides` | ware→price | Player-set prices (player stations only) |
| `supply/demand` | computed | Derived from stock/capacity ratios — not stored, computed on the fly |
| `ware exchange` | list | In-flight trade shipments (ware, quantity, source, destination, transport ship) |

**Data size estimate:**
- Per station economy: ~2–8 KB XML
- Total economy data: ~400 KB – 4 MB XML, ~100 KB – 1 MB binary
- Trade offers: ~10–200 active offers, ~50 bytes each = ~500 bytes – 10 KB
- In-flight shipments: ~50–500 active, ~100 bytes each = ~5–50 KB

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Ware stock | Per trade (seconds–minutes) | Continuous flux in active economies |
| Production cycle | Per cycle (10s–minutes) | Discrete transitions |
| Trade offers | Per trade completion | Offers appear/disappear |
| Price overrides | Rare (player input) | Only on manual adjustment |
| In-flight shipments | Per trade start/end | Appears/disappears |
| Computed prices | Continuous (derived) | Recalculated from stock levels |

**Delta compression: Good, but with caveats.** Ware stock changes are quantized (integer units) and can be delta-encoded. Production cycle progress is monotonic within a cycle. Trade offers appear/disappear as discrete events. The main challenge is the sheer number of stations (200–800) each with 5–20 wares — but most don't change between ticks. A "dirty flag" approach (only sync stations with changed stock) would be very effective.

**Key insight:** Prices are NOT stored — they're computed from stock/capacity ratios. This means you don't need to sync prices; you sync stock levels and both clients compute prices identically. The computation must be deterministic (same formula, same inputs → same output).

---

## 4. Faction State

**Save XML Structure:** `<factions>` section in save XML contains faction relations. Individual faction data includes: reputation with player, relations with other factions, budget, war/peace status.

**Faction data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `faction ID` | string | e.g., "argon", "paranid", "teladi", "xenon" |
| `player reputation` | int | -30 (mortal enemy) to +30 (trusted friend) |
| `faction relations` | map | Faction→faction: enemy, neutral, ally |
| `war/peace status` | enum | Per faction pair: at_war, at_peace, ceasefire |
| `faction budget` | int | Credits available to faction for ship/station building |
| `faction military strength` | int | Aggregate military power rating |
| `faction license held` | bool | Whether player has trade/military license with faction |
| `known/unknown` | bool | Whether player has discovered this faction |

**Data size estimate:**
- Per faction: ~200–500 bytes XML, ~50–100 bytes binary
- Total faction data: ~20 factions × 500 bytes = ~10 KB XML, ~2 KB binary
- Faction relations matrix: ~20×20 = 400 entries × ~5 bytes = ~2 KB

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Player reputation | Per significant action (kills, trades, missions) | Changes in discrete steps |
| Faction-faction relations | Rare (diplomatic events, war declarations) | Hours between changes typically |
| War/peace status | Rare (major events) | Very infrequent |
| Faction budget | Per minute (continuous spending/building) | Gradual drain/refill |
| Military strength | Per ship loss/gain | Updates when ships destroyed/built |
| License held | Rare (purchase event) | Only on license acquisition |

**Delta compression: Excellent.** Faction state is tiny (total ~10 KB) and changes very infrequently. Full-state sync every few minutes would be trivial. Delta sync of reputation changes and war/peace transitions would be near-zero bandwidth.

**Note on Dynamic Wars:** X4 has a "Dynamic Wars" system where faction relations can shift based on player actions and AI decisions. Mods like "Reactive Factions" and "War Pact" modify this. The base game's dynamic war system means faction relations can change without player input, but still slowly (hours-scale).

---

## 5. Player State

**Save XML Structure:** `<player>` section in save XML, plus `<stats>` for statistics, plus scattered references throughout the universe tree (owned ships/stations have `owner="player"`).

**Player data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `money` | int (64-bit) | Current credits; stored in `<account>` and `<stat id="money_player">` |
| `name` | string | Player name |
| `location` | entity ref | Currently controlled ship/station |
| `ships owned` | list | All ships with owner=player (derived from universe scan) |
| `stations owned` | list | All stations with owner=player |
| `blueprints` | list | Known ship/station/module blueprints (from scans/purchases) |
| `research` | list | Completed/active research projects (Terran plot) |
| `reputation` | map | Per-faction reputation values (also in faction section) |
| `licenses` | list | Trade/military licenses per faction |
| `inventory` | list | Personal items (software, upgrades, collectibles) |
| `known items` | list | Discovered wares, ship types, station types |
| `stats` | key-value | ~50+ tracked statistics (kills, trades, distance, etc.) |
| ` Headquarters` | entity ref | Player HQ station (if built) |
| `agents` | list | Diplomatic agents (v8.0+ Diplomacy update) |
| `active conversation` | state | Current dialog/NPC interaction state |

**Data size estimate:**
- Player core data: ~5–20 KB XML, ~1–5 KB binary
- Blueprint list: ~1–2 KB (list of ~200 possible blueprints, boolean owned)
- Stats: ~2 KB (50+ stat entries)
- Total player state: ~10–25 KB XML, ~3–8 KB binary

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Money | Per trade/mission (seconds–minutes) | Very frequent in active play |
| Location | Per teleport/docking (seconds–minutes) | When player switches controlled ship |
| Ships/Stations owned | Rare (purchase/destruction) | Discrete events |
| Blueprints | Rare (scan/purchase) | Discrete events |
| Research | Per research completion (minutes–hours) | Slow progress |
| Reputation | Per faction action | Discrete steps |
| Licenses | Rare | Purchase events only |
| Inventory | Rare (acquisition) | Adding/removing items |
| Stats | Per relevant action | Incremented on events |
| Agents | Per agent action | Diplomatic missions |

**Delta compression: Excellent.** Player state is small and mostly slow-changing. Money is the most frequent change (every trade), but it's a single 64-bit integer — trivial to delta-encode. Blueprint/research/license changes are discrete events with no intermediate state.

---

## 6. Mission/Quest State

**Save XML Structure:** Missions are tracked in the MD (Mission Director) system. Active missions have: mission ID, type, objectives, progress, timer state, triggered conditions. The `<missions>` section in save XML stores all active and recently completed missions.

**Mission data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `mission ID` | string | Unique mission identifier |
| `mission type` | enum | Plot, generic, faction, guild |
| `objectives` | list | Ordered objectives with completion state |
| `progress` | key-value | Per-objective progress variables |
| `triggers` | list | Conditional triggers (location, faction, timer) |
| `timer state` | int | Remaining time (if timed mission) |
| `reward state` | enum | Pending, claimed |
| `offering entity` | entity ref | NPC/station offering the mission |
| `completion state` | enum | Not started, active, completed, failed |
| `mission parameters` | key-value | Custom params (target sector, target faction, etc.) |

**Data size estimate:**
- Per mission: ~500–2000 bytes XML, ~100–500 bytes binary
- Typical active missions: 5–30 (player usually has 5–20 active)
- Total mission data: ~5–50 KB XML, ~1–10 KB binary

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Objectives | Per objective completion | Discrete events |
| Progress | Per relevant game action | Triggered by kills, scans, trades, etc. |
| Timers | Per tick (if timed) | Continuous countdown |
| Triggers | Per game event evaluation | MD evaluates conditions every tick |
| Mission offers | Per new offer (minutes–hours) | New missions appear periodically |

**Delta compression: Good.** Mission state is small and mostly event-driven (discrete changes). Timer-based missions need continuous sync but the data is tiny (a single integer). The MD system evaluates triggers every tick, but the trigger conditions themselves are static — only the trigger evaluation result (fired/not-fired) changes.

**Key insight for co-op:** Mission triggers are evaluated by the MD system on every game event. For co-op, mission state would need to be authoritative on one client (likely the host) to prevent duplicate trigger firings. Mission offers could be shown to both players but acceptance should be serialized.

---

## 7. UI State

**Save XML Structure:** UI state is partially in save files (map discovered state, some preferences) but mostly ephemeral/in-memory only.

**UI data fields:**
| Field | Type | Notes |
|-------|------|-------|
| `map view` | state | Zoom level, pan position, active filters, selected sector |
| `selected entities` | list | Currently selected ships/stations on map |
| `active menus` | state | Which menus/dialogs are open |
| `camera position` | vec3 + quat | First-person camera location and orientation |
| `active tabs` | state | Which property/info tabs are open |
| `notice board` | state | Read/unread notices |
| `overlay settings` | state | Which map overlays are enabled (trades, ships, etc.) |
| `discovered` | list | Scanned stations, discovered sectors |
| `satellite coverage` | map | Satellite-monitored areas per sector |

**Data size estimate:**
- Per client UI state: ~1–5 KB
- Total: trivial

**Change frequency:**
| Field | Frequency | Notes |
|-------|-----------|-------|
| Map view | Per frame (continuous) | Very high frequency, purely local |
| Selected entities | Per click | User-driven, discrete |
| Active menus | Per open/close | User-driven, discrete |
| Camera position | Per frame (continuous) | Very high frequency, purely local |
| Discovered/scanned | Rare | One-time discoveries |
| Satellite coverage | Per satellite deployment | Rare |

**Delta compression: Not needed — UI state should NOT be synced.** UI state is per-client and purely cosmetic. Each co-op player should have their own independent UI. The only UI-adjacent state that matters for co-op is:
- **Discovered stations/sectors**: Should be shared (if one player discovers a station, both should see it)
- **Satellite coverage**: Should be shared (both players benefit from satellite intel)
- **Scanned station data**: Should be shared (production modules, cargo levels from scans)

---

## Save File Structure Summary

Based on analysis of save file XML structure (via x4-savegame-parser, save editing guides, and XML analyzer tools):

```
<savegame>
  <info>                          — Game version, DLCs, mods, playtime
  <stats>                         — Player statistics (~50+ tracked stats)
  <player>                        — Player account, inventory, blueprints
    <account>                     — Money (appears in multiple places for consistency)
    <inventory>                   — Personal items
    <blueprints>                  — Known blueprints
    <research>                    — Research progress
  <universe>                      — The entire game world
    <component class="galaxy">    — Galaxy root
      <connections>               — Cluster connections (superhighways, gates)
        <component class="cluster">
          <sectors>
            <component class="sector">
              <zones>
                <component class="zone">
                  <connections>   — Docking bays, gates within zone
              <factions>          — Faction relations matrix
    <component class="ship_*">    — Individual ships (hundreds)
    <component class="station">   — Individual stations (hundreds)
    <component class="object">    — Drones, collectables, debris
  <factions>                      — Detailed faction data, relations, budgets
  <logbook>                       — Event log (can be very large)
  <missions>                      — Active and completed missions
  <md>                            — Mission Director state (cue trees, variables)
  <signals>                       — Pending AI signals/events
  <ai>                            — AI state aggregates
</savegame>
```

**File sizes observed by community:**
- Early game: ~20-50 MB compressed (gz), ~100-300 MB uncompressed XML
- Mid game: ~50-100 MB compressed, ~300 MB - 1 GB uncompressed
- Late game (large empire): ~100-200+ MB compressed, ~1-2+ GB uncompressed
- The x4-savegame-parser project explicitly handles "1+ GB" XML files
- Uncompressed XML can be loaded directly by the game (no recompression needed)

---

## State Access Methods

### Method 1: Save File Parsing (Offline/Batch)
- **How:** Decompress .gz save, parse XML
- **Tools:** PHP XMLReader (Mistralys parser), Python xml.sax (xixas analyzer), any XML parser
- **Granularity:** Full game state snapshot
- **Latency:** Seconds (decompress + parse)
- **Use case:** Initial state sync, save comparison, offline analysis
- **Limitation:** Save files are only written on autosave/manual save (every few minutes), not real-time

### Method 2: Lua/MD Scripting API (In-Game Real-Time)
- **How:** Lua scripts and MD (Mission Director) XML run inside the game engine
- **Available APIs** (from Lua function dump):
  - `GetComponentData()` - entity attributes (position, rotation, hull, etc.)
  - `GetCargoAfterTradeOrders()` - cargo state
  - `GetSubordinates()` - fleet relationships
  - `GetCommander()` / `GetAllCommanders()` - command hierarchy
  - `GetNPCs()` / `GetNPCBlackboard()` - NPC state
  - `GetClusters()` / `GetSectors()` / `GetGates()` - universe structure
  - `GetFactionData()` - faction info
  - `GetPlayerMoney()` / `GetAccountData()` - player finances
  - `GetStatData()` - statistics
  - `GetMissionDetails()` / `GetActiveMission()` - mission state
  - `GetTradeList()` / `GetTradeOrders()` - active trades
  - `GetProductionModuleData()` / `GetProductionModules()` - station production
  - `GetStorageData()` / `GetWareData()` / `GetWareCapacity()` - ware storage
  - `GetInventory()` / `GetPlayerInventory()` - inventory items
  - `GetLogbook()` - event log
  - `AddMoney()` / `RemoveCargo()` / `SetOrderParam()` - write operations
- **Latency:** Per-frame (scripts run in game loop)
- **Use case:** Real-time state extraction and modification
- **Limitation:** Scripts must be loaded as mods; can't inject at runtime without mod installation

### Method 3: Named Pipes IPC (Real-Time External)
- **How:** `sn_mod_support_apis` mod provides OS named pipe client in Lua; external server (Python, etc.) creates pipe and communicates with game
- **Architecture:** External process <- named pipe <- Lua client <- MD/Lua scripts <- game engine
- **Latency:** ~1 frame minimum (Lua->MD delay), practical ~16-33ms
- **Use case:** Real-time bidirectional communication with external process
- **Features:**
  - Read and write to pipes
  - Continuous read mode for streaming data
  - Error recovery on save/reload/UI reload
  - Multiple pipes supported
  - Python host server plugin system for dynamic module loading
- **Limitation:** Windows-only for Lua pipe implementation; Linux requires Proton compatibility mode

### Method 4: HTTP Client Library (Network)
- **How:** `djfhe/x4_http` mod provides HTTP client capabilities from within the game
- **Architecture:** Game Lua -> HTTP POST/GET -> External server -> Response
- **Latency:** Network-dependent (~10-100ms localhost)
- **Use case:** RESTful state synchronization with external service
- **Used by:** X4 External App (mycumycu) - demonstrates real-time logbook/mission/player data extraction via HTTP
- **Limitation:** Beta status; requires Protected UI Mode disabled

---

## Data Volume Summary

| Category | Per-Entity Size (binary) | Typical Entity Count | Total Size (binary) | Primary Change Frequency |
|----------|-------------------------|---------------------|---------------------|------------------------|
| Ships | 500-2000 B | 200-2,000 | 100 KB - 4 MB | Position: per tick; Cargo/orders: per minute |
| Stations | 1-5 KB | 200-800 | 200 KB - 4 MB | Production: per cycle; Cargo: per trade |
| NPCs | ~100 B | 500-5,000 | 50 KB - 500 KB | Rare (skills/assignments) |
| Sectors | 2-10 KB | 40-60 | 80 KB - 600 KB | Very rare (ownership changes) |
| Economy (distributed) | (included in stations) | -- | 100 KB - 1 MB | Per trade (seconds-minutes) |
| Factions | 50-100 B | ~20 | 2-10 KB | Rare (war/peace, reputation) |
| Player | 1-5 KB | 1 | 3-8 KB | Money: per trade; Location: per teleport |
| Missions | 100-500 B | 5-30 | 1-10 KB | Per objective/event |
| UI (local only) | ~1-5 KB | 1 per client | N/A (not synced) | Per frame (local) |
| **Total Syncable** | -- | -- | **~0.5-10 MB** | -- |

---

## Delta Compression Assessment

| Category | Delta Quality | Rationale |
|----------|--------------|-----------|
| Ship position/rotation | **Excellent** | Continuous small deltas; vec3 delta = 2-6 bytes/axis |
| Ship hull/shield | **Good** | Slow-changing except combat; int deltas |
| Ship cargo | **Good** | Quantized integer changes, infrequent |
| Ship orders | **Good** | Discrete queue changes, rare |
| Ship blackboard (AI) | **Fair** | Semi-random key-value updates; hard to predict |
| Station production | **Good** | Monotonic progress within cycles; discrete transitions |
| Station cargo | **Good** | Quantized integer changes per trade |
| Station build queue | **Good** | Monotonic progress; discrete additions/removals |
| NPC state | **Excellent** | Almost entirely static |
| Sector structure | **Excellent** | Nearly immutable; only ownership changes |
| Economy (ware stocks) | **Good** | Dirty-flag approach: only sync changed stations |
| Faction state | **Excellent** | Tiny total size, infrequent changes |
| Player money | **Excellent** | Single int64 delta |
| Player blueprints/research | **Excellent** | Boolean list, rare changes |
| Mission state | **Good** | Small, mostly event-driven |
| Logbook | **Poor** | Append-only, can grow large; consider ring buffer |

---

## Co-op Sync Strategy Implications

### Tier 1: One-Time Sync (Game Load)
- Sector structure (gates, zones, celestial bodies)
- Station placements (positions, types)
- Asteroid field definitions
- Faction definitions
- Player blueprints, licenses, research
- NPC skills and names

These are essentially static. Sync once at game start and never again (unless conquest/building events occur).

### Tier 2: Low-Frequency Sync (Every 10-60 seconds)
- Faction reputations and relations
- Faction budgets
- Player money
- Mission state (objectives, progress)
- Ship/station ownership changes
- Station build queue progress
- NPC assignments

Small data, infrequent changes. Full-state sync every 30-60 seconds would cost <10 KB/sync.

### Tier 3: Medium-Frequency Sync (Every 1-5 seconds)
- Station cargo levels (only dirty stations)
- Production cycle states
- Active trade offers
- In-flight trade shipments
- Player location (which ship/station controlled)
- Mission timers

Moderate data, second-to-minute changes. Delta sync of dirty stations only: ~5-50 KB/sync.

### Tier 4: High-Frequency Sync (Every tick / 10-30 Hz)
- Ship positions and rotations (high-attention ships only)
- Ship hull/shield (in combat)
- Ship AI blackboard (if needed for visible behavior)
- Active combat state (weapon fire, missile positions)

This is the bandwidth bottleneck. Only ships in high-attention zones (near either player) need this. Estimate: 10-50 ships x 20 bytes/ship/tick x 20 Hz = ~4-20 KB/s. Manageable.

### Tier 5: No Sync Required
- UI state (map views, camera, selected entities)
- Input settings
- Graphics settings
- Logbook (each client maintains own)

### Critical Design Considerations

1. **High/Low Attention Boundary:** The biggest architectural challenge. Only one "high attention bubble" exists per game instance. In co-op, either:
   - Host runs high-attention for both players' locations (heavy load)
   - Two high-attention bubbles (game engine doesn't natively support this - would require mod/engine hack)
   - One player is always "OOS" (poor experience)
   - Players must stay in the same sector (restrictive but simplest)

2. **Economic Determinism:** Since prices are computed from stock levels, both clients must have identical stock data to show identical prices. The economic simulation (production cycles, trade AI) must run authoritatively on one client (host) with results propagated to the other.

3. **Mission Director State:** The MD system evaluates triggers every tick based on game events. For co-op, the MD should run on the host only; mission offers and trigger results propagate to the guest client.

4. **NPC Simulation:** NPCs don't have independent world simulation - they're tied to ships/stations. Syncing parent entity state implicitly syncs NPC location. Only NPC skills/assignments need separate sync (and those are rare changes).

5. **Combat Resolution:** High-attention combat uses the physics engine and hit detection. This can't be meaningfully split across two clients without engine modifications. Options:
   - Host-authoritative: Host simulates combat, sends results to guest
   - Deterministic lockstep: Both clients simulate, must stay in sync (hard with floating-point)
   - Sector-lock: Only one player can be in a combat sector at a time

6. **Named Pipes as Sync Transport:** The named pipes API is the most viable real-time communication channel. Architecture would be:
   ```
   Host X4 <- Lua/MD <- Named Pipe <- Sync Server <- Named Pipe <- Lua/MD <- Guest X4
   ```
   The sync server (external Python/C++ process) would manage state diffing, throttling, and propagation between the two game instances.

---

## Sources

- **x4-savegame-parser** (Mistralys): PHP tool parsing 1+ GB save XML - reveals full XML structure and collection types (ships, stations, NPCs, sectors, zones, celestial bodies, people, event log, blueprints, Khaa'k stations, player info)
  https://github.com/Mistralys/x4-savegame-parser

- **Save XML Analyzer** (xixasdev): Python SAX parser for X4 savegame XML - confirms tag structure and attribute types
  https://gist.github.com/xixasdev/f0632aab83972985adcc7d2e11bdd6fe

- **Save Editing Guide** (Steam Community): Confirms XML structure - `<component class="ship">`, `<component class="station">`, entity IDs, account money tags, faction relations
  https://steamcommunity.com/sharedfiles/filedetails/?id=2156433664

- **Cheat/Savegame Editing Notes** (Mistralys): Details on money tags (`<account>`, `<stat id="money_player">`), entity search by `code` attribute
  https://github.com/Mistralys/x4-game-notes/blob/main/cheats-savegame-editing.md

- **Named Pipes API** (bvbohnen/x4-projects): Full documentation of MD-based named pipe IPC - read/write/continuous mode, error recovery, Python host server plugin system
  https://github.com/bvbohnen/x4-projects/blob/master/extensions/sn_mod_support_apis/documentation/Named_Pipes_API.md

- **X4 External App** (mycumycu): Working example of real-time game state extraction via HTTP - logbook, missions, player info served to external browser
  https://github.com/mycumycu/X4-External-App

- **djfhe/x4_http**: HTTP client library mod for X4 - enables RESTful communication from within game
  https://github.com/djfhe/x4_http

- **Lua API Functions** (NodusCursorius): Comprehensive list of 200+ Lua functions available to X4 mods - reveals all accessible game state
  https://gist.github.com/NodusCursorius/b61f26177fdcb490d2456e353d57f363

- **Economy Primer** (Steam Community Guide): How X4's simulated economy works - supply chains, ware stocks, production cycles, price determination
  https://steamcommunity.com/sharedfiles/filedetails/?id=2968249103

- **High/Low Attention Mode** (Reddit/Steam discussions): Explains OOS vs IS simulation fidelity differences, ~200km high-attention radius
  https://www.reddit.com/r/X4Foundations/comments/16g26g7/

- **Reddit Save Editing Guide**: Confirms save structure divisions - playerinfo section, universe section (sectors/ships/NPCs), logbook/misc section
  https://www.reddit.com/r/X4Foundations/comments/jzuw8k/guide_to_editing_savegame/
