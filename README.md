# Klipper Z Tramming Macro
This is a Z Tramming macro created for the Sovol SV06 Plus.
It can also be used in any dual motor single driver cartesian printer, given some changes documented here.

**NOTE**: This is a work in progress. If you don't understand about klipper and macros, you shouldn't use this as it **may damage your printer** in some circumstances. So make sure you read the documentation fully and understand it. I'm not responsible for any damage cause to your printer.

## How does it work?
This macro measures the opposite ends of the X axis to determine if the Z screws are misaligned.

This can be affected by a bent/misaligned bed, so take that into consideration during usage. But it should be a great tool to use instead of the G34 Mechanical Gantry Alignment, given that it does the calibration based on the bed and doesn't force the steppers to skip like the original G34.

**Any change made to your Z alignment will change your bed mesh!**
It's advised to run a new bed mesh calibration every time you correct it. You may also use it in conjunction with [Klipper Adaptive Meshing & Purging](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging).

## How to use it?
This macro has only one visible command, the `Z_TRAMMING` macro.
This allows you to use the macro in 2 different ways:

### Standalone Z calibration
When executed directly via the `Z_TRAMMING` command, the macro will home the printer, if not done yet, and proceed to probe both sides of the bed, right above the bed screws, to determine the difference in height.
Then, it'll print to the console (see [config section](#configuring-the-macro) for Popup Helper) the defined tolerance, the current difference, the correction needed and will disable the steppers for the correction to be made.
The adjustment is done in hours and minutes, same as the SCREW_TILT_ADJUST macro. Meaning that "minutes" refers to "minutes of a clock face".
So, for example, 15 minutes is a quarter of a full turn. 01:20 means 1 full turn and 20 minutes. As for direction, CW means clockwise and CCW means counter-clockwise.
**NOTE:** The correction calculation is still a work in progress, as turning one side affects both in a way a don't completely understand yet. But having the way to turn it and the current height difference will help you understand what you got to do.

