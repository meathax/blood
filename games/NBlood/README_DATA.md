# Game data

No Blood game data is distributed here.

Place legally obtained Blood data in `/media/fat/games/NBlood/` (or a subdirectory chosen by the final launcher). Upstream NBlood documentation is authoritative for the exact required files for the revision you build.

Typical original Blood data includes `BLOOD.RFF`, `BLOOD.INI`, art/resource files, sound resources, and optional Cryptic Passage files. Do not copy these into source control or release packages.

The supplied handler keeps game data in that directory and stores persistent
state in `/media/fat/games/NBlood/Saves/`:

- `game00xx.sav` — Blood's native manual and quick-save slots;
- `blood.cfg`, `blood_cvars.cfg`, and `blood_settings.cfg` — controller
  bindings, stick sensitivity/dead-zone/inversion settings, and other normal
  Blood configuration.

On first launch the handler non-destructively copies compatible legacy saves
and configuration found in the game-data root into `Saves/`, but never replaces
an existing file there. Keep the `Saves/` directory when updating the core or
game binary.

The NBlood release also contains the pinned MiSTer Frontier `Master_Daemon.sh`
and its installer. Copy the release's inner folders to `/media/fat/`, then run
`/media/fat/Scripts/Install_MiSTer_Frontier.sh` once. The daemon discovers this
folder's `_handler.sh` automatically when the `NBlood` RBF is loaded.

There is no Audio OSD page. Music is always on and always OPL3 emulation,
which measures roughly 60% of one ARM core during playback (see
`docs/PERFORMANCE.md`); there is no on/off switch, device choice, or
soundfont to configure.
