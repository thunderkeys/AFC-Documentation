## Tip-Forming Related Commands

These commands are available when using AFC with a toolchanger setup and `[AFC_Toolchanger <name>]` has been defined in the configuration.

-----
[AFC_SELECT_TOOL]
::: AFC_Toolchanger.AfcToolchanger.cmd_AFC_SELECT_TOOL
    options:
      docstring_style: numpy
      heading_level: 3

-----
[AFC_UNSELECT_TOOL]
::: AFC_Toolchanger.AfcToolchanger.cmd_AFC_UNSELECT_TOOL
    options:
      docstring_style: numpy
      heading_level: 3

-----
[AFC_SET_TOOLHEAD_LED]
::: AFC_Toolchanger.AfcToolchanger.cmd_AFC_SET_TOOLHEAD_LED
    options:
      docstring_style: numpy
      heading_level: 3

## Bondtech INDX Commands

These commands are registered when `[AFC_Toolchanger <name>]` uses `backend: indx`. They are called by
the INDX macros installed with AFC and are not meant to be run by hand. See
[Bondtech INDX](../../toolchanger/indx.md).

-----
[AFC_INDX_TOOL_CHANGED]
::: AFC_indx.AfcIndx.cmd_AFC_INDX_TOOL_CHANGED
    options:
      docstring_style: numpy
      heading_level: 3

-----
[AFC_INDX_PRE_LATCH]
::: AFC_indx.AfcIndx.cmd_AFC_INDX_PRE_LATCH
    options:
      docstring_style: numpy
      heading_level: 3