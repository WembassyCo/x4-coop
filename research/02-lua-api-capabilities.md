# X4: Foundations Lua Modding API Capabilities & Sandbox Restrictions

**Research Ticket #2 — x4-coop wayfinder map**
**Date:** 2026-10-10
**Status:** Complete

---

## Executive Summary

X4: Foundations runs **LuaJIT** (Lua 5.1 + Just-In-Time compiler) in its UI layer. The Lua environment has partial access to standard Lua libraries but is **not a full unrestricted Lua sandbox**. Critically, LuaJIT's **FFI (Foreign Function Interface)** is available, which — when Protected UI Mode is disabled — provides a **complete escape hatch** to the operating system. This means a sufficiently motivated mod can call arbitrary Windows API functions (kernel32, user32, etc.) directly from Lua.

The game also has a second scripting system — the **Mission Director (MD)**, an XML-based scripting language — which has deep game-state manipulation capabilities (spawning entities, controlling AI, editing economy) that Lua alone cannot directly access.

**For the coop project, the key finding is: the x4_coop mod by DanyWany22 has already proven that all necessary capabilities exist.** It uses LuaJIT FFI to call Windows named pipe APIs, a Python bridge for TCP/UDP networking, MD scripts for entity creation/simulation control, and Lua UI function hooking for order/state synchronization. This is a working proof-of-concept for cooperative multiplayer in X4.

---

## 1. Can Lua scripts call `os.execute()` or spawn external processes?

**Short answer:** Not via standard Lua `os` library, but **yes via LuaJIT FFI**.

### Standard Lua `os` library
- The `os` library (including `os.execute`, `os.popen`) is **not reliably available** in X4's Lua environment. The standard library is partially stripped/sandboxed.
- `require` is available but limited. Before X4 7.50, `require` could load arbitrary Lua modules. After 7.50's **Protected UI Mode**, `require` is whitelisted to a handful of modules (e.g., `"ffi"`).

