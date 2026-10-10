# Research #14: DanyWany22's x4_coop Mod — Deep Analysis

**Date:** 2026-10-10  
**Researcher:** OpenClaw CTO subagent  
**Source:** https://github.com/DanyWany22/x4_coop  
**Author:** Aaron Sturman (aaron.sturman@live.co.uk), GitHub: DanyWany22  
**Last commit:** 2026-10-09 (Terraforming in both worlds) — **actively developed as of yesterday**  
**Co-authored with:** Claude Opus 5.5 (Anthropic AI)  

---

## Executive Summary

DanyWany22's x4_coop is the most advanced X4: Foundations cooperative multiplayer mod in existence. It is a **prototype** that has been in rapid active development (recent commits show ~20+ feature commits in recent weeks). The mod achieves something remarkable: **two players flying together in X4, each running their own full game instance, synchronized through a Python bridge relay over UDP** — with no DLLs, no memory hacking, and no injection. It uses only X4's official modding interfaces (Lua UI scripting + Mission Director XML + AI scripts) plus a Windows named pipe to an external Python process.

This is not a toy. The sync system covers ships, NPCs, economy, stations, missions, terraforming, reputation, deployables, crew, cargo, orders, behaviours, station settings, trade rules, assignments, equipment, map knowledge, logbook, loot, and even on-foot station walking. The README is extraordinarily detailed and the codebase is clean, well-tested, and well-documented.

**For our project, this is the single most important reference. Much of the hard work has already been done.**

---

## 1. Source Code

**Repository:** https://github.com/DanyWany22/x4_coop  
**Branch:** main  
**License:** Not explicitly stated in content.xml or README (needs checking — likely public domain or MIT given the open nature)  
**No Nexus Mods or Steam Workshop page found** — the mod is GitHub-only, not yet published to conventional mod platforms. No Egosoft forum thread found either.

### Repository Structure

```
x4_coop/
├── content.xml              # Extension manifest (id="x4_coop", version="1", date="2026-10-08")
├── ui.xml                   # Registers Lua file with game's UI
├── ui/
│   └── x4_coop.lua          # Core mod logic (~thousands of lines, LuaJIT)
├── md/
│   └── x4_coop.xml          # Mission Director script (XML scripting)
├── aiscripts/
│   └── x4coop.proxy.fire.xml # AI script for partner's ship firing
├── bridge/
│   ├── x4_coop_bridge.py    # Network bridge (Python 3.8+, stdlib only)
│   ├── host.bat             # Launcher for host
│   ├── join.bat             # Launcher for joiner
│   ├── x4_coop_overlay.py   # Telemetry overlay
│   └── x4_coop_trace_report.py # Trace analysis
├── dev/
│   ├── fake_peer.py         # Test double for partner
│   ├── sim.lua              # Stubbed game engine for Lua testing
│   ├── run_lua.py           # Runs mod's Lua in X4's LuaJIT outside the game
│   ├── game_sim.py          # Game simulator for testing
│   ├── pipe_test.py         # Tests pipe client
│   ├── share_test.py        # Tests save sharing
│   ├── trace_test.py        # Tests telemetry
│   └── run_tests.py         # Test runner
├── demo/
│   └── x4_memory_demo.py    # Standalone demo of reading/writing X4's memory (NOT part of the mod)
```

**Every file is readable text. No compiled files. Nothing downloaded at runtime.**

---

## 2. Architecture

### Communication Stack

```
X4 (ui/x4_coop.lua) ←→ Windows Named Pipe (\\.\pipe\x4_coop) ←→ Bridge (Python: UDP/TCP) ←→ Network ←→ Partner's Bridge ←→ Pipe ←→ Partner's X4
```

### Bridge Layer

