# X4 Co-op Mod

A multiplayer mod for X4: Foundations that enables true 2-player co-op through a host-authoritative architecture.

## How It Works

- **Host** runs the full X4 universe simulation (economy, NPC AI, world state)
- **Guest** runs a second X4 instance with simulation suppressed, receiving state from the host
- **Bridge Service** (Node.js) handles TCP/UDP networking between the two instances
- **Lua Mods** read/write game state and communicate with the bridge via file I/O

## Status

🚧 In Planning — see [PRD.md](./PRD.md) and the [Wayfinder Map](https://github.com/WembassyCo/x4-coop/issues/1)

## Architecture

```
Host X4 ←→ Host Bridge ←─TCP/UDP─→ Guest Bridge ←→ Guest X4
         (state export)              (state injection)
         (command exec)              (input capture)
```

## Requirements

- X4: Foundations (latest version)
- Node.js 22+
- Both players need X4 installed
- Direct P2P connection (or optional relay for NAT traversal)

## License

MIT