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

INDX tools can be fed from an AFC unit such as a BoxTurtle. The lane modes are the ones in
[Toolchanger Configuration](configuration.md#lane-to-toolhead-connection-modes), and one printer
can mix them:

| Tool | Setup |
|---|---|
| Standalone | No lane. Spool straight to the tool, as above. |
| One lane (direct) | One lane with `hub: direct` or `direct_load`, `dist_hub` set to the full bowden length to the tool. |
| Several lanes (hub) | Lanes through an `[AFC_hub]` that feeds only this tool. |

The installer does not set this up, add the lanes to your config by hand. Install the unit with
[Install Additional System](../installation/main-menu.md#a-install-additional-system) in the
installer's main menu first, then edit its lanes as below.

### What is different on INDX

Set lanes up as in the toolchanger guide, with these changes:

- **Point lanes at the AFC tool name.** Use `extruder: tool<n>` (the `[AFC_extruder]` name), not
  `extruder`. Every INDX tool shares the klipper `extruder`, so only the AFC name tells them
  apart. Once a tool has a lane it is no longer standalone.
- **The buffer is the tool sensor.** Passive tools have no sensors, and a lane-fed tool can't use
  `pin_tool_start: virtual`. Give each fed tool its own TurtleNeck style buffer and set
  `pin_tool_start: buffer`. The lane pushes until the filament reaches the DX gears and
  compresses the buffer, and that ends the bowden load.
- **Set the buffer on the tool.** Put `buffer:` on the `[AFC_extruder]`, and every lane on that
  tool uses it. A `buffer:` on a lane overrides it, keep every lane on one tool on the same
  buffer.
- **`map` goes on the lanes.** The lanes provide the tool's `T<n>` commands, the tool's own `map`
  is only used while it is standalone.
- **One hub per tool.** An `[AFC_hub]` can only feed one tool. AFC stops at startup if a hub has
  lanes on two tools (this applies to every toolchanger).
- **Lengths end at the DX gears.** `dist_hub` (direct) or `afc_bowden_length` (hub) runs to the
  point where the buffer triggers, and `tool_stn` from there to the nozzle.
- **Tip forming, no cutter.** INDX tools have no cutter. Set `form_tip: True` and
  `tool_cut: False` on tools fed by several lanes. These override the `AFC.cfg` values for that
  tool only.
- **No `tool` or `led_name`.** The tool is found by `tool_index`, and all tools share the Smart
  Head neopixel.

### Loading only happens on the head

Lanes never push filament into a docked tool. There is a gap between the end of the PTFE on the
passive tool and the DX gears, and without the DX there filament can spool out into the head with
nothing to detect it. AFC picks the tool up first for every `TOOL_LOAD`, `T<n>`, `CHANGE_TOOL`
and `AFC_CALIBRATION` bowden or direct lane calibration, and a `direct_load` lane picks its tool
up when a spool is inserted. The printer must be homed for the pickup, and the tool stays on the
head after a calibration.

`LANE_MOVE` does not pick the tool up. Pick it up first with `T<n>` before moving a lane past its
hub.

### Example

A 4 lane BoxTurtle feeding four INDX tools:

- `lane1` + `lane2` → `hub_t0` → `tool0` (hub)
- `lane3` → `tool1` (direct)
- `lane4` → `tool2` (direct)
- `tool3` standalone

**`AFC_Hardware.cfg`:**

```ini
[AFC_extruder tool0]
extruder_name: extruder
tool_index: 0
toolchanger_unit: INDX
pin_tool_start: buffer
buffer: TN_t0
tool_stn: 72
tool_stn_unload: 100
# fed by two lanes, tip form when switching between them
form_tip: True
tool_cut: False

[AFC_extruder tool1]
extruder_name: extruder
tool_index: 1
toolchanger_unit: INDX
pin_tool_start: buffer
buffer: TN_t1
tool_stn: 72
tool_stn_unload: 100

[AFC_extruder tool2]
extruder_name: extruder
tool_index: 2
toolchanger_unit: INDX
pin_tool_start: buffer
buffer: TN_t2
tool_stn: 72
tool_stn_unload: 100

# standalone, unchanged from the installed template
[AFC_extruder tool3]
extruder_name: extruder
tool_index: 3
map: T4
toolchanger_unit: INDX
pin_tool_start: virtual
standalone_auto_load_unload: False
tool_stn: 72
tool_stn_unload: 100
```

**`AFC_Turtle_1.cfg`** (only the options that change are shown, keep the pins from the
template):

```ini
[AFC_BoxTurtle Turtle_1]
hub: hub_t0                 # default for lanes that don't set their own
extruder: tool0

[AFC_hub hub_t0]
switch_pin: ^Turtle_1:HUB
afc_bowden_length: 900      # hub to the DX gears on tool0
move_dis: 75
cut: False

[AFC_stepper lane1]
unit: Turtle_1:1
dist_hub: 155
map: T0
...

[AFC_stepper lane2]
unit: Turtle_1:2
dist_hub: 80
map: T1
...

[AFC_stepper lane3]
unit: Turtle_1:3
hub: direct
extruder: tool1
dist_hub: 1050              # lane to the DX gears on tool1
load_to_hub: False
map: T2
...

[AFC_stepper lane4]
unit: Turtle_1:4
hub: direct
extruder: tool2
dist_hub: 1050
load_to_hub: False
map: T3
...
```

The `T<n>` numbers follow the lanes, not the tools: `T0` and `T1` are both on `tool0`, and the
standalone `tool3` is `T4`.

Each `TN_t<n>` is an `[AFC_buffer]` section, see
[Buffer Overview](../installation/buffer-overview.md).

### Tool changes

What a tool change does depends on what the target tool holds:

- **Tool already holds the lane**: pickup only. Nothing is unloaded, loaded or purged.
- **Same tool, other lane** (hub): tip form, unload, load and purge with the tool on the head.
- **Other tool holding another lane**: park, pick the tool up at the temperature of the lane it
  holds, unload that lane, then load the new one at its own temperature and purge.
- **Empty tool**: pickup, then load.
- **Runout with infinite spool to a lane on another tool**: the remaining filament is unloaded
  from the tool on the head before it is parked, so a docked tool never keeps a short end.

Changes between two lanes on different tools take longer than on other toolchangers, since the
single heater has to reheat for the unload and again for the load. Slicer time estimates will be
low.

The filament stays in a docked tool and its lane motor runs free until the tool is picked up
again. The DX drives the latch and may grip the filament, so every latch move unsyncs and
disables the lane motors and their buffers. The lane in the tool on the head is enabled again
once the change finishes.

### Startup checks

AFC stops at startup if:

- a lane feeds a tool listed in `no_heat_tools`
- a hub feeds lanes on more than one tool
- a lane-fed tool uses `pin_tool_start: virtual`

It warns about any enabled AFC park, poop, kick, brush or cut position on the dock side of
`clearance_y` in `indx.cfg`, since a tool on the head can hit the docked tools there.

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

- Lanes only move filament into a tool while it is on the head, see
  [Loading only happens on the head](#loading-only-happens-on-the-head).
- The installer does not assign lanes to tools, edit the config by hand.
- `led_name` should not be set on INDX tools. All tools share the Smart Head neopixel.

## Updating the INDX macros

See [INDX Macro Resync](indx_macro_resync.md).