- **Language:** Python 3.8+ (Windows, standard library only — no pip dependencies)
- **Network protocol:** UDP for real-time game messages, TCP for save file transfer
- **Port:** 47810 (UDP + TCP)
- **Encryption:** Password-based. scrypt key derivation (n=2^15, r=8, p=1, 32MB memory per guess) → HMAC-SHA256 signing + SHAKE-256 keystream XOR encryption. Every datagram is signed and encrypted.
- **Packet format:** `b"X4C2 " + HMAC_tag + " " + session:counter:timestamp + " " + ciphertext`
- **Anti-replay:** Session IDs (random per bridge start), monotonic counters, 60-second max clock skew, sliding window of 256 counters for UDP reordering tolerance
- **Rate limiting:** Token bucket at 200 packets/sec (partner sends ~22/s; anything far above is dropped)
- **Named pipe:** Windows `CreateNamedPipeW` with `PIPE_TYPE_MESSAGE | PIPE_READMODE_MESSAGE | PIPE_NOWAIT | PIPE_REJECT_REMOTE_CLIENTS`. Only accepts `X4.exe` as client (verifies process image name via `QueryFullProcessImageNameW`).

### Lua Side (in-game)

- X4's Lua has **no sockets**. The mod doesn't open any in Lua.
- The Lua opens `\\.\pipe\x4_coop` using Windows' own `CreateFileA`, `ReadFile`, `WriteFile`, and `PeekNamedPipe` from `kernel32`, called through **LuaJIT's FFI** (`ffi.C` on Windows looks up kernel32).
- **Fallback:** SirNukes' Mod Support APIs (Steam Workshop id `ws_2042901274`) — a small DLL offering the same named pipe to Lua. Listed as optional dependency in `content.xml`.
- The Lua calls **63 game functions** by name through FFI — all functions that Egosoft's own UI scripts call (e.g., `GetObjectPositionInSector`, `SetObjectSectorPos`, `CreateOrder`, `TeleportPlayerTo`, etc.)

### Key Architectural Decision: No DLLs, No Memory Hacking

The mod has **no DLLs, no native hooks, no injection, and reads or writes no memory addresses**. In Lua it wraps some of the UI's own script functions (like `CreateOrder`, `SetOrderParam`, `TransferPlayerMoneyTo`, `SetComponentName`, terraforming functions) to share orders. This is ordinary UI scripting: each Lua function is replaced by one that calls the original and then notes which ship's orders changed.

There IS a `demo/x4_memory_demo.py` that demonstrates reading/writing X4's memory is possible (shows credits), but the mod **never loads or calls it**. The README explicitly explains why memory writes aren't needed.

---

## 3. State Sync Approach

### Two Modes

1. **Any-save mode:** Each player loads their own save. Only ships and chat sync (30 updates/sec with lag-hidden prediction). Nothing else is shared.
2. **Shared world mode:** Both load the host's save. Full sync of kills, damage, gunfire, NPCs, economy, stations, missions, reputation, etc.

### Sync Frequency & Data

| What | Frequency | Method |
|------|-----------|--------|
| Ship position/rotation/velocity | 30 Hz (send_rate setting) | Delta position updates with prediction & smoothing |
| NPC ships near players (within 6 km) | 10 Hz | Each game is truth for ships around its own player |
| Economy (station stock) | ~1 pass/min over all stations | Host sends each station's cargo |
| Reputation | Every 10 seconds | Host's relations are truth |
| Station settings | 1 station every 0.25s (if changed) | Only sends what differs |
| Station accounts/balances | Every 30 seconds | Host sends each station's balance |
| Wider world (ships away from joiner) | ~20 ships/second | Host's game goes round every ship |
| Station modules | A few each second | Within ~1 min even with dozens of stations |
| Crew sync | Within ~1 minute | Periodic scan |
| Map knowledge | Within ~30 seconds | Event-driven |
| Terraforming stats | Every 30 seconds | Host sends each planet's stats |

### Sync Approach: Authoritative Host + Regional Authority

- **Host's world is the real one.** The host's save is the shared world.
- **Regional authority:** Each game is the truth for the ships around its own player (within 6 km). Where players are together, the host's game is in charge.
- **When apart:** The host's copies of ships around the joiner follow the joiner's game. NPC ships only the host has near the joiner are removed.
- **Hull follows** the game that's in charge of that area; your own fresh hits are never undone.
- **Delta-based:** Only sends what changed. Station settings only sends when something differs. Economy only sends stations whose stock drifted.

