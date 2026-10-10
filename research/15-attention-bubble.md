# Research #15: X4 High/Low Attention Bubble System & Co-op Impact

**Date:** 2026-10-10
**Status:** Complete
**Risk Level:** HIGH — This is the single biggest technical risk for the co-op project

---

## Executive Summary

X4: Foundations uses a two-tier simulation model: **high attention (HA)** for the sector the player/camera is in, and **low attention (LA)** for everything else. The engine can actually support **multiple HA bubbles** — the Live Stream feature and NPC communications both create additional HA bubbles. This means the attention bubble problem is **not a hard engine limit** but a design decision that can potentially be worked around.

For co-op, the x4_coop mod by DanyWany22 already handles players being in different sectors by using a "each game is authoritative near its own player" model, with the host's world being the persistent truth. This works but has limitations for combat in distant sectors.

---

## 1. What is High Attention? What Does the Engine Simulate Differently?

### High Attention (HA) — Full Simulation
The current player/camera zone and its neighbours are fully simulated. Zones are cube sections of space extending up to 50km per axis. The closest zone center determines which zone an object belongs to. HA therefore varies between 50km and 150km radius, roughly approximating to a **~100km bubble centered on the player/camera**.

**What HA simulates:**
- **Physics:** Full Newtonian physics, collision detection, momentum
- **AI:** Full AI behavior — dogfighting maneuvers, evasive patterns, pathfinding with collision avoidance
- **Combat:** Individual projectiles are simulated, turret aiming, missile paths, shield/hull damage per-hit
- **Docking:** Full docking animations and physical ship approach
- **Mining:** Individual asteroids are simulated, collision exists, physical laser/drill mining
- **Movement:** Travel drive, boost, precise 3D positioning

### Low Attention (LA) — Simplified Simulation
Everything outside the HA bubble switches to a much simpler model.

