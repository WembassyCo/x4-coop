# X4 Co-op Mod

A multiplayer mod for X4: Foundations that enables true 2-player co-op using a host-authoritative architecture with the guest running a second X4 instance as a thin client.

## Language

**Host**:
The player running the authoritative simulation. All economy, NPC AI, and world state originate here.
_Avoid_: Server, authoritative client, player 1

**Guest**:
The second player. Runs a second X4 instance with local simulation suppressed. Receives state from the host and sends input commands back.
_Avoid_: Client (except in code), player 2, spectator

**Relay**:
An optional intermediary service that handles NAT traversal when direct P2P connection isn't possible.
_Avoid_: Matchmaker, proxy server

**State Delta**:
The compressed difference between two simulation snapshots, sent from host to guest over TCP.
_Avoid_: Snapshot, diff, patch

**Position Update**:
A high-frequency UDP message containing entity positions and rotations for smooth rendering on the guest.
_Avoid_: Transform sync, movement packet

**Suppression Mod**:
The X4 mod component that pauses/suppresses the guest's local simulation, preventing it from running economy, NPC AI, or world logic independently.
_Avoid_: Freeze mod, pause mod

**Injection Mod**:
The X4 mod component that receives host state and writes it into the guest's local game state, making the guest's X4 render the host's universe.
_Avoid_: Sync mod, state writer

**Bridge Service**:
The external Node.js process that handles networking (TCP/UDP), serialization, and file I/O between the host and guest X4 instances.
_Avoid_: Relay service (that's the NAT relay), networking layer, comm layer

**Command**:
An input action from the guest (fly ship, order fleet, build station) sent to the host for execution in the authoritative simulation.
_Avoid_: Input, action, request

**Session**:
A single co-op play instance — one host, one guest, connected and sharing a universe.
_Avoid_: Game, match, lobby

**Entity Ownership**:
Which player controls a given entity (ship, station). Host owns the universe; guest owns ships they've claimed.
_Avoid_: Control, possession