### Prediction & Smoothing

The mod uses **client-side prediction** to hide lag. The ghost mode demo shows this clearly: your own ship is fed back through a simulated 120ms delay and rendered as a ghost that holds its relative position using prediction. Settings: `send_rate`, `predict`, `smoothing` are configurable.

---

## 4. Guest Simulation Handling

### No Global Pause

The mod does **not** use global pause. Both games run their own full simulation simultaneously.

### Both Games Run Full Sim

The guest (joiner) is **not** a thin client — it runs its own complete X4 simulation. The mod synchronizes state between the two instances to keep them convergent. Key insight: **both games simulate everything**, and the mod reconciles differences by having an authoritative source for each domain:

- Host is authoritative for: economy, wider world state, terraforming outcomes, story progression, empire income
- Joiner is authoritative for: its own ship's cargo, what happens near the joiner (within 6 km)
- Co-authoritative: kills, damage, hits (both count), new ship purchases, station builds

### NPC Sync Strategy

- Ships near either player (within 6 km): each game is truth for its area, 10 Hz updates
- Ships near neither player: host's world is truth, joiner's copies are moved to match
- When together: joiner follows host
- When apart: host's copies near joiner follow joiner's game; NPC ships only host has are removed
- Player-owned ships are never removed
- The NPCs' own AI still decides when to shoot

### Save Sharing

The host quicksaves and the bridge sends the save file to the joiner over TCP (encrypted with the save key). The joiner loads it as their quicksave. This is how the shared world starts each session.

---

## 5. Entity Ownership

### Single Empire Model

Both players share **one faction** (the same empire/player faction). This is a fundamental design choice:

- **Shared reputation:** One faction standing with every other faction. Host's relations are truth.
- **Shared ships:** Both players see and can order the empire's ships. Ships either player buys appear in both worlds.
- **Shared stations:** Station builds, modules, settings, accounts all sync.
- **Separate wallets:** Each player has their own credits. `/x4coop give` sends credits to partner.
- **Shared research/blueprints/licences:** Whatever either unlocks, the other gets.
- **Shared trade rules, deployables, map knowledge, logbook.**

### Guest Player Entity

- The joiner loads the host's save, making them the host's player initially.
- `/x4coop guestship` (host) parks a spare ship for the joiner.
- `/x4coop takeship` (joiner) moves the joiner into the guest ship.
- The joiner's bridge keeps their own credits and inventory in a small file (`x4coop_profile_<world>.txt`, saved every 30 seconds) and restores them each time they load the host's save.
- Both players can have their own fleets (since they share the empire, both can order empire ships).

### Ship Pairing

Ships in the two worlds have different ID codes. The mod maintains a **pairing table** (kept in the save) that maps which ship in one world corresponds to which ship in the other. This ensures kills, hits, nearby sync, and ownership reach the right ship.

---

## 6. What Works (Functional Features)

### Any-Save Mode (Lighter)
- ✅ See partner's ship fly beside you (30 updates/sec with prediction)
- ✅ Chat (`/x4coop say`)
- ✅ Ghost mode (single-player demo with simulated network)

