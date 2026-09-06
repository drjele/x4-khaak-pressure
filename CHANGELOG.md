# Changelog

All notable changes to this project will be documented in this file.

## [v1.0.0] - 2026-09-06 - Initial release

### Added

- **Hive reach** slider, 3 to 10 gate jumps, writing `$HiveStationGateRange` on the vanilla Kha'ak manager and rebuilding `$WatchedSectors` so the change takes effect without waiting for a station to be created or destroyed.
- **Outpost cost** slider, 10 000 to 50 000 mining activity, writing `$SpawnOutpostThreshold`, with `$MinSpawnOutpostThreshold` held at half of it.
- **Ambush chance multiplier** slider, 1x to 100x, scaling `$BaseSpawnChance` through a single XPath patch on `md/khaak_activity.xml`.
- In-game options menu through SirNukes Mod Support APIs, with file constants and global overrides as the fallback when it is not installed.
- Existing-save support: settings are written into the vanilla manager's own variables rather than patched into a cue that has already run.

### Notes

- `$MinSpawnOutpostThreshold` is held at half the trigger, where vanilla's ratio is 0.8. At the top of the slider outposts therefore spread wider than in the stock game, though not more often. See the README.
- `extension/preview.jpg` is not in the repository yet and is required before the first Workshop upload.

[v1.0.0]: https://github.com/drjele/x4-khaak-pressure/releases/tag/v1.0.0
