# Design #18: Headless Server / Spectator Bot Mod

**Wayfinder Ticket:** #18  
**Date:** 2026-10-10  
**Status:** DRAFT  
**Author:** OpenClaw CTO subagent  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Key Findings](#3-key-findings)
4. [Architecture Overview](#4-architecture-overview)
5. [Mod Design](#5-mod-design)
   - 5.1 [Spectator Bot Lua (`ui/x4coop_spectator.lua`)](#51-spectator-bot-lua-uix4coop_spectatorlua)
   - 5.2 [Mission Director Script (`md/x4coop_spectator.xml`)](#52-mission-director-script-mdx4coop_spectatorxml)
   - 5.3 [Content Manifest (`content.xml`)](#53-content-manifest-contentxml)
   - 5.4 [Extension Registration (`ui.xml`)](#54-extension-registration-uixml)
6. [Server Operational Procedures](#6-server-operational-procedures)
   - 6.1 [Startup Sequence](#61-startup-sequence)
   - 6.2 [Auto-Save Loop](#62-auto-save-loop)
   - 6.3 [Crash Recovery](#63-crash-recovery)
   - 6.4 [Performance Profile](#64-performance-profile)
   - 6.5 [Display Handling](#65-display-handling)
7. [Remote Management](#7-remote-management)
8. [Monitoring](#8-monitoring)
9. [Open Questions & Risks](#9-open-questions--risks)
10. [Implementation Plan](#10-implementation-plan)

---

## 1. Executive Summary

This design describes a "spectator bot" mod for X4: Foundations that enables the game to run as a **dedicated server** with no human player at the keyboard. The server's X4 instance runs the full universe simulation (economy, NPC AI, faction wars, missions) while a co-op bridge relays state to connected client instances. The "player" in the server's X4 is a spectator — present because the engine requires one, but not controlling any ship and not receiving input.

The mod consists of two pieces:
1. **A Lua UI script** that manages auto-saving, suppresses the player's need for a ship, and provides a health-check heartbeat to the bridge.
2. **An MD (Mission Director) script** that runs periodic timers for the auto-save loop and reports game state to the bridge.

Combined with server automation scripts (systemd service or Windows Task Scheduler), this turns a stock X4 installation into a persistent world server that starts on boot, auto-saves every 5–10 minutes, and auto-restarts on crash.

---

## 2. Problem Statement

X4: Foundations is a single-player game. The engine requires:
- A **player entity** (the game creates one at startup; the `player` object is fundamental to the engine)
- A **player ship** (`player.occupiedship` — the engine assigns one during game start)
- A **display** (X4 uses DirectX 11; it does not have a headless rendering mode)
- A **window** (the game window must exist, even if minimized)

The x4_coop mod by DanyWany22 has proven that two X4 instances can synchronize over a network using Lua FFI → named pipe → Python bridge → UDP. The host runs the full simulation; the guest receives state updates.

For a **dedicated server**, we need the host's X4 to:
1. Start automatically with no human interaction
2. Run the simulation without anyone flying a ship
3. Auto-save periodically for crash recovery
4. Connect to the bridge and serve state to clients
5. Restart automatically if it crashes

---

## 3. Key Findings

### 3.1 Does X4 need a player entity to run the simulation?

**Yes — but the player doesn't need to do anything.**

The `player` object is fundamental to X4's engine. The Mission Director, economy, faction logic, and all game systems reference `player.galaxy`, `player.occupiedship`, `player.character`, etc. The simulation will not run without a loaded game, and a loaded game always has a player entity.

**However**, once the game is loaded, the simulation runs autonomously. The economy ticks, NPC AI makes decisions, factions fight wars, and missions spawn — all without player input. The player can sit idle in a ship indefinitely. The simulation doesn't care whether the player is actively doing anything.

**Conclusion:** We cannot remove the player entity, but we don't need to. We just need the player to be idle and harmless.

### 3.2 Can we remove the player ship entirely?

**No — the engine forces a player ship on game start.**

Every game start (including creative mode) assigns `player.occupiedship` to a ship. The engine's save/load system, the UI, and many game systems assume `player.occupiedship` is valid. Setting it to null or 0 causes crashes and undefined behavior.

**However**, we can make the player ship irrelevant:
- Park it in a safe sector (e.g., Argon Prime, away from the action)
- Make it invulnerable via MD script (`set_object_min_hull` with a very high value)
- Strip its weapons so it doesn't attract combat
- Never issue orders to it
- The player character stays on this ship and never leaves

**Alternative: Use a small, cheap ship.** A Scout ship (ship_s) in a safe zone has minimal impact on simulation performance. It exists, the engine is happy, and it doesn't participate in the game.

**Conclusion:** Keep the player ship, but make it a passive, invulnerable spectator platform parked in a safe location.

### 3.3 Can we use MD scripts to automate save game creation on a timer?

**Yes.** The Lua function `SaveGame(filename, description)` is available in the UI scripting environment. The existing x4_coop mod uses it for save sharing:

```lua
SaveGame("quicksave", "X4 Co-op shared world")
```

The MD can trigger Lua events on a timer via `raise_lua_event`, and the Lua side calls `SaveGame()`. The MD timer uses `<delay>` cues with `instantiate="true"` to create a repeating loop.

**Key constraint:** `SaveGame()` can only be called from the UI Lua environment (not from MD directly). The MD must raise a Lua event, and the Lua handler calls `SaveGame()`.

**Another constraint:** The game must not be in a "busy" state (loading, cutscene, etc.) when `SaveGame()` is called. The Lua code should check `IsSaveListLoadingComplete()` and retry on the next frame if not ready.

### 3.4 What's the minimum X4 startup sequence?

1. Launch `X4.exe` with appropriate command-line flags
2. The game loads to the main menu
3. A save game must be loaded (either via auto-load or via MD script on game start)
4. Once the save is loaded, the simulation begins
5. The Lua mod activates and connects to the bridge

**The challenge:** Getting from the main menu to a loaded save without human input. X4 has no "auto-load last save" command-line flag.

**Solutions:**
- **MD script on game start:** The MD can detect the main menu state and trigger a save load via Lua. However, MD cues typically fire after a game is loaded (`event_game_started`/`event_game_loaded`), not from the main menu.
- **Simulated input:** Send a synthetic keypress/click to the main menu to load the last save. This can be done via Windows `SendInput` API from the launcher script, or via AutoHotkey.
- **Custom game start:** Create a custom game start that automatically loads into a specific save configuration. X4's game start system is XML-based and could be modded to auto-load.
- **Most reliable: Simulated input from the launcher script.** The launcher sends keypresses to navigate the main menu: Continue → select last save → Enter. This is fragile to UI changes but simple and effective.

### 3.5 How does X4 handle having no player ship?

X4 does not handle this gracefully. If `player.occupiedship` becomes 0 (e.g., the player's ship is destroyed and they're in spacesuit), the game enters a "spacesuit" mode where the player character floats in space. The simulation continues, but the UI shows spacesuit controls.

If the player character dies (spacesuit destroyed), the game ends — it doesn't respawn.

**For our purposes:** We must ensure the player ship is never destroyed. The MD script sets `set_object_min_hull` to a very high value, making it effectively invulnerable. We also park it in a safe sector away from combat.

---

## 4. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│ Server Machine (Windows)                                            │
│                                                                     │
│  ┌──────────────┐    ┌──────────────────────┐    ┌──────────────┐  │
│  │ Launcher     │    │ X4 Server Instance   │    │ Bridge       │  │
│  │ (Python)     │───▶│                      │───▶│ (Python)     │  │
│  │ - Starts X4  │    │ x4coop_spectator.lua │    │ x4coop_      │  │
│  │ - Sends      │    │  - Auto-save timer   │    │ bridge.py    │  │
│  │   menu       │    │  - Health heartbeat  │    │              │  │
│  │   keypresses │    │  - Player ship mgmt  │    │ UDP 47810    │  │
│  │ - Monitors   │    │                      │    │ TCP 47810    │  │
│  │   process    │    │ x4coop_spectator.xml │    │              │  │
│  │ - Restarts   │    │  (MD)                │    │ ←── Named    │  │
│  │   on crash   │    │  - Invuln player ship│    │     Pipe ──→ │  │
│  │              │    │  - Auto-save trigger │    │              │  │
│  │              │    │  - State reporting   │    │ To Clients   │  │
│  └──────────────┘    └──────────────────────┘    └──────┬───────┘  │
│                                                           │          │
└───────────────────────────────────────────────────────────┼──────────┘
                                                            │
                                              UDP/TCP 47810 │
                                                            │
                    ┌───────────────────────────────────────┘
                    │
                    ▼
           ┌──────────────────┐
           │ Client Bridges    │
           │ (Player 1, 2, ...)│
           └──────────────────┘
```

### Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| `x4coop_spectator.lua` | Lua (LuaJIT, in-game) | Auto-save, heartbeat, player ship management, bridge connection |
| `x4coop_spectator.xml` | MD XML (in-game) | Timer cues, invulnerability, state queries |
| `x4coop_server_launcher.py` | Python 3 (external) | Start X4, send menu keypresses, monitor process, restart on crash |
| `x4coop_bridge.py` | Python 3 (external) | Network bridge (forked from DanyWany22's bridge) — relays state to clients |
| `x4coop_monitor.py` | Python 3 (external) | Health check endpoint, log analysis, web dashboard |
| `x4coop_server.service` | systemd / Task Scheduler | Auto-start on boot |

---

## 5. Mod Design

### 5.1 Spectator Bot Lua (`ui/x4coop_spectator.lua`)

This is the core mod logic. It runs inside X4's LuaJIT environment alongside the existing x4_coop mod (or standalone if the server doesn't need full co-op sync).

#### Responsibilities

1. **Bridge connection** — Connect to the named pipe (`\\.\pipe\x4_coop`) and communicate with the Python bridge
2. **Auto-save loop** — Call `SaveGame()` on a timer (every 5–10 minutes, configurable)
3. **Player ship management** — Park the player ship, make it invulnerable, strip weapons
4. **Health heartbeat** — Send periodic health pings to the bridge for monitoring
5. **Save rotation** — Rotate between N save slots to prevent corruption (save to slot A, then B, then A, so a crash during save doesn't destroy the last good save)
6. **No-input mode** — Ensure no keyboard/mouse input is needed after initial load

#### FFI Declarations

```lua
local ffi = require("ffi")

-- Game functions (same pattern as x4_coop.lua)
ffi.cdef[[
    typedef unsigned long long UniverseID;

    UniverseID GetPlayerID(void);
    UniverseID GetPlayerOccupiedShipID(void);
    const char* GetPlayerName(void);
    const char* GetSaveFolderPath(void);
    bool IsSaveListLoadingComplete(void);
    bool IsSaveValid(const char* filename);
    void ReloadSaveList(void);
    bool IsGamePaused(void);
    bool IsComponentClass(UniverseID id, const char* classname);
    bool GetComponentData(UniverseID id, const char* field);
    void SetObjectMinHull(UniverseID id, float hull);
    float GetObjectHull(UniverseID id);
    -- ... additional game functions as needed
]]

local C = ffi.C
```

#### State Machine

```lua
local S = {
    -- Lifecycle
    state = "init",         -- "init" -> "menu" -> "loading" -> "running" -> "saving"
    md_ready = false,       -- MD script has reported ready
    
    -- Auto-save
    autosave_interval = 300,    -- seconds (5 minutes default)
    autosave_next = 0,          -- next save time (elapsed time)
    autosave_slots = { "server_a", "server_b" },
    autosave_slot_idx = 1,      -- rotate between slots
    autosave_in_progress = false,
    last_save_at = 0,
    
    -- Player ship
    player_ship_parked = false,
    player_ship_id = 0,
    safe_sector = " macro.sector_argonprime",  -- Argon Prime
    
    -- Health
    heartbeat_at = 0,
    heartbeat_interval = 5,     -- seconds between heartbeats to bridge
    frames_since_heartbeat = 0,
    
    -- Bridge
    pipe_connected = false,
    pipe = nil,
    
    -- Stats
    started_at = 0,
    save_count = 0,
    last_error = nil,
}
```

#### Auto-Save Logic

```lua
local function do_autosave(now)
    if S.autosave_in_progress then return end
    if not C.IsSaveListLoadingComplete() then
        -- Retry next frame
        S.autosave_next = now + 1
        return
    end
    
    -- Rotate save slots: server_a -> server_b -> server_a ...
    local slot = S.autosave_slots[S.autosave_slot_idx]
    S.autosave_slot_idx = S.autosave_slot_idx % #S.autosave_slots + 1
    
    S.autosave_in_progress = true
    local ok, err = pcall(function()
        SaveGame(slot, "X4 Co-op Server auto-save")
    end)
    
    if ok then
        S.last_save_at = now
        S.save_count = S.save_count + 1
        S.autosave_in_progress = false
        -- Notify bridge
        net_send("H|save|" .. slot .. "|" .. tostring(now))
        DebugError(string.format("[x4coop_spectator] auto-saved to %s (save #%d)", 
            slot, S.save_count))
    else
        S.autosave_in_progress = false
        S.last_error = tostring(err)
        DebugError("[x4coop_spectator] save failed: " .. tostring(err))
        -- Retry in 30 seconds
        S.autosave_next = now + 30
    end
    
    -- Schedule next save
    S.autosave_next = now + S.autosave_interval
end
```

#### Player Ship Management

```lua
local function manage_player_ship(now)
    local ship = C.GetPlayerOccupiedShipID()
    if ship == 0 then return end
    
    if not S.player_ship_parked then
        -- Make the player ship invulnerable
        -- Setting min hull to an extremely high value prevents destruction
        C.SetObjectMinHull(ship, 1e9)
        
        -- Strip weapons by removing all weapon entries
        -- (This is done via MD since Lua can't directly remove equipment)
        -- The MD script handles this on game start
        
        S.player_ship_parked = true
        S.player_ship_id = ship
        DebugError("[x4coop_spectator] player ship parked and made invulnerable")
    end
    
    -- Periodically re-assert invulnerability (in case of reload)
    if now - (S.last_invuln_check or 0) > 60 then
        S.last_invuln_check = now
        C.SetObjectMinHull(ship, 1e9)
    end
end
```

#### Health Heartbeat

```lua
local function send_heartbeat(now)
    if now - S.heartbeat_at < S.heartbeat_interval then return end
    S.heartbeat_at = now
    
    local msg = string.format("H|server|%d|%.0f|%d|%s",
        os.time(),           -- real time
        now,                 -- game time
        S.save_count,        -- total saves
        S.state              -- current state
    )
    net_send(msg)
end
```

#### Main Tick

```lua
local function tick(now, dt)
    -- 1. Check MD ready
    if not S.md_ready then
        -- Keep requesting MD ready
        if now - (S.last_handshake or 0) > 2 then
            S.last_handshake = now
            AddUITriggeredEvent("x4coop_spectator", "ready", {})
        end
        return
    end
    
    -- 2. Manage player ship
    manage_player_ship(now)
    
    -- 3. Auto-save
    if S.state == "running" and now >= S.autosave_next then
        do_autosave(now)
    end
    
    -- 4. Heartbeat
    send_heartbeat(now)
    
    -- 5. Network tick (pipe I/O)
    -- Read any commands from the bridge
    -- Forward state to the bridge (reuses x4_coop's sync logic)
end
```

### 5.2 Mission Director Script (`md/x4coop_spectator.xml`)

The MD script handles things Lua cannot: entity manipulation, periodic scans, and game event hooks.

```xml
<?xml version="1.0" encoding="utf-8"?>
<mdscript name="X4Coop_Spectator" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="md.xsd">
  <cues>
    <cue name="Root" version="1">
      <actions>
        <set_value name="$DebugChance" exact="100"/>
        <set_value name="$SaveInterval" exact="300s"/>  <!-- 5 minutes -->
        <set_value name="$InvulnInterval" exact="60s"/>  <!-- re-assert every 60s -->
      </actions>
      <cues>

        <!-- Game started or save loaded: initialize spectator mode -->
        <cue name="OnGameStart" instantiate="true">
          <conditions>
            <check_any>
              <event_game_started/>
              <event_game_loaded/>
            </check_any>
          </conditions>
          <actions>
            <!-- Make player ship invulnerable -->
            <do_if value="player.occupiedship.exists">
              <set_object_min_hull object="player.occupiedship" exact="1000000000"/>
              <!-- Strip weapons: remove all weapon modules -->
              <!-- (Careful: this prevents the ship from fighting back if attacked) -->
              <remove_all_equipment object="player.occupiedship" tags="'weapon'"/>
              <!-- Cancel any orders the player ship might have -->
              <cancel_all_orders object="player.occupiedship"/>
            </do_if>
            <!-- Tell Lua that MD is ready -->
            <raise_lua_event name="'x4coop_spectator.md_ready'"/>
          </actions>
        </cue>

        <!-- Lua (re)started: re-handshake -->
        <cue name="OnLuaReady" instantiate="true">
          <conditions>
            <event_ui_triggered screen="'x4coop_spectator'" control="'ready'"/>
          </conditions>
          <actions>
            <raise_lua_event name="'x4coop_spectator.md_ready'"/>
          </actions>
        </cue>

        <!-- Re-assert player ship invulnerability on a timer -->
        <cue name="InvulnLoop" instantiate="true">
          <conditions>
            <event_game_started/>
          </conditions>
          <actions>
            <set_value name="$NextInvuln" exact="player.age + 60s"/>
          </actions>
          <cues>
            <cue name="InvulnTick" instantiate="true">
              <conditions>
                <check_value value="player.age ge @$NextInvuln and player.occupiedship.exists"/>
              </conditions>
              <actions>
                <set_object_min_hull object="player.occupiedship" exact="1000000000"/>
                <set_value name="$NextInvuln" exact="player.age + 60s"/>
                <!-- Re-trigger this cue (instantiate=true makes a new instance) -->
                <raise_lua_event name="'x4coop_spectator.invuln_checked'"/>
              </actions>
            </cue>
          </cues>
        </cue>

        <!-- Auto-save trigger: MD timer raises event to Lua, Lua calls SaveGame() -->
        <!-- Note: SaveGame() is a Lua function, not available in MD. So MD raises
             a Lua event, and the Lua handler calls SaveGame(). -->
        <cue name="AutoSaveLoop" instantiate="true">
          <conditions>
            <event_game_started/>
          </conditions>
          <actions>
            <set_value name="$NextSave" exact="player.age + 300s"/>
          </actions>
          <cues>
            <cue name="AutoSaveTick" instantiate="true">
              <conditions>
                <check_value value="player.age ge @$NextSave"/>
              </conditions>
              <actions>
                <raise_lua_event name="'x4coop_spectator.autosave'"/>
                <set_value name="$NextSave" exact="player.age + 300s"/>
              </actions>
            </cue>
          </cues>
        </cue>

        <!-- Report player ship state to Lua for monitoring -->
        <cue name="StateReport" instantiate="true">
          <conditions>
            <event_game_started/>
          </conditions>
          <actions>
            <set_value name="$NextReport" exact="player.age + 10s"/>
          </actions>
          <cues>
            <cue name="StateReportTick" instantiate="true">
              <conditions>
                <check_value value="player.age ge @$NextReport"/>
              </conditions>
              <actions>
                <do_if value="player.occupiedship.exists">
                  <set_value name="player.entity.$x4coop_spectator_state"
                    exact="'ship=' + player.occupiedship.idcode + 
                           ' sector=' + player.occupiedship.sector.name +
                           ' hull=' + player.occupiedship.hull.fraction"/>
                </do_if>
                <set_value name="$NextReport" exact="player.age + 10s"/>
              </actions>
            </cue>
          </cues>
        </cue>

      </cues>
    </cue>
  </cues>
</mdscript>
```

**Note on MD cue persistence:** An MD cue without `instantiate="true"` fires once per game and its completed state is saved with the game. Using `instantiate="true"` ensures the cue fires on every load. This is critical for the auto-save and invulnerability loops — they must re-activate after every save/load cycle.

### 5.3 Content Manifest (`content.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<content id="x4coop_spectator" name="X4 Co-op Spectator Bot" version="1" 
         date="2026-10-10" save="0" description="Headless server / spectator bot for X4 Co-op">
  <text language="44" name="X4 Co-op Spectator Bot" 
        description="Allows X4 to run as a dedicated server with no human player. Auto-saves, manages player ship, and reports health to the bridge."/>
</content>
```

**`save="0"`** means the mod doesn't require a new save game — it can be added to an existing save. This is important for crash recovery: if the mod is installed, the server can load any save.

### 5.4 Extension Registration (`ui.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<ui>
  <script file="ui/x4coop_spectator.lua"/>
</ui>
```

### 5.5 Integration with Existing x4_coop Mod

The spectator bot can run alongside the existing x4_coop mod. The key integration points:

1. **Pipe sharing:** Both mods use the same named pipe (`\\.\pipe\x4_coop`). The spectator bot adds its messages to the same pipe. The bridge must be extended to handle spectator-specific message types.

2. **Message protocol additions:**

| Code | Meaning | Fields | Direction |
|------|---------|--------|-----------|
| `H\|server` | Server heartbeat | real_time, game_time, save_count, state | Game → Bridge |
| `H\|save` | Save notification | slot_name, game_time | Game → Bridge |
| `H\|status` | Full status report | ship, sector, hull, fps, uptime | Game → Bridge |
| `C\|autosave` | Command: trigger save now | (none) | Bridge → Game |
| `C\|load` | Command: load a specific save | slot_name | Bridge → Game |
| `C\|shutdown` | Command: graceful shutdown | (none) | Bridge → Game |

3. **Bridge extension:** The Python bridge (`x4_coop_bridge.py`) needs a "server mode" that:
   - Accepts multiple client connections (1:N instead of 1:1)
   - Forwards state from the server X4 to all clients
   - Accepts commands from clients and forwards to the server X4
   - Exposes a health-check HTTP endpoint

4. **Co-op mod role:** The server X4 is always the "host" in x4_coop terms. It never receives `/x4coop join` commands — it only serves state. The spectator bot mod sets the role to "host" automatically on game start.

---

## 6. Server Operational Procedures

### 6.1 Startup Sequence

The server startup is a multi-stage process orchestrated by the launcher script (`x4coop_server_launcher.py`):

```
Stage 1: Boot → Launcher starts (systemd / Task Scheduler)
    ↓
Stage 2: Launcher starts the Python bridge
    ↓
Stage 3: Launcher starts X4.exe with low-graphics settings
    ↓
Stage 4: X4 loads to main menu (wait for window title change)
    ↓
Stage 5: Launcher sends simulated keypresses to load last save:
    - Press ENTER (Continue)
    - Wait 2s
    - Press ENTER (select last save)
    - Wait for save to load (detect via log file or window state)
    ↓
Stage 6: X4 save loaded, simulation running
    ↓
Stage 7: Spectator bot mod activates:
    - MD fires OnGameStart → raises md_ready event
    - Lua connects to named pipe
    - Lua reports "running" state to bridge
    ↓
Stage 8: Bridge accepts client connections
    ↓
Stage 9: Steady state — server running, clients connecting
```

#### Launcher Script (`x4coop_server_launcher.py`)

```python
#!/usr/bin/env python3
"""
X4 Co-op Server Launcher

Starts X4 with the spectator bot mod, loads the last save via simulated
input, monitors the process, and restarts on crash.
"""
import subprocess
import time
import sys
import os
import ctypes
import logging

# Configuration
X4_PATH = r"C:\Program Files (x86)\Steam\steamapps\common\X4 Foundations\X4.exe"
BRIDGE_PATH = r"C:\x4_coop\bridge\x4_coop_bridge.py"
SAVE_NAME = "server_a"  # or "server_b" — the launcher loads the most recent
GRAPHICS_QUALITY = "0"  # lowest
LOG_FILE = r"C:\x4_coop\server\launcher.log"

# Windows API for sending keypresses
user32 = ctypes.WinDLL("user32")
kernel32 = ctypes.WinDLL("kernel32")

def start_bridge():
    """Start the Python bridge in server mode."""
    return subprocess.Popen([
        sys.executable, BRIDGE_PATH,
        "--host",
        "--server-mode",
        "--pipe", "x4_coop"
    ])

def start_x4():
    """Start X4 with low graphics settings."""
    return subprocess.Popen([
        X4_PATH,
        "-debug", "error",
        "-logfile", r"C:\x4_coop\server\x4.log",
        # Graphics quality is set in the game's config, not command line
    ])

def send_key(key_code):
    """Send a keypress to the foreground window."""
    user32.keybd_event(key_code, 0, 0, 0)        # key down
    time.sleep(0.1)
    user32.keybd_event(key_code, 0, 0x0002, 0)   # key up (KEYUP flag)

def load_last_save():
    """Navigate the main menu to load the last save."""
    time.sleep(10)  # Wait for X4 to reach main menu
    
    # Press ENTER to select "Continue"
    send_key(0x0D)  # VK_RETURN
    time.sleep(2)
    
    # Press ENTER to confirm save selection
    send_key(0x0D)
    
    # Wait for save to load (check log file for "game loaded" message)
    wait_for_save_load()

def wait_for_save_load(timeout=120):
    """Wait for the save to finish loading by monitoring the log file."""
    start = time.time()
    while time.time() - start < timeout:
        try:
            with open(r"C:\x4_coop\server\x4.log", "r") as f:
                content = f.read()
                if "Mission Director" in content and "game loaded" in content.lower():
                    return True
        except:
            pass
        time.sleep(2)
    return False

def main():
    logging.basicConfig(filename=LOG_FILE, level=logging.INFO,
                        format="%(asctime)s %(levelname)s %(message)s")
    
    while True:  # Auto-restart loop
        logging.info("Starting X4 Co-op Server")
        
        bridge = start_bridge()
        time.sleep(2)
        
        x4 = start_x4()
        load_last_save()
        
        # Monitor process
        ret = x4.wait()
        logging.error(f"X4 exited with code {ret}")
        
        # Clean up bridge
        bridge.terminate()
        bridge.wait()
        
        # Wait before restart (avoid crash loop)
        time.sleep(10)
        logging.info("Restarting X4 Co-op Server")
```

### 6.2 Auto-Save Loop

The auto-save loop is driven by the MD script's `AutoSaveTick` cue, which raises a Lua event every 5 minutes. The Lua handler calls `SaveGame()` with a rotating slot name.

**Save rotation strategy:**

```
Time 0:00  → Save to "server_a" 
Time 5:00  → Save to "server_b"
Time 10:00 → Save to "server_a" (overwrites previous)
Time 15:00 → Save to "server_b" (overwrites previous)
...
```

This ensures that if a crash happens during a save (corrupting the save file), the previous good save is still intact in the other slot.

**Save file locations:**
```
Documents\Egosoft\X4\<steam_id>\save\server_a.xml.gz
Documents\Egosoft\X4\<steam_id>\save\server_b.xml.gz
```

**On crash recovery:** The launcher loads the most recent valid save. It checks `IsSaveValid()` via a small Lua script, or simply loads by modification time and falls back to the other slot if the first fails.

### 6.3 Crash Recovery

```
┌─────────────────────────────────────────────────────────┐
│ Crash Detection                                         │
│                                                         │
│  1. Launcher monitors X4 process (x4.wait())           │
│  2. Bridge monitors pipe connection                     │
│  3. Monitor script checks HTTP health endpoint          │
│                                                         │
│ Any of these triggers:                                   │
│  → Kill any remaining X4 process                         │
│  → Kill bridge process                                   │
│  → Wait 10 seconds (avoid crash loop)                    │
│  → Restart launcher loop (Stage 2)                       │
│  → Load most recent valid save (server_a or server_b)    │
│  → Bridge reconnects, clients reconnect                  │
└─────────────────────────────────────────────────────────┘
```

**Client-side reconnection:** Clients detect server loss when:
- No UDP packets for 10 seconds (bridge timeout)
- TCP connection reset

Clients then enter "waiting for server" mode, showing a reconnecting screen. When the server comes back, the bridge re-establishes the connection, and clients receive a full state snapshot.

**Save integrity check:** Before loading a save, the launcher can verify it:
```python
import gzip
def is_save_valid(path):
    try:
        with gzip.open(path, 'rb') as f:
            data = f.read(1024)  # Try to decompress first 1KB
            return b'<savegame' in data
    except:
        return False
```

### 6.4 Performance Profile

X4 on the server should run with minimum graphics settings to reduce GPU load:

#### Graphics Settings (in `Documents\Egosoft\X4\config.xml`)

| Setting | Value | Reason |
|---------|-------|--------|
| GfxQuality | 0 (Low) | Reduce shader complexity |
| Shadow | 0 (Off) | Shadows are GPU-expensive |
| SSAO | 0 (Off) | Ambient occlusion unnecessary |
| SoftShadows | 0 (Off) | Not needed |
| LOD | 0 (Low) | Reduce level-of-detail work |
| Glow | 0 (Off) | Post-processing not needed |
| Distortion | 0 (Off) | Post-processing not needed |
| EffectDistance | 0 (Low) | Reduce particle effects |
| TrafficDensity | 0 (Low) | Reduce civilian traffic (minor sim impact) |
| CharacterDensity | 0 (Low) | Fewer NPC models rendered |
| GfxQuality (resolution) | 1280x720 | Minimal resolution |
| Fullscreen | 0 (Windowed) | Easier to manage headless |

#### Additional Performance Measures

- **SETA not used** — The server runs at normal game speed (1x). SETA would accelerate the simulation but also increase CPU load.
- **No mods that add visual content** — No ship visual replacers, no texture mods.
- **Minimal client count** — Each client adds bridge bandwidth and Lua processing overhead. Start with 2, test scaling.
- **High-attention area minimized** — The player ship is parked in a quiet sector. Only one sector is in "high attention" mode at a time (the player's sector). This limits the full-physics simulation to one sector.

#### Expected Resource Usage

| Resource | Estimate | Notes |
|----------|----------|-------|
| CPU | 2-4 cores | X4 is CPU-heavy (economy, AI, physics) |
| GPU | Minimal | Low settings, 720p, no post-processing |
| RAM | 8-16 GB | X4 uses ~6-12 GB for a large universe |
| Disk | 500 MB - 2 GB per save | Gzipped XML save files |
| Network | 2-50 KB/s per client | Depending on activity level |

### 6.5 Display Handling

X4 requires a display. It uses DirectX 11 and does not have a headless rendering mode. Options:

#### Option A: Virtual Framebuffer (Linux via Proton/Wine) — NOT RECOMMENDED

X4 is a Windows game. Running under Wine/Proton with Xvfb is theoretically possible but:
- Egosoft doesn't support Linux
- The mod uses Windows named pipes (`kernel32` FFI)
- Wine's DirectX 11 implementation has issues with X4
- **Verdict: Too fragile for a production server**

#### Option B: RDP Session (Windows Server) — RECOMMENDED

Run X4 inside an RDP session on Windows Server:
- RDP provides a virtual display
- Windows named pipes work normally
- The game runs in a window at 1280x720
- RDP session persists even when disconnected
- **Setup:** Windows Server 2022, RDP session auto-connects on boot
- **Verdict: Most reliable option for Windows**

#### Option C: Virtual GPU (Windows VM) — VIABLE

Run X4 in a Windows VM with a virtual GPU:
- VMware ESXi with vGPU (NVIDIA GRID or AMD MxGPU)
- Or Hyper-V with GPU passthrough (DDA)
- Or a cloud Windows instance (AWS, Azure) with GPU
- **Verdict: Good for cloud deployments, more expensive**

#### Option D: Headless GPU (Windows with no monitor) — WORKABLE

Run X4 on a Windows PC with a GPU but no monitor:
- Use a dummy HDMI plug (EDID emulator) to make the GPU think a display is attached
- Remote in via RDP, VNC, or Parsec for setup
- The game runs in a window, rendered to the virtual display
- **Verdict: Simplest for a home server — use a $5 HDMI dummy plug**

**Recommended: Option D (dummy HDMI plug) for home servers, Option B (RDP on Windows Server) for cloud.**

---

## 7. Remote Management

### 7.1 SSH Access

The server machine should have SSH access (via OpenSSH on Windows or a Linux management VM) for:
- Checking logs
- Restarting the server
- Updating mods
- Running diagnostics

### 7.2 RDP Access

RDP for:
- Initial setup (installing X4, mods, Python)
- Debugging visual issues
- Manually loading a different save
- Monitoring the game window

### 7.3 Web Dashboard (`x4coop_monitor.py`)

A lightweight web dashboard provides at-a-glance server status:

```
┌────────────────────────────────────────────────────┐
│  X4 Co-op Server Dashboard                        │
│                                                   │
│  Status: ● RUNNING                                │
│  Uptime: 4h 23m                                   │
│  Players: 2/4 connected                           │
│  Last save: server_b, 3 minutes ago (#47)         │
│  Game time: 2d 14h 33m                            │
│  Player ship: ZVF-031 in Argon Prime (100% hull)  │
│  FPS: 30                                          │
│  Bridge: ● Connected                              │
│                                                   │
│  [Save Now]  [Load Save]  [Restart Server]        │
│                                                   │
│  Recent log:                                      │
│  01:14:05 [INFO] Auto-saved to server_b (#47)     │
│  01:09:05 [INFO] Auto-saved to server_a (#46)     │
│  01:04:12 [INFO] Client "Player1" connected       │
│  01:04:05 [INFO] Heartbeat: running, 30 fps       │
└────────────────────────────────────────────────────┘
```

**Implementation:** A simple Flask/FastAPI app that reads from:
- The bridge's health endpoint
- The X4 log file
- The save directory (modification times)
- The bridge's client list

**Actions:**
- **Save Now:** Sends `C|autosave` command through the bridge
- **Load Save:** Sends `C|load|<slot>` command through the bridge
- **Restart Server:** Kills X4 process (launcher auto-restarts)

### 7.4 Bridge HTTP Health Endpoint

The bridge exposes a simple HTTP endpoint (separate from the UDP game protocol):

```
GET http://server:47811/health
→ 200 OK
{
    "status": "running",
    "uptime_seconds": 15643,
    "x4_connected": true,
    "clients": [
        {"name": "Player1", "address": "192.168.1.50:47810", "connected_at": "..."},
        {"name": "Player2", "address": "203.0.113.7:47810", "connected_at": "..."}
    ],
    "last_save": {"slot": "server_b", "at": "2026-10-10T01:14:05Z", "count": 47},
    "x4_stats": {"fps": 30, "game_time": 234567, "state": "running"}
}
```

Port 47811 (one above the game protocol port 47810) serves HTTP. This is the monitoring endpoint used by the web dashboard and external monitoring tools.

---

## 8. Monitoring

### 8.1 Health Check Endpoint

The bridge's HTTP endpoint (port 47811) is the primary health check. It's polled every 10 seconds by the monitor script:

```python
# x4coop_monitor.py (simplified)
import requests, time, logging

SERVER = "http://localhost:47811/health"
ALERT_AFTER = 3  # consecutive failures before alerting

def check_health():
    try:
        r = requests.get(SERVER, timeout=5)
        data = r.json()
        if data["status"] == "running" and data["x4_connected"]:
            return True, data
        return False, data
    except:
        return False, None

def main():
    failures = 0
    while True:
        ok, data = check_health()
        if ok:
            failures = 0
            logging.info(f"Server healthy: {data['clients']} clients, save #{data['last_save']['count']}")
        else:
            failures += 1
            logging.warning(f"Health check failed ({failures}/{ALERT_AFTER})")
            if failures >= ALERT_AFTER:
                logging.error("Server unhealthy — triggering restart")
                # The launcher will detect X4 crash and restart
                # Or we can force-kill X4 here
        time.sleep(10)
```

### 8.2 Log Analysis

X4's log file (`x4.log` with `-debug error`) contains:
- Engine errors and warnings
- MD script errors
- Lua errors (from `DebugError()` calls)
- Save/load events

**Key patterns to watch:**

| Pattern | Meaning | Action |
|---------|---------|--------|
| `[x4coop_spectator] auto-saved` | Successful save | Log to dashboard |
| `[x4coop_spectator] save failed` | Save error | Alert, check disk space |
| `Lua Error` / `Script Error` | Mod error | Alert, may need restart |
| `Out of memory` | RAM exhaustion | Alert, restart X4 |
| `Crash` / `Unhandled exception` | Engine crash | Auto-restart by launcher |
| `AIDirector::Move(): Possibly infinite wake request loop` | Known AI bug | Log, monitor for freeze |

### 8.3 Metrics to Track

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| FPS | Heartbeat from Lua | < 10 for 60s |
| Save count | Heartbeat | No increase in 15 min |
| Bridge connection | Bridge health endpoint | Disconnected |
| Client count | Bridge health endpoint | Unexpected disconnect |
| X4 process alive | Launcher | Process exited |
| Disk space | OS | < 5 GB free (saves need space) |
| RAM usage | OS | > 90% |
| CPU usage | OS | > 95% for 60s |
| Save file size | Filesystem | Sudden drop (possible corruption) |
| Save file age | Filesystem | No save in 15 minutes |

---

## 9. Open Questions & Risks

### 9.1 X4 Player Entity — Can the Simulation Run with an Idle Player?

**Confidence: HIGH.** The simulation runs autonomously. Evidence:
- The x4_coop mod's host runs the full simulation while the human is at the keyboard but not actively issuing commands
- NPC factions build stations, fight wars, and trade without player intervention
- The economy is a stock-level simulation that runs on its own timers
- The player can sit idle in a ship for hours and the universe evolves

**Risk:** The player ship might be attacked in its parking sector. Mitigation: invulnerability via `set_object_min_hull`, parking in a core sector (Argon Prime) where hostile NPCs are rare.

### 9.2 Main Menu → Auto-Load Without Human Input

**Confidence: MEDIUM.** There is no documented command-line flag to auto-load a save. The launcher must send simulated keypresses to navigate the menu.

**Risk:** X4 UI changes between versions could break the keypress sequence. Mitigation:
- Keep the sequence simple (ENTER, ENTER)
- Detect menu state from log output
- Fall back to RDP manual intervention if automation fails
- Future: Investigate whether an MD cue can fire from the main menu (before a game is loaded) to trigger auto-load via Lua

**Alternative approach:** Create a custom game start XML that loads directly into the desired configuration. X4's game start system is moddable — a custom start could skip the menu entirely. This needs further investigation.

### 9.3 Save Game Size and Frequency

**Confidence: MEDIUM.** X4 saves are 50–200 MB compressed. Saving every 5 minutes means:
- ~12 saves per hour
- Each save takes 2–5 seconds (game freezes during save)
- Disk I/O: ~1 GB/hour in saves (but rotated, so only 2 files on disk)

**Risk:** Saving causes a brief simulation freeze (2–5 seconds). This is normal in single-player but may cause client desync in co-op. Mitigation:
- The bridge can buffer state during save and send a "server saving" notification to clients
- Clients pause their state application during server save
- Save interval is configurable (5 min default, can be increased to 10–15 min)

### 9.4 Named Pipe on Windows Server

**Confidence: HIGH.** Windows named pipes work identically on Windows Server and desktop Windows. The x4_coop mod's FFI pipe client will work on any Windows version.

### 9.5 X4 Updates Breaking the Mod

**Confidence: MEDIUM.** X4 updates can change:
- Lua function signatures (rare but possible)
- MD cue behavior
- Save format
- Main menu layout

Mitigation:
- Pin X4 to a specific version (Steam beta branches)
- Test mod updates on a staging server before deploying to production
- The launcher's keypress sequence should be version-checked

### 9.6 Multi-Client Scaling

**Confidence: MEDIUM.** The existing x4_coop bridge handles 1:1 (one host, one joiner). Extending to 1:N requires:
- Bridge modification to accept multiple UDP clients
- State broadcast instead of point-to-point relay
- Client authentication and session management
- Per-client state tracking (what each client has acknowledged)

This is a significant bridge modification but doesn't affect the spectator bot mod itself — the mod just reports state to the bridge; the bridge handles distribution.

### 9.7 Does `remove_all_equipment` Exist in MD?

**Confidence: LOW.** The MD script attempts to strip weapons from the player ship using `<remove_all_equipment>`. This MD command may not exist or may have a different name. Alternatives:
- Use Lua to call equipment removal functions
- Simply leave weapons on — the ship won't fire without orders
- Use `<set_object_min_hull>` to make it invulnerable and ignore weapon stripping

**Recommended:** Skip weapon stripping. An invulnerable ship with weapons that has no orders and no target won't fire. It's simpler and avoids potential MD API issues.

---

## 10. Implementation Plan

### Phase 1: Minimal Spectator Bot (1–2 days)

**Goal:** X4 starts, loads a save, and runs with no input. Auto-save works.

- [ ] Create `content.xml` and `ui.xml`
- [ ] Write `ui/x4coop_spectator.lua` with:
  - FFI declarations for game functions
  - Auto-save timer (calls `SaveGame()` every 5 min)
  - Save slot rotation
  - MD ready handshake
  - Basic health heartbeat to bridge
- [ ] Write `md/x4coop_spectator.xml` with:
  - `OnGameStart` cue → raise `md_ready` event
  - `AutoSaveTick` cue → raise `autosave` event every 5 min
  - `InvulnLoop` cue → re-assert player ship invulnerability every 60s
- [ ] Write `x4coop_server_launcher.py`:
  - Start X4 process
  - Send ENTER keypresses to load last save
  - Monitor process, restart on exit
- [ ] Test: Can X4 start, load a save, and run for 30 minutes with auto-saves?

### Phase 2: Bridge Integration (1–2 days)

**Goal:** Server X4 connects to bridge, clients can connect and see the server's state.

- [ ] Extend `x4_coop_bridge.py` with server mode:
  - Accept multiple client connections (1:N)
  - Forward server state to all clients
  - HTTP health endpoint on port 47811
  - "Server saving" notification to clients during save
- [ ] Add spectator bot message types to bridge protocol
- [ ] Test: One client can connect and see the server's universe

### Phase 3: Crash Recovery (1 day)

**Goal:** Server auto-restarts on crash, loads most recent valid save.

- [ ] Implement save validity check in launcher
- [ ] Implement crash detection (process monitoring + health check)
- [ ] Implement automatic restart with save recovery
- [ ] Test: Kill X4 process → launcher restarts and loads correct save

### Phase 4: Monitoring Dashboard (1–2 days)

**Goal:** Web dashboard shows server status, can trigger actions.

- [ ] Write `x4coop_monitor.py` (Flask/FastAPI app):
  - Read bridge health endpoint
  - Parse X4 log file
  - Display status, clients, saves, uptime
  - Action buttons: Save Now, Restart Server
- [ ] Test: Dashboard correctly shows server state, actions work

### Phase 5: Performance Tuning (1 day)

**Goal:** Server runs at minimum GPU load, stable FPS.

- [ ] Configure X4 graphics settings for minimum load
- [ ] Test with HDMI dummy plug (no monitor attached)
- [ ] Measure FPS, CPU, RAM, GPU usage over 1 hour
- [ ] Tune save interval for optimal balance of safety vs. performance
- [ ] Document recommended server hardware specs

### Phase 6: Production Deployment (1 day)

**Goal:** Server starts on boot, survives crashes, remotely manageable.

- [ ] Create Windows Task Scheduler task for auto-start on boot
- [ ] Configure RDP access for remote management
- [ ] Set up SSH (OpenSSH on Windows) for CLI access
- [ ] Write deployment documentation
- [ ] Final end-to-end test: cold boot → server starts → client connects → play for 1 hour → kill X4 → auto-restart → client reconnects

---

## Appendix A: File Structure

```
x4coop_spectator/                    # The X4 mod (goes in extensions/)
├── content.xml
├── ui.xml
├── ui/
│   └── x4coop_spectator.lua         # Core Lua logic
└── md/
    └── x4coop_spectator.xml         # MD timers and game events

x4coop_server/                       # Server infrastructure (lives on the server)
├── x4coop_server_launcher.py        # Starts X4, sends keypresses, restarts on crash
├── x4coop_monitor.py                # Web dashboard + health monitoring
├── x4coop_bridge.py                 # Extended bridge (server mode, 1:N, HTTP health)
├── config/
│   ├── graphics.xml                 # Low graphics settings
│   └── server.conf                  # Save interval, ports, passwords
├── logs/
│   ├── launcher.log
│   ├── x4.log
│   └── bridge.log
└── saves/                           # Backup copies of auto-saves
    ├── server_a.xml.gz
    └── server_b.xml.gz
```

## Appendix B: Command-Line Reference

### X4 Launch Command

```bash
# Windows
"C:\Program Files (x86)\Steam\steamapps\common\X4 Foundations\X4.exe" \
    -debug error \
    -logfile "C:\x4coop_server\logs\x4.log"
```

### Bridge Launch Command (Server Mode)

```bash
python x4coop_bridge.py \
    --host \
    --server-mode \
    --pipe x4_coop \
    --port 47810 \
    --health-port 47811 \
    --password "<shared_password>"
```

### Launcher Command

```bash
python x4coop_server_launcher.py \
    --x4-path "C:\Program Files (x86)\Steam\steamapps\common\X4 Foundations\X4.exe" \
    --bridge-path "C:\x4coop_server\x4coop_bridge.py" \
    --save-slot server_a \
    --restart-delay 10 \
    --log "C:\x4coop_server\logs\launcher.log"
```

### Monitor Command

```bash
python x4coop_monitor.py \
    --bridge-url http://localhost:47811 \
    --web-port 8080 \
    --log "C:\x4coop_server\logs\monitor.log"
```

## Appendix C: MD Script — Full Draft

The MD script in Section 5.2 is a draft. Key areas that need validation against the actual X4 MD API:

1. **`<remove_all_equipment>`** — May not exist. Alternative: skip weapon stripping.
2. **`<set_object_min_hull>`** — Used in x4_coop.xml, confirmed to work.
3. **`player.occupiedship.exists`** — Standard MD condition, used in x4_coop.xml.
4. **`player.age`** — Standard MD time reference, used extensively in x4_coop.xml.
5. **`<raise_lua_event>`** — Standard MD action, used in x4_coop.xml.
6. **Repeating cue pattern** — The `instantiate="true"` + re-triggering pattern needs testing. An alternative is a single cue with `<delay>` that re-queues itself.

**Testing approach:** Use the x4_coop mod's `dev/run_lua.py` and `dev/sim.lua` to test MD/Lua interaction without launching X4. Then validate on a real X4 instance with the mod installed.

---

*Design document for wayfinder ticket #18. Created 2026-10-10.*