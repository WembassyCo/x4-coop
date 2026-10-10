# #20 — Server Infrastructure Research

**Ticket:** #20 (Wayfinder)  
**Project:** x4-coop  
**Date:** 2026-10-10  
**Status:** Research Complete

---

## Executive Summary

X4: Foundations does **not** have a headless/dedicated server mode. Any "server" for the x4-coop project must run a **full X4 client instance** with rendering active. The simulation is **not decoupled** from the frame rate — the renderer waits for the economy simulator each frame, meaning low FPS directly slows the simulation. A real GPU with Vulkan support and ≥3GB VRAM is required. This dramatically narrows the infrastructure options.

**Top recommendation:** A **dedicated Windows PC with a low-end dedicated GPU** (~$300–500 upfront) or a **Linux PC running X4 natively** (same price range). Cloud GPU VMs are too expensive for 24/7 operation ($380+/month). Parallels on Mac Mini is **not viable** (no Vulkan support in Windows ARM VMs).

---

## X4: Foundations — Technical Background

### System Requirements (Official)

| Tier | CPU | RAM | GPU | OS | Storage |
|------|-----|-----|-----|----|---------|
| **Minimum** | Intel i5-4590 3.3GHz / AMD equiv. | 8 GB | Nvidia GTX 780/970 or AMD equiv. (Vulkan required) | Windows 10 64-bit | 12 GB |
| **Recommended** | Intel Core i7-6700 / AMD equiv. | 16 GB | Nvidia GTX 1070 or AMD equiv. (Vulkan required) | Windows 10 64-bit | 12 GB |

**Additional notes from Egosoft:**
- Requires at least **3 GB of free dedicated GPU memory** — integrated GPUs are explicitly not supported
- Vulkan API support is mandatory (no DirectX fallback)
- 12 GB disk space (SSD strongly recommended for load times)

### Headless / Dedicated Server Mode

**❌ Does not exist.** X4: Foundations has no dedicated server binary. The game runs as a full client with rendering, audio, and UI. There is no official headless mode, no server-only binary, and no configuration option to disable rendering.

**Implications for x4-coop:**
- The "server" must run a full X4 client instance
- A GPU is mandatory (even if nobody is looking at the screen)
- The window can be minimized or run at lowest settings, but the render loop still runs
- Remote management requires RDP/VNC/Steam Remote Play — no SSH-only management

### Performance Characteristics

**X4 is heavily CPU-bound** for simulation:
- The economy simulator (thousands of ships, stations, trade routes) is the primary bottleneck
- The renderer waits for the simulation to compute each frame — they are **NOT decoupled**
- Even high-end machines (RTX 4090, modern CPU) get 20 FPS dips in busy sectors with SETA
- The Steam community describes it as: *"The graphics renderer waits for the economy simulator to draw a frame. This is stupid and should have been decoupled years ago."*
- Opening the map menu causes severe FPS drops because the sim + map rendering both hit CPU hard

**GPU requirements:**
- A GPU is required for Vulkan rendering, but it does not need to be powerful
- The simulation is CPU-bound, so a low-end dedicated GPU (GTX 1050/1650) would suffice if paired with a decent CPU
- The GPU must support Vulkan and have ≥3GB VRAM
- Integrated GPUs (Intel UHD, etc.) are explicitly not supported

**SETA (Singularity Engine Time Accelerator):**
- 6x speed multiplier — increases CPU load significantly
- A server running SETA continuously will need a strong CPU to maintain acceptable sim tick rate

### Platform Support

| Platform | Support | Notes |
|----------|---------|-------|
| **Windows x64** | ✅ Native | Primary platform, best supported |
| **Linux x64** | ✅ Native | Egosoft provides a native Linux build! Runs well per ProtonDB reports |
| **macOS** | ❌ Not supported | No native Mac build |
| **Windows ARM** | ❌ Not supported | No ARM64 build; x64 emulation + Vulkan translation = non-viable |
| **Steam Deck** | ✅ Works | Linux native version runs well on Steam Deck |

---

## Option 1: Windows VM on Mac Mini (Parallels/VMware/UTM)

**Hardware:** M4 Mac Mini, 24GB RAM

### Verdict: ❌ NOT VIABLE

