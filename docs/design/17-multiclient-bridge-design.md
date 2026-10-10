# Design Document #17: Multi-Client Bridge Extension

**Wayfinder Ticket:** #17  
**Project:** x4-coop  
**Status:** Draft  
**Date:** 2026-10-10  

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Overview](#2-current-architecture-overview)
3. [Design Goals & Constraints](#3-design-goals--constraints)
4. [Connection Model](#4-connection-model)
5. [Architecture Diagram](#5-architecture-diagram)
6. [Protocol Extension: X4C3](#6-protocol-extension-x4c3)
7. [Client Session Management](#7-client-session-management)
8. [State Broadcast](#8-state-broadcast)
9. [Command Routing](#9-command-routing)
10. [Client Lifecycle](#10-client-lifecycle)
11. [Authentication](#11-authentication)
12. [Backward Compatibility](#12-backward-compatibility)
13. [Sequence Diagrams](#13-sequence-diagrams)
14. [Error Handling & Edge Cases](#14-error-handling--edge-cases)
15. [Implementation Phases](#15-implementation-phases)

---

## 1. Executive Summary

The current x4_coop bridge is a 1:1 P2P relay: two bridges, one host and one joiner, exchange UDP datagrams encrypted with a shared password. Each bridge sits between a local X4 instance (via named pipe) and a remote partner bridge (via UDP). This document designs the extension to a **1:N server model** where one server bridge manages N client connections, enabling cooperative play among more than two players.

The design preserves backward compatibility with the existing 1:1 P2P mode, extends the wire protocol with client identifiers and routing metadata, and addresses the fundamental challenges of multi-client state synchronization in X4's single-player architecture.

**Key decisions:**

| Question | Decision |
|----------|----------|
| Connection model | Single server bridge with N TCP+UDP connections |
| Transport | TCP for control plane; UDP for real-time game state |
| Broadcast strategy | Server-side fan-out (unicast to each client) |
| View distances | Per-client NPC bubble; server prunes per recipient |
| Command routing | Server applies client commands to authoritative state, then broadcasts |
| Client lifecycle | Connect → authenticate → snapshot → delta stream → disconnect → cleanup |
| Auth | Player IDs + session tokens + ban list, backwards-compatible with shared password |
| Backward compat | Server bridge supports `--p2p` flag for original 1:1 mode |

---

## 2. Current Architecture Overview

### 2.1 Bridge (`x4_coop_bridge.py`)

```
X4 (game) ←→ Named Pipe ←→ Bridge ←→ UDP ←→ Partner Bridge ←→ Named Pipe ←→ X4 (game)
```

**Current characteristics:**

- **One bridge per player**, running on the same PC as X4
- **Pipe:** Windows named pipe (`\\.\pipe\x4_coop`), message-mode, non-blocking. Only X4.exe may connect (`PIPE_REJECT_REMOTE_CLIENTS`)
- **UDP:** Single socket on port 47810. Host waits for any partner; joiner sends to host's address. One partner at a time.
- **TCP:** Used only for save file transfer (host → joiner). One-time, encrypted with a separate key derived from the password.
- **Protocol (X4C2):** `b"X4C2 " + HMAC_tag + " " + session:counter:timestamp + " " + ciphertext`
  - Encryption: XOR keystream from SHAKE-256(enc_key, header)
  - Authentication: HMAC-SHA256, first 32 hex chars
  - Anti-replay: session ID (random per bridge start) + monotonically increasing counter + timestamp window (±60s)
  - Rate limiting: token bucket at 200 packets/s (partner sends ~22/s)
- **Save sharing:** Host quicksaves, offers via UDP (`X|offer|size|sha256|token`), serves over TCP. Joiner fetches, verifies, replaces quicksave.
- **Roles:** Host (shared world from host's save) and joiner (loads host's save). Told to game via `R|host` / `R|join`.
- **Keepalives:** `H|host|pid|os|t|echo|hold` every 1s. Partner timeout at 10s.

### 2.2 Lua Mod (`x4_coop.lua`)

The Lua side runs inside X4 and handles:

- **Sampling:** Local player ship pose (sector space) at `send_rate` (30 Hz default)
- **Proxy:** A "proxy" ship representing the partner, positioned via snapshots
- **Shared world:** When both sides load the same save (identified by world ID), kills/hits/owners/orders/economy/etc. are mirrored
- **NPC bubble:** Ships near each player are listed (md `BubbleScan`) and poses exchanged at `npc_rate` (10 Hz)
- **Message types:** S (snapshot), P/Q (ping/pong), M (chat), R (role), L (world link), K (kill), D (damage), F (fire), B (NPC bubble), E (economy), C (credits), T (trades), V (relations), U (unlocks), Z (SETA), O (ownership), Y (new ships), G (orders), J (behaviours), I (on foot), b (stations), s (settings), c (commands), r (rules), a (accounts), m (missions), e (module changes), q/p/n/l/w/g/h/d/f/t (various sync), X (save handoff), W/N (bridge notices), H (keepalive)

### 2.3 Limitations of 1:1

- Only two players can share a universe
- Host is single point of failure — no failover
- No concept of per-client state or view frustum
- Save sharing is one-shot host→joiner, no incremental updates
- NPC bubble assumes exactly two areas (host's + joiner's)

---

## 3. Design Goals & Constraints

### Goals

1. **Support 2–8 concurrent clients** (practical upper bound for X4 co-op)
2. **Minimize wire protocol changes** — extend, don't replace X4C2
3. **Preserve 1:1 P2P mode** as a first-class option
4. **Server authority:** The server's X4 universe is the shared world. All clients sync to it.
5. **Per-client view:** Each client sees ships/objects relevant to their area, not the entire universe
6. **Graceful degradation:** Client disconnect should not affect other clients' games
7. **Security:** Per-client authentication, replay protection, rate limiting

### Constraints

- X4 is single-player at its core; the mod works by spawning proxy ships and mirroring state
- Named pipes are local only (same PC) — each client still runs their own bridge + X4
- UDP packet size limited to ~64KB; snapshot messages are small (~200 bytes) but NPC bubbles can be large
- The Lua mod runs per-frame and must not stall on network I/O
- Python standard library only (no external dependencies in current bridge)
- Windows-only for the pipe API (ctypes/kernel32)

---

## 4. Connection Model

### Decision: Single Server Bridge with N Client Connections

**Rationale:**

The alternative — one bridge per client on the server — would require N bridge processes, N named pipes, and N X4 connections on the server machine. This is impractical: the server runs one X4 instance with one pipe. A single server bridge multiplexes N clients over one pipe to one X4.

```
Option A (CHOSEN): Single server bridge, N connections
  Server PC:  X4 ←pipe→ Server Bridge ←┬─ UDP/TCP ─→ Client Bridge 1 ←pipe→ X4 (Player 2)
                                        ├─ UDP/TCP ─→ Client Bridge 2 ←pipe→ X4 (Player 3)
                                        └─ UDP/TCP ─→ Client Bridge 3 ←pipe→ X4 (Player 4)

Option B (REJECTED): One bridge per client on server
  Server PC:  X4 ←pipe→?  (only one pipe client allowed)
             Bridge 1, Bridge 2, Bridge 3 all need their own pipe to the same X4 — impossible
```

### Transport Layer

| Channel | Protocol | Purpose |
|---------|----------|---------|
| Control plane | TCP | Authentication, join/leave, save transfer, reliable commands |
| Data plane | UDP | Real-time snapshots, NPC bubbles, fire events, pings |

**Why both:** TCP ensures reliable delivery for control messages (auth, save handoff, ack-based reliable commands) while UDP provides low-latency delivery for high-frequency state updates (snapshots at 30 Hz, NPC poses at 10 Hz). The current bridge already uses this split for save transfer only; we extend it to all control-plane traffic.

### Socket Layout (Server)

```
Server bridge:
  - UDP socket: 0.0.0.0:47810  (shared for all clients; source addr identifies client)
  - TCP listener: 0.0.0.0:47810 (control channel; one accepted connection per client)
```

The server uses a single UDP socket (as it does now). Client addresses are tracked per-session. TCP is used for control connections that need reliability.

### Socket Layout (Client)

```
Client bridge:
  - UDP socket: 0.0.0.0:<random or configured>  (sends to server's UDP)
  - TCP connection: server_ip:47810  (control channel)
```

---

## 5. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SERVER PC                                      │
│                                                                             │
│   ┌─────────────┐    Named Pipe     ┌──────────────────────────────┐        │
│   │   X4.exe    │◄──────╔════╗──────►│      Server Bridge           │        │
│   │  (Host)     │       ║pipe║       │  ┌────────────────────┐     │        │
│   │  x4_coop.lua│       ╚════╝       │  │  Pipe Multiplexer  │     │        │
│   └─────────────┘                    │  │  (tags msgs with   │     │        │
│        ▲                             │  │   client IDs)      │     │        │
│        │ game state                  │  └────────┬───────────┘     │        │
│        │ (snapshots, NPC,            │           │                 │        │
│        │  economy, etc.)             │  ┌────────▼───────────┐     │        │
│        │                             │  │  State Router       │     │        │
│        │                             │  │  (fan-out, prune,   │     │        │
│        │                             │  │   alias, debounce)  │     │        │
│        │                             │  └────────┬───────────┘     │        │
│        │                             │           │                 │        │
│        │                             │  ┌────────▼───────────┐     │        │
│        │                             │  │  Session Manager    │     │        │
│        │                             │  │  ┌───┐ ┌───┐ ┌───┐ │     │        │
│        │                             │  │  │C1 │ │C2 │ │C3 │ │     │        │
│        │                             │  │  └───┘ └───┘ └───┘ │     │        │
│        │                             │  └────────┬───────────┘     │        │
│        │                             │           │                 │        │
│        │                             │  ┌────────▼───────────┐     │        │
│        │                             │  │  Codec (X4C3)       │     │        │
│        │                             │  │  Per-client keys    │     │        │
│        │                             │  └────────┬───────────┘     │        │
│        │                             └───────────┼─────────────────┘        │
│        │                                         │                          │
│        │                             UDP 47810   │   TCP 47810             │
└────────┼─────────────────────────────────────────┼──────────────────────────┘
         │                                         │
    ╔════╧════╗                               ╔════╧════╗
    ║   UDP   ║                               ║   TCP   ║
    ╚════╤════╝                               ╚════╤════╝
         │                                         │
    ┌────┼─────────────┬─────────────┐            │
    │    │             │             │            │
    ▼    ▼             ▼             ▼            ▼
┌──────┬──────┐  ┌──────┬──────┐  ┌──────┬──────┐
│ UDP  │ TCP  │  │ UDP  │ TCP  │  │ UDP  │ TCP  │
│      │      │  │      │      │  │      │      │
├──────┴──────┤  ├──────┴──────┤  ├──────┴──────┤
│   Client    │  │   Client    │  │   Client    │
│   Bridge 1  │  │   Bridge 2  │  │   Bridge 3  │
│  (Joiner)   │  │  (Joiner)   │  │  (Joiner)   │
├─────────────┤  ├─────────────┤  ├─────────────┤
│  X4.exe     │  │  X4.exe     │  │  X4.exe     │
│  Player 2   │  │  Player 3   │  │  Player 4   │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## 6. Protocol Extension: X4C3

### 6.1 Magic Byte

```
X4C2:  b"X4C2 "  (current, 1:1 only)
X4C3:  b"X4C3 "  (multi-client, superset of X4C2)
```

The magic byte change ensures old bridges reject new traffic cleanly (as they already do for non-X4C2 magic). X4C3 is a superset: all X4C2 message types are valid in X4C3, with added routing fields.

### 6.2 Datagram Format

```
X4C2 (current):
  MAGIC + HMAC_tag + " " + session:counter:timestamp + " " + ciphertext

X4C3 (extended):
  MAGIC + HMAC_tag + " " + client_id:session:counter:timestamp + " " + ciphertext
         └── 8 hex ──┘    └── 16 hex ──┘ └── 8 hex ──┘ └── 10 digits ─┘
```

**New field: `client_id`** (4 bytes, 8 hex chars)

- Server-assigned, unique per client connection
- `00000000` = server itself (for messages originating from the server bridge)
- Present in every datagram for routing
- Backward compat: when `client_id` is absent (X4C2), the bridge treats it as a 1:1 message (no routing needed)

### 6.3 Plaintext Message Extension

The plaintext (after decryption) gains an optional routing prefix:

```
X4C2 message:  "S|seq|t|sector|ship|x|y|z|..."
X4C3 message:  ">SRC>Dst|S|seq|t|sector|ship|x|y|z|..."
```

**Routing prefix:** `>SRC>DST|` where:
- `SRC` = 8-char hex client ID of the sender
- `DST` = 8-char hex client ID of the recipient, or `*` for broadcast, or `S` for server
- The prefix is stripped before delivering to the pipe (X4's Lua sees the same message format as before)

**Examples:**
```
>00000001>00000002|S|...    Client 1's snapshot, destined for Client 2
>00000001>*|S|...            Client 1's snapshot, broadcast to all other clients
>00000000>00000002|N|...     Server notice to Client 2
>00000002>S|R|join            Client 2 sends role message to server
```

### 6.4 Control-Plane Messages (TCP)

New control-plane message types, sent over TCP for reliability:

```
AUTH|client_id|session_token|player_name|protocol_version
AUTH_OK|assigned_client_id|world_id|role
AUTH_DENY|reason
JOIN|client_id|save_hash|save_size
JOIN_OK|snapshot_token
JOIN_SNAPSHOT|offset|total|data...    (chunked world snapshot over TCP)
LEAVE|client_id|reason
HEARTBEAT|client_id|timestamp
HEARTBEAT_ACK|client_id|timestamp|server_time
KICK|client_id|reason
BAN_LIST|player_ids...
```

These are plaintext JSON or pipe-delimited, encrypted with the session's AES key (negotiated during auth), framed with a 4-byte length prefix for TCP.

### 6.5 Message Kind Extensions

New message kinds for multi-client awareness (in the pipe-delimited format):

```
J|join|client_id|player_name|sector_macro     (client joined notification, to other clients)
J|leave|client_id|reason                       (client left notification)
J|players|count|id1:name1;id2:name2;...        (roster update, server → all clients)
```

The `J` kind was previously used for "default behaviours and order queues." To avoid collision, multi-client `J` messages use a `J|join|...` / `J|leave|...` / `J|players|...` sub-protocol that the Lua side distinguishes by the second field. The existing behaviour-sync `J` messages use `J|msg|id|...` and `J|ack|id|...` which never collide with `join`/`leave`/`players`.

---

## 7. Client Session Management

### 7.1 Data Structures

```python
# Server-side session manager

@dataclass
class ClientSession:
    """One connected client."""
    client_id: str           # 8 hex chars, server-assigned
    player_id: str           # persistent player identifier (auth)
    player_name: str         # display name from X4
    udp_addr: tuple          # (ip, port) — may change with NAT
    tcp_conn: socket         # TCP control connection (or None if dropped)
    role: str                # "join" for all clients; "host" for server's own X4
    
    # Crypto
    session_key: bytes       # per-client encryption key (derived from password + nonce)
    codec: Codec             # per-client codec instance (counter, replay window)
    
    # State
    connected_at: float      # monotonic time
    last_heard: float        # last UDP or TCP message
    last_keepalive: float    # last keepalive sent
    world_id: str            # shared world ID
    
    # Game state
    sector_macro: str        # current sector (for view-distance pruning)
    ship_macro: str          # current ship type
    ship_idcode: str         # ID code of player's ship
    snapshot_seq: int        # last snapshot sequence number from this client
    npc_radius: int          # this client's NPC bubble radius
    
    # Save
    save_hash: str           # sha256 of the save this client loaded
    save_transferred: bool   # whether initial save transfer completed
    
    # Rate limiting
    rate_limiter: RateLimit  # per-client token bucket
    
    # Reliable message tracking
    reliable_out: dict       # id → {msg, first, last} — pending reliable sends
    reliable_seen: dict      # key → timestamp — dedup for received reliable msgs
    
    # Status
    authenticated: bool      # passed AUTH handshake
    snapshot_received: bool  # received at least one snapshot
    banned: bool             # in ban list


class SessionManager:
    """Manages all client sessions on the server."""
    
    clients: dict[str, ClientSession]      # client_id → session
    by_player_id: dict[str, str]           # player_id → client_id (prevent double-join)
    by_udp_addr: dict[tuple, str]          # udp_addr → client_id (fast packet lookup)
    ban_list: set[str]                     # banned player IDs
    
    next_client_id: int                    # monotonically increasing counter
    
    def assign_client_id(self) -> str:
        """Generate next 8-hex-char client ID."""
        self.next_client_id += 1
        return f"{self.next_client_id:08x}"[:8]
    
    def add_client(self, session: ClientSession):
        self.clients[session.client_id] = session
        self.by_player_id[session.player_id] = session.client_id
        self.by_udp_addr[session.udp_addr] = session.client_id
    
    def remove_client(self, client_id: str, reason: str = ""):
        session = self.clients.pop(client_id, None)
        if session:
            self.by_player_id.pop(session.player_id, None)
            self.by_udp_addr.pop(session.udp_addr, None)
            # Notify other clients
            self.broadcast(f"J|leave|{client_id}|{reason}", exclude=client_id)
    
    def get_by_addr(self, addr: tuple) -> ClientSession | None:
        return self.clients.get(self.by_udp_addr.get(addr))
    
    def broadcast(self, msg: str, exclude: str | None = None):
        """Send a message to all clients except the excluded one."""
        for cid, session in self.clients.items():
            if cid != exclude and session.authenticated:
                self.send_to(session, msg)
    
    def send_to(self, session: ClientSession, msg: str):
        """Send a message to one client over UDP."""
        data = session.codec.seal(msg)
        self.sock.sendto(data, session.udp_addr)
    
    def tick(self, now: float):
        """Periodic maintenance: timeouts, keepalives, reliable retries."""
        for cid, session in list(self.clients.items()):
            if now - session.last_heard > PARTNER_TIMEOUT_S:
                self.remove_client(cid, "timeout")
            elif now - session.last_keepalive > KEEPALIVE_S:
                self.send_to(session, f"H|server|{os.getpid()}|server|{int(now*1000)}|0|0")
                session.last_keepalive = now
```

### 7.2 Client-Side Session State

```python
@dataclass
class ServerConnection:
    """Client bridge's view of the server."""
    server_addr: tuple        # (ip, port)
    client_id: str            # assigned by server
    player_id: str            # our player ID
    session_token: str        # for reconnection
    tcp_conn: socket          # TCP control connection
    udp_sock: socket          # UDP socket for data
    
    # Other clients (for proxy management)
    known_clients: dict       # client_id → {name, sector, ship, last_seen}
    
    # Codec (shared with server, per-session key)
    codec: Codec
    
    # Reconnection
    last_snapshot: dict       # for resume after reconnect
    reconnect_backoff: float  # exponential backoff
```

---

## 8. State Broadcast

### 8.1 Broadcast Strategy: Server-Side Fan-Out (Unicast)

**Decision:** Server sends individually to each client over UDP. No IP multicast.

**Rationale:**

1. **NAT traversal:** Most clients are behind NAT. IP multicast doesn't cross NAT boundaries.
2. **Per-client pruning:** Each client only needs state for its own area + partner areas. Multicast sends the same data to everyone.
3. **Per-client encryption:** Each client has its own session key. Multicast can't encrypt differently per recipient.
4. **Scale:** With 2–8 clients, fan-out overhead is negligible. At 30 Hz × 200 bytes × 8 clients = 48 KB/s — trivial.

### 8.2 What Gets Broadcast

| Message Type | Broadcast? | Pruning |
|-------------|------------|---------|
| S (snapshot) | Yes, to all other clients | Always: small, needed for proxy positioning |
| B (NPC bubble) | Yes, to all other clients | **Per-recipient:** server filters ships outside recipient's sector + npc_radius |
| K/D/F (kill/damage/fire) | Yes, to all other clients | Only to clients whose NPC bubble includes the target |
| E (economy) | Yes, to all clients | Same as 1:1: host (server) sends stock data to all joiners |
| C/T/V/U (credits/trades/relations/unlocks) | Routed, not broadcast | Sender → server → specific recipient (or server processes) |
| M (chat) | Broadcast | Always: small, everyone should see chat |
| L (world link) | Server processes, doesn't forward | Used for link negotiation |
| P/Q (ping/pong) | Server responds directly | No forwarding |
| Z (SETA) | Broadcast | All clients must sync speed |

### 8.3 Per-Client View Distance Pruning

The server maintains each client's current sector and NPC bubble radius. For NPC bubble messages (B), the server filters the ship list before forwarding:

```python
def prune_bubble_for_recipient(bubble_msg: str, recipient: ClientSession) -> str:
    """Remove ships from the bubble that are outside the recipient's view."""
    fields = bubble_msg.split("|")
    # B|t|sector_macro|radius|complete|ship1;ship2;...
    sector = fields[2]
    if sector != recipient.sector_macro:
        # Recipient is in a different sector — they still need it for proxy positioning
        # but can skip NPC detail. Send a compressed version.
        return bubble_msg  # full bubble for cross-sector awareness
    
    radius = int(fields[3])
    entries = (fields[5] or "").split(";")
    kept = []
    for entry in entries:
        parts = entry.split(",")
        if len(parts) >= 7:
            x, y, z = float(parts[4]), float(parts[5]), float(parts[6])
            # Check if within recipient's radius
            dist = sqrt((x - recipient.pos.x)**2 + ...)
            if dist <= recipient.npc_radius + radius:
                kept.append(entry)
    
    fields[5] = ";".join(kept)
    return "|".join(fields)
```

### 8.4 Snapshot Aggregation

When the server's X4 sends a snapshot (S), the server bridge broadcasts it to all clients. When a client sends a snapshot, the server:

1. **Delivers to the server's X4** (so the host sees the proxy)
2. **Forwards to all other clients** (so they see each other's proxies)

```python
def route_snapshot(msg: str, sender: ClientSession, sessions: SessionManager):
    """Forward a snapshot from one client to all others + server's X4."""
    # Deliver to server's own X4
    tell_game(msg)
    
    # Forward to all other clients with routing prefix
    routed = f">{sender.client_id}>*|{msg}"
    for cid, session in sessions.clients.items():
        if cid != sender.client_id and session.authenticated:
            sessions.send_to(session, routed)
```

---

## 9. Command Routing

### 9.1 Authority Model

The server's X4 instance is the **authoritative universe**. All clients are joiners who load the server's save. Commands that modify the universe (orders, ownership, economy, station building, etc.) flow:

```
Client A → Server Bridge → Server's X4 (authoritative) → Broadcast effect to Clients B, C
```

### 9.2 Command Flow

```python
def route_message(msg: str, sender: ClientSession, sessions: SessionManager, game_pipe: GamePipe):
    """
    Route a decoded message from a client.
    
    Most messages follow one of these patterns:
    1. To server's X4 only (role, ping, save handoff, economy ack)
    2. To server's X4 + broadcast to other clients (snapshots, kills, damage, fire)
    3. To server's X4 + specific client routing (credits, trades, relations)
    """
    kind = msg[0] if msg else ""
    
    if kind in ("S",):  # Snapshot: deliver to X4 + broadcast
        tell_game(msg)
        sessions.broadcast(f">{sender.client_id}>*|{msg}", exclude=sender.client_id)
    
    elif kind in ("K", "D", "F"):  # Combat: deliver to X4 + broadcast
        tell_game(msg)
        sessions.broadcast(f">{sender.client_id}>*|{msg}", exclude=sender.client_id)
    
    elif kind == "B":  # NPC bubble: deliver to X4 + pruned broadcast
        tell_game(msg)
        for cid, session in sessions.clients.items():
            if cid != sender.client_id and session.authenticated:
                pruned = prune_bubble_for_recipient(msg, session)
                sessions.send_to(session, f">{sender.client_id}>{cid}|{pruned}")
    
    elif kind in ("C", "T", "V", "U"):  # Economy/credits/relations/unlocks: to server's X4
        tell_game(msg)
        # Server's X4 processes and may generate responses that get broadcast
    
    elif kind in ("O", "Y", "G", "J", "b", "s", "c", "r", "a", "m", "e"):  # World mods
        tell_game(msg)
        # These modify the authoritative world; server's X4 handles and effects propagate
        # through the server's own outbound messages (which the server bridge broadcasts)
    
    elif kind == "M":  # Chat: broadcast to all
        sessions.broadcast(f">{sender.client_id}>*|{msg}", exclude=sender.client_id)
        tell_game(msg)  # Server player sees it too
    
    elif kind == "P":  # Ping: respond directly
        sessions.send_to(sender, f"Q|{msg[2:]}")
    
    elif kind == "L":  # World link: server processes
        handle_world_link(msg, sender)
    
    elif kind == "Z":  # SETA: broadcast
        tell_game(msg)
        sessions.broadcast(f">{sender.client_id}>*|{msg}", exclude=sender.client_id)
    
    elif kind == "H":  # Keepalive: update last_heard, don't forward
        sender.last_heard = time.monotonic()
```

### 9.3 Alias Resolution in Multi-Client

The current 1:1 system uses ID code aliases (`alias_in`/`alias_out`) to map ships that exist in both worlds under different ID codes. In multi-client, this becomes an **N-way alias table**:

```python
# Server maintains: (client_a_idcode) → {client_id: their_idcode}
# When Client 1 references ship "ABC-123", server knows:
#   - In server's X4: ship is "ABC-123" (authoritative)
#   - In Client 2's world: ship is "DEF-456"
#   - In Client 3's world: ship is "GHI-789"

class AliasTable:
    """Multi-client ship ID alias mapping."""
    
    # server_idcode → {client_id: client_idcode}
    forward: dict[str, dict[str, str]]
    # (client_id, client_idcode) → server_idcode
    reverse: dict[tuple[str, str], str]
    
    def add(self, server_code: str, client_id: str, client_code: str):
        self.forward.setdefault(server_code, {})[client_id] = client_code
        self.reverse[(client_id, client_code)] = server_code
    
    def to_client(self, server_code: str, client_id: str) -> str:
        return self.forward.get(server_code, {}).get(client_id, server_code)
    
    def from_client(self, client_id: str, client_code: str) -> str:
        return self.reverse.get((client_id, client_code), client_code)
```

When the server forwards a message from Client 1 to Client 2, it rewrites ID codes:

```python
def rewrite_for_recipient(msg: str, sender_id: str, recipient_id: str, aliases: AliasTable) -> str:
    """Rewrite ship ID codes in a message from sender's codes to recipient's codes."""
    fields = msg.split("|")
    kind = fields[0]
    
    # Message kinds that contain ID codes in specific positions
    id_fields = {
        "K": [2],      # K|code|macro|sector
        "D": [2],      # D|code|macro|sector|hull|enemy
        "F": [2],      # F|code|macro|sector
        "B": ["ships"], # B|t|sector|radius|complete|ship1;ship2;... (each ship starts with code)
        "O": [3],      # O|what|code|macro|sector|owner
        # ... etc
    }
    
    # For each ID code field, translate: sender_code → server_code → recipient_code
    for pos in id_fields.get(kind, []):
        if pos == "ships":
            # Handle B message ship list
            entries = (fields[5] or "").split(";")
            rewritten = []
            for entry in entries:
                parts = entry.split(",")
                server_code = aliases.from_client(sender_id, parts[0])
                recipient_code = aliases.to_client(server_code, recipient_id)
                parts[0] = recipient_code
                rewritten.append(",".join(parts))
            fields[5] = ";".join(rewritten)
        elif pos < len(fields):
            server_code = aliases.from_client(sender_id, fields[pos])
            fields[pos] = aliases.to_client(server_code, recipient_id)
    
    return "|".join(fields)
```

---

## 10. Client Lifecycle

### 10.1 Join Sequence

```
1. Client bridge starts, connects to server over TCP
2. Client sends AUTH|player_id|password_hash|protocol_version|player_name
3. Server validates:
   - Check ban list → reject if banned
   - Check protocol version → reject if mismatch
   - Check password → reject if wrong
   - Check if player_id already connected → reject old connection
4. Server assigns client_id, sends AUTH_OK|client_id|world_id|role=join
5. Server sends current save offer (X|offer|size|sha256|token) over TCP
6. Client fetches save over TCP, verifies, installs as quicksave
7. Client's X4 loads the save (Lua mod triggers /x4coop loadshared)
8. Client sends first snapshot (S|...) and world link (L|...)
9. Server registers client in SessionManager
10. Server broadcasts J|join|client_id|player_name|sector to all other clients
11. Server sends J|players|count|roster to the new client (so it knows who's online)
12. Other clients spawn proxy ships for the new player
```

### 10.2 Disconnect Sequence

```
1. Client bridge detects TCP disconnect OR no UDP for PARTNER_TIMEOUT_S (10s)
2. Server removes client from SessionManager
3. Server broadcasts J|leave|client_id|reason to remaining clients
4. Remaining clients despawn the leaving player's proxy ship
5. Server's X4 is notified (N|client disconnected|player_name)
6. Server persists client state (player profile: credits, wares) for reconnection
```

### 10.3 Reconnect Sequence

```
1. Client bridge reconnects over TCP with same player_id + session_token
2. Server recognizes player_id, restores session
3. Server sends AUTH_OK with same client_id (or new one if expired)
4. If save has changed since disconnect, server offers new save
5. If save is the same, client skips save download and resumes delta stream
6. Server broadcasts J|join (same as new join, but other clients may keep proxy if brief)
7. Client's X4 may need to reload save if world state has diverged
```

### 10.4 State Persistence

The server bridge maintains a lightweight profile per player (extending the current `SaveShare._profile_*` mechanism):

```python
@dataclass
class PlayerProfile:
    """Persisted between sessions for each player."""
    player_id: str
    name: str
    credits: int           # joiner's own wallet
    wares: str             # joiner's inventory
    last_sector: str       # last known sector
    last_ship_macro: str   # last known ship type
    last_seen: float       # timestamp of last session
    total_playtime: float  # cumulative
    sessions: int          # count of sessions
```

Stored as `x4coop_profile_<player_id>.txt` in the server's save directory (same format as current, but keyed by player_id instead of world).

---

## 11. Authentication

### 11.1 Player IDs

Each player has a persistent **player_id** — a random 16-char hex string generated on first run and stored in the X4 save (via the Lua mod's blackboard). This persists across sessions and survives save reloads.

```lua
-- In x4_coop.lua
local PLAYER_ID_KEY = "$x4coop_player_id"

local function get_player_id()
    local id = GetNPCBlackboard(S.player, PLAYER_ID_KEY)
    if type(id) ~= "string" or not id:match("^%x+$") or #id ~= 16 then
        id = string.format("%08x%08x", math.random(0, 0xFFFFFFFF), math.random(0, 0xFFFFFFFF))
        SetNPCBlackboard(S.player, PLAYER_ID_KEY, id)
    end
    return id
end
```

### 11.2 Session Tokens

On successful authentication, the server generates a **session_token** (32 hex chars, random) and returns it with the `AUTH_OK` message. The client stores it and presents it on reconnection. Session tokens expire after 300 seconds of disconnection.

```python
def authenticate(self, player_id: str, password_proof: bytes, protocol: int) -> str | None:
    """Returns session token on success, None on failure."""
    if player_id in self.ban_list:
        return None
    if protocol != PROTOCOL_VERSION:
        return None
    if not hmac.compare_digest(password_proof, self.expected_proof):
        return None
    if player_id in self.by_player_id:
        # Kick existing session
        old_client_id = self.by_player_id[player_id]
        self.remove_client(old_client_id, "replaced by new connection")
    
    token = os.urandom(16).hex()
    self.session_tokens[token] = (player_id, time.time())
    return token
```

### 11.3 Password Verification

The shared password model is preserved for backward compatibility. The proof is HMAC-SHA256 of the player_id using the shared password's signing key:

```python
# Client:
proof = hmac.new(signing_key, player_id.encode(), hashlib.sha256).digest()

# Server:
expected = hmac.new(signing_key, player_id.encode(), hashlib.sha256).digest()
```

This proves the client knows the password without sending it in cleartext, and binds the proof to the player_id to prevent token theft.

### 11.4 Ban List

A simple text file on the server: `x4coop_banlist.txt`, one player_id per line. The server loads it on startup and checks on each auth attempt. Can be modified at runtime via a server bridge command:

```
python x4_coop_bridge.py --ban <player_id>     # add ban
python x4_coop_bridge.py --unban <player_id>   # remove ban
python x4_coop_bridge.py --list-bans            # show ban list
```

### 11.5 Per-Client Encryption Keys

While the current bridge derives one set of keys from the shared password, the multi-client server derives **per-client session keys** using a key exchange:

```python
def derive_session_key(shared_key: bytes, client_id: str, server_nonce: bytes, client_nonce: bytes) -> bytes:
    """Derive a per-client encryption key from the shared key and nonces."""
    material = shared_key + server_nonce + client_nonce + client_id.encode()
    return hashlib.sha256(material).digest()
```

This means even if one client's session is compromised, other clients' traffic remains secure. The nonces are exchanged during the TCP auth handshake.

---

## 12. Backward Compatibility

### 12.1 P2P Mode

The server bridge retains full 1:1 P2P capability via a `--p2p` flag (or auto-detection):

```bash
# P2P mode (current behavior, unchanged):
python x4_coop_bridge.py --host --password secret
python x4_coop_bridge.py --join 203.0.113.7 --password secret

# Multi-client server mode:
python x4_coop_bridge.py --server --password secret --max-clients 8

# Multi-client client mode:
python x4_coop_bridge.py --connect 203.0.113.7 --player-id <id> --password secret
```

**Auto-detection:** If `--server` or `--connect` is specified, multi-client mode is used. If `--host` or `--join` is specified, P2P mode is used. The two modes are mutually exclusive.

### 12.2 Protocol Negotiation

When a bridge connects, the first datagram's magic byte determines the mode:

- `X4C2` → P2P mode (current behavior, no routing prefix, single partner)
- `X4C3` → Multi-client mode (routing prefix, session management)

A server bridge in `--server` mode accepts only X4C3 connections. A bridge in `--host` mode accepts only X4C2. This prevents accidental cross-mode connections.

### 12.3 Lua Mod Compatibility

The Lua mod requires **no changes for P2P mode**. For multi-client mode, the mod needs extensions to:

1. **Track multiple proxy ships** (one per connected client) instead of a single proxy
2. **Parse the routing prefix** (`>SRC>DST|`) and strip it before processing
3. **Handle `J|join`/`J|leave`/`J|players`** messages for multi-client awareness
4. **Manage multiple NPC bubble sources** (one per other client)

These changes are additive — the Lua mod can detect multi-client mode by checking for the X4C3 magic byte or the presence of a client_id in the bridge's `W|` welcome message.

### 12.4 Save Format

The save file format is unchanged. The server's save is the shared world, same as the current host's save. All clients load it the same way (quicksave replacement + `/x4coop loadshared`).

---

## 13. Sequence Diagrams

### 13.1 Client Join

```
Client Bridge                    Server Bridge                    Server X4
     │                                │                              │
     │  TCP connect to server:47810   │                              │
     │───────────────────────────────►│                              │
     │  AUTH|player_id|proof|v5|name  │                              │
     │───────────────────────────────►│                              │
     │                                │  Check ban list              │
     │                                │  Check password proof        │
     │                                │  Check protocol version      │
     │                                │  Assign client_id=00000002   │
     │  AUTH_OK|00000002|world_id|join│                              │
     │◄───────────────────────────────│                              │
     │  X|offer|size|sha256|token     │                              │
     │◄───────────────────────────────│                              │
     │  TCP: fetch save (encrypted)   │                              │
     │───────────────────────────────►│                              │
     │  Save data (chunked)           │                              │
     │◄───────────────────────────────│                              │
     │  Verify sha256 + gzip          │                              │
     │  Install as quicksave          │                              │
     │  Tell X4: X|received|quicksave │                              │
     │  (via named pipe)              │                              │
     │                                │                              │
     │  X4 loads save                 │                              │
     │  First snapshot S|1|...        │                              │
     │  (UDP) ───────────────────────►│  tell_game(S|1|...)          │
     │  L|world_id|join|ship_code|v5  │  tell_game(L|...)            │
     │  (UDP) ───────────────────────►│                              │
     │                                │  Register in SessionManager  │
     │                                │  Broadcast J|join|00000002   │
     │                                │  ────────► Other Clients     │
     │  J|players|3|00000001:Alice;00000002:Bob;00000003:Carol      │
     │◄───────────────────────────────│                              │
     │                                │                              │
     │  Steady state: snapshots, NPC bubbles, combat events flowing  │
     │  ◄────────────── UDP ─────────►  ◄──── pipe ────►             │
```

### 13.2 Client Disconnect

```
Client 2 Bridge              Server Bridge              Other Clients          Server X4
     │                            │                          │                     │
     │  (TCP connection drops     │                          │                     │
     │   OR no UDP for 10s)       │                          │                     │
     │      ✗                     │                          │                     │
     │                            │  Detect timeout          │                     │
     │                            │  last_heard > 10s        │                     │
     │                            │                          │                     │
     │                            │  Remove from             │                     │
     │                            │  SessionManager          │                     │
     │                            │                          │                     │
     │                            │  J|leave|00000002|timeout│                     │
     │                            │─────────────────────────►│                     │
     │                            │                          │  Despawn proxy      │
     │                            │                          │  for Client 2       │
     │                            │                          │                     │
     │                            │  N|client disconnected|Bob│                     │
     │                            │──────────────────────────────────────────────►│
     │                            │                          │                     │
     │                            │  Persist PlayerProfile   │                     │
     │                            │  (credits, wares)        │                     │
     │                            │                          │                     │
```

### 13.3 Command Flow (Client A fires at NPC, Client B sees it)

```
Client A X4        Client A Bridge     Server Bridge      Server X4        Client B Bridge    Client B X4
     │                   │                  │                 │                  │                  │
     │  F|npc_code|      │                  │                 │                  │                  │
     │  macro|sector     │                  │                 │                  │                  │
     │──────────────────►│                  │                 │                  │                  │
     │                   │  >A>*|F|npc_code │                 │                  │                  │
     │                   │  (UDP, X4C3)     │                 │                  │                  │
     │                   │─────────────────►│                 │                  │                  │
     │                   │                  │  F|npc_code|    │                  │                  │
     │                   │                  │  macro|sector   │                  │                  │
     │                   │                  │  (strip prefix, │                  │                  │
     │                   │                  │   alias rewrite)│                  │                  │
     │                   │                  │────────────────►│                  │                  │
     │                   │                  │                 │                  │                  │
     │                   │                  │  Server X4      │                  │                  │
     │                   │                  │  processes fire │                  │                  │
     │                   │                  │  event          │                  │                  │
     │                   │                  │                 │                  │                  │
     │                   │                  │  Forward to B   │                  │                  │
     │                   │                  │  >A>B|F|npc_code│                  │                  │
     │                   │                  │  (rewrite to B's│                  │                  │
     │                   │                  │   alias)        │                  │                  │
     │                   │                  │─────────────────────────────────►│                  │
     │                   │                  │                 │                  │  F|npc_code|     │
     │                   │                  │                 │                  │  macro|sector    │
     │                   │                  │                 │                  │  (strip prefix)  │
     │                   │                  │                 │                  │─────────────────►│
     │                   │                  │                 │                  │                  │
     │                   │                  │                 │                  │  Proxy fires at  │
     │                   │                  │                 │                  │  NPC in Client B │
```

---

## 14. Error Handling & Edge Cases

### 14.1 Client Crash

**Scenario:** Client's X4 crashes or bridge process is killed.

**Detection:** No UDP keepalives for `PARTNER_TIMEOUT_S` (10s) + TCP connection drops.

**Handling:**
1. Server marks client as disconnected after timeout
2. Server broadcasts `J|leave|client_id|crash` to remaining clients
3. Remaining clients despawn the crashed player's proxy
4. Server's X4 continues running — the crashed player's ship becomes AI-controlled or stays put
5. Player profile is persisted (last known credits/wares)
6. On reconnect, player gets a fresh save (world may have changed significantly)

### 14.2 Server Crash

**Scenario:** Server bridge process crashes or server X4 crashes.

**Detection (Client side):**
1. TCP control connection drops immediately
2. UDP keepalives stop
3. Client bridges detect server absence within `PARTNER_TIMEOUT_S`

**Handling:**
1. Client bridges notify their X4: `N|server disconnected`
2. Client X4 removes all proxy ships (or freezes them)
3. Clients cannot re-establish the shared world without the server
4. **Recovery:** Server bridge restarts → clients auto-reconnect using stored session token → if token expired, re-authenticate → server offers latest save → clients reload

**Mitigation:** The server bridge should write a crash log and optionally auto-restart via a watchdog script. The server's X4 save is the authoritative state and survives a bridge crash.

### 14.3 Partial Disconnect (UDP alive, TCP dead)

**Scenario:** TCP connection drops but UDP packets still flowing (e.g., TCP RST from a firewall but UDP hole punching still works).

**Handling:**
1. Server continues accepting UDP from the client (data plane stays up)
2. Server attempts TCP reconnection from its end
3. If TCP cannot be re-established within 30s, server falls back to UDP-only mode for that client
4. In UDP-only mode, reliable messages use the existing retry/ack mechanism over UDP
5. Save transfer is impossible without TCP — if a new save is needed, client must reconnect

### 14.4 Client Duplication (Two connections, same player_id)

**Scenario:** Player reconnects without properly disconnecting first (e.g., network glitch causes new TCP connection while old one is half-open).

**Handling:**
1. Server detects duplicate player_id during AUTH
2. Server sends `KICK|old_client_id|replaced` to the old connection
3. Server removes old session, assigns new client_id (or reuses if within reconnect window)
4. Old connection's TCP is closed; any UDP from old address is ignored after removal
5. New connection gets a fresh session with same player profile

### 14.5 Save Divergence

**Scenario:** Client's save doesn't match server's save (e.g., client loaded their own save instead of the host's).

**Detection:** World ID check in `L` message — `partner_world != server_world`.

**Handling:**
1. Server sends `N|world mismatch: your save doesn't match the server's world`
2. Server offers current save: `X|offer|size|sha256|token`
3. Client must load the server's save to continue
4. If client refuses (keeps their save), server kicks them: `KICK|client_id|world mismatch`

### 14.6 Rate Limiting / Abuse

**Scenario:** A misbehaving client sends excessive packets (bug or deliberate flooding).

**Handling:**
1. Per-client `RateLimit` token bucket (200 packets/s + 400 burst, same as current)
2. Excess packets are dropped and counted in telemetry
3. If sustained overage > 1000 packets/s for 5 seconds, server kicks the client
4. Server logs the event and optionally adds to a temporary cooldown list

### 14.7 NAT Port Change

**Scenario:** Client's NAT remaps their UDP port mid-session (common with some carriers).

**Handling:**
1. Client's UDP source address changes
2. Server's UDP recvfrom sees packets from new address
3. Server looks up by `client_id` in the X4C3 header (not by address)
4. Server updates `session.udp_addr` to the new address
5. No disruption to the client

This is an improvement over the current 1:1 bridge, which only checks IP address for joiners and would break on a port change.

### 14.8 Clock Skew

**Scenario:** A client's system clock is significantly off from the server's.

**Handling:**
- The current `MAX_SKEW_S = 60` seconds is retained
- During TCP auth handshake, server sends its current timestamp; client adjusts its packet timestamps accordingly
- If skew > 60s even after adjustment, auth is rejected with `AUTH_DENY|clock skew too large`

### 14.9 Late Joiner With Stale Save

**Scenario:** New client joins with an old version of the server's save from a previous session.

**Handling:**
1. Server sends save offer during join
2. Client compares save_hash (sha256) with their current quicksave
3. If hashes match, client skips download and proceeds to delta stream
4. If hashes differ, client downloads new save
5. If client's save has a different world_id, they must download (see §14.5)

---

## 15. Implementation Phases

### Phase 1: Protocol Extension (X4C3)

- Add `client_id` field to datagram header
- Add routing prefix `>SRC>DST|` to plaintext messages
- Implement X4C3 magic byte negotiation
- Keep X4C2 path unchanged for P2P mode
- **Estimated effort:** 2–3 days

### Phase 2: Server Bridge Core

- Implement `SessionManager` and `ClientSession` data structures
- Add `--server` and `--connect` CLI modes
- TCP control channel with length-prefixed framing
- Authentication handshake (AUTH/AUTH_OK/AUTH_DENY)
- Per-client `Codec` instances with per-session keys
- Per-client rate limiting
- **Estimated effort:** 3–4 days

### Phase 3: State Routing & Broadcast

- Implement message routing (snapshot, NPC bubble, combat, chat)
- Per-recipient NPC bubble pruning
- Multi-client alias table with ID code rewriting
- Server-side fan-out with routing prefixes
- Pipe multiplexer (tag game messages with source client_id)
- **Estimated effort:** 3–4 days

### Phase 4: Client Lifecycle

- Join sequence (auth → save transfer → first snapshot → roster broadcast)
- Disconnect detection and cleanup
- Reconnect with session tokens
- Player profile persistence
- Ban list management
- **Estimated effort:** 2–3 days

### Phase 5: Lua Mod Extensions

- Multiple proxy ship management (array of proxies, one per connected client)
- Parse and strip routing prefix from pipe messages
- Handle `J|join`/`J|leave`/`J|players` for multi-client awareness
- Multiple NPC bubble sources (track per-client areas)
- Extended `L` message with client_id and player_name
- **Estimated effort:** 4–5 days

### Phase 6: Hardening & Testing

- Edge case testing (crash, partial disconnect, NAT change, rate limiting)
- Telemetry extensions for multi-client tracing
- Performance testing with 4+ clients
- Backward compatibility testing (X4C2 ↔ X4C2 P2P mode)
- Documentation and user guide
- **Estimated effort:** 3–4 days

### Total Estimated Effort: 17–23 days

---

## Appendix A: Protocol Version History

| Version | Magic | Key Changes |
|---------|-------|-------------|
| X4C1 | `b"X4C1"` | Unencrypted, original format |
| X4C2 | `b"X4C2 "` | HMAC + XOR encryption, session/counter/timestamp |
| X4C3 | `b"X4C3 "` | Multi-client: client_id in header, routing prefix, TCP control plane |

## Appendix B: Message Routing Reference

| Kind | Source | Server Action | Forwarded? | Pruning |
|------|--------|---------------|------------|---------|
| S | Client | Deliver to X4 | Yes, broadcast to all others | None (small) |
| B | Client | Deliver to X4 | Yes, per-recipient | Sector + radius |
| K | Client | Deliver to X4 | Yes, to clients in same sector | Sector match |
| D | Client | Deliver to X4 | Yes, to clients tracking that ship | Ship in bubble |
| F | Client | Deliver to X4 | Yes, to clients in same sector | Sector match |
| M | Client | Deliver to X4 | Yes, broadcast | None (small) |
| P | Client | Respond Q | No | N/A |
| Q | Client | Update RTT | No | N/A |
| L | Client | Process link | No | N/A |
| R | Server | Tell X4 | No (server-originated) | N/A |
| C | Client | Deliver to X4 | Routed to specific recipient | Target client |
| T | Client | Deliver to X4 | Routed to specific recipient | Target client |
| V | Client | Deliver to X4 | Broadcast (state) / routed (change) | All or target |
| U | Client | Deliver to X4 | Routed to specific recipient | Target client |
| Z | Client | Deliver to X4 | Yes, broadcast | None (small) |
| O/Y/G/J/b/s/c/r/a/m/e | Client | Deliver to X4 | Broadcast effects via server X4 output | Per-message |
| H | Client/Server | Update last_heard | No | N/A |
| J (join/leave/players) | Server | N/A | Yes, broadcast | None |
| X | Client/Server | Save handoff | TCP only | Target client |

## Appendix C: CLI Reference

```bash
# P2P mode (unchanged, backward compatible):
python x4_coop_bridge.py --host --password SECRET
python x4_coop_bridge.py --join 203.0.113.7 --password SECRET
python x4_coop_bridge.py --peer 198.51.100.4 --role host --password SECRET

# Multi-client server mode:
python x4_coop_bridge.py --server --password SECRET --max-clients 8
python x4_coop_bridge.py --server --password SECRET --max-clients 4 --port 47810

# Multi-client client mode:
python x4_coop_bridge.py --connect 203.0.113.7 --password SECRET
python x4_coop_bridge.py --connect 203.0.113.7 --password SECRET --player-id auto

# Server management:
python x4_coop_bridge.py --server --ban a1b2c3d4e5f6a1b2
python x4_coop_bridge.py --server --unban a1b2c3d4e5f6a1b2
python x4_coop_bridge.py --server --list-bans

# Telemetry (unchanged):
python x4_coop_bridge.py --server --password SECRET --trace --overlay
```

---

*End of Design Document #17*