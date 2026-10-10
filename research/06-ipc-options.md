# Research Ticket #6: X4: Foundations Lua IPC Options

**Date:** 2026-10-10
**Status:** COMPLETE
**Verdict:** Three viable IPC pathways exist. Named Pipes (via SirNukes mod) is the proven, production-ready option. LuaSocket (via djfhe/x4_http) is a proven alternative for HTTP-based communication. FFI + custom DLL is the underlying mechanism both rely on.

---

## Executive Summary

X4: Foundations runs **LuaJIT** (Lua 5.1-compatible with FFI support). The base game **sands down the Lua environment** by setting `io = nil` and `os = nil`, removing standard file I/O and os.execute. However, **LuaJIT's FFI library is available**, and mods can load external C DLLs via `ffi.load()`. This is the critical escape hatch.

**Three IPC mechanisms are proven to work in production mods:**

| # | Mechanism | Requires | Bidirectional | Windows | Linux | Production Use |
|---|-----------|----------|---------------|---------|-------|----------------|
| 1 | **Named Pipes** (SirNukes API) | Protected UI off, custom DLL | ✅ Yes | ✅ | ❌ | x4-agent (RalfHermsen) |
| 2 | **LuaSocket + HTTP** (djfhe/x4_http) | Protected UI off, luasocket/luasec DLLs | ✅ (via HTTP) | ✅ | ✅ (likely) | X4-External-App (mycumycu) |
| 3 | **FFI + custom C DLL** | Protected UI off | Depends on DLL | ✅ | ✅ | Underlies both above |

**Recommended for X4 Co-op:** Named Pipes (SirNukes API) for the in-game bridge, with a Python host process on the game machine that forwards to the co-op server over standard TCP/HTTP.

---

## 1. File I/O: Can Lua read/write arbitrary files?

**Answer: NO — the `io` library is set to nil in X4's Lua sandbox.**

X4's Lua environment removes the standard `io` module by setting it to `nil` after initializing the sandbox. This means:
- `io.open()` → not available
- `io.read()` / `io.write()` → not available
- No standard file read/write from Lua

**However, there is a limited file output mechanism:**
- `<debug_to_file>` in MD scripts can write to log files when the game is launched with `-scriptlogfiles` and `-logfile` switches
- `DebugError()` in Lua writes to the debug log (always on, even without `-debug` flag)
- These are **write-only, append-only** log outputs — not suitable for IPC

**Named Pipes (FIFOs) from Lua directly:** Not possible without `io` library. However, the SirNukes Named Pipes API loads a custom C DLL via FFI that calls the Windows CreateNamedPipe API directly, bypassing the missing `io` library.

