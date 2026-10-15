### Utilities Menu Overview

![Utilities Menu](../assets/images/utilities_menu.png){ loading=lazy }

The `Utilities` menu allows you to change certain aspects of your system. The options shown depend on what is installed.

### Menu Options

#### `R. Rename existing unit`

This option will allow you to rename an existing unit. For example, by default, when you install a BoxTurtle system,
it is named `Turtle_1`. This value is displayed in the Mainsail/Fluidd GUI along with multiple places in the 
configuration files.

This option will prompt you for the following information:

![Rename-detail](../assets/images/rename-detail.png){ loading=lazy }

    - Name of the unit you want to rename
    - New name of the unit

A validation check will be performed to ensure that the name meets specific requirements

??? note "Name Validation Requirements"

        Specifically, the name must meet the following regex:
        
        `^[a-zA-Z0-9_]+$`. It also must be less than 24 characters in length.

Once the name is validated, the system will rename the unit and update the configuration files accordingly. 

#### `T. Set INDX tool count`

This option changes the number of Bondtech INDX tools. It is only shown when the INDX macros are installed
(`AFC/indx/indx.cfg` exists). It keeps `AFC_Hardware.cfg` and `indx.cfg` in step: it adds or removes
`[AFC_extruder toolN]` sections and `T<n>` wrappers, updates `tool_count`, and adds placeholder dock positions for
new tools. See [Bondtech INDX](../toolchanger/indx.md#configuration) for details. Restart Klipper afterwards.

#### `M. Return to Main Menu`

Select this option to return to the main menu.

#### `Q. Quit`

Select this option to exit the AFC script.