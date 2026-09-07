# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Changed

- Standardize repository settings, development checks, code style and documentation.
- Detailed Steam Workshop description, covering every setting, where to change it, and the required and optional dependencies.

### Added

- `publish.sh update` options `--minor`, `--namedesc` and `--readback`, for an update that leaves the version number alone, one that also pushes the name and description to Steam, and one that writes Steam's own text back into `content.xml.steam`.

### Fixed

- `publish.sh` passes `-batchmode`, so an upload no longer waits for a keypress a non-interactive run cannot give it.
- `publish.sh` runs the workshop tool inside the Steam snap's mount namespace. The snap has a private `/tmp`, so the Steam client IPC the tool needs is unreachable from outside it and the upload failed on a Steamworks assertion.
- `publish.sh` shows the tool's output, which Proton otherwise discards, and reads success or failure out of it rather than out of an exit code Proton does not pass on.
- `publish.sh` restores the local installation even when the upload fails, instead of leaving it holding the staged copy with the Workshop id in it.
- `publish.sh update` sends the preview image too, so a refreshed `extension/preview.jpg` reaches the Workshop item instead of leaving the one from the first upload in place.

## [v1.0.0] - 2026-09-06 - Initial release

### Added

- **Hive reach** slider, 3 to 10 gate jumps, writing `$HiveStationGateRange` on the vanilla Kha'ak manager and rebuilding `$WatchedSectors` so the change takes effect without waiting for a station to be created or destroyed.
- **Outpost cost** slider, 10 000 to 50 000 mining activity, writing `$SpawnOutpostThreshold`, with `$MinSpawnOutpostThreshold` held at half of it.
- **Ambush chance multiplier** slider, 1x to 100x, scaling `$BaseSpawnChance` through a single XPath patch on `md/khaak_activity.xml`.
- In-game options menu through SirNukes Mod Support APIs, with file constants and global overrides as the fallback when it is not installed.
- Existing-save support: settings are written into the vanilla manager's own variables rather than patched into a cue that has already run.

### Notes

- `$MinSpawnOutpostThreshold` is held at half the trigger, where vanilla's ratio is 0.8. At the top of the slider outposts therefore spread wider than in the stock game, though not more often. See the README.

[v1.0.0]: https://github.com/drjele/x4-khaak-pressure/releases/tag/v1.0.0
