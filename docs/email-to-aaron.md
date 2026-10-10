# Email to Aaron Sturman — Collaboration Proposal

**To:** aaron.sturman@live.co.uk
**From:** Chris McIntosh (chrismcintosh@wembassy.com)
**Subject:** x4_coop — I'd like to help extend your mod

---

Hi Aaron,

I came across your x4_coop mod on GitHub and I'm genuinely impressed. The architecture is elegant — using LuaJIT's FFI to call kernel32 for named pipes, the Python bridge with stdlib-only networking, the regional authority model, and the fact that you've achieved all of this without any DLLs or memory hacking is remarkable. The ghost mode test harness is particularly clever.

I'm Chris McIntosh, a developer working with AI-assisted tooling (I use an AI CTO agent built on OpenClaw + Claude/GLM models — sounds like you're working with Claude Opus 5.5, so we have that in common). I've done a thorough review of your codebase — the bridge, the Lua sync layer, the MD scripts, the test infrastructure. It's some of the cleanest mod code I've seen.

I'd like to contribute to the project rather than fork it. Specifically, I'm interested in extending x4_coop in a direction that I think is the natural evolution of your work: **a persistent universe server.**

The idea: instead of two players starting a session together, a headless X4 instance runs as a dedicated server (spectator mode, no player ship, simulation runs continuously). Players connect their X4 client, play, disconnect. The universe keeps living — economy shifts, wars happen, stations get built. Next time they connect, the world has moved forward. Think Ark or Rust, but for X4.

This requires three extensions to your existing architecture:
1. **Multi-client bridge** — your 1:1 relay becomes 1:N (one server, many clients)
2. **Headless server mod** — a spectator bot that runs X4 with no player input, auto-saving for persistence
3. **Mid-game join protocol** — clients connect to a running universe instead of loading a shared save

Your core work — the FFI pipe client, the sync protocol, the entity pairing, the regional authority model — all still applies directly. This is an extension, not a rewrite.

Concrete contributions I'd like to make:
- **Add a LICENSE file** (MIT) — I noticed the repo doesn't have one yet, and it's a prerequisite for any community contribution
- **Performance stress testing** — run full-scale sessions with your telemetry tools and publish results
- **Multi-client bridge extension** — design and implement the 1:N server architecture
- **Headless server mod** — the spectator bot Lua/MD scripts
- **Cross-platform client support** — Unix domain sockets for macOS/Linux clients
- **Packaging** — Nexus Mods / Steam Workshop page with install guide

I've set up a project repo at https://github.com/WembassyCo/x4-coop with the research I've done so far (architecture analysis, state structure mapping, simulation suppression investigation). You're welcome to look at it — the research documents are in docs/research/.

Would you be open to this? I'm happy to work however suits you best — PRs against your repo, a shared org, or just coordination via email/GitHub. I'm not looking to take over or redirect your work — I want to build on it and contribute back.

Let me know what you think.

Best,
Chris McIntosh
chrismcintosh@wembassy.com
https://github.com/cmcintosh