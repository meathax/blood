# NBlood for MiSTer

A native MiSTer port of **NBlood** — the actively-maintained source port of **Blood** (Monolith/Nightdive, 1997) built on Ken Silverman's BUILD engine — for the [DE10-Nano](https://mister-devel.github.io/MkDocs_MiSTer/).

NBlood's classic software renderer runs as ARM software on MiSTer's HPS. The FPGA handles everything that needs to feel like a real MiSTer core: native video scanout, audio presentation (including a hardware OPL3 synth), controller/keyboard/mouse capture, and the OSD. This is **not** an FPGA recreation of BUILD, and not an x86/DOS emulator — the game engine itself is real, compiled NBlood source.

> **Preview release:** this repository is actively developed. The release payload is complete and copy-ready, but game-specific hardware acceptance remains a work in progress.

No Blood game data is included. You need your own legally obtained copy — this port targets the GOG **Blood: Fresh Supply** data layout.

---

## Install guide

**You need:**
- A MiSTer (DE10-Nano) with a working SD card
- Your own copy of **Blood: Fresh Supply** (e.g. from GOG)

**Steps:**

1. Copy the contents of this repo's [`releases/`](releases/) folder to the root of your MiSTer SD card. This places:
   - `/media/fat/Mister_NBlood` — the core's launcher
   - `/media/fat/_Computer/NBlood.rbf` — the FPGA core
   - `/media/fat/games/NBlood/NBlood` — the game engine
