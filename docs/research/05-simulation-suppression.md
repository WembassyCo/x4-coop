# Research Ticket #5: X4 Foundations Simulation Suppression via Modding API

**Date:** 2026-10-10  
**Status:** COMPLETE  
**Risk Assessment:** ⚠️ HIGH RISK — Global pause is available but selective subsystem suppression is NOT

---

## Executive Summary

X4: Foundations has a **global pause** mechanism accessible via Lua (`Pause()` / `Unpause()`) and the keyboard (P key / ESC menu). When the system menu pause is active, **all simulation stops** — economy, NPC AI, missions, and faction logic all freeze. Rendering, input, and UI continue to function.

However, there is **no API for selective subsystem suppression**. You cannot pause only the economy while letting AI continue, or freeze NPC ships while production cycles keep running. The pause is all-or-nothing.

This is the riskiest component for the coop mod because the "guest-as-thin-client" approach requires the host's simulation to remain frozen while the guest interacts with the game world. The global pause CAN achieve this, but it means the host player also experiences a fully frozen game (no economy, no NPC movement) while the guest is interacting.

---

## Findings by Question

### 1. Can economy ticks be paused (stop trade simulation, production cycles, NPC trading)?

**YES — but only via global pause, not selectively.**

- The system menu pause (ESC → Options, or `Pause()` in Lua) freezes all simulation including economy ticks, production cycles, and NPC trading.
- There is NO API to selectively pause only the economy.
- **Static modification** is possible: Production cycle times can be modified via XML (wares/factories), and the X4 Customizer can adjust these. But this is a load-time modification, not a runtime pause/resume.
- Example: The "3X Production Cycle Speed" mod on Nexus modifies production cycle durations via XML. This approach could theoretically set production times to infinity (effectively pausing production), but it requires a save/reload to apply.

**Relevant APIs:**
- `Pause()` (Lua, menu environment) — freezes ALL simulation including economy
- No `PauseEconomy()` or similar selective API exists
- XML modification of `<ware>` production elements — static, load-time only

---

### 2. Can NPC AI be paused (stop NPC ships from moving, fighting, trading)?

**YES — via global pause only.**

- The system menu pause freezes all NPC AI. Ships stop moving, fighting, and trading.
- There is no selective "freeze AI" API.
- The AIDirector (internal engine system) handles NPC ship orders and movement. It respects the global pause but cannot be independently suspended via modding.
- **Bug evidence**: A known X4 bug ("AIDirector::Move(): Possibly infinite wake request loop") causes all ships to freeze when the AI gets stuck. This confirms that AI movement is tick-based and would be frozen by a global pause.
- **Job system**: NPC ship counts are controlled by the jobs XML file. The X4 Customizer's `Adjust_Job_Count()` can set multipliers to 0 (effectively removing NPC ships), but this is a static load-time change, not a dynamic pause.

**Relevant APIs:**
- `Pause()` — freezes all AI as part of global pause
- `Adjust_Job_Count()` (X4 Customizer) — static XML modification of NPC ship counts
- No `FreezeAI()` or similar selective API exists

---

### 3. Can the mission system be paused (stop new missions from spawning)?

**PARTIALLY — via global pause, or via MD cue suppression.**

- The global pause freezes mission spawning as part of the all-or-nothing freeze.
- The **Mission Director (MD)** is an XML-based scripting system that controls mission offers and spawning. MD cues fire based on conditions and events. Theoretically, a mod could:
  - Override mission-offering cues with always-false conditions (preventing new missions)
  - Use `<do_if>` blocks with a global "paused" flag to skip mission creation actions
  - But this would require patching every mission-offering cue in the game, which is invasive
- The X4 Customizer offers `Adjust_Mission_Rewards()` and `Adjust_Mission_Reward_Mod_Chance()` — these modify reward parameters, not mission spawning.
- There is no `PauseMissions()` API.

**Relevant APIs:**
- `Pause()` — freezes mission spawning as part of global pause
- MD cue conditions — can theoretically suppress individual missions
- No `PauseMissionSystem()` or similar API exists

