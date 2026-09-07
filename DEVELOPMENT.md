# Development

## Checks and formatting

Use Python 3.10 or newer and Bash. Install the pinned tools in a virtual environment:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
export PATH="$PWD/.venv/bin:$PATH"
python3 scripts/check.py
```

Run `python3 scripts/check.py --fix` to format Python and shell and normalize text whitespace. The same checks run on pushes and pull requests. XML is checked for well-formedness; game schemas, XPath matches and gameplay require separate X4 validation. Blender scripts are parsed and linted without importing Blender.

Use UTF-8, LF, a final newline, spaces and no trailing whitespace. Indent code with four spaces and workflow YAML with two. Use descriptive names, uppercase shell variables, constant-first equality comparisons and explicit boolean checks. Ruff's E712 rule is disabled to retain explicit boolean comparisons. Keep shell free of prose comments. Keep only short, non-obvious constraints in code; put explanations here. XML continuation attributes may align with their opening attribute. Preserve XPath selectors, savegame identifiers and embedded game expressions when applying formatting.

## Installation and publishing helpers

`install.sh` and `publish.sh` both source `lib/find_x4.sh`. The library searches usual Steam roots and additional library folders. `X4_PATH`, `X_TOOLS_PATH`
and `PROTON_PATH` override discovery. Proton Experimental is preferred when found; otherwise the helper uses the last matching Proton directory it encounters.

Installation replaces the extension directory with a copy of `extension/`. Refresh it after edits; X4 enumerates real extension directories, so a symlink does not substitute for installation. Restart the game after installing or removing.

Publishing stages a separate copy inside the game's extensions directory. The repository keeps its readable extension id; `steam/workshop-id` holds the numeric Workshop id. The helper changes only the staged manifest, runs the interactive WorkshopTool and restores the manual installation after success. On Linux it runs WorkshopTool through Proton and maps paths through drive Z. A failed upload can leave the staged copy behind; rerun `./install.sh` to restore it.

## Release metadata

`content.xml` uses an integer version multiplied by 100 and an ISO release date. The date matches the corresponding released entry in `CHANGELOG.md`. Development changes belong under `Unreleased`; they do not advance the manifest's release version or date. An unreleased scaffold may retain its initial creation date until its first release. Keep existing extension ids stable.

## Implementation constraints

### extension/md/drjele_khaak_pressure.xml

Configuration. Re-read on every savegame load, so editing a value below and reloading is enough - no new game required. Each one can also be overridden at runtime without touching this file, by setting the matching global variable:
&lt;set_value name="global.$DrJeleKhaakHiveGateRange" exact="6"/&gt; Those globals are also what the in-game options menu writes - see drjele_khaak_pressure_options.xml.

Two of the three settings are applied by writing into the vanilla Kha'ak manager's own variables (see Apply below) rather than by patching its script. That is deliberate: the numbers live on a cue that runs once per game, so an XPath patch adding a value to its actions would only ever reach a new game, and it would collide with the other mods that patch the same file. Writing the variable reaches an existing savegame too.

The third one, the spawn chance factor, is a local inside a vanilla library that is reset on every call, so it cannot be written from outside - that one does need a patch, and it lives in khaak_activity.xml next to this file.

Applied on every savegame load, and again whenever a slider moves.

The parent instantiates on md.Setup.Start, which the game signals on a new game and on every load; the child then waits for the vanilla manager to have run. Waiting matters:
both cues trigger off the same signal and the order between two scripts is not defined, so the settings would land on variables that do not exist yet perhaps half the time.

$HiveStationGateRange existing is also the test for "the Kha'ak are active at all". The vanilla manager only initialises in the main galaxy, so in a Timelines scenario, a tutorial or the workshop map the check never passes and this mod stays completely inert.

Every setting is looked up the same way: the global the options menu writes wins, otherwise the table Configuration publishes, otherwise the vanilla literal. All of them are global variables, so a configuration that fails to load degrades to stock behaviour rather than breaking the Kha'ak. The table is tested for existence rather than the field inside it, so a value edited down to the vanilla number is still read as configured.

Half, as configured. Note this is not the vanilla ratio: vanilla pairs a trigger of 50000 with a candidate floor of 40000, so even at the top of the slider the pool of sectors an outpost may land in is wider here than in the stock game. The rate at which outposts appear is unchanged by that on its own - the trigger is what gates it

- but they spread further from the single hottest sector.

The gate range is the expensive one, so it is only touched when it actually moved. The manager derives $WatchedSectors from it, but only ever recomputes that set when a station is created or destroyed - so changing the number alone would leave the watched set stale until the next hive dies. Rebuilding it here is what makes the slider take effect immediately.

Recomputes $WatchedSectors the way the vanilla StationDestroyed_v2 cue does: every sector within gate range of a hive, plus the sector of every outpost. Sectors that fall out of range lose their accumulated activity as well, which is again what vanilla does when a station dies - leaving a stale score behind would let a sector the Kha'ak no longer watch hand them a free outpost the moment it comes back into range.

The group is snapshotted into a list before the removal pass; a group cannot be mutated while it is being iterated.

### extension/md/drjele_khaak_pressure_options.xml

In-game options, through SirNukes Mod Support APIs. Everything that touches that API lives in this file, so if the API is not installed nothing here ever runs and the mod keeps working off the constants in drjele_khaak_pressure.xml.

All three callbacks only ever write the global override variables the rest of the mod already reads, so neither the apply step nor the patched vanilla library knows anything about menus.

The API stores an option's value in the savegame under its $id, and hands it straight back to the widget as its start value. A stored value outside the slider's range fails widget validation and takes the whole Extension Options menu down with it, so the range and the unit of an option must never change under a $id that has already shipped - give it a new one instead. That is why these ids carry their unit.

The two applied settings are pushed into the vanilla manager, so moving either slider has to signal the apply cue; the ambush multiplier is read at the moment the vanilla library rolls, so writing the global is all it needs.

### extension/md/khaak_activity.xml

One patch, for the one setting that cannot be applied by writing a variable.

$BaseSpawnChance is a local of GetSuitableBaseStationForShipSpawn, assigned fresh on every call, so anything written into it from outside is overwritten before it is ever read. Scaling it right after vanilla sets it is therefore the only place the multiplier can go.

Vanilla's chance, per resource-depletion event in a sector that holds a Kha'ak outpost, is

chance = (sector Kha'ak activity / 500000) * 0.03 * units mined * resource factor

as a percentage, with the resource factor 1.2 for ore and silicon and 2.7 for nividium. The multiplier lands on the 0.03, so it scales the whole thing linearly; the game clamps the result at a certain hit, which a high multiplier reaches easily on a large mining haul.

The setting is read the same way as the rest: the global the options menu writes wins, otherwise the table drjele_khaak_pressure.xml publishes, otherwise the vanilla literal 1. All of them are global variables, so a configuration script that fails to load leaves the vanilla chance intact rather than breaking Kha'ak ship spawning.