**Why it fails:**
1. **No Vulkan support in Parallels:** Parallels Desktop on Apple Silicon does not support the Vulkan API in its Windows ARM video driver. Multiple users report X4 fails to launch: *"it won't launch, seems parallels desktop does not support Vulkan"* (Reddit, July 2023)
2. **No ARM64 build of X4:** The game is x64-only. Running it requires x64 emulation through Windows ARM, adding significant overhead
3. **Parallels forum confirms:** *"Vulkan API support in Parallels Windows ARM video driver is required to support any Vulcan based game on Windows ARM VMs"* — this has been requested since 2022 and is still not implemented
4. **Multiple translation layers needed:** x64 emulation (Windows ARM → x64) + Vulkan translation (if it existed) + Parallels GPU virtualization = performance would be abysmal even if it worked

**UTM/QEMU alternative:**
- UTM can run full x86 Windows emulation, but without GPU acceleration for Vulkan
- Performance would be a fraction of native — X4 already struggles on real hardware
- Not practical for 24/7 server operation

**VMware Fusion:**
- Similar limitations — no Vulkan support in Windows ARM guest
- VMware Fusion Pro is now free from Broadcom, but the GPU virtualization gap remains

**Conclusion:** The fundamental blocker is Vulkan API support in ARM Windows VMs. Until Parallels/VMware implement Vulkan in their Windows ARM drivers, this path is closed. Even if they did, the double emulation (ARM→x64 + API translation) would likely produce unplayable performance.

---

## Option 2: Dedicated Windows PC (Build/Buy)

### Verdict: ✅ RECOMMENDED — Best cost/performance for 24/7 operation

### Minimum Specs for Headless X4 Server

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| **CPU** | Intel i5-4590 (4c/4t) | Intel i5-10400 / Ryzen 5 3600 (6c/12t) | CPU is the primary bottleneck for X4 simulation; more cores = better sim tick rate |
| **RAM** | 8 GB | 16 GB DDR4 | 8 GB is the bare minimum; 16 GB recommended for large universes |
| **GPU** | GTX 1050 (2GB VRAM) | GTX 1650 / GTX 1050 Ti (4GB VRAM) | Needs Vulkan + 3GB VRAM. GTX 1050 Ti 4GB is the cheapest safe option |
| **Storage** | 120 GB SSD | 500 GB NVMe SSD | SSD is critical for X4 load times and save operations |
| **PSU** | 300W | 400W 80+ Bronze | Low power draw with budget GPU |
| **Case** | Any | Small form factor | For headless operation, size/cooling matters more than looks |

### Cost Estimates

**Option A: Refurbished Office PC + GPU (~$200–350)**
- Refurbished Dell OptiPlex / HP EliteDesk (i5-6500, 8GB, 128GB SSD): ~$100–150
- Used GTX 1050 Ti 4GB: ~$50–80
- RAM upgrade to 16GB: ~$20–30
- Total: **~$170–260**

**Option B: Budget New Build (~$350–500)**
- CPU: Intel i3-12100F or Ryzen 5 4500 (~$80–100)
- Motherboard: H610 or A520 (~$70)
- RAM: 16GB DDR4 (~$35)
- GPU: GTX 1650 or RX 6400 (~$100–130)
- SSD: 500GB NVMe (~$35)
- PSU + Case: (~$60–80)
- Total: **~$380–475**

**Option C: Used Gaming PC (~$300–500)**
- Look for used systems with GTX 1060/1660 on eBay/Facebook Marketplace
- Often includes Windows license
- Ready to go with minimal setup

### Pros
- **Lowest total cost** for 24/7 operation (one-time purchase)
- **Full performance** — no virtualization overhead
- **Reliable** — dedicated hardware, no shared resources
- **Easy remote management** via RDP, Chrome Remote Desktop, or AnyDesk
- **Auto-restart** via Windows Task Scheduler or a simple script
- **Low power consumption** — budget build draws ~100–150W under load

### Cons
- **Upfront cost** — $200–500 initial investment
- **Physical space** — needs a spot with power and Ethernet
- **Maintenance** — Windows updates, driver updates, hardware failures
- **Noisy** — even a quiet fan is audible; can be mitigated with SFF build
- **Cooling** — needs decent airflow for 24/7 operation

### Setup Complexity: Low
1. Install Windows 10/11
2. Install Steam + X4: Foundations
3. Configure X4 for lowest graphics settings
4. Set up RDP for remote access
5. Create auto-start script (Windows Task Scheduler → launch Steam → launch X4)
6. Install mods via Steam Workshop