### LuaJIT FFI escape hatch
- **FFI is available** (it's part of LuaJIT and is whitelisted even in Protected UI Mode).
- With Protected UI Mode **disabled**, `ffi.C` can look up and call functions from any loaded shared library, including **kernel32.dll** on Windows.
- The x4_coop mod demonstrates calling these Windows functions directly from Lua:
  - `CreateFileA` — open named pipes
  - `ReadFile` / `WriteFile` — pipe I/O
  - `PeekNamedPipe` — non-blocking pipe reads
  - `CloseHandle` — cleanup
- The x4_coop README states: *"The Lua opens `\\.\pipe\x4_coop` with Windows' own `CreateFileA`, `ReadFile`, `WriteFile` and `PeekNamedPipe` from kernel32. It calls them through LuaJIT's FFI: on Windows, `ffi.C` also looks up kernel32."*

### Spawning external processes
- While `os.execute` isn't directly available, FFI can call `CreateProcessA` or `ShellExecuteA` from kernel32/shell32.
- The **SirNukes Mod Support APIs** mod provides a DLL that wraps pipe-related Windows functions, and includes a **Named Pipes API** specifically for inter-process communication.
- The **X4-External-App** by mycumycu demonstrates a complete external application integration pattern.
- The **X4-rest-server** by Alia5 provides a REST HTTP API by running an external server alongside the game.

### Verdict
| Method | Available? | Notes |
|--------|-----------|-------|
| `os.execute()` | ❌ Not directly | `os` library partially stripped |
| LuaJIT FFI → `CreateProcessA` | ✅ Yes | Requires Protected UI Mode OFF |
| SirNukes Named Pipes API → external process | ✅ Yes | Requires Protected UI Mode OFF, provides DLL wrapper |
| External bridge program (Python, etc.) | ✅ Yes | Standard pattern used by multiple mods |

---

## 2. Can Lua scripts do file I/O (read/write files outside the X4 save directory)?

**Short answer:** Limited via standard Lua, but **yes via FFI**.

### Standard Lua file I/O
- `io.open` and related functions are **partially available** but the environment restricts where files can be accessed.
- The game's own UI scripts use `GetSaveFolderPath()`, `IsSaveValid()`, `ReloadSaveList()` — suggesting file access is channeled through game-provided functions rather than raw `io`.
- Debug logging via `-logfile debuglog.txt` works, writing to the player's Documents/Egosoft/X4 directory.

### Via FFI
- With FFI and Protected UI Mode off, `CreateFileA`, `ReadFile`, `WriteFile` from kernel32 provide **unrestricted file system access**.
- Named pipes (which are OS-level pseudo-files) work through FFI as demonstrated by x4_coop.
- The SirNukes Named Pipes API provides a cleaner abstraction for pipe-based I/O.

### Via MD scripts
- MD scripts have `<debug_text>` for logging to the debug log file.
- MD scripts can use `<save>` actions (limited).
- No general-purpose file I/O in MD.

### Verdict
| Method | Available? | Notes |
|--------|-----------|-------|
| `io.open()` | ⚠️ Partial | Limited to game directories, not reliable for arbitrary paths |
| FFI → `CreateFileA`/`ReadFile`/`WriteFile` | ✅ Yes | Full filesystem access, requires Protected UI Mode OFF |
| Game API functions (`GetSaveFolderPath` etc.) | ✅ Yes | Limited to save/game directories |
| Named pipes via FFI or SirNukes DLL | ✅ Yes | Inter-process communication, not disk files |

---

## 3. Can Lua scripts open network sockets (TCP/UDP)?

**Short answer:** **Not directly in Lua**, but **yes via external bridge process**.

### LuaSocket
- LuaSocket is **not bundled** with X4 and cannot be loaded as a standard module.
- Pre-7.50, it might have been possible to `require("socket")` if a LuaSocket DLL was placed in the right location, but this was never a standard approach.
- Post-7.50 Protected UI Mode makes loading arbitrary DLLs via `require` impossible without disabling protection.

### FFI socket access
- In theory, FFI could call `ws2_32.dll` (Winsock) functions like `socket()`, `connect()`, `send()`, `recv()` directly.
- However, Winsock initialization (`WSAStartup`) and the complexity of async I/O makes this impractical compared to named pipes + external bridge.

### The proven pattern: Named pipe → External bridge → Network
This is the architecture used by **x4_coop**, **X4-rest-server**, **X4-External-App**, and mods using SirNukes' Named Pipes API:

```
X4 (Lua via FFI) ↔ Windows named pipe ↔ External bridge (Python/C++) ↔ TCP/UDP sockets ↔ Network
```

- **x4_coop** uses Python (standard library only) as the bridge, with encrypted UDP between two players.
- **X4-rest-server** runs an HTTP REST API server alongside the game.
- **SirNukes Named Pipes API** provides a Python pipe server (`X4_Python_Pipe_Server`) as a general-purpose bridge.

### Verdict
| Method | Available? | Notes |
|--------|-----------|-------|
| LuaSocket (`require("socket")`) | ❌ No | Not bundled, blocked by Protected UI Mode |
| FFI → Winsock (`ws2_32.dll`) | ⚠️ Theoretically possible | Very complex, no known mod does this |
| Named pipe → External bridge → Network | ✅ Yes | **Proven pattern**, used by x4_coop and others |
| SirNukes Named Pipes API + Python server | ✅ Yes | General-purpose IPC bridge |

---

## 4. What game state APIs are available?

**Short answer:** **Extensive** — via both Lua UI functions and MD script properties/actions.

### Lua UI Game State Functions
X4's own UI scripts (in `ui/addons/ego_detailmonitor/`, `ui/addons/ego_chatwindow/`, etc.) use a rich set of Lua functions. The x4_coop mod documented **63 game functions** called through FFI, all of which are also used by vanilla UI scripts:

**Entity/Position:**
- `GetObjectPositionInSector` — get any object's position
- `SetObjectSectorPos` — set object position (used by map editor)
- `GetPlayerOccupiedShipID` — which ship the player is piloting
- `GetPlayerID` — player entity ID
- `GetObjectIDCode` — unique object identifier
- `GetCurrentGameTime` — in-game time
- `IsComponentOperational` — check if component is working
- `IsComponentClass` — check component type
- `CanTeleportPlayerTo` / `TeleportPlayerTo` — player teleportation

**Ship Orders/Commands:**
- `CreateOrder` — create orders for ships
- `SetOrderParam` — modify order parameters
- `EnableOrder` / `EnablePlannedDefaultOrder` — enable/disable orders
- `GetDefaultOrder` / `GetOrders` — read current orders
- `GetNumOrders` / `RemoveAllOrders2` — order count and clearing
- `GetMissionIDObjective2` / `GetMissionIDObjectiveTarget` — mission data

**Faction Data:**
- `GetNumAllFactions` / `GetAllFactions` — enumerate all factions
- Faction relationship data accessible via MD properties

**Economy/Trade:**
- `GetContainerWareIsBuyable` / `GetContainerWareIsSellable` — trade flags
- `GetContainerBuyLimit` / `GetContainerSellLimit` — trade limits
- `SetContainerBuyLimitOverride` / `SetContainerSellLimitOverride` — set limits
- `GetContainerWarePrice` / `SetContainerWarePriceOverride` — pricing
- `SetContainerStockLimitOverride` — stock limits
- `AddTradeWare` / `RemoveTradeWare` — manage trade wares
- `GetContainerBuildPriceFactor` / `SetContainerBuildPriceFactor` — build prices
- `GetContainedStationsByOwner` — list stations by owner

**Trade Rules:**
- `CreateTradeRule` / `UpdateTradeRule` / `RemoveTradeRule`
- `GetTradeRuleInfo` / `GetTradeRuleInfoCounts`
- `SetContainerTradeRule` / `GetContainerTradeRuleID`
- `IsPlayerTradeRuleDefault` / `SetPlayerTradeRuleDefault`

**Station Management:**
- `TransferPlayerMoneyTo` / `TransferMoneyToPlayer` — player account
- `SetMinBudget` / `SetMaxBudget` — station budgets
- `SetComponentName` — rename ships/stations
- `ShouldContainerFillWorkforceCapacity` / `SetContainerWorkforceFillCapacity`

**Terraforming:**
- `StartTerraformingProject` / `AbortActiveTerraformingProject`
- `CanStartTerraformingProject` / `CanAbortActiveTerraformingProject`
- `GetTerraformingActiveProject`

**Save/Game State:**
- `GetSaveFolderPath` / `IsSaveValid` / `ReloadSaveList` / `IsSaveListLoadingComplete`
- `IsGamePaused` — check pause state
- `GetPlayerName` — player name

**Context/Lookup:**
- `GetContextByClass` — find game objects by class type

### MD Script Properties (read-only game state)
The MD system provides extensive read access to game state through property expressions:
- `player.age` — time since game start
- `player.occupiedship` — current ship
- `player.money` — player credits
- `entity.isclass.{class.ship}` — class checks
- `entity.sector` — which sector an entity is in
- `entity.position.x/y/z` — spatial coordinates
- `entity.hull` / `entity.shield` — damage state
- `faction.relation` — faction relationships
- Station cargo, ware amounts, production, etc.
- Event-driven data: `event.param` on events like `event_object_destroyed`, `event_player_killed_object`, `event_object_docked`, etc.

### Verdict
**Game state access is comprehensive.** Lua provides direct function calls for reading and modifying game state. MD provides event-driven conditions and property lookups. The combination covers virtually all game state needed for coop sync.

---

## 5. Can Lua scripts create new entities (spawn ships, create sectors)?

**Short answer:** **Not directly in Lua**, but **yes via MD scripts** which Lua can trigger.

### Lua limitations
- Lua UI scripts don't have direct "spawn ship" or "create sector" functions.
- Lua can call game functions like `CreateOrder`, `SetComponentName`, `TeleportPlayerTo`, etc., but entity creation is not in the Lua API.

### MD script capabilities
MD scripts **can create and destroy entities**:

- **`<create_ship>`** — spawn ships (used by x4_coop to create "proxy" ships for the partner player)
- **`<create_station>`** — create stations
- **`<create_object>`** — generic object creation
- **`<destroy_object>`** / **`<remove_entity>`** — destroy/remove entities
- **`<restore_object>`** — restore destroyed objects
- MD scripts can also set object positions, assign AI behaviors, set ownership, etc.

### The Lua → MD bridge
Lua and MD communicate through **`raise_lua_event`** (Lua → MD) and **`event_ui_triggered`** (MD → Lua):

1. **Lua raises events to MD:** `RaiseEvent("MyEvent", paramTable)` triggers MD cues with `<event_ui_triggered screen="'MyLuaScreen'" control="'MyEvent'"/>`
2. **MD signals Lua:** `<raise_lua_event name="'MyAction'" param="$data"/>` triggers Lua event handlers
3. **MD signals MD:** `<signal_cue cue="md.SomeCue" param="$data"/>` for MD-to-MD communication

This is exactly how x4_coop works:
- Lua detects player position/orders → raises event to MD
- MD creates proxy ship for partner → assigns AI script to follow partner's movements
- MD handles kill sync, damage sync, NPC position sync
- AI scripts (`aiscripts/`) handle weapons firing

### Sector creation
- Creating entirely new sectors is possible via MD but extremely complex — it requires defining the sector map, connections, gates, and economy.
- Most mods that "add sectors" do this through extension files (map definitions) rather than runtime MD scripts.
- For coop purposes, **sector creation is not needed** — both players exist in the same universe.

### Verdict
| Capability | Lua Direct | MD Script | Lua → MD Bridge |
|-----------|-----------|-----------|-----------------|
| Spawn ships | ❌ | ✅ `<create_ship>` | ✅ Via `raise_lua_event` |
| Create stations | ❌ | ✅ `<create_station>` | ✅ Via `raise_lua_event` |
| Destroy entities | ❌ | ✅ `<destroy_object>` | ✅ Via `raise_lua_event` |
| Create sectors | ❌ | ⚠️ Complex/runtime | Not practical at runtime |
| Set entity positions | ✅ `SetObjectSectorPos` | ✅ MD actions | Both paths work |
| Assign AI behaviors | ❌ | ✅ AI scripts | ✅ Via MD/AI scripts |

---

## 6. Can Lua scripts suppress or pause the simulation (freeze NPC AI, economy ticks)?

**Short answer:** **Partial** — pause detection and SETA sync are possible, but full simulation freeze is not directly controllable.

### Game pause
- `IsGamePaused()` exists as a Lua function — the game can detect pause state.
- The game pauses automatically when the player opens certain menus.
- MD scripts can detect pause via conditions but **cannot force-pause** the simulation.
- The **SirNukes Time API** provides real-time timers that continue during pause, but doesn't pause the game.

### SETA (Singularity Engine Time Accelerator)
- SETA accelerates game time (typically 10x).
- x4_coop **syncs SETA between players**: "whichever of you switches it on or off, the other's game follows, at the same speed."
- This is done through Lua event detection → pipe message → bridge → partner's bridge → partner's Lua → game SETA toggle.
- If the other player can't follow (no SETA, enemies nearby), it's switched off for both.

### NPC AI freeze
- **No direct "freeze all AI" function** exists in Lua or MD.
- MD scripts can set entities to "suspended" state via `<set_suspended>` (referenced in MD actions).
- Individual AI behaviors can be overridden with custom AI scripts (`.xml` files in `aiscripts/`).
- x4_coop uses custom AI scripts (e.g., `aiscripts/x4coop.proxy.fire.xml`) to make partner ships fire at targets — demonstrating AI behavior override.
- The game's "rough simulation" mode (for distant sectors) already reduces simulation detail — MD can influence which sectors get full vs. rough simulation.

### Economy tick control
- **No direct control over economy ticks** from Lua or MD.
- The economy runs on its own simulation loop.
- However, x4_coop **overrides economy state**: "every station's stock, galaxy-wide, follows the host's" — by periodically writing station cargo amounts via MD/Lua. This doesn't freeze the economy but **overwrites its results** to match the host's state.

### Verdict
| Capability | Available? | Method |
|-----------|-----------|--------|
| Detect pause | ✅ | `IsGamePaused()` in Lua |
| Force pause | ❌ | Not available via API |
| Sync SETA between players | ✅ | Proven by x4_coop |
| Freeze individual NPC AI | ⚠️ Partial | MD `<set_suspended>` or custom AI scripts |
| Freeze all NPC AI | ❌ | No global freeze function |
| Control economy ticks | ❌ | No direct control |
| Override economy results | ✅ | Periodic state writes (x4_coop pattern) |

---

## 7. Can Lua scripts read/write another player's input state?

**Short answer:** **No direct input API**, but **UI function hooking provides equivalent capability**.

### No direct input access
- X4's Lua environment does not expose raw keyboard/mouse/joystick input state.
- The SirNukes **Hotkey API** can capture key presses, but only for registered hotkeys, not arbitrary input.
- There is no API to read "what buttons the player is pressing" in real-time.

### UI function hooking (the x4_coop approach)
Instead of reading raw input, x4_coop **hooks the UI functions that the player's input triggers**:

- `CreateOrder` — when the player gives a ship an order, the hooked function calls the original then notifies the partner
- `SetOrderParam` — order modifications
- `buttonDefaultOrderConfirm` — map menu order confirmation
- `TransferPlayerMoneyTo` / `TransferMoneyToPlayer` — station account transfers
- `SetMinBudget` / `SetMaxBudget` — budget changes
- `SetComponentName` — ship/station renames
- `buttonStartProject` / `buttonAbortProject` — terraforming actions

This is **function-level interception**, not input-level. It captures the *result* of player input (the game command) rather than the raw keystroke. This is actually **better for coop sync** because:
1. It's semantic (you know *what* the player did, not just which key they pressed)
2. It's UI-agnostic (works regardless of how the player triggered the action)
3. It captures AI-initiated actions too (orders given by the game's own AI)

### What cannot be captured
- **Flight controls** (throttle, steering, mouse aim) — these are handled at the engine level, not the Lua UI level
- x4_coop solves this by reading **ship position/orientation directly** via `GetObjectPositionInSector` at 30Hz and sending that to the partner, rather than trying to capture joystick input

### Verdict
| Capability | Available? | Notes |
|-----------|-----------|-------|
| Read raw keyboard/mouse state | ❌ | No API for this |
| Capture registered hotkeys | ✅ | SirNukes Hotkey API |
| Hook UI functions (orders, trades, etc.) | ✅ | **Proven by x4_coop** — semantic interception |
| Read ship position/orientation | ✅ | `GetObjectPositionInSector` at high frequency |
| Write flight input to another player's game | ❌ | Not possible — position sync used instead |
| Write orders to another player's game | ✅ | `CreateOrder` etc. from Lua |

---

## 8. What's the Mission Director (MD) XML system capable of that Lua isn't?

### MD strengths (things MD can do that Lua cannot)

1. **Entity creation/destruction** — `<create_ship>`, `<create_station>`, `<create_object>`, `<destroy_object>`, `<restore_object>`
2. **Event-driven programming** — Rich event system: `event_object_destroyed`, `event_object_docked`, `event_player_killed_object`, `event_cue_signalled`, `event_ui_triggered`, `event_object_sector_changed`, etc.
3. **AI script assignment** — Can assign custom AI behaviors to ships (order lists, behaviors, resupply triggers)
4. **Persistent background processes** — Cues can run indefinitely, checking conditions on intervals, responding to events
5. **Game state modification** — Set values, modify relationships, control missions, trigger dialogue
6. **Signal/communication system** — Cues can signal each other with parameters (`<signal_cue>`, `<signal_cue_instantly>`)
7. **Condition checking** — Rich condition system: check ship ownership, faction relations, object proximity, object class, player money, game time, etc.
8. **Library/parameter system** — Reusable parameterized libraries for modular code
9. **Savegame integration** — MD state is saved with the game; cues persist across save/load
10. **AI script system** — Separate `aiscripts/` XML files define behaviors that can be assigned to ships (patrol, trade, follow, fire weapons, etc.)

### Lua strengths (things Lua can do that MD cannot)

1. **Arbitrary computation** — Lua is a full programming language; MD is declarative XML with limited control flow
2. **FFI access** — Call OS-level functions (named pipes, file I/O, etc.)
3. **UI function hooking** — Replace game UI functions with wrappers
4. **High-frequency polling** — Lua runs every frame; MD check intervals are slower (seconds)
5. **Complex data structures** — Tables, arrays, string manipulation, math
6. **External communication** — Via FFI named pipes, bridge programs
7. **String/data processing** — JSON parsing, encryption, protocol handling

### How they work together (the coop pattern)

The x4_coop mod demonstrates the ideal architecture:

```
┌─────────────────────────────────────────────────────┐
│ Lua (ui/x4_coop.lua)                                 │
│  • Runs every frame (30+ Hz)                        │
│  • Reads player ship position via GetObjectPosition  │
│  • Hooks UI functions (CreateOrder, etc.)           │
│  • Opens named pipe via FFI (kernel32 calls)        │
│  • Serializes/deserializes network protocol         │
│  • Sends/receives messages through pipe             │
│  • Raises events to MD via RaiseEvent()              │
└────────────────────┬────────────────────────────────┘
                     │ raise_lua_event / event_ui_triggered
┌────────────────────▼────────────────────────────────┐
│ MD (md/x4_coop.xml)                                  │
│  • Creates proxy ships for partner (<create_ship>)   │
│  • Updates proxy positions every tick                │
│  • Syncs kills/destruction (<destroy_object>)        │
│  • Syncs economy (station cargo writes)              │
│  • Manages shared world state                        │
│  • Persists across save/load                         │
└─────────────────────────────────────────────────────┘
                     │ assigns behavior
┌────────────────────▼────────────────────────────────┐
│ AI Scripts (aiscripts/x4coop.proxy.fire.xml)         │
│  • Makes partner's proxy ship fire at targets        │
│  • Custom behavior override                          │
└─────────────────────────────────────────────────────┘
```

### Verdict
**MD and Lua are complementary, not competing.** MD handles game world manipulation and persistence; Lua handles I/O, computation, and high-frequency updates. The coop project must use both.

---

## 9. Existing mods that demonstrate these capabilities

### 9.1 x4_coop (DanyWany22) — **THE reference implementation**
- **GitHub:** https://github.com/DanyWany22/x4_coop
- **What it does:** Two-player cooperative multiplayer for X4
- **Capabilities demonstrated:**
  - ✅ External process communication (FFI named pipes → Python bridge → UDP)
  - ✅ Network sockets (via Python bridge, encrypted UDP)
  - ✅ Entity creation (MD `<create_ship>` for proxy ships)
  - ✅ Entity destruction sync (MD `<destroy_object>`)
  - ✅ Game state read (position, orders, hull, shields, faction relations)
  - ✅ Game state write (station cargo, trade rules, ship names, orders)
  - ✅ AI behavior override (custom aiscripts for proxy ship firing)
  - ✅ UI function hooking (CreateOrder, TransferPlayerMoneyTo, etc.)
  - ✅ SETA sync between players
  - ✅ Save game sharing (host's save sent to joiner via bridge)
  - ✅ Economy sync (station cargo overwrite, periodic)
  - ✅ NPC position sync (10Hz for nearby ships)
  - ✅ On-foot position sync (room-level, with stand-in NPC)
  - ✅ Ship equipment sync
  - ✅ Crew sync
  - ✅ Reputation sync
  - ✅ New ship purchase sync
  - ✅ Kill/damage sync
  - ✅ Gunfire sync
- **Architecture:** Lua (FFI pipes + UI hooks) + MD (entity management) + AI scripts (weapon firing) + Python bridge (networking)
- **Status:** Prototype, actively developed, X4 9.00 compatible
- **Key insight:** This mod proves the entire coop concept is viable using only X4's modding interfaces (no memory hacking)

### 9.2 x4p2pmultiplayer (Yxel69)
- **GitHub:** https://github.com/Yxel69/x4p2pmultiplayer
- **What it does:** P2P multiplayer mod for X4
- **Status:** Early stage, minimal README
- **Notes:** Less documentation than x4_coop; appears to be a separate approach

### 9.3 X4-rest-server (Alia5)
- **GitHub:** https://github.com/Alia5/X4-rest-server
- **What it does:** REST HTTP API server for X4
- **Capabilities:**
  - ✅ External communication (HTTP REST API)
  - ✅ Game state export (ship data, sector data, economy)
  - ✅ External process (Node.js or similar server alongside game)
- **Architecture:** Likely uses named pipes or similar IPC between game and server
- **Use case:** External tools, dashboards, AI integration

### 9.4 SirNukes Mod Support APIs (bvbohnen)
- **GitHub:** https://github.com/bvbohnen/x4-projects
- **What it does:** Collection of modding support APIs
- **Components:**
  - **Named Pipes API** — DLL-based Windows named pipe access from Lua/MD
  - **Simple Menu API** — Custom menu creation from MD
  - **Interact Menu API** — Custom right-click menu actions
  - **Hotkey API** — Custom hotkey registration
  - **Time API** — Real-time timers (continue during pause)
  - **Lua Loader API** — `require` support in Protected UI Mode
- **Capabilities:**
  - ✅ Inter-process communication (named pipes)
  - ✅ Custom UI/menus
  - ✅ Input capture (hotkeys)
  - ✅ Real-time timing
- **Critical for coop:** This is the fallback for FFI pipe access if Protected UI Mode restrictions prevent direct kernel32 calls

### 9.5 X4-External-App (mycumycu)
- **GitHub:** https://github.com/mycumycu/X4-External-App
- **What it does:** Framework for external application integration with X4
- **Capabilities:**
  - ✅ External process communication
  - ✅ Game state access
- **Notes:** Demonstrates the general pattern of external app ↔ X4 communication

### 9.6 X4 HTTP Library (djfhe)
- **GitHub:** https://github.com/djfhe/x4_http
- **What it does:** HTTP client library for X4 Lua
- **Capabilities:**
  - ✅ HTTP requests from Lua (likely via FFI or external bridge)
  - ✅ External communication
- **Notes:** Provides HTTP client functionality, potentially useful for REST-based coop communication

### 9.7 God Mode Ship and Station (alexalsp)
- **Nexus Mods:** Steam Workshop
- **What it does:** Invulnerability for ships and stations
- **Capabilities:**
  - ✅ Game state modification (set minimum shields)
  - Uses MD `<set_object_min_shield>` actions
- **Notes:** Demonstrates MD's ability to directly modify entity state

### 9.8 X4 Warehouse Fleets (mbleichner)
- **GitHub:** https://github.com/mbleichner/x4-warehouse-fleets
- **What it does:** Warehouse fleet management
- **Capabilities:**
  - ✅ MD-based persistent background processes
  - ✅ Custom order/behavior system
- **Notes:** Shows MD cue-based persistent background processing pattern

### 9.9 KUDA AI tweaks
- **Nexus Mods:** https://www.nexusmods.com/x4foundations/mods/839
- **What it does:** AI behavior modifications
- **Capabilities:**
  - ✅ AI behavior modification via MD and AI scripts
  - ✅ Ship behavior overrides (carriers, fighters, resupply)

### 9.10 UI Extensions (kuertee)
- **GitHub:** https://github.com/kuertee/x4-mod-ui-extensions
- **What it does:** UI extension framework
- **Capabilities:**
  - ✅ Custom UI elements
  - ✅ Extended menu system
  - Requires Protected UI Mode OFF

---

## Protected UI Mode (Critical Context)

### What it is
Introduced in X4 7.50 (2025), Protected UI Mode is a security feature that restricts Lua's capabilities:

- **`require` is whitelisted** to a small set of modules (e.g., `"ffi"`)
- Loading arbitrary DLLs via `require` is blocked
- Some FFI operations may be restricted
- Mods that use patched UI functions need it disabled

### How to disable
- Settings → Extensions → Protected UI Mode (toggle off)
- Mods that need it off will show a warning
- Some mods (like kuertee's UI Extensions) require it off

### Impact on coop mod
- **The coop mod will require Protected UI Mode OFF** — this is consistent with x4_coop, SirNukes APIs, and other IPC mods
- FFI is still available even with Protected UI Mode ON (it's whitelisted), but the ability to load additional modules is restricted
- The x4_coop README states: "If `/x4coop check` says the pipe functions can't be reached, turn Protected UI Mode off (Settings → Extensions)"

---

## Key Architectural Recommendations for x4-coop

Based on this research, the coop project should:

1. **Use Lua + MD + AI scripts together** (proven by x4_coop)
2. **Use LuaJIT FFI for named pipe access** (kernel32 `CreateFileA`/`ReadFile`/`WriteFile`)
3. **Use an external Python bridge for TCP/UDP networking** (standard library only, no dependencies)
4. **Use MD scripts for entity creation/destruction and persistent state**
5. **Use Lua UI function hooking for order/command sync**
6. **Use `GetObjectPositionInSector` at 30Hz for position sync**
7. **Use MD `<destroy_object>` for kill sync**
8. **Use periodic MD writes for economy sync** (station cargo overwrite)
9. **Require Protected UI Mode OFF**
10. **Consider SirNukes' Named Pipes API as a fallback** for FFI pipe access

### The x4_coop mod is the most valuable reference
Rather than building from scratch, the coop project should:
- Study x4_coop's architecture in detail
- Identify its limitations (prototype quality, 2-player limit, some desync risks)
- Build on its proven patterns
- Consider whether to fork/extend or build alongside

---

## Sources

- x4_coop README: https://github.com/DanyWany22/x4_coop (full README fetched, ~20KB)
- x4p2pmultiplayer: https://github.com/Yxel69/x4p2pmultiplayer
- X4-rest-server: https://github.com/Alia5/X4-rest-server
- SirNukes Mod Support APIs: https://github.com/bvbohnen/x4-projects
- X4-External-App: https://github.com/mycumycu/X4-External-App
- X4 HTTP library: https://github.com/djfhe/x4_http
- X4 Warehouse Fleets: https://github.com/mbleichner/x4-warehouse-fleets
- UI Extensions: https://github.com/kuertee/x4-mod-ui-extensions
- Egosoft MD Guide: https://wiki.egosoft.com/X%20Rebirth%20Wiki/Modding%20support/Mission%20Director%20Guide/
- Egosoft Modding Wiki: https://wiki.egosoft.com/X4%20Foundations%20Wiki/Modding%20Support/
- SirNukes README (Named Pipes, Hotkey, Time APIs): https://github.com/bvbohnen/x4-projects/blob/master/extensions/sn_mod_support_apis/Readme.md
- Reddit r/X4Foundations: Multiple threads on modding, Protected UI Mode
- Steam Community: Protected UI Mode discussions
- X4 Lua functions gist (NodusCursorius): https://gist.github.com/NodusCursorius/b61f26177fdcb490d2456e353d57f363
- X4 FFI gist (NodusCursorius): https://gist.github.com/NodusCursorius/b238ff17e33e51110c252680c63abfcb

---

## Researcher's Assessment

**The x4_coop mod by DanyWany22 is a complete proof-of-concept for cooperative multiplayer in X4: Foundations.** It demonstrates every capability the coop project needs:

- External communication ✅ (FFI named pipes → Python bridge → encrypted UDP)
- Entity creation ✅ (MD `<create_ship>`)
- Game state sync ✅ (position 30Hz, economy 1/min, reputation 10s, kills instant)
- UI hooking ✅ (63 game functions wrapped)
- AI override ✅ (custom aiscripts for proxy ship firing)
- Save sharing ✅ (host save sent to joiner via bridge)
- SETA sync ✅
- On-foot sync ✅ (room-level with NPC stand-in)

The main limitations of x4_coop are:
1. **2-player only** — the protocol is point-to-point
2. **Prototype quality** — still in development, some desync risks
3. **Windows only** — FFI calls are to kernel32 (Windows named pipes)
4. **Requires Protected UI Mode OFF**
5. **Same DLC requirement** — both players need same DLCs

For a coop project, these are acceptable constraints. The architecture is proven. The research question is not "can it be done?" but "how do we build on what x4_coop has already proven?"