**Save file location (for the workaround idea):**
- Windows: `Documents\Egosoft\X4\<SteamID>\save\`
- Format: Gzip-compressed XML
- Contains: player info, universe state (sectors, ships, NPCs), logbook
- Can be parsed/modified externally (x4-agent does streaming parse), but the game only reads it on load and writes on save — not a real-time channel

---

## 2. os.execute(): Can Lua call external commands?

**Answer: NO — the `os` library is set to nil in X4's Lua sandbox.**

The standard `os.execute()`, `os.exit()`, `os.getenv()` are all unavailable because X4 sets `os = nil`. This means:
- No shelling out to `curl`, `nc`, `python`, or any external command
- No environment variable access
- No process spawning from Lua directly

**Workaround via FFI:** Since LuaJIT's FFI is available, you could theoretically call `system()` or `CreateProcess()` via FFI:
```lua
local ffi = require("ffi")
ffi.cdef[[ int system(const char *command); ]]
ffi.C.system("curl http://example.com")
```
However, `system()` is synchronous and would block the game thread. The SirNukes approach (custom DLL with non-blocking pipe communication) is the correct architecture.

---

## 3. LuaSocket: Is the LuaSocket library available?

**Answer: NOT BUILT IN — but can be loaded as external DLLs via FFI, with Protected UI disabled.**

The djfhe/x4_http mod proves this works. It bundles compiled `luasocket` and `luasec` DLLs and loads them via LuaJIT's `require()`:

```lua
local socket = require("luasocket.socket")
local https = require("luasec.https")
local ltn12 = require("luasocket.ltn12")
```

The x4_http library wraps these synchronous calls in a non-blocking callback interface that polls every 50ms, making it safe for use in the game loop.

**Key constraints:**
- **Protected UI Mode must be DISABLED** — this is the single most important requirement
- The DLLs must be compiled against X4's specific LuaJIT build (Lua 5.1 API)
- The mod provides HTTP client functionality (GET, POST, JSON parsing)
- This is the approach used by the X4-External-App mod (mycumycu) which connects X4 to an external web dashboard

**For TCP sockets directly:** LuaSocket gives you raw TCP/UDP socket access too, so you could implement a custom binary protocol over TCP without HTTP overhead.

---

## 4. ffi.load: Does X4's LuaJIT have FFI enabled?

**Answer: YES — FFI is available, and `ffi.load()` works to load external C DLLs.**

X4 uses LuaJIT (Lua 5.1-compatible), and the FFI library is **not disabled** in the sandbox. This is the fundamental escape hatch that enables all real IPC in X4 mods.

**Evidence:**
1. **SirNukes Named Pipes API** loads `winpipe_64.dll` via FFI to provide Windows named pipe communication
2. **djfhe/x4_http** loads `luasocket` and `luasec` DLLs via `require()` (which internally uses FFI on LuaJIT)
3. **X4-rest-server** (Alia5) loads custom C++ DLLs to implement a REST server inside the game

**Requirement:** Protected UI Mode must be disabled. When enabled, the game prevents loading of external DLLs from mod directories.

**Example pattern (from SirNukes winpipe.lua):**
```lua
local ffi = require("ffi")
local winpipe = ffi.load("extensions/sn_mod_support_apis/ui/c_library/winpipe_64")
```

**What you could load via FFI:**
- `libcurl` / `libsocket` for networking
- Custom C libraries for shared memory, pipes, sockets, IPC
- Windows API functions directly via `ffi.C` (default C library namespace)
- Any precompiled `.dll` (Windows) or `.so` (Linux) placed in the mod directory

---

## 5. MD (Mission Director) External Calls

**Answer: MD scripts cannot directly execute external programs or read URLs.**

The Mission Director (MD) is an XML-based scripting system that runs in the game engine. It does NOT have:
- File I/O capabilities (beyond debug_to_file log writing)
- HTTP/web request capabilities
- Process execution capabilities
- Any direct external communication

**However, MD scripts CAN communicate with Lua via `raise_lua_event`:**
```xml
<raise_lua_event name="'my_event'" param1="value1" />
```
This is the bridge between MD and Lua. The typical pattern is:
1. MD script detects game state change → raises Lua event
2. Lua handler receives event → uses FFI/pipe/socket to send message externally
3. External process responds → Lua handler receives response → calls back into MD via `Helper` functions

**The x4-agent architecture proves this pattern:**
- MD script (XML) reads game state and raises Lua events
- Lua scripts use SirNukes Named Pipes to communicate with Python host
- Python host sends orders back through the pipe → Lua → MD execution

---

## 6. Shared Memory

**Answer: Not directly available, but achievable via FFI.**

There is no built-in shared memory mechanism in X4's Lua environment. However, since FFI is available:
- On Windows: `ffi.C` can call `CreateFileMapping()`, `MapViewOfFile()` from kernel32.dll
- On Linux: `ffi.C` can call `shm_open()`, `mmap()` from librt
- A custom C DLL could wrap shared memory operations

**Practical assessment:** Shared memory is overkill for the co-op use case. Named pipes or sockets are simpler, proven, and sufficient for the message volumes involved.

---

## 7. Workaround: Save File as Communication Channel

**Answer: Possible but impractical for real-time communication.**

Save files are gzip-compressed XML at `Documents\Egosoft\X4\<SteamID>\save\`. An external process can:
- **Read** the save file to extract game state (x4-agent does this with streaming XML parse)
- **Modify** the save file to inject changes (risky, can corrupt saves)
- The game only reads saves on **explicit load** and writes on **explicit save**

This is NOT a real-time channel. The latency is measured in minutes (manual save/load cycle). However, it has one advantage: it works with **Protected UI Mode enabled** — no mods required for the read side.

**Best use case:** One-shot state extraction for initialization or save analysis, not ongoing communication.

---

## Proven Architectures from Existing Projects

### Architecture A: Named Pipes (x4-agent pattern)

```
X4 (Windows) <--named pipes--> Python host (Windows) <--LAN/HTTP--> External services
   MD + Lua                       Pipe host                    Co-op server