2. Copy your own Blood data into `/media/fat/games/NBlood/` (same folder as the `NBlood` engine above). From a Fresh Supply install this is everything at the top level plus the `movie/` folder — `BLOOD.RFF`, `blood.ini`, `sounds.rff`, `tiles*.art`, `blood0*.ogg`, etc. Cryptic Passage's `addons/Cryptic Passage/` files go in the **same** `games/NBlood/` folder too (flattened, not in a subfolder) — copy its patched `tiles007.ART`/`tiles015.ART` over the base versions last.
3. Add this to `/media/fat/MiSTer.ini` (create the file if it doesn't already exist):
   ```ini
   [NBlood]
   main=Mister_NBlood

   [Mister_NBlood]
   main=Mister_NBlood
   ```
4. Reboot, or rescan/reload the menu, then start **NBlood** from MiSTer's `_Computer` menu.

That's it — no separate install script, no daemon to start. `Mister_NBlood` creates `games/NBlood/Saves/` for saves and per-user config on first launch; keep that folder when updating.

**Transferring the binaries:** `Mister_NBlood` and `games/NBlood/NBlood` are extensionless ARM executables. If you copy them with an FTP client, use **binary** transfer mode — text/ASCII mode corrupts them.

### Why the `mister.ini` line

Normally, picking a core's `.rbf` from the MiSTer menu leaves the generic `MiSTer` program running your whole session — it drives the OSD, reads your controller, and so on, the same way for every core. NBlood needs more than that (it also has to launch and supervise a full ARM game engine), so `mister.ini`'s `main=` line tells MiSTer: once `NBlood.rbf` is loaded, hand control to `Mister_NBlood` instead of continuing as itself. `Mister_NBlood` then takes over the OSD/controller job *and* launches the game engine. See [How it works](#how-it-works) below and `docs/MISTER_MAIN_WRAPPER.md` (local developer notes, not part of this published tree) for the full mechanics.

### Automatic installation (downloader)

When NBlood is included in the Meathax downloader bundle, add this to `/media/fat/downloader.ini` and run **Update All**:

```ini
[meathax/meatcores]
db_url = https://raw.githubusercontent.com/meathax/meatcores/db/db.json.zip
```

Until that payload is published, copying `releases/` by hand (above) is the canonical install method.

---

## How it works

```text
NBlood classic software renderer (ARM, HPS)
        │ indexed frame + palette / PCM / OPL3 register writes / input
        ▼
shared DDR3 transport (memory-mapped register + ring-buffer protocol)
        ▼
FPGA fabric (Cyclone V SoC)
        ├── native video scanout (raster generation, palette, scaling)
        ├── PCM audio output + hardware OPL3 synth
        └── hps_io: controller/keyboard/mouse capture, OSD
```

The ARM renderer publishes completed frames through shared DDR3; the FPGA displays only a completed buffer at a frame boundary. Audio and input each use their own explicit ring/snapshot contract over the same shared window. This keeps the port close to a real MiSTer core's feel without putting Build-engine game logic into FPGA fabric — the DDR3 protocol is the only thing crossing the ARM/FPGA boundary.

### What runs on the ARM (HPS, dual-core Cortex-A9)

| Process | Role |
| --- | --- |
| `Mister_NBlood` | The core's actual MiSTer "front end" — a small [Main_MiSTer](https://github.com/MiSTer-devel/Main_MiSTer) derivative (see [Recent changes](#recent-changes) below). Owns hps_io service (OSD, controller/keyboard/mouse) for the session, launches and supervises the game engine below. |
| `NBlood` | The real, compiled NBlood/BUILD engine — game logic, classic renderer, level/save/config handling — talking to the FPGA only through the shared DDR3 window. |

### What's on the FPGA (Cyclone V SoC, DE10-Nano)

| Function | Implementation |
| --- | --- |
| Video | Custom reader FSM streams the ARM's published indexed-8 frame + palette out of DDR3 and drives native raster timing — 320×240 at 15.72 kHz (15 kHz CRT-compatible) or 640×480 at 33.56 kHz, selectable in the OSD |
| Audio (PCM) | 48 kHz signed 16-bit stereo ring, read out of DDR3 and mixed to MiSTer's audio path |
| Audio (OPL3 music) | A real hardware OPL3/YMF262 core ([`opl3_fpga`](https://github.com/gtaylormb/opl3_fpga) by Greg Taylor, LGPL-3.0) driven by register writes the ARM sends over the DDR3 window — not software FM emulation |
| Input / OSD | Standard MiSTer `hps_io`, feeding a shared-memory joystick/keyboard/mouse/OSD-status snapshot the ARM engine reads every frame |

Current build (`fpga/`, Quartus 17.0.2.602): **11,315 / 41,910 ALMs (27%)**, 18,922 registers, 481,294 / 5,662,720 block memory bits (8%), 34 / 112 DSP blocks (30%), 3 / 6 PLLs.

---

## Recent changes

This core used to need a background daemon (`MiSTer_Frontier`/`Master_Daemon.sh`, borrowed from the community OpenBOR/PICO-8 "hybrid core" pattern) polling for when its RBF loaded, to launch the ARM engine. **That daemon is gone.** In its place:

- **`platform/mister/wrapper/` — `Mister_NBlood`.** A genuine [Main_MiSTer](https://github.com/MiSTer-devel/Main_MiSTer) derivative, selected via the native `mister.ini` `main=` core-exception mechanism (the same mechanism [Mister_Duke3d](https://github.com/neofreno/Mister_Duke3d) uses). It takes over the OSD/controller role for the session and launches the game engine itself — no polling, no separate install script, no `_handler.sh`. See `docs/MISTER_MAIN_WRAPPER.md` (local developer notes) for the full mechanism and provenance.
- **OPL3 music no longer stutters.** The FPGA only serviced OPL3 register-write handshakes once per video frame (~16.7 ms) — far too coarse for a real-time synth bus, so note bursts could stall the ARM for hundreds of milliseconds at a time, dropping frames and producing a stuck-note tone. The reader FSM (`fpga/rtl/openbor_video_reader.sv`) now also services that handshake once per raster line (240–480× per frame instead of once), using DDR-port time that state already had idle. Verified against the project's Verilator per-line timing testbenches and confirmed on real hardware — spin dropped from ~44–99% of wall clock to under 1%, and OPL3 writes/s hold steady instead of decaying to zero.

---

## Features in the OSD

Press **F12** or the MiSTer menu button.

| Page | Setting | Choices |
| --- | --- | --- |
| Video | Aspect ratio | Original, Full Screen, ARC1, ARC2 |
| Video | Scale | Normal, V-Integer, Narrower/Wider HV-Integer, HV-Integer |
| Video | Resolution | 320×240 15 kHz, 640×480 31 kHz |
| Video | HUD size | Full, Compact, Minimal, Off |
| Controls | Autorun | On, Off |
| Controls | Crosshair | Off, On |
| Controls | Aim sensitivity | Neutral, 20 finer steps, 20 coarser steps |
| Controls | Aim curve | Linear, Exponential |
| System | Reset | Restart the core |

The gamepad exposes Fire, Alt Fire, Use, Jump, Crouch, weapon selection, Map, inventory, and Menu. Both analogue sticks, keyboard events, and relative mouse movement are forwarded to NBlood.

## Supported games

| Game | Status |
| --- | --- |
| Blood | Supported target. A user-owned game-data install is required. |

Cryptic Passage and other optional content follow the normal NBlood game-data convention (see the install guide above); no commercial content is shipped here.

## PCB accuracy

| Area | Evidence |
| --- | --- |
| Original Blood arcade hardware | Not applicable. This project is an ARM software port with an FPGA presentation bridge, and does not claim to reproduce an original Blood PCB — Blood never had one. |

## Building from source

The repository tracks source, build tooling, licence notices, and the ready-to-copy `releases/` payload. Local dependencies, build products, reports, captures, evidence, and agent/workspace files are ignored.

```bash
make smoke
make fetch
make donor
make prepare
make integrate
make arm
make wrapper
make tools-arm
make fpga
make package
```

`make package` refreshes `releases/` directly and also writes a convenience archive under ignored `dist/`. A package must contain matching ARM, wrapper, and FPGA outputs; do not combine parts from different builds.

## Repository layout

```text
fpga/                    MiSTer Quartus project, RTL, constraints, and vendored framework source
  rtl/                     Core RTL: video reader/timing, input, OPL3 bridge, vendored opl3_fpga
  sim/                     Verilator testbenches for the DDR3 transport, video timing, and OPL3 bridge
integration/             NBlood audio/platform integration source (build-time patches applied to upstream NBlood)
platform/                ARM-to-FPGA bridge and kernel-module source
  mister/                  Shared-DDR3 transport, video/audio/input glue (frontier_*.c, mister_*.c)
  mister/kmod/             Optional write-combining kernel module for the DDR3 window
  mister/wrapper/          Mister_NBlood: the mister.ini `main=` core-exception HPS launcher (see Recent changes)
scripts/                 Fetch, integration, build, package, and deployment scripts
tools/                   Host checks and MiSTer diagnostic source
releases/                Copy-ready /media/fat payload: Mister_NBlood, _Computer/NBlood.rbf, games/NBlood/NBlood
```

## Credits

- [NBlood](https://github.com/nukeykt/NBlood) — Blood source port and BUILD runtime.
- [MiSTer_OpenBOR_7533](https://github.com/MiSTerOrganize/MiSTer_OpenBOR_7533) — Frontier donor used for the ARM/FPGA bridge direction.
- [opl3_fpga](https://github.com/gtaylormb/opl3_fpga) by Greg Taylor — LGPL-3.0 SystemVerilog OPL3/YMF262 implementation.
- [3s-mister-arm](https://github.com/kimchiman52/3s-mister-arm) and [sonic-mania-mister](https://github.com/kimchiman52/sonic-mania-mister) — MiSTer ARM + FPGA scanout references.
- [Mister_Duke3d](https://github.com/neofreno/Mister_Duke3d) — `mister.ini` `main=` core-exception wrapper pattern donor for `platform/mister/wrapper/`.
- [MiSTer FPGA](https://github.com/MiSTer-devel/Main_MiSTer) — platform framework and core conventions, and original upstream for `platform/mister/wrapper/`.

## License

The project source is GPL-2.0-or-later. Imported components retain their own notices:

- `fpga/rtl/opl3_fpga/` is LGPL-3.0 and includes its licence text.
- `platform/mister/wrapper/` is GPL-3.0-only and includes its own `LICENSE` copy.
- MiSTer framework and Frontier components retain their upstream licences.
- Blood game data is not part of this project and remains subject to its owners' terms.