### Auto-Restart on Crash
- **Windows Task Scheduler:** Can be configured to restart Steam/X4 if the process dies
- **PowerShell script:** Check for `X4.exe` process, relaunch if missing
- **Steam launcher:** Can be configured to auto-launch X4 on Steam startup
- **External monitor:** A Raspberry Pi or similar could ping the server and trigger restart

---

## Option 3: Cloud Windows VM with GPU

### Verdict: ⚠️ Possible but expensive — only viable short-term or for testing

### Cost Comparison (24/7 Monthly)

| Provider | Instance | GPU | vCPUs | RAM | On-Demand $/hr | Monthly (730hr) | Notes |
|----------|----------|-----|-------|-----|----------------|-----------------|-------|
| **AWS EC2** | g4dn.xlarge | NVIDIA T4 (16GB) | 4 | 16 GB | $0.526 | **~$384** | Windows adds ~$0.06/hr |
| **AWS EC2** | g4dn.2xlarge | NVIDIA T4 (16GB) | 8 | 32 GB | $0.752 | **~$549** | Better CPU for sim |
| **AWS EC2** | g4dn.xlarge (Spot) | NVIDIA T4 | 4 | 16 GB | $0.289 | **~$211** | Not reliable for 24/7 (can be terminated) |
| **AWS EC2** | g4dn.xlarge (1yr Reserved) | NVIDIA T4 | 4 | 16 GB | $0.331 | **~$242** | Requires 1-year commitment |
| **Vultr** | L40S 1gpu | NVIDIA L40S (48GB) | 16 | 180 GB | $1.67 | **~$1,219** | Overkill; no cheaper T4 option |
| **Vultr** | A40 1gpu | NVIDIA A40 (48GB) | 24 | 120 GB | $1.71 | **~$1,248** | Overkill |
| **Azure** | NV4as_v4 (est.) | NVIDIA T4 (16GB) | 4 | 14 GB | ~$0.45 | **~$328** | Estimated; varies by region |
| **Paperspace/DO** | Various | Various | — | — | from $0.51 | **~$370+** | Windows VMs available |
| **Shadow PC** | Base tier | RTX equivalent | — | — | — | **~$38** | Consumer cloud gaming, not a server |

### Provider Notes

**AWS EC2 g4dn (T4 GPU):**
- The most affordable cloud GPU option for 24/7
- T4 supports Vulkan (required for X4)
- 1-Year Reserved drops to ~$242/month
- Spot pricing (~$211/month) is tempting but instances can be terminated at any time — not suitable for a persistent server
- Windows licensing adds ~$0.06/hr (~$44/month) on top of Linux pricing

**Vultr:**
- GPU instances are aimed at AI/ML workloads — very expensive for gaming
- No budget T4-class GPU option
- Cheapest GPU plan (L40S) is ~$1,219/month — not practical

**Azure NV-series:**
- T4-based VMs available
- Similar pricing to AWS
- Azure Hybrid Benefit can reduce Windows licensing cost if you have existing Windows Server licenses

**Shadow PC:**
- At ~$38/month, this is the cheapest "cloud Windows PC with GPU"
- **However:** Shadow is a consumer cloud gaming service, not a server
- Sessions may time out, not designed for 24/7 unattended operation
- No RDP access — you use their client app
- **Not suitable** for a persistent server, but could be used for development/testing

**Hetzner:**
- GEX44 dedicated GPU server: ~€184/month (~$211) — but this is **Linux only**
- No Windows GPU servers at competitive prices
- If X4 runs on Linux natively (it does!), this becomes interesting (see Option 4)

### Pros
- **No upfront hardware cost**
- **No physical space/power/cooling needed**
- **Scalable** — can upgrade instance type
- **Professional infrastructure** — UPS, redundant networking

### Cons
- **Very expensive for 24/7** — $380+/month is $4,560+/year
- **Network latency** — depends on datacenter location
- **No persistent local state** — need EBS/storage for saves
- **Windows licensing** adds $30–50/month
- **GPU may be overprovisioned** on shared instances

### Recommendation
Cloud GPU VMs are only worth it for:
- Short-term testing/development
- Temporary events (multi-day sessions)
- Prototyping the x4-coop mod before investing in hardware

For 24/7 persistent operation, a dedicated PC pays for itself in **1–2 months** vs. cloud.

---

## Option 4: Linux + Native X4 (Wine/Proton not needed!)

### Verdict: ✅ HIGHLY RECOMMENDED — Best option if comfortable with Linux