```

**Components:**
- **In-game:** MD script (XML) + Lua scripts using SirNukes Named Pipes API
- **Host:** Python process running on the same Windows machine as X4
- **External:** Python host forwards to remote services via standard TCP/HTTP

**Pros:**
- Proven in production (x4-agent plays the game autonomously)
- Bidirectional, low-latency
- Python host can do heavy computation before sending to/from game
- SirNukes API is MIT licensed, well-maintained

**Cons:**
- Windows-only (named pipes are a Windows IPC mechanism)
- Requires Protected UI Mode disabled
- Requires Python host process running on the game machine
- One pipe owner at a time (must manage stale hosts)
- Back-to-back commands can drop the pipe (need spacing)

**Key files:**
- SirNukes API: https://github.com/bvbohnen/x4-projects/tree/master/extensions/sn_mod_support_apis
- Named Pipes docs: https://github.com/bvbohnen/x4-projects/tree/master/extensions/sn_mod_support_apis/documentation
- x4-agent: https://github.com/RalfHermsen/x4-agent
- x4-agent IPC docs: https://github.com/RalfHermsen/x4-agent/blob/main/docs/phase2-ipc.md

### Architecture B: HTTP via LuaSocket (X4-External-App pattern)

```
X4 (any OS) <--HTTP--> External web server <---> Frontend/other services
   Lua + luasocket        Python/Node.js
```

**Components:**
- **In-game:** Lua scripts using djfhe/x4_http library (which wraps luasocket/luasec)
- **External:** Any HTTP server reachable from the game machine

**Pros:**
- Cross-platform (luasocket works on Windows and Linux)
- HTTP is firewall-friendly
- No host process needed on game machine — Lua talks directly to remote server
- Non-blocking callback interface (polls every 50ms)

**Cons:**
- Requires Protected UI Mode disabled (for DLL loading)
- HTTP overhead (headers, TCP handshake) — higher latency than pipes
- Requires compiling luasocket/luasec DLLs against X4's LuaJIT
- djfhe/x4_http is still in beta

**Key files:**
- x4_http: https://github.com/djfhe/x4_http
- X4-External-App: https://github.com/mycumycu/X4-External-App

### Architecture C: REST Server Inside Game (X4-rest-server pattern)

```
External clients <--HTTP--> X4 (running REST server inside game)
                     C++ DLL loaded via FFI