**What LA simulates (differently):**
- **Physics:** **No physics simulation** — ships are essentially running in the background without physics
- **Combat:** No projectiles simulated. Uses estimated hit chance and damage based on **sustained DPS** of weapons. Ship turrets only have a **30% chance to fire** in LA (vs. normal behavior in HA). This is a dice-roll system, not a physics simulation
- **AI:** No pathfinding, no collision avoidance, no dogfighting maneuvers. Ships use simplified movement calculations
- **Docking:** Physics-free model handles docking (and ironically, often works *better* than HA docking because X4's docking AI is notoriously buggy in HA)
- **Mining:** Individual asteroids not simulated, collision doesn't exist. Simplified collection makes LA mining significantly more efficient
- **Defense drones:** Still contribute to LA battle simulation (increase DPS, absorb damage) despite not being visually launched — a common misconception is they do nothing

### Key Differences — "Like Two Different Games"
The differences are so vast that each mode functions like an entirely different game:
- **Turret-heavy ships** (e.g., Xenon destroyers) have drastically reduced DPS in LA due to the 30% turret fire chance
- Weapons that are devastating in HA may be useless in LA, or vice versa
- Combat outcomes for the same engagement can differ dramatically based on simulation mode
- The IS/OOS (In-Sector/Out-of-Sector) terminology from older X games is misleading — X4 has no loading screens, it's all about the attention bubble

**Sources:**
- Steam Guide: "Hidden Mechanics & Penalties" (steamcommunity.com/sharedfiles/filedetails/?id=3463911140)
- Reddit: r/X4Foundations multiple threads on HA vs LA
- Steam community discussions on docking AI

---

## 2. Can We Have Two High-Attention Bubbles?

### **YES — The engine already supports this.**

This is the most critical finding of this research. The engine is NOT limited to a single HA bubble.

### Evidence: Live Stream Feature
The Live Stream feature (introduced in X4 6.00) creates a **second high attention bubble** at the livestreamed object's location:

> "If you livestream to a location, that location becomes a second high attention bubble as if you were on board of the livestreamed object."
> — Reddit: r/X4Foundations, "Current State of High Attention vs Low Attention mode"

> "It turns it into 'in sector' high attention combat, and the results of a fight can change."
> — Reddit: r/X4Foundations, "Live stream: sleeper hit feature of 6.0"

> "This is a cinematic camera that focuses on ships, stations, or gates. In addition to being something to look at while having your captain fly you around, it can also be used if you want to force 'high attention' simulation into an area without actually being there yourself."
> — Steam community discussion

### Evidence: NPC Communications
Opening communications with an NPC also extends the HA bubble to the NPC's location:

> "The 'camera' includes the location of the Live Stream and an NPC when opening communications with them."
> — Hidden Mechanics & Penalties guide

### Implications for Co-op
- **The engine is NOT hard-limited to one HA bubble.** It can track multiple simultaneously.
- The Live Stream proves a second bubble works for combat, physics, and AI
- The mechanism appears to be tied to "camera" locations — wherever the game considers the camera to be, that location gets HA
- **Potential workaround:** If we can trick the engine into treating the guest player's sector as a "camera" location (similar to Live Stream), we could get HA simulation for both players' sectors simultaneously

### Unknowns
- Whether the engine supports more than 2 simultaneous HA bubbles (2 is confirmed via Live Stream + player)
- Whether the Live Stream's HA bubble is truly equivalent to the player's HA bubble in all simulation aspects, or if it's a partial HA
- Performance impact of running 2+ HA bubbles (likely significant — HA is CPU-intensive)
- Whether the modding API exposes any function to set/query attention mode directly (no such function was found in searches)

---

## 3. Low-Attention Behavior When Guest is Paused

### Key Insight: If the guest's simulation is paused, LA may be sufficient for rendering

In our co-op architecture, the guest receives state from the host. If the guest's X4 engine is:
- **Paused** (not running its own simulation)
- **Receiving all state from the host** (positions, health, AI states)
- **Only rendering** the received state

Then the guest's X4 doesn't need HA simulation at all. The LA mode would be fine because:
- The guest's engine doesn't need to calculate combat — the host does that
- The guest's engine doesn't need physics — positions come from the host
- The guest's engine doesn't need AI — the host runs AI
- The guest only needs to **render** ships at the positions the host sends

### However:
- X4's rendering is tied to its simulation. Ships don't just "appear" at positions — they need to exist as game objects with positions, orientations, velocities
- Even in LA, the engine still tracks positions and basic movement
- The question is whether we can suppress the guest's own simulation while still having it render objects at host-provided positions
- **This approach would require the guest's engine to not independently simulate anything** — all state must come from the host

### x4_coop's approach:
The x4_coop mod does NOT pause the guest's simulation. Instead, it runs both simulations in parallel and synchronizes them:
- "Each game is the truth for the ships around its own player"
- "When you're apart, even in different sectors: the host's copies of the ships around the joiner follow the joiner's game"
- Both games run their own simulations, and the mod reconciles differences

---

## 4. Sector Loading on Guest When Paused

### Unknown — but likely manageable

When the guest is paused:
- **Sectors should still load** around the guest's position because the rendering engine needs sector geometry to display anything
- **Sector loading is a rendering concern**, separate from simulation
- The guest's position **does matter** for sector loading — X4 loads/unloads sector geometry based on the player's location
- If the guest is paused and receiving host state, the guest's X4 still needs to:
  - Load the sector the guest is visually in (for rendering)
  - Display ships at host-provided positions
  - NOT independently simulate AI, physics, or combat

### Practical approach:
- The guest's player character/ship should be positioned in the sector the guest is visually viewing
- Sector geometry loads around that position
- All dynamic objects (ships, stations) get their state from the host
- The guest's engine runs in a degraded/paused simulation mode

---

## 5. Host Maintaining Two HA Sectors

### The critical question: Can the host's X4 simulate two sectors at HA simultaneously?

**Based on the Live Stream evidence — YES, this should be possible.**

The Live Stream feature proves the engine can maintain HA in two locations simultaneously:
1. The player's actual location (primary HA bubble)
2. The Live Stream target (secondary HA bubble)

Both locations get full physics, AI, and projectile simulation.

### For co-op, the host would need:
- HA for the host player's sector (primary bubble)
- HA for the guest player's sector (secondary bubble, treated like a Live Stream target)

### Challenges:
- **Performance:** Two HA bubbles is CPU-intensive. HA simulation is the most expensive part of X4's processing. Performance may degrade significantly with two active HA bubbles, especially in busy sectors
- **Scale:** If both sectors have large fleet battles, the combined simulation load could overwhelm a single PC
- **Modding API:** There is no documented modding API function to create a HA bubble. The Live Stream feature is built into the engine's C++ code. Mods may not be able to invoke it programmatically
- **No known modding function** like `SetSectorAttention()` or `ForceHighAttention()` was found in any search

### Potential approaches:
1. **Use the Live Stream mechanism** — if the modding API can trigger a Live Stream to the guest's sector, the engine might give it HA
2. **NPC comm trick** — opening comms with an NPC extends the HA bubble; could we open comms with an NPC in the guest's sector?
3. **Memory hacking** — the x4_coop mod's demo shows memory reading/writing is possible; could we set the attention flag directly in memory? (x4_coop explicitly avoids this, but our project might consider it)

---

## 6. Impact on Combat in Different Sectors

### If the host only runs HA for the host's sector:
- Combat in the guest's sector runs in **LA on the host's machine**
- LA combat is **significantly different** from HA combat:
  - 30% turret fire chance (vs. normal in HA)
  - No individual projectiles — just DPS calculations
  - No dogfighting maneuvers
  - No physics-based collision
  - Different weapon effectiveness profiles
- **The host's simulation would NOT accurately resolve combat** in the guest's sector if it runs in LA
- This means: if the guest is in a dogfight in a different sector, the host's LA simulation would produce a different outcome than what the guest experiences in HA

### If the host runs HA for both sectors (using Live Stream-like mechanism):
- Combat in both sectors would be fully simulated
- The host's simulation would accurately resolve combat in the guest's sector
- The guest would receive accurate combat results from the host
- **This is the ideal scenario** but requires solving the two-HA-bubble problem

### x4_coop's approach to combat across sectors:
- "Kills and damage: a ship one of you kills dies in the other's world too, wherever it is, even in a sector the other game is only simulating roughly"
- "Hits lower its hull on the other side, so both players' damage adds up"
- Each game runs its own simulation near its player, and syncs kills/damage to the other
- **When apart, combat near the joiner is simulated by the joiner's game** (in HA), and results are sent to the host
- The host's world near the joiner follows the joiner's game
- **This means the host doesn't need HA for the guest's sector — the guest's own game provides HA there**

### For our project:
If we use a "pause guest simulation, host provides all state" architecture, the host MUST run HA for the guest's sector for combat to work properly. If we use a "both games simulate, sync differences" architecture (like x4_coop), each game provides HA for its own area.

---

## 7. How x4_coop Handles the Attention Bubble Problem

### Architecture: Dual-simulation with synchronization
x4_coop does NOT try to solve the attention bubble problem. Instead, it **works around it** by running two full simulations in parallel:

**Each game is authoritative near its own player:**
- The host's game is the truth for ships near the host
- The joiner's game is the truth for ships near the joiner
- Both games send their area's ships to the other 10 times/second
- The other game moves its copies to match

**When together (same sector):**
- The joiner follows the host
- Ships only the host has get a stand-in on the joiner's side
- Ships only the joiner has, well inside the host's range, are removed there

**When apart (different sectors):**
- "Even in different sectors: the host's copies of the ships around the joiner follow the joiner's game, in the same way"
- The joiner's copies around the host follow the host
- Hull follows the game that's in charge of that area
- **Both players do NOT need to stay in the same sector** — the mod explicitly supports being apart

**Combat sync:**
- Kills sync across both worlds regardless of sector
- Damage syncs — hits lower hull on the other side
- Partner's ship fires at what they shoot at (proxy fire)
- NPCs' own AI decides when to shoot

**Economy sync:**
- Every station's stock follows the host's, galaxy-wide
- Host sends each station's cargo (~1 pass/minute)
- Joiner's trades count in the host's world

### Key design decision:
x4_coop **embraces LA** for sectors away from either player. The joiner's own game provides HA for the joiner's area, and the host's game provides HA for the host's area. Away from both players, things still differ between the two worlds — the host's world is the "real" one, and they re-share saves next session.

### Limitations:
- "Away from both of you, what ships carry, their exact routes between the host's reports, and fights there still differ between the two worlds"
- The economy is synchronized approximately (~1 pass/minute), not tick-perfect
- Both players must run the host's save (shared world mode) for full sync
- No "pause one game" option — both games run full simulations

---

## 8. Workarounds and Techniques

### 8.1 Live Stream (Vanilla Feature)
- **What it does:** Creates a second HA bubble at the livestreamed object's location
- **Confirmed to work:** Changes combat outcomes (HA combat vs LA combat)
- **Limitation:** Requires the player to actively view the stream; not a persistent background HA
- **Modding potential:** If the modding API can trigger Live Stream programmatically, we could force HA on the guest's sector

### 8.2 NPC Communications (Vanilla Feature)
- **What it does:** Opening comms with an NPC extends the HA bubble to their location
- **Limitation:** Requires an NPC to comm with in the target sector; comms UI is intrusive
- **Modding potential:** Minimal — too hacky for persistent use

### 8.3 OOS Turret Fire Chance Change (Mod)
- Steam Workshop mod (id: 3249159201)
- **What it does:** Reduces the 30% LA turret fire penalty, making LA combat more closely match HA combat
- **Relevance:** Doesn't solve the core problem but reduces the disparity between HA and LA combat results
- **Could help:** If the host must run the guest's sector in LA, this mod would make combat results closer to what they should be

### 8.4 KUDA AI Tweaks (Mod)
- Nexus Mods (id: 839)
- **What it does:** Improves AI behavior in both HA and LA, including "event listeners so that ships in low-attention function like in visible attention"
- **Relevance:** Could reduce the HA/LA behavioral gap

### 8.5 No Known Modding API for Direct Attention Control
- Searches for `SetSectorAttention`, `ForceHighAttention`, `attentionmode`, `SetAttention` etc. returned no results
- The Egosoft Mission Director Guide mentions an `attention` enumeration type, but no function to set it
- The attention system appears to be controlled at the engine level (C++), not exposed to the MD/Lua modding API
- **This is a significant gap** — without API access, mods cannot directly control attention mode

### 8.6 Memory Hacking (Theoretical)
- x4_coop's demo (`demo/x4_memory_demo.py`) proves reading/writing X4's memory is possible
- Could theoretically set attention flags directly in memory
- **x4_coop explicitly avoids this** for safety and compatibility reasons
- Our project could consider it as a last resort, but it's fragile (memory addresses change with game updates)

---

## 9. Practical Impact for Co-op Play

### How often would players be in different sectors?

**Frequent scenarios where players would be apart:**
1. **Division of labor:** One player manages station building while another explores or fights
2. **Multi-front war:** Defending against Xenon in two sectors simultaneously
3. **Trading vs. combat:** One player runs trade routes while another does combat missions
4. **Exploration:** One player explores new sectors while another manages existing operations
5. **Mining operations:** Different mining locations in different sectors

**Frequent scenarios where players would be together:**
1. **Joint combat operations** — the most fun co-op activity
2. **Station construction** — both players contributing to the same build
3. **Story missions** — playing through the story together
4. **Boarding operations** — coordinated captures
5. **Early game** — both players likely stick together when they have few assets

### Assessment:
For a **casual co-op playthrough**, players would likely be together **60-70% of the time**. The attention bubble problem is NOT an edge case — it would arise regularly, especially as the empire grows and players split responsibilities.

For a **competitive/efficiency-focused playthrough**, players would be apart **frequently** — min-maxers would split up to cover more ground.

### When apart matters most:
- **Combat** is the critical case. If one player is in a fight while the other is elsewhere, the fight must be simulated correctly
- **Economy** is less critical — slight desyncs in station stock are tolerable and self-correcting
- **Exploration** is trivial — no simulation needed, just map knowledge sharing

---

## Architectural Recommendations for Our Co-op Project

### Option A: Dual-Simulation (x4_coop approach)
- Both games run full simulations
- Each is authoritative near its own player
- Sync kills, damage, positions, economy periodically
- **Pros:** No need to solve the attention bubble problem; proven to work
- **Cons:** Two simulations can diverge; requires periodic re-sharing of saves; complex sync logic; both PCs need full processing power

### Option B: Host-Authitative with Live Stream Trick
- Host's game runs HA for both sectors (using Live Stream-like mechanism)
- Guest's game is paused/limited, receives all state from host
- Guest only renders
- **Pros:** Single source of truth; no divergence; guest PC can be lower-spec
- **Cons:** Requires solving the two-HA-bubble problem (may need memory hacking or undocumented API); host PC needs to handle 2x HA load; complex to implement

### Option C: Hybrid — Dual-Sim When Apart, Host-Auth When Together
- When in the same sector: host is authoritative, guest receives state
- When apart: both games simulate their own area, sync differences (like x4_coop)
- **Pros:** Best of both worlds; simpler when together; flexible when apart
- **Cons:** Two different sync modes to implement; mode switching complexity

### Recommended: Option C (Hybrid)
This matches what x4_coop already does and adds a "pause guest sim" mode for when players are together. The attention bubble problem is sidestepped rather than solved — each game provides HA for its own player's area.

---

## Sources

1. **Hidden Mechanics & Penalties** — Steam Guide (id: 3463911140) — Comprehensive HA/LA explanation
2. **x4_coop mod** — GitHub: DanyWany22/x4_coop — Full README with shared world details
3. **Reddit: r/X4Foundations** — Multiple threads on Live Stream HA bubble, combat calculations, LA behavior
4. **Steam Community Discussions** — Threads on docking AI, LA vs HA, multiplayer discussions
5. **Reddit: "Live stream: sleeper hit feature of 6.0"** — Confirms Live Stream creates HA bubble
6. **Reddit: "Does live stream move the high attention bubble?"** — Confirms HA combat via Live Stream
7. **Reddit: "Live Stream feature and combat calculation"** — Community discussion of HA via Live Stream
8. **Reddit: "Current State of High Attention vs Low Attention mode"** — Detailed HA/LA comparison
9. **Reddit: "AI ship simulation"** — Confirms no physics/pathfinding in LA
10. **OOS Turret Fire Chance Change** — Steam Workshop mod (id: 3249159201)
11. **KUDA AI tweaks** — Nexus Mods (id: 839)

---

## Key Takeaways

| Question | Answer |
|----------|--------|
| Is HA a hard engine limit? | **No** — Live Stream proves multiple HA bubbles work |
| Can modding API control attention? | **No documented API found** |
| Does x4_coop require same sector? | **No** — explicitly supports being apart |
| Is the attention bubble problem solvable? | **Yes, via workarounds** — dual-sim or Live Stream trick |
| Is this the biggest risk? | **Still high risk** — but less of a blocker than originally thought |
| Can the host sim two sectors at HA? | **Theoretically yes** — engine supports it, but performance and API access are concerns |
| How often would co-op players be apart? | **30-40% of the time** — not an edge case |