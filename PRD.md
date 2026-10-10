# X4 Co-op Mod — Technical Specification

## Overview

A multiplayer mod for X4: Foundations enabling 2-player co-op through a host-authoritative architecture. The host runs the full universe simulation; the guest runs a second X4 instance with simulation suppressed, receiving state updates and sending commands back.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Host's X4 Instance                                              │
│                                                                 │
│  ┌──────────────┐     ┌───────────────────┐     ┌────────────┐ │
│  │ Host Bridge  │◀──▶│ State Export Mod  │     │            │ │
│  │ (Node.js)    │     │ (Lua — reads game │     │  X4 Engine │ │
│  │ - TCP server │     │  state per tick,  │     │  (full sim │ │
│  │ - UDP pos    │     │  writes commands  │     │   runs     │ │
│  │   broadcast  │     │  back to game)    │     │   normally)│ │
│  └──────┬───────┘     └───────────────────┘     └────────────┘ │
│         │                                                       │
└─────────┼───────────────────────────────────────────────────────┘
          │ TCP (state deltas) + UDP (positions)
          │
┌─────────┼───────────────────────────────────────────────────────┐
│ Guest's X4 Instance                                             │
│         │                                                       │
│  ┌──────▼───────┐     ┌───────────────────┐     ┌────────────┐ │
│  │ Guest Bridge │     │ Suppression Mod   │     │            │ │
│  │ (Node.js)    │───▶│ + Injection Mod   │───▶│  X4 Engine │ │
│  │ - TCP client │     │ (Lua — pauses sim,│     │  (render + │ │
│  │ - UDP recv   │     │  injects host     │     │   UI only, │ │
│  │ - sends cmds │     │  state, forwards  │     │   sim      │ │
│  │   to host    │     │  guest input)     │     │  suppressed)│ │
│  └──────────────┘     └───────────────────┘     └────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Decisions (Locked)

| # | Decision | Resolution |
|---|----------|------------|
| 1 | Concurrent players | 2 only (host + guest) |
| 2 | Guest autonomy | Full second player — own ship, fleet, station building |
| 3 | Simulation authority | Host-authoritative. Guest is thin client. |
| 4 | Viability target | Playable by us first, publishable later |
| 5 | Guest rendering | Second X4 instance with sim suppressed + state injection |
| 6 | Network transport | TCP for state, UDP for positions (10-20Hz) |
| 7 | X4 version | Latest only (track patches as they come) |
| 8 | Relay architecture | P2P direct, optional relay for NAT traversal |
| 9 | Testing | Chris tests on weekends; Spock builds + automates |

## Component Breakdown

### 1. Bridge Service (Node.js)
External process running alongside each X4 instance.

**Host Bridge:**
- TCP server (listens for guest connection)
- UDP broadcaster (sends position updates at 10-20Hz)
- State serializer (reads exported game state, compresses to deltas)
- Command receiver (accepts guest commands, writes to command queue file)
- File I/O (reads state export file, writes command queue file)

**Guest Bridge:**
- TCP client (connects to host)
- UDP receiver (listens for position updates)
- State deserializer (receives deltas, writes to injection file)
- Command sender (reads guest input, sends to host)
- File I/O (writes injection state file, reads command ACK file)

### 2. X4 Lua Mods
**Host-side mods:**
- `x4coop_host_state_export` — Reads game state each tick, writes to export file
- `x4coop_host_command_executor` — Reads command queue, executes commands in-game

**Guest-side mods:**
- `x4coop_guest_suppression` — Pauses local simulation (economy, NPC AI, world logic)
- `x4coop_guest_state_injection` — Reads injected state, updates local entities
- `x4coop_guest_input_capture` — Captures guest player input, forwards to bridge

### 3. Network Protocol
- **Wire format:** Protocol Buffers (protobuf)
- **TCP messages:** StateDelta, Command, CommandAck, Handshake, SyncRequest
- **UDP messages:** PositionUpdate (entity ID + position + rotation)
- **Delta compression:** Only send changed fields since last acknowledged state

### 4. State Sync Model
- Host exports full universe state at configurable interval (default: 2Hz)
- Position updates sent via UDP at 10-20Hz for entities near guest player
- Guest applies state injections, interpolating between updates for smoothness
- Host processes guest commands and reflects results in next state export

## Entity Ownership Model

- Host owns: All NPC factions, economy, stations (unless built by guest), universe state
- Guest owns: Ships they're piloting, fleets they've commanded, stations they've built
- Ownership tracked by entity ID → owner mapping in host's state export
- Guest can only issue commands for entities they own

## Connection Lifecycle

1. Host starts X4 + bridge service, bridge listens for connection
2. Guest starts X4 + bridge service, connects to host
3. Handshake: exchange player info, mod versions, sync initial state
4. Guest's suppression mod activates, simulation pauses
5. Host sends initial full state snapshot
6. Guest's injection mod applies snapshot, rendering begins
7. Steady state: deltas + position updates + commands flow continuously
8. Disconnect: guest bridge detects loss, suppression deactivates, guest returns to single-player

## Error Handling

- **Desync detection:** Host includes state hash in each delta; guest compares against local state hash
- **Reconnection:** Guest can reconnect within 60s without full resync; after 60s, full snapshot required
- **Command rejection:** Host validates all guest commands; invalid commands return error ACK
- **Mod version mismatch:** Handshake rejects if mod versions don't match

## File Structure

```
x4-coop/
├── CONTEXT.md
├── README.md
├── docs/
│   └── adr/
├── bridge/                    # Node.js bridge service
│   ├── package.json
│   ├── src/
│   │   ├── host-bridge.ts     # Host-side bridge
│   │   ├── guest-bridge.ts    # Guest-side bridge
│   │   ├── protocol/          # Protobuf definitions
│   │   ├── transport/         # TCP/UDP transport layer
│   │   └── state/             # State serialization/delta
│   └── tests/
├── mods/                      # X4 mod packages
│   ├── x4coop_host_state_export/
│   │   ├── content.xml
│   │   ├── libraries/
│   │   └── md/                # Mission Director scripts
│   ├── x4coop_host_command_executor/
│   ├── x4coop_guest_suppression/
│   ├── x4coop_guest_state_injection/
│   └── x4coop_guest_input_capture/
├── proto/                     # Shared protobuf definitions
│   └── x4coop.proto
└── scripts/                   # Build/deploy scripts
    ├── package-mods.sh
    └── install-mods.sh
```