### Key Discovery: X4 Has Native Linux Support!

X4: Foundations has a **native Linux build** available on Steam. This is a major advantage — no Wine/Proton translation layer needed.

**ProtonDB reports:** Users report the native Linux version runs well, even on Steam Deck. One user: *"X4 has native Linux support and the Linux version runs much better and more stable on the steam deck."*

**GamingOnLinux (Feb 2026):** Egosoft is actively improving Linux support, including cross-platform cloud saves and plans to use the native Linux version on Steam Deck by default.

### What This Enables

A **Linux dedicated server with a GPU** opens up much cheaper options:

**Option A: Hetzner GEX44 Dedicated Server (~$211/month)**
- NVIDIA GPU + dedicated hardware
- Full root access
- 24/7 operation in a datacenter
- But still $211/month — cheaper than AWS but more than a local PC

**Option B: Linux Mini PC + GPU (~$300–450 upfront)**
- Any Linux box with a Vulkan-capable GPU
- Same hardware as Option 2 but running Linux instead of Windows
- No Windows licensing cost
- SSH access for remote management
- X4 native Linux build + SteamCMD for headless-ish operation

**Option C: Linux Cloud VPS + GPU passthrough**
- Some providers offer GPU VPS instances
- Still expensive but cheaper without Windows licensing

### Practical Linux Setup

```
# Install SteamCMD
sudo apt install steamcmd

# Install X4 via SteamCMD (native Linux build)
steamcmd +login <account> +app_update 392160 +quit

# Run X4 with lowest settings, virtual display
# Use Xvfb or a headless X server for rendering without a monitor
```

### Challenges with Linux
- **SteamCMD + X4:** X4 is a full GUI game — running it truly headless requires a virtual framebuffer (Xvfb) or a dummy display plug
- **Vulkan on Linux:** Requires proper GPU drivers (NVIDIA proprietary or Mesa/AMD)
- **Steam integration:** Mods via Steam Workshop work on Linux, but SteamCMD has some limitations
- **Remote management:** SSH + VNC/xrdp for GUI access
- **Auto-restart:** systemd service or cron-based monitoring

### Pros
- **No Windows licensing cost** (save ~$100+)
- **Native Linux build** = no translation overhead
- **Better remote management** via SSH
- **Lower resource overhead** (no Windows bloat)
- **More stable for 24/7** (Linux servers are proven for always-on workloads)

### Cons
- **More complex setup** — virtual display, GPU drivers, SteamCMD quirks
- **Less community support** for Linux X4 issues
- **GUI game in headless environment** requires workarounds (Xvfb)
- **Mod compatibility** — most mods work on Linux but some may have Windows-specific dependencies

---

## Option 5: Existing Hardware

### Chris's Known Hardware

| Machine | Specs | X4 Viability |
|---------|-------|--------------|
| **M4 Mac Mini** | M4 ARM, 24GB RAM | ❌ Not viable (no Vulkan in ARM Windows VMs, no native Mac build) |
| **Other machines** | Unknown | Needs investigation |

**Action item:** Check if Chris has any other Windows/Linux PCs with dedicated GPUs. Even an older gaming laptop could serve as a temporary server.

### GeForce NOW as a Temporary Solution
- X4 is available on GeForce NOW (confirmed by Reddit users)
- Could be used for development/testing without any local hardware
- Not suitable for 24/7 server operation (sessions time out)
- But could be useful for testing the x4-coop mod during development

---

## Comparison Matrix

| Factor | Mac Mini VM | Dedicated Windows PC | Cloud GPU VM | Linux PC (Native X4) | Existing Hardware |
|--------|-------------|---------------------|--------------|---------------------|-------------------|
| **Upfront Cost** | $0 | $200–500 | $0 | $200–450 | $0 |
| **Monthly Cost** | $0 | $5–10 (power) | $380–550 | $5–10 (power) | $5–10 (power) |
| **Performance** | ❌ Won't run | ✅ Full | ✅ Good (T4) | ✅ Full | Varies |
| **Reliability** | N/A | ✅ High | ⚠️ Medium (cloud) | ✅ High | Varies |
| **24/7 Suitability** | ❌ | ✅ Yes | ⚠️ Yes but costly | ✅ Yes | Varies |
| **Setup Complexity** | N/A | 🟢 Low | 🟡 Medium | 🟡 Medium-High | Varies |
| **Remote Management** | N/A | RDP/AnyDesk | RDP | SSH + VNC | Varies |
| **Auto-Restart** | N/A | Task Scheduler | Cloud auto-restart | systemd/cron | Varies |
| **Vulkan Support** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | Varies |

