# Khaak Pressure for X4: Foundations

<p align="center">
  <img src="extension/preview.jpg" alt="Khaak Pressure" width="512">
</p>

The Kha'ak in X4 are a mining alarm, not an enemy. They watch the sectors within three gate jumps of one of their hives, add up how much ore, silicon and nividium is pulled out of each one, and when a sector's score crosses fifty thousand they drop an outpost on it. Everything about that is a number, and every one of those numbers is tuned for a galaxy where you are not supposed to notice them much.

This mod turns three of those numbers into sliders: how far a hive reaches, how much mining an outpost costs them, and how often mining an asteroid drops a swarm on your miner's head.

**Requires X4: Foundations 9.00.** No DLC required, and no hard dependency on other mods. Works on an existing savegame.

## Install

```bash
./install.sh
```

The helper copies `extension/` into the game's `extensions/<extension-id>`
directory, using the id in `extension/content.xml`. It searches the usual Steam layouts and additional library folders. To choose an installation:

```bash
X4_PATH="/path/to/X4 Foundations" ./install.sh
```

Restart X4 after installing or updating. To remove the manual installation:

```bash
./install.sh --uninstall
```

## The sliders

| Slider                       | Range           | Vanilla | What it is                                                                                                                                                                  |
|------------------------------|-----------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Hive reach**               | 3 – 10 jumps    | 3       | `$HiveStationGateRange` — how far from a hive the Kha'ak watch sectors and may place an outpost                                                                             |
| **Outpost cost**             | 10 000 – 50 000 | 50 000  | `$SpawnOutpostThreshold` — the mining activity that triggers a placement. `$MinSpawnOutpostThreshold`, the floor a sector needs to be a candidate, follows at **half** this |
| **Ambush chance multiplier** | 1x – 100x       | 1x      | scales `$BaseSpawnChance`, the chance that mining an asteroid spawns a Kha'ak group                                                                                         |

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

| Patch                   | Change                                                            |
|-------------------------|-------------------------------------------------------------------|
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

## Debugging

Add this to the game's launch options — Steam, right click X4, **Properties → General → Launch Options**:

```
-debug all -logfile debuglog.txt
```

The log lands next to your savegames: `$HOME/.config/EgoSoft/X4/<userid>/debuglog.txt` on Linux, `Documents\Egosoft\X4\<userid>\debuglog.txt` on Windows. If Steam is installed as a snap it runs the game with a redirected home, which puts both under `~/snap/steam/common/`. `all` turns on every one of the engine's debug channels; a narrower filter only makes sense once you know which channel a message uses, and the engine prints `Unknown debug filter` for a name it does not recognise.

The mod itself is silent by default. Set `$DebugChance` to `100` in the configuration cue of `extension/md/drjele_khaak_pressure.xml`, re-run `./install.sh` and restart, and every applied change is written out:

```
DrJele Khaak Pressure: gate range 3 -> 6, outpost threshold 50000 (min 25000)
DrJele Khaak Pressure: watching 214 sectors after rebuild
```

A patch whose XPath finds nothing is reported without any of that, at startup, by file and selector — so `grep -i drjele debuglog.txt` after a start is the quick check that the one patch here landed. Silence means it did.

## Removing it

Take the extension out and the sliders stop being applied — but the values already written into the vanilla manager stay in the savegame, because they are the manager's own variables. They are reset the next time you start a new game, and can be put back at any time by reinstalling this mod with the vanilla numbers in the configuration cue. Nothing else of this mod persists; it adds no cues to the Kha'ak manager and stores no state of its own.

## Status

Working, verified in game on 9.00, on a savegame several hundred hours old — the case the whole design is built around. Settings for the run: reach 5 jumps, outpost cost 10 000, ambush multiplier 50x.

| Check           | Result                                                                                                                              |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Applied on load | the two written settings land on the vanilla manager and are still there after eight hours of uninterrupted play                    |
| Reach slider    | moving it rebuilt the watched set immediately: 108 sectors at 4 jumps, 124 at 5                                                     |
| Outposts placed | three in about eight and a half hours of game time, in Pious Mists XI, Trinity Sanctum III and Wretched Skies X                     |
| Patch           | the XPath selector matches exactly one node, the merged XML is structurally valid, and the game reports no patch error for the file |
| Script errors   | none                                                                                                                                |

What the run does not measure is how much of that rate comes from the lowered cost and how much from the wider reach; both were changed at once. The half-ratio floor is likewise unconfirmed on its own — it changes *where* placements land rather than how many, and separating that from ordinary mining pressure needs a controlled comparison that has not been run.

## Publishing to the Steam Workshop

Install **X Tools** (Steam app 282160) and keep Steam running and logged in with an account that owns X4. On Linux, install Proton as well; on Windows, run the helper from Git Bash, MSYS or Cygwin.

```bash
./publish.sh publish
./publish.sh update "what changed"
```

Use `publish` once, then `update` with a change note. `X4_PATH`,
`X_TOOLS_PATH` and `PROTON_PATH` override automatic discovery. The staging location must contain an `extensions` directory.

The first upload records the numeric id in `steam/workshop-id`; retain that file for future updates. The readable id in the repository's `content.xml`
stays unchanged. After publishing, open the printed Workshop URL, complete any required Steam agreement and choose the item's visibility. Avoid keeping both the manual installation and a subscription to the same mod enabled.

Update the manifest version and release date together with `CHANGELOG.md`
when releasing. See [Development](DEVELOPMENT.md) for staging, platform and release conventions.

## Development

See [DEVELOPMENT.md](DEVELOPMENT.md) for setup, code style, validation and release conventions.

## Legal

MIT, see [`LICENSE`](LICENSE). Non-commercial fan project; X4: Foundations belongs to Egosoft GmbH and this project is not affiliated with or endorsed by Egosoft.