You should repeat the process several times until it is within the set tolerance.
You can check and set the tolerance together with the other configs in the [config section](#configuring-the-macro).

### Automatic Z alignment check
This second way to use it was made in order to automate the printing process.
To do that, you'd put `Z_TRAMMING` into your `PRINT_START` macro.
The macro will then perform the same check as before, and proceed as normal if the the difference is within the set tolerance.

If it is out of tolerance, the printer will stop and show you the corrections needed through the console, same as before.
Once you've done the necessary corrections, you should run `Z_TRAMMING` again and the process will repeat.
Once the macro checks that everything is within tolerance, it'll re-home the printer and you can restart the print.

## Configuring the macro
The first configuration is the UI Popup Helper. For now, it is only implemented via the Macro Prompts in Mainsail 2.9.0.
If you have this version or higher, you should set the `variable_mainsail` to `1` in the `Z_TRAMMING` macro, just as the tolerance settings below.

If you have any other UI set, you should keep it as `0` and use it through the console.

This script was made for the stock Sovol SV06 Plus, if that is your printer, you don't need to change anything else.
But optionally, you can set the tolerance in millimeters at the beginning of the `Z_TRAMMING` macro.
```yaml
[gcode_macro Z_TRAMMING]
variable_mainsail: 0          # Set 1 if you're using Mainsail 2.9 or higher
variable_tolerance: 0.05      # <----- Set the tolerance in mm
variable_screw_pitch: 4       # Set lead screw pitch (See documentation for how to calculated).
variable_advanced_mode: 0     # If you do not know what this does, DO NOT CHANGE IT!
description: Measures the opposite ends of the X axis to determine if the Z screws are misaligned.
gcode:
```

### Configuring for another printer
You can use this macro in any cartesian printer with the same dual stepper single driver setup, **BUT SOME CHANGES ARE REQUIRED!**

The first changes are the bed size and probe location.
The probing is done right above the bed screws in order to minimize bed bending variations.
You can use any 2 points on the bed, but above the screws seams to be the most reliable to avoid bending variations.

If you have `screws_tilt_calculate` setup, you should already have the right values in your `printer.cfg`. Just select the correct screws and input the values as shown bellow.

If for some reason you don't have it yet, just home the printer, position the probe right above one of the screws and get the Y and X absolute position.
The Y position should be the same for both screws, just change the X axis until you're in the right spot.

With those in hand, there is only one measurement left, the lead screw pitch.

To measure this, put your printer on Z10 using `G1 Z10` or the UI, mark the position of the groove on the leadscrew coupler and raise the Z axis until you have a full turn.
Then, get the difference between the Z10 and the current position. This is your leadscrew pitch.

Now, you're gonna input those measurements in their respective places at the start of the `Z_TRAMMING` macro.
```yaml
[gcode_macro Z_TRAMMING]
variable_mainsail: 0          # Set 1 if you're using Mainsail 2.9 or higher
variable_tolerance: 0.05      # Set the tolerance in mm
variable_screw_pitch: 4       # <----- Set lead screw pitch (See documentation for how to calculated).
variable_advanced_mode: 0     # If you do not know what this does, DO NOT CHANGE IT!
description: Measures the opposite ends of the X axis to determine if the Z screws are misaligned.
gcode:
    # Points to probe (on the SV06+)
    {% set LEFT_X = 5 %}      # <----- Front left X screw position
    {% set RIGHT_X = 244 %}   # <----- Front right X screw position
    {% set Y_AXIS = 55 %}     # <----- Y axis screws position
    {% set SPEED = 6000 %}    # Travel speed between probes
```

You can also change the travel speed between probes as seen above, but it shouldn't make much of a difference.

## Advanced configuration
If you don't understand what is happening in this section, **DO NOT USE IT. IT MAY DAMAGE YOUR PRINTER!**

If you've done Gcode for marlin, you're probably familiar to the ideia of turning off and then homing a single axis.
Unfortunately, Klipper as of the writing of this does not support single axis disabling. Both `M18` and `M84` disable all steppers, even if only one is specified.

If you've understood this macro, you can already see the problem.
Every time you have to adjust the Z, every axis is disabled and has to be homed again.

That's were the `SET_STEPPER_ENABLE` command comes in, but it has it's caveats.
This command disables the stepper, but does not reset it's "homed" state. This means the printer thinks it knows were the stepper is, but it reality, it has been moved.
If you're interested on this, you can learn more on the original [issue on github](https://github.com/Klipper3d/klipper/issues/906) that lead to this command.

The implementation here is simple, but it can be dangerous if done wrong.

To activate it, you only need to change the `variable_advanced_mode` to `1`.

```yaml
[gcode_macro Z_TRAMMING]
variable_mainsail: 0          # Set 1 if you're using Mainsail 2.9 or higher
variable_tolerance: 0.05      # Set the tolerance in mm
variable_screw_pitch: 4       # Set lead screw pitch (See documentation for how to calculated).
variable_advanced_mode: 1     # <----- If you do not know what this does, DO NOT CHANGE IT!
description: Measures the opposite ends of the X axis to determine if the Z screws are misaligned.
```

With this done, the macro is ready, but you should **always run the macro until you have made it within tolerance**.

In the following code you can see why:
```yaml
{% if printer.toolhead.homed_axes == "xyz" %}
  G28 Z
{% else %}
  G28
{% endif %}
```

If the printed "is homed", the `SET_STEPPER_ENABLE` method has been used, so it homes only the Z axis.
If not, `M18` has been used and everything needs homing again.

If the printer is not homed on the Z axis again before moving, it may crash into the bed for believing it is higher than it actually is.
And it will only home Z again when it pass the tolerance check, only moving side to side to probe on every execution of the `Z_TRAMMING`.

This was done to cut back on iteration time when adjusting, and `M18` was set up as a failsafe option for the less tech savvy.
Keep that in mind and use it at your own risk. As I've said before, I'm not responsible for any damage cause to your printer.
If you don't understand this macro, **DO NOT USE IT!**

## Contributing
Contributions are welcomed. If you feel like you can improve on this or suggest improvements for me to implement you can open an issue.
I'm not guaranteed to implement it or merge it, but I'm open to suggestions.

Also, if you see a typo or maybe know a better way to explain this to new people, feel free to help with this documentation as well.
I'm just a single Brazilian trying to give back to this awesome community.