---

### 4. Can faction logic be paused (stop faction war/peace calculations, reputation changes)?

**YES — via global pause only. No selective API.**

- Faction logic (war/peace calculations, reputation changes) is part of the simulation that freezes during global pause.
- There is no documented API to selectively pause faction logic.
- Faction behavior is controlled by internal engine systems (GodDirector, AIDirector) that are not directly accessible via Lua or MD.
- Reputation changes from player actions (trading, combat) would also be frozen during pause since the underlying transactions don't process.

**Relevant APIs:**
- `Pause()` — freezes faction logic as part of global pause
- No `PauseFactionLogic()` or similar API exists

---

### 5. Can rendering stay active while simulation is paused?

**YES — this is confirmed.**

- When the game is paused (system menu pause or P key pause), rendering continues. The 3D scene is still drawn, the UI is responsive, and input is processed.
- Players routinely pause the game to issue orders, read the encyclopedia, build stations, and scan ships — all of which require active rendering and input.
- The SirNukes Time API documentation notes: "Some frames may be missed while the game is paused" — indicating the engine still runs frames during pause, but simulation ticks may not advance.
- The MD system can detect frame changes even during pause via `<event_ui_triggered screen="'Time'" control="'Frame_Advanced'" />`, confirming the render loop continues.

**Implication for coop mod**: The host's game can be paused (freezing simulation) while rendering continues, allowing the guest's camera/view to still render the scene. This is critical for the thin-client approach.

---

### 6. Can sector loading stay active (sectors load/unload based on player position)?

**LIKELY YES — but needs verification.**

- Sector loading is tied to the render/visibility system, not the simulation tick system. When the player moves to a new sector, the engine loads it regardless of pause state.
- The global pause does not appear to affect sector loading/unloading since this is a rendering/engine function, not a simulation function.
- During pause, players can still open the map, select objects in other sectors, and issue orders (which will execute when unpaused). This suggests sector data remains accessible.
- **Risk**: If the guest's camera can move to sectors the host hasn't loaded, there may be issues. However, if the guest is constrained to the host's current sector or loaded sectors, this should work fine.

**Needs testing**: Verify that sector loading works correctly during pause by using the map to jump to a distant sector while paused.

---

## Pause Mechanism Deep Dive

### Two Types of Pause

X4 has **two distinct pause systems**:

1. **System Menu Pause** (ESC menu / Options menu)
   - Triggered by opening the ESC menu or calling `Pause()` in a menu's `onShowMenu` handler
   - **Pauses EVERYTHING** — all simulation stops
   - Only the system menu can trigger this full pause
   - Non-system menus (map, trade, etc.) do NOT trigger a full pause

2. **Player Pause** (P key / Pause key)
   - Triggered by pressing P or Pause/Break key
   - Also pauses all simulation
   - Independent from system menu pause — P cannot lift a menu pause, and vice versa

### Lua API: Pause() and Unpause()

```lua
-- Available in the menu environment (UI Lua context)
-- These are globals available to menu scripts, not to MD Lua

Pause()   -- Freezes all simulation
Unpause() -- Resumes all simulation
```

**Critical caveat from the pause-on-load mod (mkoester/x4-mod-pause-on-load)**:
> "Earlier builds called `Pause()` alone: P did nothing visible, and only ESC ESC freed the game, because opening and closing Options ran its `Unpause()`. Opening the Options menu gives the pause an owner, and ESC is its sanctioned exit."