---

## Recommendations

### Tier 1: Best Option — Dedicated Windows PC ($200–500 upfront)
- **Cheapest path to 24/7 operation**
- Lowest setup complexity
- Best community support (most X4 users are on Windows)
- Steam Workshop mod management works seamlessly
- RDP for remote access is built into Windows
- Auto-restart via Task Scheduler is trivial

### Tier 2: Best "No Windows" Option — Linux PC with GPU ($200–450 upfront)
- Same hardware cost, no Windows license needed
- Native X4 Linux build performs well
- Better for always-on server workloads
- More complex setup (virtual display, SteamCMD)
- SSH access is superior to RDP for server management

### Tier 3: Best "No Hardware" Option — Cloud GPU VM ($380+/month)
- Only viable for short-term testing or development
- AWS g4dn.xlarge with 1-Year Reserved (~$242/month) is cheapest
- Consider for prototyping before buying hardware
- Shadow PC ($38/month) could work for development but not 24/7 production

### Tier 4: Not Viable — Mac Mini VM
- Blocked by lack of Vulkan support in Parallels/VMware ARM Windows drivers
- No ARM64 build of X4 exists
- Even if Vulkan were supported, double emulation overhead would be prohibitive

---

## Next Steps

1. **Check if Chris has an existing Windows/Linux PC** with a dedicated GPU that could serve as the server
2. **If not, decide on dedicated PC vs Linux PC** — budget $300–400 for hardware
3. **For immediate testing:** Consider using Shadow PC ($38/month) or GeForce NOW to prototype the x4-coop mod before investing in hardware
4. **For the cooperative server architecture:** The server will run a full X4 client at lowest settings. The x4-coop mod (using Nakama for networking) will handle player connections separately from X4's rendering
5. **Auto-restart infrastructure:** Regardless of platform, implement a watchdog (Task Scheduler on Windows, systemd on Linux) that:
   - Monitors the X4 process
   - Restarts it if it crashes
   - Logs restart events
   - Optionally sends a Discord webhook notification

---

## Research Sources

- [X4 System Requirements (official)](https://gamesystemrequirements.com/game/x4-foundations) — Egosoft published specs
- [ProtonDB — X4: Foundations](https://www.protondb.com/app/392160) — Native Linux support confirmed
- [Reddit — X4 on Parallels](https://www.reddit.com/r/X4Foundations/comments/15avt14/) — "won't launch, Parallels doesn't support Vulkan"
- [Parallels Forum — Vulkan API request](https://forum.parallels.com/threads/356887/) — Vulkan not supported in ARM Windows VMs
- [Reddit — X4 on Parallels M1](https://www.reddit.com/r/X4Foundations/comments/14eg7vp/) — Launch failure confirmed
- [Steam Community — X4 Performance](https://steamcommunity.com/app/392160/discussions/0/3725071921960185002/) — "renderer waits for economy simulator" (not decoupled)
- [Reddit — X4 Performance](https://www.reddit.com/r/X4Foundations/comments/ckufl5/) — CPU-bound simulation confirmed
- [Reddit — X4 Integrated GPU](https://www.reddit.com/r/X4Foundations/comments/olpaue/) — "3 GB free dedicated VRAM required, not for integrated GPUs"
- [GamingOnLinux — X4 Linux support](https://www.gamingonlinux.com/2026/02/x4-foundations-cross-platform-cloud-saves/) — Native Linux build, Steam Deck support coming
- [AWS EC2 g4dn pricing](https://instances.vantage.sh/aws/ec2/g4dn.xlarge) — $0.526/hr on-demand
- [Vultr GPU pricing](https://www.vpsbenchmarks.com/gpu_plans/vultr) — L40S from $1.67/hr
- [Hetzner GPU servers](https://www.hetzner.com/dedicated-root-server/matrix-gpu/) — GEX44 at €184/month
- [Shadow PC pricing](https://shadow.tech/us/shadowpc/offers/) — From $37.99/month
- [GitHub — carrascodev/x4-mods](https://github.com/carrascodev/x4-mods) — Nakama-based multiplayer mod proof of concept
- [MacGamingDB — X4](https://macgamingdb.app/games/392160) — Mac compatibility info