# [AFC_Toolchanger.cfg] Configuration Overview

The `AFC_Toolchanger.cfg` file is used to define the configuration when using AFC on toolchangers.

## [AFC_Toolchanger toolchanger] Section

Registers a toolchanger unit with AFC.

Only one `[AFC_Toolchanger]` section is required per toolchanger system.

Lanes from an AFC unit can feed toolchanger tools. Each `[AFC_hub]` can only feed lanes on one
tool, since it merges them into a single path. AFC refuses to start when a hub feeds lanes on
more than one tool. Lanes going straight to a tool use `hub: direct`.

``` cfg
[AFC_Toolchanger <toolchanger name>]
backend: ktc
#    Default: ktc
#    Toolchanger type. Valid options:
#      ktc  - klipper-toolchanger, or custom_tool_swap/custom_unselect macros
#      indx - Bondtech INDX, see the Bondtech INDX page
```

### Bondtech INDX Settings

The following options are only used with `backend: indx`. Only set them if you renamed the INDX macros.
See [Bondtech INDX](../toolchanger/indx.md) for the full setup.

``` cfg
indx_change_macro: INDX_CHANGE_TOOL
#    Default: INDX_CHANGE_TOOL
#    Macro AFC runs to pick up a tool, called with TOOL=<n> TEMP=<t>.
indx_park_macro: PARK_TOOL
#    Default: PARK_TOOL
#    Macro AFC runs to park the tool on the head.
indx_positions_macro: TOOL_POSITIONS
#    Default: TOOL_POSITIONS
#    Macro holding the INDX tool_count and no_heat_tools variables.
```