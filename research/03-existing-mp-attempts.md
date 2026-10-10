# 03 — Existing Multiplayer / Co-op / State Sync Mod Attempts

**Research Ticket:** #3 (x4-coop wayfinder map)
**Date:** 2026-10-10
**Status:** Complete

---

## Executive Summary

**No multiplayer or co-op mod exists for X4: Foundations.** The community consensus is that true multiplayer is beyond modder scope due to the game's single-threaded simulation architecture and lack of networking code. However, several projects demonstrate **partial building blocks** that could be repurposed for a state-synchronization approach — particularly the LuaSocket/named-pipes IPC infrastructure and external companion apps that achieve real-time data exchange with the running game.

---

## 1. Direct Multiplayer / Co-op Mod Attempts

### X4: Foundations — None Found

- **Nexus Mods:** No multiplayer/co-op mods exist. Searches across the full X4 Nexus catalog (2,400+ mods) return zero results for "multiplayer," "co-op," or "networked."
- **Steam Workshop:** Same — no multiplayer mods. The Workshop is actually more limited than Nexus because X4's Steam Workshop doesn't support mods requiring `Protected UI Mode` to be disabled (which networking mods would need).
- **Egosoft Forums:** Multiple threads (2018–2026) asking for multiplayer. The official response is consistently "not possible with the current engine architecture." Community sentiment ranges from "it would require a complete rework" to "it's financial, not technical."

### Reddit Thread Highlights

