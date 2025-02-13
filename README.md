*This readme is work in progress...*
# ERCF-Software-V3 "Happy Hare"
I love my ERCF and building it was the most fun I've had in many years of the 3D-printing hobby. Whilst the design is brilliant I found a few problems with the software and wanted to add some features and improve user friendliness.  This became especially true after the separation of functionality with the introduction of the "sensorless" branch. I liked the new python implementation as a Klipper plug-in but wanted to leverage my (very reliable) toolhead sensor.  So I rewrote the software behind ERCF - it still has the structure and much of the code of the original but, more significantly, it has many new features, integrates the toolhead sensor and sensorless options.  I'm calling it the **"Happy Hare"** release or v3.

## Major new features:
<ul>
<li>Support all options for both toolhead sensor based loading/unloading and the newer sensorless filament homing (no toolhead sensor)
<li>Supports sync load and unloading steps moving the extruder and gear motor together, including a config with toolhead sensor that can work with FLEX materials!
  <li>Fully implements “EndlessSpool” with new concept of Tool --> Gate mapping.  This allows empty gates to be identified and tool changes subsequent to runout to use the correct filament spool.  It has the added advantage for being able to map gates to tools in case of slicing to spool loading mismatch.
<li>Measures “spring” in filament after extruder homing for more accurate calibration reference
<li>Adds servo_up delay making the gear to extruder transition of filament more reliable (maintains pressure)
<li>Ability to secify empty or disabled tools (gates).
<li>Formal support for the filament bypass block with associated new commands and state if using it.
<li>Ability to reduce gear current (currently TMC2209 only) during “collision” homing procedure to prevent grinding, etc.
</ul>

## Other features:
<ul>
<li>Optional fun visual representation of loading and unloading sequence
<li>Reworks calibration routine to average measurements, add compensation based on spring in filament (related to ID and length of bowden), and considers configuration options.
<li>Runtime configuration via new command (ERCF_TEST_CONFIG) for most options which avoids constantly restarting klipper or recalibrating during setup
<li>Workarond to some of the ways to provoke Klipper “Timer too close” errors (although there are definitely bugs in the Klipper firmware)
<li>More reliable “in-print” detection so tool change command “Tx” g-code can be used anytime and the user does not need to resort to “ERCF_CHANGE_TOOL_STANDALONE”
<li>New LOG_LEVEL=4 for developer use.  BTW This is useful in seeing the exact stepper movements
<li>Experimental logic to use stallguard filament homing (Caveat: not easy to setup using EASY-BRD and not compatible with sensorless selector homing option)
</ul>
  
## Other benefits of the code cleanup / rewrite:
<ul>
<li>Vastly increased error detection/checking.
<l1>Consistent handling of errors. E.g. use exceptions to avoid multiple calls to _pause()
<li>Wrapping of all stepper movements to facilitate “DEVELOPER” logging level and easier debugging
<li>Renewed load and unload sequences (to support all build configurations) and effectively combine the sensor and sensorless logic
</ul>
 
<br>
  To try it out I recommend you save your old configuration and then take the supplied `ercf_parameters.cfg` file as a starting point and edit back some of your known settings.  The replace ercf.py in the Klipper extra folder and restart. 
<br>

