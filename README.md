# NBlood MiSTer — Downloader Database

This repository is a [MiSTer Downloader](https://github.com/MiSTer-devel/Downloader_MiSTer) custom database. It distributes the ready-to-run **NBlood for MiSTer** core files, so `downloader`/**Update All** can install and keep them up to date automatically — no manual file copying required.

Source and development happen in [meathax/nblood](https://github.com/meathax/nblood); this repo just publishes that project's build output.

## What NBlood is

A native MiSTer port of **Blood** running on **NBlood**, the actively-maintained BUILD-engine source port. The classic game engine runs as ARM software on the DE10-Nano's HPS; the FPGA handles native video scanout, audio (including a real hardware OPL3 synth), controller/keyboard/mouse capture, and the OSD. See [meathax/nblood](https://github.com/meathax/nblood) for full architecture details, hardware notes, and source.

No Blood game data is distributed here or anywhere in this project — you need your own legally obtained copy (this port targets the GOG **Blood: Fresh Supply** data layout).

## What this repository contains

| File | Purpose |
| --- | --- |
| `Mister_NBlood` | The core's launcher (ARM). Selected via `mister.ini`'s `main=` core exception; owns the OSD/controller session and starts the game engine. |
| `_Computer/NBlood.rbf` | The FPGA core image. |
| `games/NBlood/NBlood` | The NBlood game engine (ARM). |
| `games/NBlood/README_DATA.md` | What game data to supply and where. |

## Installing via the Downloader

Add this to `/media/fat/downloader.ini`:

```ini
[meathax/blood]
db_url = https://raw.githubusercontent.com/meathax/blood/db/db.json.zip
```

Then run **Update All** (or `downloader`). This installs/updates the three files above.

**One manual step is still required** — the Downloader can copy files, but it can't edit `MiSTer.ini` for you. Add this once, yourself:

```ini
[NBlood]
main=Mister_NBlood

[Mister_NBlood]
main=Mister_NBlood
```

Then supply your own Blood game data in `/media/fat/games/NBlood/` (see `games/NBlood/README_DATA.md`), and start **NBlood** from MiSTer's `_Computer` menu.

## License

`Mister_NBlood` is GPL-3.0-only; the NBlood engine and FPGA core are GPL-2.0-or-later with imported components under their own notices. See [meathax/nblood](https://github.com/meathax/nblood)'s README for the full breakdown. Blood game data is not part of this project and remains subject to its owners' terms.
