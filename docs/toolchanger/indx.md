# Bondtech INDX

AFC supports the [Bondtech INDX](https://github.com/BondtechAB/INDX) toolchanger with
`[AFC_Toolchanger]` `backend: indx`. Kalico is the supported firmware, since INDX support
(`[indx]`, the `indx` sensor type and `INDX_EXTRUDER_MOVE`) is built into it.

INDX differs from other toolchangers in one important way: the passive tools have no
electronics. There is a single `[extruder]` (heater, IR thermistor and DX stepper) on the Smart
Head, and every tool uses it while it is on the head. AFC models each INDX tool as an
`[AFC_extruder]` that shares `extruder_name: extruder` with the other tools, and keys it by its
AFC name instead of the klipper extruder name.

Tool changes are done by the INDX macros, vendored in [`templates/indx_macros`](https://github.com/AFCProject/AFC-Klipper-Add-On/tree/main/templates/indx_macros) with a small set
of AFC patches, and installed to `AFC/indx/`.

## Install

Run `install-afc.sh` and pick **Bondtech INDX** as the installation type (see the
[Install Menu](../installation/install-menu.md#t-installation-type)). The installer:

- copies `AFC_Hardware_INDX.cfg` to `AFC/AFC_Hardware.cfg`
- copies the INDX macros to `AFC/indx/`. `indx.cfg` (your settings) is never overwritten.
- adds `[save_variables]` and `[respond]` unless your config already has them
- leaves `[include indx/homing.cfg]` commented out if you already have a `[homing_override]`.
  Merge the INDX homing sequence into yours in that case.

Then configure the Smart Head in `printer.cfg` (`[mcu indxmcu]`, `[indx]`, `[extruder]`,
`[load_cell_probe]` ...) as described in the [INDX README](https://github.com/BondtechAB/INDX), and set your dock positions in
`AFC/indx/indx.cfg`.

## Configuration

```ini
[AFC_Toolchanger INDX]
backend: indx               # ktc (default) or indx
#indx_change_macro: INDX_CHANGE_TOOL   # macro AFC runs to pick up a tool (TOOL=<n> TEMP=<t>)
#indx_park_macro: PARK_TOOL            # macro AFC runs to park the tool on the head
#indx_positions_macro: TOOL_POSITIONS  # macro holding tool_count and no_heat_tools

[AFC_extruder tool0]
extruder_name: extruder     # every INDX tool uses the Smart Head extruder
tool_index: 0               # INDX tool number (t0_x / t0_dock_y in indx.cfg)
map: T0                     # T command the slicer sends for this tool
toolchanger_unit: INDX
pin_tool_start: virtual     # passive tools have no sensors
standalone_auto_load_unload: False
```

There is one `[AFC_extruder]` per installed tool, matching `tool_count` in `indx.cfg`. The
installer asks for the number of tools (option **J** in the install menu). To change it later, use
**Set INDX tool count** in the installer's [utilities menu](../installation/utilities-menu.md#t-set-indx-tool-count), or run:

```shell
~/AFC-Klipper-Add-On/utilities/indx_tools.py set-count 6
~/AFC-Klipper-Add-On/utilities/indx_tools.py show
```

This keeps `AFC_Hardware.cfg` and `indx.cfg` in step:

- New `[AFC_extruder toolN]` sections copy the settings of your lowest numbered tool, with
  `tool_index` and `map` changed. Lowering the count removes the extra sections.
- In `indx.cfg` it updates `tool_count`, adds a `[gcode_macro T<n>]` wrapper per tool and removes
  wrappers past the new count.
- New tools get placeholder dock positions (`t<n>_x`, `t<n>_dock_y`), spaced like your last two
  docks. Measure them with `CALIBRATE_DOCK_X TOOL=<n>`. Adding a dock sets `docks_calibrated` to
  `False` so INDX does not drive into a placeholder position. Set it back to `True` once every dock
  is measured. Dock positions of removed tools are left in place.

Restart Klipper afterwards.

AFC checks at startup that `[save_variables]` and the INDX macros are loaded, that every tool
has a unique `tool_index` below `tool_count`, and that all tools use the same extruder.

## How it works

- **Tool on the head.** INDX stores it in the `active_tool` save variable, and AFC reads it from
  there. Every INDX write goes through `_INDX_SET_ACTIVE_TOOL`, which calls
  `AFC_INDX_TOOL_CHANGED`, so AFC follows changes it did not start: the `G28` T0 fetch,
  `MANUAL_TOOL_SEAT`, `MANUAL_TOOL_REMOVE`, `TC_CYCLE_ALL` and the calibration macros.
- **T commands.** AFC claims `T0`, `T1` ... through each tool's `map`. The `[gcode_macro T<n>]`
  wrappers in `indx.cfg` exist so Mainsail and Fluidd show tool buttons. AFC sets their `active`
  variable for the tool on the head.
- **Tool changes.** AFC runs `INDX_CHANGE_TOOL TOOL=<n> TEMP=<t>` (the INDX `CHANGE_TOOL`, renamed
  because AFC owns `CHANGE_TOOL`) and `PARK_TOOL`, then checks `active_tool`. If you renamed
  these macros, point AFC at them with `indx_change_macro`, `indx_park_macro` and
  `indx_positions_macro`. A failed pickup or
  park stops AFC with the INDX error.
- **Heating.** There is one heater, so AFC never preheats the next tool or cools the last one.
  The incoming tool's temperature is passed to INDX as `TEMP=`. AFC's `M104`/`M109` apply the
  INDX rules: a `T<n>` for a tool that is not on the head is ignored, an empty head is not heated,
  and tools in `no_heat_tools` (`indx.cfg`) are held at 0.
- **Latch moves.** The DX extruder also drives the latch. `_TC_LATCH_MOVE` calls
  `AFC_INDX_PRE_LATCH`, which unsyncs and disables every AFC lane stepper on an INDX tool, and
  lanes only sync to the extruder while their tool is on the head.

Both hook commands are documented under [Toolchanger Commands](../klipper/internal/toolchanger.md).

## Feeding tools from an AFC unit

INDX tools can be fed from an AFC unit such as a BoxTurtle, and one printer can mix three kinds
of tool:

| Tool | Setup |
|---|---|
| Standalone | Spool straight to the tool, as above. |
| One lane (1:1) | One lane with `hub: direct`, `dist_hub` set to the full bowden length to the tool. |
| Several lanes (N:1) | Lanes through an `[AFC_hub]` that feeds only this tool. |

Point a lane at its tool with `extruder: tool<n>` in the lane or unit section. Once a tool has a
lane it is no longer standalone, and the lanes provide its `T<n>` maps instead of the tool's
`map`. The PTFE ends on the passive tool, so the filament stays in a docked tool and the lane
motor runs free until the tool is picked up again.

Give each lane-fed tool its own TurtleNeck style buffer and use it as the tool sensor:

```ini
[AFC_extruder tool0]
extruder_name: extruder
tool_index: 0
toolchanger_unit: INDX
pin_tool_start: buffer
buffer: buf_t0
# INDX tools have no cutter, tip form when a lane on this tool is swapped
form_tip: True
tool_cut: False

[AFC_BoxTurtle Turtle_1]
hub: hub_t0
extruder: tool0              # lanes 1 and 2 feed tool0 through hub_t0

[AFC_stepper lane3]
unit: Turtle_1:3
hub: direct
extruder: tool1              # lane 3 feeds tool1 on its own
dist_hub: 1050
...
```

What a tool change does depends on what the target tool holds:

- **Tool already holds the lane**: pickup only. Nothing is unloaded, loaded or purged.
- **Same tool, other lane** (N:1): tip form, unload, load and purge with the tool on the head.
- **Other tool holding another lane**: park, pick the tool up at the temperature of the lane it
  holds, unload that lane, then load the new one at its own temperature and purge.
- **Empty tool**: pickup, then load.
- **Runout with infinite spool to a lane on another tool**: the remaining filament is unloaded
  from the tool on the head before it is parked, so a docked tool never keeps a short end.

`tool_cut` and `form_tip` on an `[AFC_extruder]` override the `AFC.cfg` values for that tool, so
a printer can cut on one toolhead and tip form on its INDX tools.

AFC checks at startup that:

- no lane feeds a tool listed in `no_heat_tools`
- each hub feeds only one tool (checked for every toolchanger)

A lane-fed tool can't use `pin_tool_start: virtual`, as with any AFC toolhead.

It warns about any enabled AFC park, poop, kick, brush or cut position on the dock side of
`clearance_y` in `indx.cfg`, since a tool on the head can hit the docked tools there.

The DX drives the latch and may grip the filament, so every latch move unsyncs and disables the
lane motors and their buffers. The lane in the tool on the head is enabled again once the change
finishes.

## Slicer

Use the same tool change G-code as with AFC on any toolchanger (see [Slicer Config](slicer_config.md)),
for example in OrcaSlicer:

```gcode
T{next_extruder}
```

To heat each tool straight to its own temperature, pass it through AFC:

```gcode
CHANGE_TOOL LANE=tool{next_extruder} NEW_EXTRUDER_TEMP={new_filament_temp}
```

## Current limits

- Lanes only move filament into a tool while it is on the head. Pushing filament into a
  docked tool ahead of the pickup is planned.
- `led_name` should not be set on INDX tools. All tools share the Smart Head neopixel.

## Updating the INDX macros

See [INDX Macro Resync](indx_macro_resync.md).