## Summary of new commands:
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_STATUS | Report on ERCF state, cababilities and Tool-to-Gate map | DETAIL=\[0\|1\] Displays TTG map and gate status (automatic if EndlessSpool is  configured) |
  | ERCF_TEST_CONFIG | Dump / Change essential load/unload config options at runtime | Many. Best to run ERCF_TEST_CONFIG without options to report all parameters that can be specified |
  | ERCF_DISPLAY_TTG_MAP | Displays the current Tool - to - Gate mapping (can be used all the time but generally designed for EndlessSpool  | DETAIL=\[0\|1\] Whether to also show the gate availability |
  | ERCF_REMAP_TTG | Reconfiguration of the Tool - to - Gate (TTG) map.  Can also set gates as empty! | TOOL=\[0..n\] <br>GATE=\[0..n\] Maps specified tool to this gate (multiple tools can point to same gate) <br>AVAILABLE=\[0\|1\]  Marks gate as available or empty |
  | ERCF_SELECT_BYPASS | Unload and select the bypass selector position if configured | None |
  | ERCF_LOAD_BYPASS | Does the extruder loading part of the load sequence - designed for bypass filament loading | None |
  | ERCF_TEST_HOME_TO_EXTRUDER | For calibrating extruder homing - TMC current setting, etc. | RETURN=\[0\|1\] Whether to return the filament to the approximate starting position after homing - good for repeated testing |
  
  Note that some existing commands have been enhanced a little.  See the [command reference](#ercf-command-reference) at the end of this page.
  
<br>

## New features in detail:
### Config Loading and Unload sequences explained
Note that if a toolhead sensor is configured it will become the default filament homing method and home to extruder an optional but unecessary step. Also note the home to extruder step will always be performed during calibration of tool 0 (to accurately set `ercf_calib_ref`) regardless of the `home_to_extruder` setting. For accurate homing and to avoid grinding, tune the gear stepper current reduction `extruder_homing_current` as a % of the default run current.

#### Possible loading options (ercf_parameters.cfg configuration) WITH toolhead sensor:

    Extruder homing config          Toolhead homing config     Notes
    ----------------------          ----------------------     -----
    
    1. home_to_extruder=0           toolhead_homing_max=20     This is probably the BEST option and can work with FLEX
                                    toolhead_homing_step=1     Filament can load close to extruder gear, then is pulled
                                    sync_load_length=1         through to home on toolhead sensor by synchronized gear
                                                               and extruder motors
    
    2. home_to_extruder=0           toolhead_homing_max=20     Not recommended but can avoid problems with sync move.  The
                                    toolhead_homing_step=1     initial load to end of bowden must press the filament to
                                    sync_load_length=0         create spring so that extruder will pick up the filament
    
    3. home_to_extruder=1           toolhead_homing_max=20     Not recommended. The filament will be rammed against extruder
       extruder_homing_max=50       toolhead_homing_step=1     to home and then synchronously pulled through to home again on
       extruder_homing_step=2       sync_load_length=1         toolhead sensor (no more than 20mm away in this example)
       extruder_homing_current=50
    
    4. home_to_extruder=1           toolhead_homing_max=20     A bit redundant to home twice but allows for reliable filament
       extruder_homing_max=50       toolhead_homing_step=1     pickup by extruder, accurate toolhead homing and avoids 
       extruder_homing_step=2       sync_load_length=0         problem problems with sync move (Timer too close)
       extruder_homing_current=50

#### Possible loading options WITHOUT toolhead sensor:
    5. home_to_extruder=1          sync_load_length=10         BEST option without a toolhead sensor.  Filament is homed to
       extruder_homing_max=50                                  extruder gear and then the initial move into the extruder is
       extruder_homing_step=2                                  synchronised for accurate pickup
       extruder_homing_current=50
  
    6. home_to_extruder=1          sync_load_length=0          Same as above but avoids the synchronous move.  Can be
       extruder_homing_max=50                                  reliable with accurate calibration reference length and
       extruder_homing_step=2                                  accurate encoder
       extruder_homing_current=50

*Obviously the actual distances shown above may be customized*
  
  **Advanced options**
When not using synchronous load move the spring tension in the filament held by servo will be leverage to help feed the filament into the extruder. This is controlled with the `delay_servo_release` setting. It defaults to 2mm and is unlikely that it will need to be altered. An option to home to the extruder using stallguard `homing_method=1` is avaiable but not recommended: (i) it is not necessary with current reduction, (ii) it is not readily compatible with EASY-BRD and (iii) is currently incompatible with sensorless selector homing which hijacks the gear endstop configuration.
  
  **Note about post homing distance**
Regardless of loading settings above it is important to accurately set `home_to_nozzle` distance.  If you are not homing to the toolhead sensor this will be from the extruder entrance to nozzle.  If you are hoing to toolhead sensor, this will be the (smaller) distance from sensor to nozzle.  For example in my setup of Revo & Clockwork 2, the distance is 72mm or 62mm respectively.
  
#### Possible unloading options:
This is much simplier than loading. The toolhead sensor, if installed, will automatically be leveraged as a checkpoint when extracting from the extruder.
`sync_unload_length` controls the mm of synchronized movement at start of bowden unloading.  This can make unloading more reliable and will act as what Ette refers to as a "hair pulling" step on unload.  This is an optional step, set to 0 to disable.
  
<br>

### Tool-to-Gate (TTG) mapping and EndlessSpool application
When changing a tool with the `Tx` command ERCF would by default select the filament at the gate (spool) of the same number.  The mapping built into this *Angry Hare* driver allows you to modify that.  There are 3 primarly use cases for this feature:
<ol>
  <li>You have loaded your filaments differently than you sliced gcode file. No problem, just issue the appropriate remapping commands prior to printing
  <li>Some of "tools" don't have filament and you want to mark them as empty
  <li>Most importantly, for EndlessSpool - when a filament runs out on one gate (spool) then next in the sequence is automatically mapped to the original tool.  It will therefore continue to print on subsequent tool changes.  You can also replace the spool and update the map to indicate avaiablity mid print
</ol>

*Note that the initial availability of filament at each gate can also be specfied in the `ercf_parameters.cfg` file by updating the `gate_status` list. E.g.
>gate_status = 1, 1, 0, 0, 1, 0, 0, 0, 1

  on a 9-gate ERCF would mark gates 2, 3, 5, 6 & 7 as empty
 
To view the current mapping you can use either `ERCF_STATUS DETAIL=1` or `ERCF_DISPLAY_TTG_MAP`
  
![ERCF_STATUS](doc/ercf_status.png "ERCF_STATUS")

<br>
  
### Visualization of filament position
  The `log_visual` setting turns on an off the addition of a filament tracking visualization. Can be nice with log_level of 0 or 1 on a tuned and functioning setup.
  
![Bling is always better](doc/visual_filament.png "Visual Filament Location")
  
<br>

### Filament bypass
If you have installed the optional filament bypass block your can configure its selector position by setting `bypass_selector` in `ercf_parameters.cfg`. Once this is done you can use the following command to unload any ERCF controlled filament and select the bypass:
  > ERCF_SELECT_BYPASS
  
  Once you have filament loaded upto the extruder you can load the filament to nozzle with
  > ERCF_LOAD_BYPASS

### Adjusting configuration at runtime
  All the essential configuration and tuning parameters can be modified at runtime without restarting Klipper. Use the `ERCF_TEST_CONFIG` command to do this:
  
  <img src="doc/ercf_test_config.png" width="500" alt="ERCF_TEST_CONFIG">
  
  Any of the displayed config settings can be modifed.  E.g.
  > ERCF_TEST_CONFIG home_position_to_nozzle=45
  
  Will update the distance from homing postion to nozzle.  The change is designed for testing was will not be persistent.  Once you find your tuned settings be sure to update `ercf_parameters.cfg`
  
### Updated Calibration Ref
  Setting the `ercf_calib_ref` is slightly different in that it will, by default, average 3 runs and compsensate for spring tension in filament held by servo. It might be worth limiting to a single pass until you have tuned the gear motor current. Here is an example:
  
  <img src="doc/Calibration Ref.png" width="500" alt="ERCF_CALIBRATION_SINGLE TOOL=0">
  
<br>

## My Testing:
  This software is largely rewritten as well as being extended and so, despite best efforts, has probably introducted some bugs that may not exist in the official driver.  It also lacks extensive testing on different configurations that will stress the corner cases.  I have been using successfully on Voron 2.4 / ERCF with EASY-BRD.  I use a self-modified CW2 extruder with foolproof microswitch toolhead sensor. My day-to-day configuration is to load the filament to the extruder in a single movement (num_moves=1), then home to toolhead sensor with synchronous gear/extruder movement (option #1 explained above).  I use the sensorless selector and have runout and EndlessSpool enabled.
  
> Klipper Host Version: v0.10.0-594
> <br>Primary MCU Klipper version: v0.10.0-594
> <br>EASY-BRD firmware version: v0.10.0-220

<br>

### My Setup:
<img src="doc/My Voron 2.4 and ERCF.jpg" width="400" alt="My Setup">

### Some setup notes based on my learnings:
<ul>
  <li>Firstly the importance of a reliable and fairly accurate encoder should not be under estimated. If you cannot get very reliable results from `ERCF_CALIBRATE_ENCODER` then don't proceed with setup - address the encoder problem first. Note that I had really good luck with this https://discord.com/channels/460117602945990666/909743915475816458/1023873076095615036 approach of blackening the encoder wheel and then adjusting the sensor is little *further* away from the gear.
  <li>If using a toolhead sensor, that must be reliable too.  The hall effect based switch is very awkward to get right because of so many variables: strength of magnet, amount of iron in washer, even temperature, therefore I strongly recommend a simple microswitch based detection.  They work first time, every time.
  <li>Eliminate all points of friction in the filament path.  There is lots written about this already but I found some unusual places where filament was rubbing on plastic and drilling out the path improved things a good deal.
  <li>This version of the driver software both, compensates for, and exploits the spring that is inherently built when homing to the extruder.  The `ERCF_CALIBRATE_SINGLE TOOL=0` (which calibrates the *ercf_calib_ref* length) averages the measurement of multiple passes, measures the spring rebound and considers the configuration options when recommending and setting the ercf_calib_ref length.  If you change basic configuration options it is advisable to rerun this calibration step again.
  <li>The dreaded "Timer too close" can occur but I believe I have worked around these cases.  The problem is not always an overloaded mcu as often cited -- there are a couple of bugs in Klipper that will delay messages between mcu and host and thus provoke this problem.
  <li>The servo problem where a servo with move to end position and then jump back can occur due to bug in Klipper just like the original software. The workaroud is increase the same servo "dwell" config options in small increments until the servo works reliably.
  <li>I highly recommend Ette's "sensorless selector" option -- it works well and provides for additional recovery abilities if filment gets stuck in encoder preventing selection of a different gate.
</ul>

Good luck and hopefully a little less *enraged* printing.  You can find me on discord as *moggieuk#6538*

  
  ---
  
# ERCF Command Reference
  
  *Note that some of these commands have been enhanced from the original*

  ## Logging and Stats
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_RESET_STATS | Reset the ERCF statistics | None |
  | ERCF_DUMP_STATS | Dump the ERCF statistics | None |
  | ERCF_SET_LOG_LEVEL | Sets the logging level and turning on/off of visual loading/unloading sequence | LEVEL=\[1..4\] <br>VISUAL=\[0\|1\] Whether to also show visual representation |
  | ERCF_STATUS | Report on ERCF state, cababilities and Tool-to-Gate map | DETAIL=\[0\|\1] Displays TTG map and gate status (automatic if EndlessSpool is  configured) |
  | ERCF_DISPLAY_ENCODER_POS | Displays the current value of the ERCF encoder | None |
  <br>

  ## Calibration
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_CALIBRATE | Complete calibration of all ERCF tools | None |
  | ERCF_CALIBRATE_SINGLE | Calibration of a single ERCF tool | TOOL=\[0..n\] <br>REPEATS=\[1..10\] How many times to repeat the calibration for reference tool T0 (ercf_calib_ref) |
  | ERCF_CALIB_SELECTOR | Calibration of the selector for the defined tool | TOOL=\[0..n\] |
  | ERCF_CALIBRATE_ENCODER | Calibration routine for ERCF encoder | DIST=.. Distance to measure over. Longer is better, defaults to calibration default length <br>RANGE=.. Number of times to average over <br>SPEED=.. Speed of gear motor move. Defaults to long move speed <br>ACCEL=.. Accel of gear motor move. Defaults to motor setting in ercf_hardware.cfg |
  <br>

  ## Servo and motor control
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_SERVO_DOWN | Enguage the ERCF gear | None |
  | ERCF_SERVO_UP | Disengage the ERCF gear | None |
  | ERCF_MOTORS_OFF | Turn off both ERCF motors | None |
  | ERCF_BUZZ_GEAR_MOTOR | Buzz the ERCF gear motor and report on whether filament was detected | None |
  <br>

  ## Core ERCF functionality
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_UNLOCK | Unlock ERCF operations | None |
  | ERCF_HOME | Home the ERCF selector and optionally selects gate associated with the specified tool | TOOL=\[0..n\] |
  | ERCF_SELECT_TOOL | Selects the gate associated with the specified tool | TOOL=\[0..n\] The tool to be selected (technically the gate associated with this tool will be selected) |
  | ERCF_SELECT_BYPASS | Unload and select the bypass selector position if configured | None |
  | ERCF_LOAD_BYPASS | Does the extruder loading part of the load sequence - designed for bypass filament loading | None |
  | ERCF_CHANGE_TOOL | Perform a tool swap (generally called from 'Tx' macros) | TOOL=\[0..n\] |
  | (ERCF_CHANGE_TOOL_STANDALONE) | Deprecated, 'ERCF_TOOL_CHANGE' can handle. Was: Perform a tool swap outside of print | TOOL=\[0..n\] |
  | ERCF_EJECT | Eject filament and park it in the ERCF | None |
  | ERCF_PAUSE | Pause the current print and lock the ERCF operations | None |
  <br>

  ## User Testing
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_TEST_GRIP | Test the ERCF grip of the currently selected tool | None |
  | ERCF_TEST_SERVO | Test the servo angle | VALUE=.. Angle value sent to servo |
  | ERCF_TEST_MOVE_GEAR | Move the ERCF gear | LENGTH=..\[200\] Length of gear move in mm <br>SPEED=..\[50\] Stepper move speed50 <br>ACCEL=..\[200\] Gear stepper accel |
  | ERCF_TEST_LOAD_SEQUENCE | Soak testing of load sequence. Great for testing reliability and repeatability| LOOP=..\[10\] Number of times to loop while testing <br>RANDOM=\[0 \|1 \] Whether to randomize tool selection <br>FULL=\[0 \|1 \] Whether to perform full load to nozzle or short load just past encoder |
  | ERCF_TEST_LOAD | Test loading filament | LENGTH=..[100] Test load the specified length of filament into selected tool |
  | (ERCF_LOAD) | Identical to ERCF_TEST_LOAD | |
  | ERCF_TEST_UNLOAD | Move the ERCF gear | LENGTH=..[100] Lenght of filament to be unloaded <br>UNKNOWN=\[0\|1\] Whether the state of the extruder is known. Generally 0 for standalone use, 1 simulates call as if it was from slicer when tip has already been formed |
  | ERCF_TEST_HOME_TO_EXTRUDER | For calibrating extruder homing - TMC current setting, etc. | RETURN=\[0\|1\] Whether to return the filament to the approximate starting position after homing - good for repeated testing |
  | ERCF_TEST_CONFIG | Dump / Change essential load/unload config options at runtime | Many. Best to run ERCF_TEST_CONFIG without options to report all parameters than can be specified |
  <br>

  ## Tool to Gate map  and Endless spool
  | Commmand | Description | Parameters |
  | -------- | ----------- | ---------- |
  | ERCF_ENCODER_RUNOUT | Filament runout handler that will also implement EndlessSpool if enabled | None |
  | ERCF_DISPLAY_TTG_MAP | Displays the current Tool -> Gate mapping (can be used all the time but generally designed for EndlessSpool  | DETAIL=\[0\|1\] Whether to also show the gate availability |
  | ERCF_REMAP_TTG | Reconfiguration of the Tool - to - Gate (TTG) map.  Can also set gates as empty! | TOOL=\[0..n\] <br>GATE=\[0..n\] Maps specified tool to this gate (multiple tools can point to same gate) <br>AVAILABLE=\[0\|1\]  Marks gate as available or empty |
  | ERCF_RESET_TTG_MAP | Reset the Tool-to-Gate map back to default | None |
  
