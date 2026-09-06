# Kha'ak Pressure for X4: Foundations

The Kha'ak in X4 are a mining alarm, not an enemy. They watch the sectors within three gate jumps of one of their hives, add up how much ore, silicon and nividium is pulled out of each one, and when a sector's score crosses fifty thousand they drop an outpost on it. Everything about that is a number, and every one of those numbers is tuned for a galaxy where you are not supposed to notice them much.

This mod turns three of those numbers into sliders: how far a hive reaches, how much mining an outpost costs them, and how often mining an asteroid drops a swarm on your miner's head.

**Requires X4: Foundations 9.00.** No DLC required, and no hard dependency on other mods. Works on an existing savegame.

## The sliders

| Slider | Range | Vanilla | What it is |
|---|---|---|---|
| **Hive reach** | 3 – 10 jumps | 3 | `$HiveStationGateRange` — how far from a hive the Kha'ak watch sectors and may place an outpost |
| **Outpost cost** | 10 000 – 50 000 | 50 000 | `$SpawnOutpostThreshold` — the mining activity that triggers a placement. `$MinSpawnOutpostThreshold`, the floor a sector needs to be a candidate, follows at **half** this |
| **Ambush chance multiplier** | 1x – 100x | 1x | scales `$BaseSpawnChance`, the chance that mining an asteroid spawns a Kha'ak group |

### One place where the defaults are not vanilla

The floor is held at half the trigger, which is what was asked for, but vanilla's ratio is 0.8, not 0.5 — it pairs a trigger of 50 000 with a floor of 40 000. So even with both sliders at the top, the pool of sectors an outpost may land in is wider here than in the stock game.

That does not make outposts appear more often: the trigger is what gates a placement, and it is unchanged at 50 000. What it changes is *where* they land. Vanilla picks from the sectors at 40 000 and up, weighted by score; this picks from 25 000 and up. Outposts spread further from whichever sector you are mining hardest instead of concentrating on it.

If you want the ratio back, change one line in `drjele_khaak_pressure.xml`:

```xml
<set_value name="$MinThreshold" exact="($Threshold * 8) / 10"/>
```

### What the numbers mean in play

**Hive reach** is the setting that changes the game most. At 3 the Kha'ak are a local nuisance around eight fixed sectors. At 6 they reach most of the core worlds. At 10 they watch nearly the galaxy, and any sector you mine seriously is a candidate. The cost is CPU: the vanilla evaluation walks every watched sector once a minute, so the top of the slider is measurably more work than the bottom. It is not a large amount either way, but it is not free.

**Outpost cost** is denominated in mining. Because your own miners count double, 50 000 is roughly 20 000 units of silicon, 28 600 of ore or 10 000 of nividium pulled out of a single sector. At 10 000 that is a fifth of it — a couple of full L miner runs.

**Ambush chance multiplier** only does anything in a sector that already holds a Kha'ak outpost; everywhere else vanilla's chance is zero no matter what this says, because the term it multiplies is scaled by a sector activity value that only accumulates while an outpost is alive. Inside such a sector the vanilla chance is 0.03 % per mined unit at full activity, times 1.2 for ore and silicon or 2.7 for nividium. High multipliers saturate on a large haul.

## Install

```bash
./install.sh
```

That finds your X4 installation — the usual Steam layouts including extra library folders — and copies `extension/` into `X4 Foundations/extensions/drjele_khaak_pressure`. If it cannot find the game, or you want a different copy of it, point it there yourself:

```bash
X4_PATH="/path/to/X4 Foundations" ./install.sh
```

**Restart X4** — extensions are only read at startup.

It copies rather than symlinks on purpose: X4 only enumerates real directories under `extensions/`, and silently ignores a symlink placed there. So re-run `./install.sh` after every edit. `./install.sh --uninstall` removes it again.

## How it works

Almost nothing here is a patch, and that is the interesting part.

The Kha'ak numbers all live as variables on a single cue, `Manager` in `md/khaak_activity.xml`, which runs **once** per game on `md.Setup.Start`. That shape rules out the obvious approach. An XPath patch that adds a `set_value` to that cue's actions would only ever fire on a new game — on a save already in progress the cue has long since run, and the patched line never executes. It would also put this mod in the same file as No Kha'ak Torture and Spawn Begone, which is where mods of this kind go to conflict with each other.

