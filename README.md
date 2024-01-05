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
When executed directly via the `Z_TRAMMING` command, the macro will home the printer, if not done yet, and proceed to probe both sides of the bed to determine the difference in height.
Then, it'll print to the console the defined tolerance, the current difference, the correction needed and will disable the steppers for the correction to be made.
The adjustment is done in hours and minutes, same as the SCREW_TILT_ADJUST macro. Meaning that "minutes" refers to "minutes of a clock face".
So, for example, 15 minutes is a quarter of a full turn. 01:20 means 1 full turn and 20 minutes. As for direction, CW means clockwise and CCW means counter-clockwise.
**NOTE:** The correction calculation is still a work in progress, as turning one side affects both in a way a don't completely understand yet. But having the way to turn it and the current height difference will help you understand what you got to do.

You should repeat the process several times until it is within the set tolerance.
You can check and set the tolerance together with the other configs in the [config section](#configuring-the-macro).

### Automatic Z alignment check
This second way to use it was made in order to automate the printing process.
To do that, you'd put the `Z_TRAMMING` macro into your `PRINT_START` macro.
The macro will then save the printing state, perform the same check as before, and proceed as normal if the the difference is within the set tolerance.

If it is out of tolerance, the printer will stop and show you the corrections needed through the console, same as before.
Once you've done the necessary corrections, you should run `Z_TRAMMING` again and the process will repeat.
Once the macro checks that everything is within tolerance, it'll re-home the printer and resume printing from the saved state.
**NOTE:** The saved state is only available during the current session. Any restart of the printer, mcu/firmware, klipper or power cycle would erase it. If that happens, just restart the print as if printing the first time.

## Configuring the macro
This script was made for the stock Sovol SV06 Plus, if that is your printer, the only change you could do is changing the tolerance.
The tolerance is set in milliliters at the beginning of the `Z_TRAMMING` macro.
```yaml
[gcode_macro Z_TRAMMING]
variable_tolerance: 0.05    # Set the tolerance in mm <-----
variable_screw_height: 4    # Set lead screw height
description: Measures the opposite ends of the X axis to determine if the Z screws are misaligned.
gcode:
```

### Configuring for another printer
You can use this macro in any cartesian printer with the same dual stepper single driver setup, **BUT SOME CHANGES ARE REQUIRED!**

The first changes are the bed size and probe location.
To calculate the probe points, first home your printer and get the middle of the Y axis. This number is probably gonna be different from the actual center of the bed, because we're centering the probe instead of the nozzle.

Next you're gonna get the X axis probe points.
Start from the side opposite to the probe, position it as far as you can without leaving the bed.
**NOTE:** If you're print head is capable of leaving the bed on both side, leave a safe margin for probing, based on the type of probe you have.

With the print head in place, you'll get the absolute X position and also the distance from the tip of the probe to the end of the bed.

In the opposite side, you'll position the print head in a way that the distance from the tip of the probe to the end of the bed is the same as the other side.
With the print head in place, get the absolute X position again.

With all of those in hand, there is only one measurement left, the lead screw pitch.
To measure this you'll need something to mark a position. Anything that is easy to hold still will work.

With your printer turned on and homed, put your printer on Z10 using `G1 Z10` or the UI, mark the position of the groove on the leadscrew coupler and raise the Z axis until you have a full turn.
Then, get the difference between the Z10 and the current position. This is your leadscrew pitch.

Now, you're gonna input those measurements in their respective places at the start of the `Z_TRAMMING` macro.
```yaml
[gcode_macro Z_TRAMMING]
variable_tolerance: 0.05    # Set the tolerance in mm
variable_screw_pitch: 4    # Set lead screw pitch (See documentation for how to calculated). <-----
description: Measures the opposite ends of the X axis to determine if the Z screws are misaligned.
gcode:
    # Points to probe (on the SV06+)
    {% set LEFT = 0 %} # Left most probe point <-----
    {% set RIGHT = 246 %} # Right equivalent probe point <-----
    {% set MIDDLE_Y = 170 %} # Middle of the bed on the Y axis <-----
    {% set SPEED = 6000 %} # Travel speed between probes
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

To activate it, you'll need to comment the `M18` command and uncomment the `SET_STEPPER_ENABLE` command on 2 place inside the `_Z_TRAMMING_EVALUATE` macro, inside the "Compare sides" section.

```jinja2
# Compare sides
    {% if left_probe - right_probe > tolerance %}
        {action_respond_info("--------------------------------------")}
        {action_respond_info("01:20 means 1 full turn and 20 minutes, CW=clockwise, CCW=counter-clockwise")}
        {% set hours = ((left_probe - right_probe) / screw_pitch)|round(1, "floor")|int %}
        {% set minutes = ((((left_probe - right_probe) / screw_pitch)* 60) % 60)|round(1, "floor")|int %}
        {action_respond_info("Right lead screw:" ~ hours ~ ":" ~ minutes ~ " CCW")}
        {action_respond_info("Right is " ~ '%0.2f'| format(left_probe - right_probe|float) ~ " mm higher")}
        M18 # <-----
        # Uncommenting the step below MAY DAMAGE YOUR PRINTER.
        # Make sure you have read the documentation and undestand the usage and the risks!
        # Use at your own risk.
        #SET_STEPPER_ENABLE STEPPER=stepper_z ENABLE=0 # <-----
        {action_respond_info("Steppers disabled and print paused. Run Z_TRAMMING again to resume.")}
        _Z_TRAMMING_ERROR
    {% elif left_probe - right_probe < tolerance * -1 %}
        {action_respond_info("--------------------------------------")}
        {action_respond_info("01:20 means 1 full turn and 20 minutes, CW=clockwise, CCW=counter-clockwise")}
        {% set hours = ((right_probe - left_probe) / screw_pitch)|round(1, "floor")|int %}
        {% set minutes = ((((right_probe - left_probe) / screw_pitch)* 60) % 60)|round(1, "floor")|int %}
        {action_respond_info("Right lead screw: " ~ hours ~ ":" ~ minutes ~ " CW")}
        {action_respond_info("Right is " ~ '%0.2f'| format(right_probe - left_probe|float) ~ " mm lower")}
        M18 # <-----
        # Uncommenting the step below MAY DAMAGE YOUR PRINTER.
        # Make sure you have read the documentation and undestand the usage and the risks!
        # Use at your own risk.
        #SET_STEPPER_ENABLE STEPPER=stepper_z ENABLE=0 # <-----
        {action_respond_info("Steppers disabled and print paused. Run Z_TRAMMING again to resume.")}
        _Z_TRAMMING_ERROR
    {% else %}
        {action_respond_info("--------------------------------------")}
        {% if left_probe > right_probe %}
            {action_respond_info("Right is " ~ '%0.2f'| format(left_probe - right_probe|float) ~ " mm higher")}
        {% else %}
            {action_respond_info("Right is " ~ '%0.2f'| format(right_probe - left_probe|float) ~ " mm lower")}
        {% endif %}
        {action_respond_info("Within tolerance.")}
    {% endif %}
    {action_respond_info("--------------Adjustment--------------")}
```

With this done, the macro is ready, but you should always run the macro until you have made it within tolerance.

In the following code you can see why:
```jinja2
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