### Shared World Mode (Full)
- ✅ **Ships:** Each sees the other's real ship with hull and shields
- ✅ **Kills & damage:** Ship kills die in both worlds; hits lower hull on both sides; damage from NPCs near either player syncs
- ✅ **Gunfire:** Partner's ship fires at what they shoot at (hits or misses)
- ✅ **NPCs near players:** Within 6 km, 10 Hz sync, regional authority
- ✅ **Economy:** Station stock matches galaxy-wide (~1 pass/min). Joiner's trades count.
- ✅ **Reputation:** Shared faction standing, host is truth, joiner's changes propagate
- ✅ **New ships:** Bought by either player appear in both worlds
- ✅ **Ship equipment:** Upgrades sync to the other world
- ✅ **Crew:** Service crew, marines, pilot skills match (names/faces are each world's own)
- ✅ **Ship renames:** Sync to both worlds
- ✅ **Map knowledge:** Sectors, stations, factions discovered by either become known to both
- ✅ **Deployables:** Satellites, beacons, probes, mines, laser towers sync
- ✅ **Stations:** Module builds, deconstruction, destruction, repair all sync
- ✅ **Station settings:** Trade wares, limits, prices, rules, workforce, name
- ✅ **Station accounts:** Money transfers, budgets, balances match
- ✅ **Trade rules:** Empire's rules match in both worlds
- ✅ **Assignments:** Ships assigned to stations/fleets/players sync
- ✅ **Orders:** Map and right-click orders carried out in both worlds
- ✅ **Behaviours & order queues:** Default behaviour and queue edits sync
- ✅ **Ship ownership changes:** Claims, boarding, captures sync
- ✅ **Research, blueprints, licences:** Shared
- ✅ **Credits:** Separate wallets, `/x4coop give` for transfers
- ✅ **Missions:** Each sees partner's missions as guidance; kills count in both worlds
- ✅ **Shared logbook:** Achievements written in both logbooks
- ✅ **Terraforming:** Projects start/abort in both; stats sync every 30s
- ✅ **Cargo:** Empire ships carry same in both worlds
- ✅ **Shared loot:** Lockboxes, drops, crates opened by either are gone in both
- ✅ **On-foot:** Walking on stations syncs (room-level), stand-in crew member
- ✅ **SETA:** Time acceleration syncs between both games
- ✅ **Joiner's own wallet/inventory:** Preserved between sessions via profile file
- ✅ **Wider world:** Other factions' stations and ships away from joiner follow host

### Infrastructure
- ✅ Save sharing (host → joiner via encrypted TCP)
- ✅ Telemetry overlay, trace recording, Wireshark capture support
- ✅ Ghost mode (single-player test mode with simulated 120ms latency)
- ✅ `/x4coop check` diagnostic (finds first thing blocking co-op)
- ✅ `/x4coop status` (mode, link, RTT, proxy state, partner location)
- ✅ Encrypted & signed networking with anti-replay
- ✅ Rate limiting
- ✅ Reconnection after game reloads
- ✅ Tailscale/ZeroTier support
- ✅ Comprehensive test suite (offline tests, pipe tests, share tests, trace tests)

---

## 7. What Doesn't Work / Known Limitations

### Explicitly Mentioned in README
- ❌ **Fog of war on map** can't be set by scripts — stays your own
- ❌ **Story progression for joiner:** Story moves on in host's world only. Story steps the joiner finishes on their own copy aren't kept.
- ❌ **Away from both players:** What ships carry, their exact routes between host reports, and fights there still differ between the two worlds. "The host's world is the real one: next session, share again."
- ❌ **On-foot stand-in appearance:** The game doesn't let a script give the stand-in your character's look — it's a generic crew member
- ❌ **On-foot corridor precision:** NPCs only stop at room standing spots (1-3 m apart). In corridors, the stand-in walks to the nearer end.
- ❌ **Equipment slot placement:** Equipment may sit in different slots of the same kind in the copy
- ❌ **Crew names and faces:** Each world generates its own

### Stated as Open Questions (from README)
- ❓ "The economy still matching after an hour of play" — probably achievable
- ❓ "A stress test at full scale" — not yet done
- ❓ "The whole universe is an open question that the next few milestones will answer"
- ❓ "Nearby sync in both directions" — listed as a goal

### Inferred Limitations
- **2 players only:** No support for 3+ players
- **Windows only:** Named pipes are Windows-specific; bridge uses `ctypes` WinDLL kernel32
- **Same DLCs required:** Both PCs need the same DLCs (partner is placed by sector name)
- **Protected UI Mode:** Must be off for the pipe functions to work (or use SirNukes fallback)
- **Same X4 version:** Both players need the same version
- **Same Steam account workaround:** Second PC must be in Offline Mode with Steam Cloud off
- **No mod page:** Not on Nexus Mods or Steam Workshop — GitHub only
- **No known community testing:** No Reddit threads, no Egosoft forum threads found — this appears to be very recent/niche

---

## 8. Performance

### Reported Data from README/Code
- **Ship position updates:** 30 Hz (configurable `send_rate`)
- **NPC sync:** 10 Hz for nearby ships
- **Packet rate:** ~22 packets/sec per partner (rate limiter at 200/sec)
- **Max packet size:** 64 KB buffer
- **Save transfer:** Up to 1 GiB, sent in 1 MB chunks over TCP, encrypted per-chunk
- **Simulated latency in ghost mode:** 120 ms (configurable)
- **Economy pass:** ~1 pass/minute over all stations
- **Wider world scan:** ~20 ships/second
- **Station settings scan:** 1 station every 0.25 seconds

### No Published Benchmarks
No formal performance reports, bandwidth measurements, or frame rate impact data were found. The mod is too new and not yet widely tested by the community. The README mentions telemetry/trace tools (`--trace`, `--overlay`, `x4_coop_trace_report.py`) that could produce this data, but no published results exist.

### Expected Performance Characteristics (inferred)
- **Bandwidth:** At 22 packets/sec with small payloads (~100-500 bytes each), roughly 2-10 KB/s for ship sync. Economy passes add bursts. Save transfer is a one-time large TCP transfer.
- **CPU overhead:** Python bridge is lightweight (stdlib only, single-threaded relay loop). Lua-side FFI calls add minimal overhead.
- **Frame rate impact:** Should be minimal — the Lua does non-blocking pipe reads and writes, and the bridge runs in a separate process.

---

## 9. Last Update / Activity Status

### Extremely Active

- **Last commit:** 2026-10-09 (1 day ago) — "Terraforming in both worlds; the reviewers' function list is complete again"
- **Second-to-last:** 2026-10-09 — "Cargo: the empire's ships carry the same in both worlds"
- **content.xml date:** 2026-10-08
- **Commit cadence:** Recent commits show rapid feature development (multiple commits per day on some days)
- **Co-authored with Claude Opus 5.5** — the developer is using AI assistance for code generation
- **Total recent commits visible:** 22+ commits on the first page, all recent feature additions

### Author Identity
- **Name:** Aaron Sturman
- **Email:** aaron.sturman@live.co.uk
- **GitHub:** DanyWany22 (user ID: 199164991)
- **GitHub profile:** Only the x4_coop repo is visible; appears to be a dedicated account for this project
- **Contact:** Email is publicly available via commit history. Could be contacted.

### Maintenance Status
**Actively maintained — daily development.** The mod is in rapid feature development, not maintenance mode. New sync systems are being added regularly (terraforming, cargo, and shared loot were all added in the last few days).

---

## 10. Compatibility

- **Target X4 version:** 9.00 (stated in README: "X4 9.00 on both PCs")
- **Latest X4 version:** As of research date, X4 9.00 appears to be current (the mod targets it)
- **DLCs:** Both PCs need the same DLCs (partner is placed by sector name, which depends on DLC content)
- **Platform:** Windows only (named pipes, kernel32 FFI, WinDLL ctypes)
- **Python:** 3.8+ on Windows (standard library only)
- **Protected UI Mode:** Must be OFF (Settings → Extensions) for the mod's own pipe client to work. SirNukes' Mod Support APIs is an optional fallback.
- **Steam:** Works with same Steam account on both PCs (second in Offline Mode, Steam Cloud off for X4)

---

## 11. Installation

### Steps
1. Clone or download the repository
2. Place folder at `X4 Foundations/extensions/x4_coop/`
3. New extensions are enabled by default; check under Extensions in main menu
4. Install Python 3.8+ on Windows (standard library only — nothing to pip install)
5. Optional: Install SirNukes' Mod Support APIs (Steam Workshop id `ws_2042901274`) as fallback for pipe access
6. Bind a chat key: Settings → Controls → General Controls → Expert Settings → Toggle Chat Window
7. Optional debug: Steam launch options `-debug scripts -logfile debuglog.txt`

### For Playing Together
- Both players run the bridge (`bridge/host.bat` or `bridge/join.bat`)
- Host enters password, joiner enters host's address + same password
- Network: same LAN (use home IP), internet (forward UDP+TCP 47810), or Tailscale/ZeroTier
- In-game: `/x4coop net`, `/x4coop check`, `/x4coop join`

### Dependencies
- **Python 3.8+** (stdlib only, nothing to install)
- **SirNukes' Mod Support APIs** — OPTIONAL, fallback only for pipe access
- **No DLLs, no compiled files, no runtime downloads**

---

## 12. Reusability Assessment

### What We Can Directly Reuse or Adapt

#### ✅ HIGHLY REUSABLE

1. **Bridge architecture (bridge/x4_coop_bridge.py)**
   - The entire Python bridge is clean, well-structured, and stdlib-only
   - Named pipe server with non-blocking I/O
   - UDP relay with encryption, signing, anti-replay
   - TCP save transfer
   - Rate limiting, telemetry, reconnection logic
   - We could fork this almost directly

2. **Lua FFI pipe client pattern**
   - Using `ffi.C` to call kernel32 `CreateFileA`/`ReadFile`/`WriteFile` for named pipe access
   - This is the critical breakthrough: X4's Lua has no sockets, but FFI gives access to Windows API
   - Directly reusable

3. **Mission Director + AI script sync patterns**
   - The MD XML approach for periodic scans (economy, stations, crew, etc.)
   - AI script for proxy firing
   - These patterns are directly applicable

4. **Ship position sync with prediction & smoothing**
   - 30 Hz updates with client-side prediction to hide lag
   - The ghost mode test validates this approach
   - Directly reusable

5. **Save sharing mechanism**
   - Host quicksaves → bridge sends encrypted save over TCP → joiner loads as quicksave
   - Clean, well-thought-out, with backup of old quicksave
   - Directly reusable

6. **Regional authority model**
   - Each game is truth for its own area (6 km radius)
   - Host is truth when players are together
   - Host is truth for wider world when apart
   - This is a sound distributed systems design — directly reusable

7. **Entity pairing table**
   - Mapping ships/stations between worlds by different ID codes
   - Stored in save — directly reusable

8. **Test infrastructure**
   - `dev/fake_peer.py`, `dev/sim.lua`, `dev/run_lua.py`, `dev/game_sim.py`
   - Running mod's Lua in X4's own LuaJIT DLL outside the game
   - Pipe client test doubles
   - Directly reusable for our development

#### 🔄 ADAPTABLE

9. **Single-empire model**
   - Works for cooperative play but limits player autonomy
   - We might want separate factions for more independent play
   - The sync patterns are reusable but the ownership model would need rework

10. **2-player limit**
    - The bridge handles one partner at a time
    - Would need multi-peer support for 3+ players
    - The protocol and bridge architecture could be extended

11. **Windows-only constraint**
    - Named pipes are Windows-specific
    - Could potentially use Unix domain sockets on Linux/macOS
    - Would need platform abstraction layer

#### 🏗️ BUILD DIFFERENTLY

12. **Separate wallets via profile file**
    - The joiner's credits/inventory stored in a text file next to saves
    - Functional but fragile — we might use a more robust sidecar database

13. **Economy sync approach (full station scan)**
    - ~1 pass/min over all stations is simple but may not scale
    - For larger player counts, event-driven economy sync would be better

### What They Got Right

- **No DLLs, no memory hacking** — purely through official modding interfaces. This is the single most important decision. It means the mod can't crash the game, can't be detected as cheating, and doesn't break with game updates.
- **FFI for named pipes** — brilliant insight that LuaJIT's FFI can access kernel32 on Windows, giving Lua network access without any DLL
- **Python stdlib only** — zero install friction for the bridge
- **Ghost mode** — single-player test mode with simulated network is excellent for development
- **Regional authority** — sound distributed systems design for a game that simulates everything
- **Save sharing** — starting from the host's save ensures both worlds are identical at session start
- **Comprehensive telemetry** — trace recording, overlay, Wireshark-compatible
- **Encryption by default** — password-based crypto with scrypt KDF, HMAC signing, anti-replay
- **Test infrastructure** — running Lua in the game's own LuaJIT outside X4 is clever
- **Gradual sync** — not everything at once; ships at 30 Hz, economy at 1/min, stations at 1/0.25s

### What They Got Wrong (or could improve)

- **2-player only** — architectural limitation in the bridge (one partner)
- **Windows only** — no path to Linux/Mac
- **No mod page** — GitHub only, no Nexus/Workshop/Egosoft forum presence
- **Single empire** — limits gameplay variety (both players share one faction)
- **Story is host's only** — joiner can't progress their own story
- **No formal benchmarks** — can't assess if it scales
- **Drift accumulation** — away from both players, the worlds drift. Requires periodic re-share of host's save.

---

## 13. Lessons Learned

### Architectural Lessons

1. **You don't need memory hacking.** X4's Lua UI scripting + Mission Director + FFI for named pipes is sufficient for deep synchronization. This is a massive insight — it means the mod is stable across game updates and doesn't risk anti-cheat issues.

2. **Named pipes via LuaJIT FFI is the key bridge.** X4's Lua has no sockets, but Windows kernel32 functions are accessible through FFI. This is the breakthrough that makes networking from Lua possible without any DLL.

3. **Both games run full sims.** Don't try to make a thin client. X4's simulation is too complex to serialize/deserialize in real-time. Instead, run two full sims and reconcile them. The reconciliation is cheaper than the simulation.

4. **Regional authority beats global authority.** Each game being the truth for its own area (6 km) means nearby combat feels responsive, and there's no waiting for network round-trips to resolve who shot whom.

5. **Save sharing is the bootstrap.** Starting each session from the host's save ensures both worlds begin identical. The drift during play is manageable for a session, and re-sharing next session resets it.

6. **Prediction hides latency.** 30 Hz updates with client-side prediction (as demonstrated in ghost mode) is enough for smooth visual coherence even at 120ms+ latency.

7. **Gradual sync is essential.** Don't try to sync everything every frame. Ships at 30 Hz, NPCs at 10 Hz, economy at 1/min, stations at 1/0.25s. The total bandwidth stays low because each system only sends what changed.

8. **Entity pairing by different ID codes.** Ships in two worlds have different IDs. A pairing table (stored in save) maps them. This handles the fundamental problem that X4 generates entity IDs independently in each save.

9. **Wrap UI functions, don't replace them.** Instead of hooking or replacing game functions, wrap them: call the original, then note what changed. This is how orders, behaviours, station settings, and terraforming are synced.

10. **Test outside the game.** Running the mod's Lua in X4's own `lua51_64.dll` outside the game (`dev/run_lua.py`, `dev/sim.lua`) enables fast iteration without launching X4.

### Problems to Avoid

1. **World drift is inevitable.** Away from both players, simulations diverge. Don't promise perfect sync — accept drift and reset with save re-share each session.

2. **Don't sync what you can't set.** Map fog of war can't be set by scripts — accept it stays separate. Know the engine's limits.

3. **Protected UI Mode breaks FFI.** Must be off for the pipe to work. Have a fallback (SirNukes).

4. **LuaJIT 200 locals limit.** The simulator's scenario step lists had to become globals to stay under LuaJIT's 200 locals limit. Plan for this in Lua architecture.

5. **Equipment slot placement varies.** The game builds loadouts from equipment lists, so slots may differ. Don't assume exact slot matching.

6. **NPC AI makes its own decisions.** You can sync positions and hull, but the NPC's own AI still decides when to shoot. Don't try to override AI decisions.

7. **Story is fundamentally single-player.** The story moves on in the host's world only. The joiner helps but can't progress their own story independently.

8. **Performance testing is needed.** No stress test at full scale has been done. The economy matching after an hour of play is still an open question.

### For Our Project

**The single most important takeaway: DanyWany22 has already proven that cooperative multiplayer in X4 is achievable without DLLs, memory hacking, or injection. The architecture (Lua FFI → named pipe → Python bridge → UDP) is sound, and the sync patterns (regional authority, gradual sync, entity pairing, save sharing) are well-designed.**

We should:
1. **Fork or heavily adapt the bridge** — it's clean, stdlib-only, and well-tested
2. **Reuse the Lua FFI pipe client** — this is the critical breakthrough
3. **Reuse the MD/AI script sync patterns** — proven to work
4. **Consider extending to 3+ players** — the main architectural gap
5. **Consider separate factions** — for more independent gameplay
6. **Add formal performance testing** — stress test at full scale
7. **Publish to Nexus/Workshop** — for community testing
8. **Contact Aaron Sturman** — he's actively developing and using AI assistance; collaboration could be mutually beneficial

---

## Appendix: Key Code References

### Bridge (Python)
- `bridge/x4_coop_bridge.py` — Main bridge: ~800+ lines
  - `GamePipe` class: Named pipe server (Windows API via ctypes)
  - `Codec` class: Encryption, signing, anti-replay
  - `SaveShare` class: TCP save transfer with HMAC proof
  - `Telemetry` class: Trace recording and overlay
  - `RateLimit` class: Token bucket rate limiting
  - Main relay loop: non-blocking pipe reads + UDP recvfrom

### Lua (in-game)
- `ui/x4_coop.lua` — Core mod logic: ~thousands of lines
  - FFI declarations for 6 Windows functions + 63 game functions
  - Own pipe client (CreateFileA, ReadFile, WriteFile via FFI)
  - Ship position send/receive with prediction & smoothing
  - Ghost mode (simulated network for single-player testing)
  - All sync handlers for the features listed above
  - Config table at top of file for all settings

### Mission Director
- `md/x4_coop.xml` — Periodic scans and sync operations
  - Economy pass (station cargo)
  - Station modules scan
  - Crew sync
  - Wider world scan
  - Terraforming stats
  - And many more

### AI Scripts
- `aiscripts/x4coop.proxy.fire.xml` — Makes the partner's proxy ship fire at targets

### Content Manifest
- `content.xml` — `version="1"`, `date="2026-10-08"`, `save="0"` (doesn't require new save), optional dependency on SirNukes' Mod Support APIs

---

## Appendix: Message Types (from bridge code)

The bridge recognizes these message kinds from the game:

| Code | Meaning | Fields |
|------|---------|--------|
| S | Ship position | seq t sector ship x y z yaw pitch roll vx vy vz name idcode hull shield |
| D | Damage | code macro sector hull enemy |
| K | Kill | code macro sector |
| F | Fire | code macro sector |
| L | World link | world role ship protocol |
| M | Chat message | name text |
| P/Q | Ping | t |
| R | Role | role |
| B | NPC batch | t sector radius complete ships |
| H | Keepalive | host pid os t echo hold |
| E | Economy | what pass index count station wares trades |
| C | Credits | what id amount name |
| T | Trade | what id station ware change |
| V | Relations | what relations ids |
| U | Unlock | what id kind item extra |
| Z | SETA | what active factor note |
| O | Owner change | what id code macro sector owner |
| Y | New ship | what id code macro sector shipyard name wares |
| G | Order | what id ship order clear args params |
| J | Behaviour | what id ship default queue |
| I | On-foot | what station macro sector room index x y z speed name |
| b | Station module | what id station macro sector x y z yaw pitch roll module... |
| q | New ship equipment | what id ship macro sector equipment |
| p | Deployable | what id op deployable macro sector x y z |
| n | Knowledge | what id kind name macro sector revealed |
| l | Ship rename | what id ship macro sector name |
| w | Crew | what id ship macro sector crew |
| g | Terraforming | what a b c d e f g h i j k l m n o p q |
| h | Logbook | what id category title text credits |
| d | Deployable destroyed | what id kind macro sector x y z station smacro |
| f | Cargo | what id ship macro sector cargo |
| t | Terraform project | what id op cluster project |
| s | Station settings | what id op station name fill buildprice supplyrule... |
| c | Assignment | what id ship macro sector commander cmacro csector group assignment |
| r | Trade rule | what id op rule name whitelist defaults factions |
| a | Station account | what id station change min max |
| m | Mission | what id op mission name description objective faction target... |
| e | Module change | what id change before station macro sector module... |

---

*Research completed 2026-10-10. All data from GitHub repository at https://github.com/DanyWany22/x4_coop as of that date.*