So the two applied settings are written straight into the vanilla manager's variables instead:

```xml
<set_value name="md.Khaak_Activity.Manager.$SpawnOutpostThreshold" exact="$Threshold"/>
```

That reaches a running savegame, needs no patch, and cannot collide with another mod's XPath.

The apply step runs on `md.Setup.Start` — signalled on a new game and on every load — but through a child cue that waits for `md.Khaak_Activity.Manager.$HiveStationGateRange` to exist first. Both scripts hang off the same signal and the order between two MD scripts is not defined, so without the wait the settings would land on variables that do not exist yet about half the time. That same existence check doubles as the test for "the Kha'ak are running at all": the vanilla manager only initialises in the main galaxy, so in a Timelines scenario, a tutorial or the workshop map this mod stays completely inert.

**Changing the reach also rebuilds the watched set.** The manager derives `$WatchedSectors` from the gate range, but only ever recomputes it when a Kha'ak station is created or destroyed. Writing the number alone would leave the old set in place until the next hive died. So when the slider moves, the mod recomputes the set the way vanilla's own `StationDestroyed_v2` does — every sector in range of a hive, plus every outpost's own sector — and drops the accumulated activity of the sectors that fall out of range, again as vanilla does. Turning the slider up therefore takes effect on the next evaluation, and turning it back down does not leave a stale score behind for a sector to cash in later.

The third setting does need a patch, and gets one:

| Patch | Change |
|---|---|
| `md/khaak_activity.xml` | Scales `$BaseSpawnChance` right after the vanilla library sets it |

`$BaseSpawnChance` is a local of `GetSuitableBaseStationForShipSpawn`, assigned fresh on every call, so a value written into it from outside is overwritten before anything reads it. Two lines inserted immediately after vanilla's assignment is the only place the multiplier can go. The selector matches exactly one node in the 9.00 file.

Every setting is read defensively — the global the options menu writes, else the table the configuration cue publishes, else the vanilla literal — so a configuration that fails to load degrades to stock Kha'ak behaviour rather than breaking them.

## Configuring without the menu

The in-game sliders need [SirNukes Mod Support APIs](https://steamcommunity.com/sharedfiles/filedetails/?id=2042901274); without it the mod runs off the constants at the top of `extension/md/drjele_khaak_pressure.xml`, which are re-read on every savegame load. Editing one and reloading is enough — no new game.

Each can also be overridden at runtime without touching the file:

```xml
<set_value name="global.$DrJeleKhaakHiveGateRange" exact="6"/>
<set_value name="global.$DrJeleKhaakOutpostThreshold" exact="20000"/>
<set_value name="global.$DrJeleKhaakSpawnChanceFactor" exact="10"/>
```

Set `$DebugChance` to 100 in the configuration cue to have every applied change written to the debug log.

## Removing it

Take the extension out and the sliders stop being applied — but the values already written into the vanilla manager stay in the savegame, because they are the manager's own variables. They are reset the next time you start a new game, and can be put back by hand at any time from the in-game script editor or another mod. Nothing else of this mod persists; it adds no cues to the Kha'ak manager and stores no state of its own.

## Publishing to the Steam Workshop

Egosoft does not publish from inside the game. Uploads go through `WorkshopTool`, shipped in the **X Tools** package — Steam app 282160, `steam://install/282160`. Steam has to be running and logged in with an account that owns X4.

```bash
./publish.sh publish                    # first upload
./publish.sh update "what changed"      # every upload after that
```

`extension/preview.jpg` is required before the first upload and is not in the repository yet.

`X_TOOLS_PATH` and `PROTON_PATH` override the automatic lookup, the same way `X4_PATH` does. The script keeps the Workshop id out of the manual-install `content.xml` so a local copy does not collide with a subscription; see the other drjele X4 mods for the long version of why.

## Status

The single XPath selector and its cardinality are verified against the installed X4 9.00 game files, and the merged XML is structurally validated. The behaviour of the two written settings is derived from reading `md/khaak_activity.xml`; it has not yet been observed over a long game.

## Legal

MIT, see [`LICENSE`](LICENSE). Non-commercial fan project; X4: Foundations belongs to Egosoft GmbH and this project is not affiliated with or endorsed by Egosoft.
