# AGENTS.md -- DataTableFramework

`DataTableFramework/run.py` builds `AAAA_DataTableFramework_P`: every composite table the game has, plus `Ch0001..Ch1000` stubs each mod can fill. The pak tree is generated, 14,000 files, and is not in this repo.

## Tools

Same toolchain as MoreTuning. UAssetGUI v1.1.1, engine version `VER_UE5_4`, mappings
`MotorTown721T1` or newer. UAssetGUI fails silently, so always check that the output file
exists and reads back as a DataTableExport, not a RawExport.
The tools import `Rebase/` from a MoreTuning checkout at `D:\MT\MoreTuning`; paths are
absolute and machine-specific on purpose.

## 0.7.20: everything ships as DataTableFramework channels

Since 0.7.20 no pak in this repo ships a part table. `DataTableFramework/run.py`
ships every composite the game has (14: VehicleParts, Vehicles, Items, Cargos,
CharacterBodyParts, Buildings, Buffs, Emotes, four `DT_Voice_*`, HelpText,
RichImageAll2 -- discovered by scanning, not listed) with its vanilla parents,
then the tables other mods already add (`registry.json` "adopted"), then
`Ch0001..Ch1000`. A mod ships only `/Game/DataTableFramework/<Composite>/ChNNNN`.
Later parents win a same-named row, so channel number is priority.

```
python Rebase/motorworks.py        # only when Maja releases: her edits -> old/0.7.19/motorworks_deltas.json
python DataTableFramework/run.py   # AAAA_DataTableFramework_P: composites + 14,000 stubs
python Rebase/channels.py          # VehicleParts Ch0001 MoreTuning, Ch0002 No Limits, Ch0003 Carbon
python TehEngines/run.py           # Teh's 16 engines -> Z_TehsEngineSoundPack_P, Ch0004
python SuspensionExpansion/run.py  # 57 suspension parts + its cost/mass edits -> Z_SuspensionExpansion_P, Ch0005
python MoreAttachments/channel.py  # Items Ch0001 + both Vendor_Garage copies
```

- **One framework pak, mounting first** (`AAAA_`), so any mod's channel file
  beats its stub. The price: a mod that still ships its own composite copy
  beats the framework and switches every channel off. Overlord, Proxy's wheels
  and cargo, VehicleLabs and Detail Works all did in 0.7.19; their tables are
  adopted (`registry.json`), so dropping the copy is all they need -- and their
  0.7.19 copies lack 0.7.20's Horns/RoofRacks/CargoBedAttachments anyway.
- **Channels are `Ch0001`, never `Ch_0001`**: `Ch_1000` is FName (`Ch`, 1000).
  Equal length is also what lets one built stub be cloned by byte swap.
- **One mod, one channel file.** Rows from different source tables share one
  import table (`channels.combine` grafts each source's hard refs across), and
  the build fails on a row name defined twice. An own row whose name vanilla now
  has in *any* parent is dropped: 0.7.20 moved the taxi/bus/limo licences from
  VehicleParts0 into License, and our 0.7.19 copies would have shadowed them.
- **Old row names stay.** Saves key parts by row name; the `C<NNNN>_` prefix is
  for rows added from now on.
- **The `No_*` mesh-less parts are retired** (`channels.RETIRED`, 2026-10-30): VehicleLabs
  makes those seven body types uninstallable instead (`vehicle_mod/tables.py` `OPTIONAL`).
- The mod's 0.7.19 tables live in `Rebase/old/0.7.19/` as migration input only.
  `rebuild.py`, `compat.py` and the `main()`s of RemoveReq/CarbonParts still write
  whole tables -- the pre-0.7.20 route below; don't ship their output.
- **Motor Works ships inside MoreTuning.** Her 7 rows are in its own tables, the
  hitch mesh is in `qxZap_MoreTuning_P/Cars/Parts/TrailerHitch`, and her edits to
  vanilla rows are kept as a per-property diff against 0.7.19 vanilla
  (`motorworks.py`) and applied onto 0.7.20 vanilla -- never her frozen tables,
  which would undo 0.7.20's balance. Her mesh refs are stored by asset path, not
  import index, and re-resolved against the target table.
- **No override channel on `Wheels`.** It would sit above Maja's Mo Wheel Works
  and put vanilla meshes back on her 93 rows. With channels the old ZZb wheel
  compat pak is unnecessary: carbon wheel twins are new rows.
- **Paint on foreign cars** (`paint.py`): a part paints where its mesh slot uses
  `M_Body`, but 388/413 AeroParts rows have no `ColorSlots` and rely on the
  slot name matching the car's. No Limits and Carbon give such parts one
  `ColorSlot` per `M_Body` slot. MoreAttachments turns the same into
  `AttachmentPaintSlots` (the 0.7.20 metallic/roughness sliders) and goes
  further: with no `M_Body` slot, every non-light slot is painted, because
  vanilla's antenna paints a plain `M_Gray` slot -- the attachment paint system
  supplies its own material, so palette-only meshes (Duro bumpers) paint too.
- **Generated attachment actors never use the lamp template.** A bumper with
  built-in lights (Duro 02/03, Duro bull bars) is still a bumper; the mesh test
  for `M_HeadLight` made them lamps with a dead ToggleLight/ChangeLightMode.
- **0.7.20 changed the part assets too.** `MHTransmissionProperty` split
  `AutoShiftComportRPM` into `_Gasoline`/`_Diesel` (vanilla's rule: old <= 2500
  went to diesel with gasoline 3500, else to gasoline with diesel 2300), which
  made all 99 mod transmissions "Corrupt data found" in game. `reserialize.py`
  migrates them (`FIELD_MIGRATIONS`). `TransmissionGenerator`'s template still
  has the old field -- regenerate it from a 0.7.20 box before generating again.
- **Class layout shifts** (`reserialize.py`): unversioned properties are stored
  by index, so a property removed mid-class shifts everything after it. 0.7.20
  removed `MTInteractableComponent.bUseServerOnPreInteraction`, which made all
  713 attachment actors read raw. The tool diffs the usmaps and round-trips
  only assets exporting a shifted class -- including a class whose own list is
  unchanged but which holds a shifted struct (that is how the transmissions
  hid: `MTTransmissionDataAsset` itself did not change).
- 0.7.20 moved the turbo's seven flat fields into `Gasoline`/`Diesel`
  sub-structs. `rebase.reshape` copies a moved field into every sub-struct that
  has it (what vanilla did to its stock turbos) and fails on one with no home.
- Wheel radius tuning (0.8-1.2) is native: `GarageTuningWidget` builds the
  sliders in code and no asset carries the range. Not a pak change.