| Thread | Date | Key Takeaway |
|--------|------|--------------|
| [r/X4Foundations: "X4 Multiplayer possible?"](https://www.reddit.com/r/X4Foundations/comments/181pa91/) | Nov 2023 | "Out of scope for modders, and even likely out of scope for this dev team" — but data bandwidth wouldn't be the issue |
| [r/X4Foundations: "Multiplayer Co-op"](https://www.reddit.com/r/X4Foundations/comments/1r2wkv0/) | Feb 2026 | "It doesn't exist and from what I have heard, is not possible with how the game is made" |
| [r/X4Foundations: "Chances of ever becoming multiplayer?"](https://www.reddit.com/r/X4Foundations/comments/t59w2v/) | Mar 2022 | "Multiplayer is supposed to be on the way to X4" (unfounded rumor); another user: "it's because it's over 9000" (referring to simulation complexity) |
| [r/X4Foundations: "Possible Co-Op?"](https://www.reddit.com/r/X4Foundations/comments/gsnpy5/) | May 2020 | "If Egosoft felt that the increased revenue from co-op multiplayer would offset the increased development costs, they could certainly do it. It is primarily financial obstacles, not technical obstacles" |
| [r/X4Foundations: "X5 Multiplayer"](https://www.reddit.com/r/X4Foundations/comments/1b7z9a6/) | Mar 2024 | "What is present in X4 is not true multiplayer. Egosoft would have to work from the ground up" |
| [r/X4Foundations: "Co-Op"](https://www.reddit.com/r/X4Foundations/comments/1nzd4js/) | Oct 2025 | "I highly doubt we could get multiplayer for X4 as it would demand a complete rework of the game" |

### X Rebirth (2013) — None Found

- Steam community discussions show players begging for co-op multiplayer (2013–2017).
- No mod was ever created. Community response: "Rebirth doesn't need a half-baked, tacked on multiplayer."
- X Rebirth's engine is even more restrictive than X4's for modding (single-ship focus, more locked-down systems).

---

## 2. Earlier Egosoft Games — X3 Multiplayer Attempts

### X3MP (X3: Albion Prelude Multiplayer)

- **GitHub:** [LaT3St/X3MP](https://github.com/LaT3St/X3MP)
- **Status:** "Very very early stage" — appears abandoned
- **Description:** A multiplayer mod for X3: Albion Prelude
- **Approach:** Not documented in detail; the repo has minimal documentation and appears to be a proof-of-concept that never progressed
- **Key Learning:** Even for X3 (which has a simpler simulation than X4), multiplayer was not achieved beyond initial experiments

### Online-Universe.net

- **Website:** [online-universe.net](http://www.online-universe.net/44home.htm)
- **Description:** An ambitious fan project from the early 2000s to create an MMO version of "X - Beyond the Frontier" and "X-Tension"
- **Status:** Never released. The project was attempted multiple times over decades by Thomas Wojtetzka
- **Current Revival (2026):** Rebooted as a **browser-based multiplayer game** using Babylon.js + Node.js + Socket.io
  - 11 API servers as "source of truth"
  - Node-servers running Babylon.js NullEngine (headless)
  - Has working stations, NPC freighters, player/NPC productions, multi-sector jumpgates
  - [Babylon.js forum thread](https://forum.babylonjs.com/t/online-universe-attempt-to-adapt-egosofts-x-into-a-browser-multiplayer/63551)
- **Key Learning:** This is a **from-scratch reimplementation**, not a mod. The developer concluded that modding the existing X engine for multiplayer was not viable, so built a new engine inspired by X. This validates that the X engine itself is the barrier.

---

## 3. External Service Communication Mods

These are the most directly relevant projects for a co-op/state-sync approach, as they demonstrate real-time data exchange between X4 and external applications.

### 3a. SirNukes Mod Support APIs — Named Pipes

- **Nexus Mods:** [Mod Support APIs](https://www.nexusmods.com/x4foundations/mods/503)
- **Steam Workshop:** [SirNukes Mod Support APIs](https://steamcommunity.com/sharedfiles/filedetails/?id=2042901274)
- **GitHub:** [bvbohnen/x4-projects](https://github.com/bvbohnen/x4-projects) (Windows) / [bekopharm/x4-projects](https://github.com/bekopharm/x4-projects) (Linux fork)
- **Components:**
  - **Lua Loader API** — Dynamic Lua file loading at runtime
  - **Named Pipes API** — Windows named pipes for IPC between X4 and external processes
  - **Linux Pipe API** — Unix domain sockets for Linux (using LuaSocket)
  - **X4 Python Pipe Server** — Python companion server that communicates with X4 via pipes
- **How it works:**
  - Creates named pipe sockets in the X4 savegame folder (the only folder Lua can unlink files from)
  - External applications connect to these pipes to send/receive data
  - Lua-side code polls pipes every game tick for incoming messages
  - Requires **Protected UI Mode disabled** (because it loads external .dll/.so files)
- **Linux specifics (from bekopharm fork):**
  - Requires `lua5.1-socket` (NOT lua5.4-socket — version matters!)
  - Socket files created in `~/.config/EgoSoft/X4/save/` with `.xml` extension (loophole in DeleteSavegame() function)
  - Uses `nc -U` (netcat) for testing: `nc -U ~/.config/EgoSoft/X4/save/x4_external_game.xml`
- **Critical Learning:** This is the **primary IPC mechanism** for X4 mods. Any co-op approach would likely build on this infrastructure.

### 3b. djfhe/x4_http — HTTP Client Library

- **GitHub:** [djfhe/x4_http](https://github.com/djfhe/x4_http)
- **Description:** HTTP/HTTPS client library for X4 using LuaSocket and LuaSec
- **Key features:**
  - Non-blocking callback-based interface (polls every 50ms)
  - GET and POST requests supported
  - JSON response parsing
  - Requires Protected UI Mode disabled (loads external .dll files)
  - Windows only (requires compiled luasocket.dll and luasec.dll for X4's Lua version)
- **Usage example:**
  ```lua
  local Request = require("djfhe.http.request")
  Request.new('POST'):setUrl("https://example.com/api")
    :setBody({ data = "value" })
    :send(function(response, err)
      if err then DebugError("Error: " .. tostring(err)) end
    end)
  ```
- **Critical Learning:** HTTP requests from within X4 Lua are possible but require sideloading native libraries. The non-blocking polling approach (50ms) means network latency is acceptable for non-tick-critical communication.

### 3c. X4 External App — Real-time Web Dashboard

- **Nexus Mods:** [X4 External App](https://www.nexusmods.com/x4foundations/mods/818)
- **GitHub:** [mycumycu/X4-External-App](https://github.com/mycumycu/X4-External-App)
- **Description:** External companion app showing real-time logbook entries, mission offers, active mission details, and player information
- **Architecture:**
  - X4 Lua mod → djfhe/x4_http → HTTP POST to local server
  - Local server (Go binary: `x4_external_app.exe` / `x4_external_app_linux`) serves a web dashboard on port 8080
  - Accessible from any device on the local network (monitor, tablet, smartphone)
  - Customizable layout, logbook filtering, mission filtering
- **Platform:** Windows (primary) and Linux (via Proton/Steam compatibility mode)
- **Critical Learning:** This proves the full pipeline works: **X4 Lua → HTTP → External Server → Web UI**. The same architecture could carry multiplayer state data.

### 3d. X4 Dashboard — Real-time Stats Tracker

- **Nexus Mods:** [X4 Dashboard](https://www.nexusmods.com/x4foundations/mods/2149)
- **Description:** Local companion dashboard for X4, tracks real-time info with customizable colors and achievement tracking
- **Status:** Heavily WIP

### 3e. X4 Foresight — Browser-based Empire Intelligence

- **Website:** [X4 Foresight](https://joyariyb.github.io/X4-Foresight/)
- **Description:** Free in-browser dashboard for fleet, economy, sectors, crew & threat alerts
- **Approach:** Save file scanner (not real-time IPC) — parses save files to extract empire data
- **Learning:** Save-file parsing is an alternative to real-time IPC for state extraction, though it's snapshot-based, not live

### 3f. X4 Savegame Analyzer / Parser

- **Nexus Mods:** [Savegame Analyzer](https://www.nexusmods.com/x4foundations/mods/779)
- **GitHub:** [BeamerMiasma/X4-Foundations](https://github.com/BeamerMiasma/X4-Foundations) (dashboard/visualization), [Mistralys/x4-savegame-parser](https://github.com/Mistralys/x4-savegame-parser) (savegame manager)
- **Approach:** Offline save file parsing tools that extract game state into HTML dashboards
- **Learning:** Multiple tools exist for extracting X4 state from saves, confirming that the save format is well-understood and parseable

---

## 4. Lua Sandbox Boundaries & Native Library Loading

### Protected UI Mode

- X4 has a **Protected UI Mode** setting (introduced in v7.0) that prevents mods from loading external native libraries
- Disabling it allows mods to load `.dll` (Windows) or `.so` (Linux) files
- Mods that need native libraries (LuaSocket, LuaSec, custom pipes) **require Protected UI Mode to be OFF**
- This is a security feature — Egosoft's way of preventing malicious mods from running arbitrary native code
- Once disabled, mods can load arbitrary native code into the game process

### Lua Standard Library Access

- **`os.execute`**: Not available/restricted in X4's Lua sandbox. No evidence of mods using it.
- **`io.open`**: Not available/restricted in X4's Lua sandbox. The named pipe approach works around this by using X4's own `DeleteSavegame()` function (a loophole) to manage socket files in the save directory.
- **`os.remove`**: Same restrictions — the save folder loophole is the only known way to delete files from Lua.
- **File I/O workaround**: The NPC Evolution API (see below) implements JSON persistence, suggesting some file I/O is possible, but details on how it bypasses restrictions are unclear (may use the save folder loophole or Mission Director XML mechanisms).

### Lua Language Server Addon (Documentation)

- A community-maintained [X4 LuaLS Addon](https://www.reddit.com/r/X4Foundations/comments/1jr6qo6/) exists for VS Code
- Documents 1938 C functions, 269 type definitions, 244 "undocumented" functions, and 39 global-access functions
- This is the most comprehensive documentation of X4's Lua API surface
- Source: [chemodun/X4-LuaLSAddonPrep](https://github.com/chemodun/X4-LuaLSAddonPrep)

### LuaSocket Side-loading

- X4 uses Lua 5.1 (NOT 5.4 — version compatibility is critical)
- LuaSocket 3.0.0 and LuaSec must be compiled against X4's specific Lua build
- On Linux: `lua5.1-socket` package works; `lua5.4-socket` does NOT
- Native libraries go in `game/ui/core/lualibs/` or `game/extensions/<mod>/ui/c_library/`
- Symlinks are **no longer accepted** since X4 v7.x

---

## 5. NPC Evolution API — JSON Persistence Engine

- **Nexus Mods:** [NPC Evolution API](https://www.nexusmods.com/x4foundations/mods/2379)
- **Description:** WIP mod providing `_G.DataCore` — a direct JSON file persistence engine
- **Key feature:** Allows mods to persist complex state as JSON files
- **Documentation:** Includes `DEVELOPER_API.md` with copy-paste Lua and Mission Director XML examples
- **Learning:** This demonstrates that structured data persistence from within X4 Lua is possible, though the exact mechanism (file I/O vs. save-folder loophole vs. Mission Director) needs further investigation.

---

## 6. Large-Scale Simulation Control Mods

These mods demonstrate that the X4 simulation can be manipulated at scale from Lua:

| Mod | What It Controls | Scale |
|-----|------------------|-------|
| **Civilian Economy** (Nexus #2405) | Galaxy-wide logistics, ware distribution, sector populations | Entire galaxy economy |
| **Keep Economy Alive** (Nexus #2059) | Periodically trims excess stock at NPC stations | Galaxy-wide economic intervention |
| **Galaxy Trader** (Nexus #1857) | Autonomous trading AI, skill progression, fleet management | Fleet-level automation |
| **Warehouse Fleets** (Nexus #939) | Direct ship routing, sector restrictions, station-level trade control | Fleet + sector management |
| **Safe Cheat Panel** (Nexus #1971) | Edit player stats, inventory, research, blueprints, faction relations, galaxy exploration state, spawning | Full sandbox control |
| **Civilian Economy** | Population-rich sectors, galaxy-wide logistics | Galaxy simulation override |

**Learning:** The Lua API provides sufficient control to read AND write the simulation state at galaxy scale. Economy, fleets, faction relations, and spawn control are all accessible. This means a co-op mod could theoretically inject state changes received from a network peer.

---

## 7. Community & Documentation Resources

### Discord Servers

| Server | URL | Members | Notes |
|--------|-----|---------|-------|
| **Egosoft Official** (unofficial but dev-attended) | [discord.gg/J8u6Kdc](https://discord.gg/J8u6Kdc) | ~17,000 | Devs hang out here; modding discussions occur |
| **X4 Star Wars mod** | (separate server) | — | Mod-specific, but demonstrates modding community organization |

No dedicated X4 modding Discord was found, and no networking-specific channels were identified in the main Discord.

### Egosoft Wiki

- **URL:** [wiki.egosoft.com](https://wiki.egosoft.com/X4%20Foundations%20Wiki/Modding%20Support/)
- **Content:** XML modding guides, Mission Director documentation
- **Gaps:** No documentation on Lua sandbox boundaries, native library loading, or networking. The wiki focuses on XML/data modding, not Lua code modding.

### Key Modder Documentation

| Resource | What It Covers |
|----------|---------------|
| [SirNukes Mod Support APIs README](https://github.com/bvbohnen/x4-projects/blob/master/extensions/sn_mod_support_apis/Readme.md) | Named pipes, Lua loading, IPC |
| [bekopharm Linux fork wiki](https://github.com/bekopharm/x4-projects/wiki/Quick-manual) | Linux pipe setup, LuaSocket, socket file management |
| [djfhe/x4_http](https://github.com/djfhe/x4_http) | HTTP client from Lua, non-blocking patterns |
| [X4 LuaLS Addon](https://github.com/chemodun/X4-LuaLSAddonPrep) | Complete Lua API surface (1938 C functions documented) |
| [upb Lua Loader](https://www.nexusmods.com/x4foundations/mods/1397) | Runtime Lua file loading patterns |

---

## 8. Key Findings for Co-op Approach

### What's Proven to Work

1. **IPC from X4 Lua to external processes** — SirNukes Named Pipes API (Windows) and bekopharm Linux Pipes both work
2. **HTTP from X4 Lua** — djfhe/x4_http library makes outbound HTTP requests with non-blocking callbacks
3. **Real-time external dashboards** — X4 External App proves X4 → HTTP → Server → Web UI pipeline
4. **Large-scale simulation read/write** — Multiple mods read and modify galaxy economy, fleets, and faction state
5. **JSON persistence from Lua** — NPC Evolution API provides structured data persistence
6. **Save file parsing** — Multiple tools extract full game state from save files

### What's NOT Been Attempted

1. **Bidirectional state synchronization** — External apps currently only read from X4 (via HTTP POST or pipe messages). No mod sends commands *back* to X4 to modify game state based on external data.
2. **Two X4 instances communicating** — No attempt to run two X4 games and synchronize state between them
3. **Conflict resolution** — No work on merging simulation state from two sources
4. **Tick-rate synchronization** — No work on aligning game simulation speeds between instances
5. **Entity ID mapping** — No system for mapping entity IDs between two game instances

### Architecture Implications

The existing infrastructure suggests a **hub-and-spoke architecture** is most feasible:
- Each X4 instance runs the game normally
- A Lua mod in each instance sends state updates to an external server (via HTTP or named pipes)
- The external server maintains the merged state
- The server pushes state updates back to each instance via the same channel
- Each instance's Lua mod applies incoming state changes to its local simulation

The key unsolved problem is **semantic merge** — how to reconcile two games' simulations when they diverge (e.g., both players destroy the same station, or one player's economy changes affect another's).

---

## 9. Recommendation: Follow-Up Research

Based on findings in this document, the following research tickets should be prioritized:

1. **Deep-dive SirNukes Named Pipes API** — Read the full source code to understand message format, throughput, and bidirectional capabilities
2. **Test djfhe/x4_http throughput** — Determine max requests/second, payload size limits
3. **Investigate NPC Evolution API persistence mechanism** — How exactly does `_G.DataCore` write JSON files from Lua?
4. **Study X4 save file format** — If save files contain full state, could save-file merging be a simpler approach than real-time sync?
5. **Read X4 LuaLS addon data** — The 1938 documented C functions may reveal APIs for state injection not yet used by any mod
6. **Contact modders** — SirNukes (bvbohnen), djfhe, and mycumycu are the most relevant experts. The Egosoft Discord is the best contact channel.

---

## Sources

- [Nexus Mods — X4: Foundations](https://www.nexusmods.com/x4foundations/)
- [Steam Workshop — X4: Foundations](https://steamcommunity.com/app/392160/workshop/)
- [Egosoft Forums](https://forum.egosoft.com/)
- [Egosoft Wiki — Modding Support](https://wiki.egosoft.com/X4%20Foundations%20Wiki/Modding%20Support/)
- [GitHub — bvbohnen/x4-projects (SirNukes)](https://github.com/bvbohnen/x4-projects)
- [GitHub — bekopharm/x4-projects (Linux fork)](https://github.com/bekopharm/x4-projects)
- [GitHub — djfhe/x4_http](https://github.com/djfhe/x4_http)
- [GitHub — mycumycu/X4-External-App](https://github.com/mycumycu/X4-External-App)
- [GitHub — LaT3St/X3MP](https://github.com/LaT3St/X3MP)
- [GitHub — chemodun/X4-LuaLSAddonPrep](https://github.com/chemodun/X4-LuaLSAddonPrep)
- [Babylon.js Forum — Online-Universe.net](https://forum.babylonjs.com/t/online-universe-attempt-to-adapt-egosofts-x-into-a-browser-multiplayer/63551)
- [Online-Universe.net](http://www.online-universe.net/44home.htm)
- [Egosoft Discord](https://discord.gg/J8u6Kdc)
- Reddit r/X4Foundations (multiple threads linked above)