This means:
- `Pause()` must be called from within a menu context (a menu's `onShowMenu` or similar handler)
- A bare `Pause()` call without a menu owner can leave the game in an unrecoverable state
- The safe pattern is to open a (possibly invisible) menu that calls `Pause()` in its `onShowMenu`

### Mission Director (MD) Pause Detection

The MD system can detect when the game is paused and can run cues during pause:

```xml
<!-- This event fires on every frame, even when paused -->
<event_ui_triggered screen="'Time'" control="'Frame_Advanced'" />
```

This is useful for the coop mod because MD scripts can continue to process network events during pause.

---

## SETA (Singularity Engine Time Accelerator)

SETA accelerates game time by **6x** — it's a time multiplier, not a pause mechanism.

- SETA affects the entire simulation uniformly (economy, AI, missions, faction logic all speed up)
- There is **no reverse SETA** — no way to slow time to 0.1x or 0x
- SETA is implemented as a time scale modifier on the simulation tick rate
- The SETA effect is applied through the player ship equipment system, not a general API
- **Cannot be repurposed for pause**: Even if you could set a fractional multiplier, there's no documented API for setting custom time scales. SETA is hardcoded to 6x.

**No useful application for simulation suppression.**

---

## SirNukes Mod Support APIs — Time API

The SirNukes Time API provides **real-time delays and frame callbacks** for mod code. It does NOT control the game's simulation speed.

Key functions:
- `Register_NewFrame_Callback(function)` — called every frame, even during pause
- `Set_Alarm(id, time, function)` — single-fire timer
- `getEngineTime()` — seconds since X4 loaded (only counts active time, not paused)
- `getSystemTime()` — real system time (works even when minimized)

**Useful for the coop mod**: Frame callbacks that work during pause can be used to process network messages and update the guest's view while the simulation is frozen.

---

## Selective Subsystem Suppression — NOT POSSIBLE

After extensive research, there is **no documented API** for selectively disabling specific simulation subsystems at runtime. The simulation is a monolithic tick that either runs or doesn't.

### Workarounds (all have major limitations):

| Approach | What It Does | Limitation |
|----------|-------------|------------|
| Global `Pause()` | Freezes everything | Cannot keep any subsystem running |
| `Adjust_Job_Count(0)` | Removes NPC ships | Static, load-time only, not dynamic |
| Modify production XML | Stops factory production | Static, load-time only |
| MD cue suppression | Prevents specific missions | Requires patching every mission cue |
| Remove all orders from ships | Stops NPC behavior | CPU-intensive, doesn't stop economy |
| SETA | Speeds up time 6x | Cannot slow down or pause |

---

## Existing Mods That Freeze Game State

### 1. Pause on Load (mkoester/x4-mod-pause-on-load)
- **Purpose**: Auto-pauses the game a few seconds after save load
- **Mechanism**: MD cue detects save load → opens Options menu → `Pause()` called in menu handler
- **Key insight**: The mod had to use the Options menu as a "pause owner" because bare `Pause()` is unreliable
- **Source**: https://github.com/mkoester/x4-mod-pause-on-load

### 2. Autocamera and Autopilot and Auto-pause (kuertee)
- **Purpose**: Cinematic camera with auto-pause features
- **Mechanism**: Uses MD + Lua to control camera and trigger pauses
- **Dependencies**: SirNukes Mod Support APIs
- **Key insight**: Confirms that pause is used in cinematic/photo contexts

### 3. X4 Customizer (bvbohnen)
- **Purpose**: Programmatic XML modification framework
- **Relevant features**: Can reduce job counts, adjust production, modify missions
- **Limitation**: All changes are static (applied at load time as diff patches)

---

## Recommendations for the Coop Mod

### Approach A: Use Global Pause (Recommended, with caveats)

1. **Host pauses the game** using `Pause()` (via a custom menu) when the guest is interacting
2. **Rendering continues** — guest's camera view works, UI is responsive
3. **MD frame events continue** — can process network messages during pause
4. **Host unpauses** when the guest is done interacting

**Caveats:**
- The host experiences a fully frozen game during guest interaction
- All simulation stops (economy, AI, missions, factions)
- The host cannot selectively keep any subsystem running
- The pause must be "owned" by a menu to be safely unpausable

**Implementation pattern:**
```lua
-- In a custom menu's Lua environment:
function onShowMenu()
    Pause()
end

function cleanup()
    Unpause()
end
```

### Approach B: "Soft Pause" via Order Stripping (Complex, not recommended)

1. Strip all orders from NPC ships (stops movement/trading)
2. Remove production from stations (stops economy)
3. Suppress mission cues via MD (stops new missions)
4. This would require constant monitoring and order-stripping every tick

**Problems:**
- Extremely CPU-intensive (checking every NPC ship every tick)
- Doesn't stop faction logic or reputation changes
- Very fragile — new ships from job system would need immediate order stripping
- No way to cleanly resume (restoring all orders is impossible)

### Approach C: Hybrid — Global Pause + Post-Pause Selective Re-enable

1. Use global `Pause()` to freeze everything
2. Use MD frame events (which fire during pause) to selectively re-apply specific orders
3. This would allow, e.g., keeping the economy running while AI is frozen

**Problems:**
- The global pause likely prevents the underlying simulation from processing even if you re-apply orders
- No evidence that selectively re-enabling subsystems during pause is possible
- Would require extensive reverse engineering of the engine's tick system

**Verdict: Not viable without engine-level modifications.**

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Global pause is the only option | **Confirmed** | High — host game fully frozen | Accept that coop sessions require full simulation freeze |
| Bare `Pause()` is unreliable | **Confirmed** | Medium — can cause unpausable state | Use the "menu owner" pattern from pause-on-load mod |
| Sector loading fails during pause | **Possible** | High — guest can't see other sectors | Test sector transitions during pause; constrain guest to loaded sectors |
| Performance impact of pause + rendering | **Low** | Low | Rendering continues normally during pause |
| Network sync issues during pause | **Possible** | Medium | Use SirNukes Time API frame callbacks for network processing |
| Engine updates break pause API | **Low** | High | `Pause()`/`Unpause()` have been stable since X4 release |

---

## Conclusion

**The global pause is viable for the coop mod's "guest-as-thin-client" approach**, but with a significant constraint: when the guest is interacting, the host's entire simulation is frozen. This means:

1. ✅ Rendering stays active (guest can see the world)
2. ✅ Input/UI stays active (guest can interact with menus, map, etc.)
3. ✅ All simulation is frozen (no desync between host and guest)
4. ❌ No selective subsystem suppression (economy, AI, missions, factions all freeze together)
5. ❌ Host experiences a fully paused game during guest interaction
6. ❓ Sector loading during pause needs testing

The recommended implementation is:
- Use a custom invisible menu that calls `Pause()` in `onShowMenu` and `Unpause()` in `cleanup`
- Use MD frame events (`event_ui_triggered screen="'Time'" control="'Frame_Advanced'"`) for network processing during pause
- Constrain the guest's camera to currently-loaded sectors
- Accept that coop sessions are "turn-based" — the host plays, then pauses for the guest to interact, then unpauses

If true selective suppression is required (e.g., keeping the economy running while the guest interacts), the only path would be **engine-level modification** (patching X4.exe), which is outside the scope of the modding API.

---

## Sources

- [X4 Lua API Gist](https://gist.github.com/NodusCursorius/b61f26177fdcb490d2456e353d57f363) — Full list of Lua globals including `Pause()` and `Unpause()`
- [pause-on-load mod](https://github.com/mkoester/x4-mod-pause-on-load) — Source code and README documenting pause mechanics
- [SirNukes Mod Support APIs](https://github.com/bvbohnen/x4-projects/tree/master/extensions/sn_mod_support_apis) — Time API documentation
- [X4 Customizer Documentation](https://github.com/bvbohnen/X4_Customizer/blob/master/Documentation.md) — Job adjustment, mission reward, and production modification APIs
- [Egosoft Forum: miscellaneous bug reports](https://forum.egosoft.com/viewtopic.php?t=426335) — "Only the system menu can pause everything"
- [Steam Discussions: pausing while in menus](https://steamcommunity.com/app/392160/discussions/0/3176733964135201626/) — Player usage of pause for orders, scanning, reading
- [kuertee Autocamera/Autopilot/Auto-pause mod](https://www.nexusmods.com/x4foundations/mods/734) — Cinematic pause mod
- [SirNukes Time API docs](https://raw.githubusercontent.com/bvbohnen/x4-projects/master/extensions/sn_mod_support_apis/documentation/Time_API.md) — Frame callbacks during pause