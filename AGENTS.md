# AGENTS.md -- DataTableFramework

`DataTableFramework/update.py` builds `AAAA_DataTableFramework_99_P`: one shared copy of every composite
table the game reads, listing vanilla's tables first and then every registered mod table. The pak tree is
generated and is not in this repo. The source of truth is the MoreTuning checkout (`D:\MT\MoreTuning\
DataTableFramework`); this repo is a copy, updated by hand after each change there.

## Tools

Same toolchain as MoreTuning. UAssetGUI v1.1.1 (`D:\MT\UAssetGUI.exe`, not the repo-local v1.0.4), engine
version `VER_UE5_4`, the current Motor Town usmap. UAssetGUI fails silently: check that the output file
exists and reads back as a DataTableExport, not a RawExport. The tools import `Rebase/` from the MoreTuning
checkout; paths are absolute and machine-specific on purpose.

## How it works

- **14 composites**, found by scanning, not listed: VehicleParts, Vehicles, Items, Cargos, CharacterBodyParts,
  Buildings, Buffs, Emotes, four `DT_Voice_*`, HelpText, RichImageAll2. Plus a Decals composite
  (`/Game/DataTableFramework/Decals/Decals`) that GameResource points the game at.
- **Adopted tables** (`registry.json` "adopted"): a mod ships its own table and is listed as a parent of the
  composite. Later parents win a same-named row, so the order in `registry.json` is the priority. A listed
  table whose mod is not installed is a missing parent, which the game skips.
- **Reserved car slots** (`registry.json` "reserved"): rtwin's ReservedVehicleFramework layout,
  `/Game/ReservedRegistration/Slots/Slot01..Slot32/<table>`, listed after every adopted table with no stubs.
  A car mod drops into a free slot without a framework rebuild. Slot01 is Suspension Tuning's brake kits.
- **The pak outranks every other pak** (`_99_P`). A mod that ships its own copy of a composite is overruled,
  so its rows only appear once its table is adopted.
- **GameResource ships here**, because only one pak can: it points Decals at the composite and carries
  Arini's world-map bounds, read from `D:\MTLiveMap\dtf_patches\gameresource_worldmap.json` (Arini's map
  build rewrites it). When the map size changes, rebuild with `gameresource.py`.
- **Channels are retired.** The first 0.7.20 build shipped `Ch0001..Ch1000` stubs per composite; every
  channel moved to an adopted table and no `ChNNNN` file is read any more. `REGISTRY.md` keeps the history.

## Rules

- **Never rename a row.** Saves store parts and vehicles by row name; a renamed row is a part gone from
  every car that wore it. Adding rows is free.
- **Keep row names unique across mods.** Two mods defining the same row: the later parent wins silently.
- **Regenerate `REGISTRY.md`** (`python DataTableFramework/registry_doc.py`) whenever `registry.json` changes.
- **Rebuild after every game update** (`update.py`): the framework outranks vanilla, so a stale composite,
  GameResource or DriveGameModeBP would undo the update's own changes.