```

**Components:**
- **In-game:** Custom C++ DLL loaded via FFI that runs an HTTP server inside the game process
- **External:** Any HTTP client can query game state

**Pros:**
- Game becomes a server — external clients connect to it
- No host process needed on game machine

**Cons:**
- Most complex to build (C++ REST server embedded in game)
- Security concerns (open port on game machine)
- POC/WIP status — less proven than the other approaches

**Key files:**
- X4-rest-server: https://github.com/Alia5/X4-rest-server

---

## Critical Requirement: Protected UI Mode

**Protected UI Mode** is X4's security feature that prevents mods from loading external DLLs. When enabled:
- `ffi.load()` for external DLLs fails
- `require()` for C extension modules fails
- Only pure-Lua mods work

**To disable:** Settings → Extensions → disable Protected UI Mode

**When disabled, the game shows a warning popup** about mod safety. This is expected and necessary for any IPC solution.

**Impact on co-op mod:** The co-op mod MUST require users to disable Protected UI Mode. This is standard for any mod that does external communication — all three proven architectures require it.

---

## Lua Environment Summary

| Property | Value |
|----------|-------|
| Lua version | Lua 5.1 API (via LuaJIT) |
| JIT | Yes — LuaJIT is the runtime |
| FFI | Available (`require("ffi")` works) |
| `io` library | **NIL** (removed by sandbox) |
| `os` library | **NIL** (removed by sandbox) |
| `require` | Available (can load C extension DLLs) |
| `DebugError()` | Available (writes to debug log) |
| `print()` | Available (writes to log when logging enabled) |
| Protected UI | Blocks FFI/require for external DLLs when enabled |
| `raise_lua_event` | MD → Lua bridge (available) |
| `<debug_to_file>` | MD log writing (available with -scriptlogfiles) |

---

## Recommendation for X4 Co-op Bridge

**Use Architecture A (Named Pipes) as primary, with Architecture B (HTTP) as fallback.**

### Primary: Named Pipes + Python Host

1. **In-game mod:** MD script + Lua scripts using SirNukes Named Pipes API
2. **Python host:** Runs on the same machine as X4, owns the named pipe
3. **Co-op server:** Python host connects to the co-op server via standard TCP/WebSocket
4. **Message flow:** MD detects game events → Lua → Named Pipe → Python host → TCP → Co-op server → TCP → Python host → Named Pipe → Lua → MD commands

### Fallback: HTTP via LuaSocket

If named pipes prove problematic (Windows-only, pipe ownership issues):
1. **In-game mod:** Lua scripts using djfhe/x4_http (luasocket/luasec)
2. **Co-op server:** HTTP endpoint that Lua talks to directly
3. **Message flow:** Lua → HTTP GET/POST → Co-op server → HTTP response → Lua callback

### Why not direct TCP via LuaSocket?

LuaSocket provides raw TCP socket access. You could skip HTTP and do a custom binary protocol. However:
- HTTP gives you structured requests/responses, JSON parsing, status codes
- A custom TCP protocol requires more error handling
- The 50ms polling interface of x4_http is already battle-tested
- For co-op, HTTP request/response maps well to the co-op message types

### Why not the save file?

- Not real-time (minutes of latency vs milliseconds)
- Requires game save/load cycle
- Write side is risky (can corrupt saves)
- Useful only for initial state extraction or save analysis tools

---

## References

| Resource | URL | Relevance |
|----------|-----|-----------|
| SirNukes Mod Support APIs | https://github.com/bvbohnen/x4-projects/tree/master/extensions/sn_mod_support_apis | Named Pipes API, the IPC foundation |
| SirNukes on Nexus | https://www.nexusmods.com/x4foundations/mods/503 | Distribution page |
| x4-agent | https://github.com/RalfHermsen/x4-agent | Full autonomous agent using named pipes |
| x4-agent IPC docs | https://github.com/RalfHermsen/x4-agent/blob/main/docs/phase2-ipc.md | IPC implementation details |
| djfhe/x4_http | https://github.com/djfhe/x4_http | HTTP client library for X4 Lua |
| X4-External-App | https://github.com/mycumycu/X4-External-App | External app using HTTP client |
| X4-rest-server | https://github.com/Alia5/X4-rest-server | REST server inside X4 (C++ DLL via FFI) |
| X4 Lua Language Server Addon | https://github.com/chemodun/X4-LuaLSAddonPrep | 244 Lua functions, 193 C functions documented |
| Egosoft Wiki - Running X4 for modding | https://wiki.egosoft.com/X4%20Foundations%20Wiki/Modding%20Support/Running%20X4%20for%20modding/ | Debug logging, scriptlogfiles |
| Reddit: ChatGPT in X4 | https://www.reddit.com/r/X4Foundations/comments/1joobjn/chatgpt_in_x4/ | Community discussion of luasocket in X4 |
| Reddit: X4 Lua Language Server addon | https://www.reddit.com/r/X4Foundations/comments/1jr6qo6/ | Documentation of X4's Lua environment |
| Nexus: SirNukes fix for 9.0 | https://www.nexusmods.com/x4foundations/mods/2373 | Community Edition compatibility patch |

---

## Appendix: Key Code Patterns

### Loading a C DLL via FFI (SirNukes pattern)
```lua
local ffi = require("ffi")
ffi.cdef[[
  -- Define C function signatures
  typedef struct { /* ... */ } PipeHandle;
  int create_pipe(const char* name);
  int write_pipe(PipeHandle* h, const char* data, int len);
  int read_pipe(PipeHandle* h, char* buf, int len);
]]
local lib = ffi.load("extensions/my_mod/winpipe_64")
```

### HTTP GET request (djfhe/x4_http pattern)
```lua
local Request = require("djfhe.http.request")
Request.new('GET'):setUrl("https://example.com/api/state")
  :send(function(response, err)
    if err then
      DebugError("Error: " .. tostring(err))
    else
      local body = response:getBody()
      -- Process response
    end
  end)
```

### MD → Lua event bridge
```xml
<!-- In MD script (XML) -->
<raise_lua_event name="'coop_send_state'" />
```
```lua
-- In Lua script
function onCoopSendState()
  -- Gather game state
  -- Send via named pipe or HTTP
end
```

### Save file parsing (external, Python)
```python
import gzip
import xml.etree.ElementTree as ET
with gzip.open(save_path, 'rb') as f:
    # Stream parse — full saves don't fit in memory as a tree
    for event, elem in ET.iterparse(f):
        # Extract player state, ships, stations, etc.
```