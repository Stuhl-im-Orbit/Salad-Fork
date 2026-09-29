# Salad Fork 180: Klipper Configuration

> **Work in Progress**
> This printer and its configuration are currently under construction. Use these files entirely at your own risk. I assume no liability for any damage to hardware, property, or personal injury resulting from their use. Please review and verify all settings carefully before applying them to your machine.

---

## Contents

- [Hardware Specifications](#hardware-specifications)
- [Software Dependencies](#software-dependencies)
- [Core Features](#core-features)
- [Slicer Configuration Guide](#slicer-configuration-guide)
  - [Orca Slicer, PrusaSlicer & SuperSlicer](#orca-slicer-prusaslicer--superslicer)
  - [Ultimaker Cura](#ultimaker-cura)
- [Filament Change Workflow](#filament-change-workflow)
- [Macro Reference](#macro-reference)
  - [Print Lifecycle & State](#print-lifecycle--state)
  - [Extruder & Filament](#extruder--filament)
  - [Movement & Homing](#movement--homing)
  - [Calibration & Tuning](#calibration--tuning)
  - [Variables, Boot & Safety Loops](#variables-boot--safety-loops)
  - [System Helpers](#system-helpers)
  - [Compatibility Aliases & Dummies](#compatibility-aliases--dummies)

---

## Hardware Specifications

| | |
|---|---|
| Kinematics | CoreXY · Max 400 mm/s · Max 12000 mm/s² · Z max 10 mm/s, 50 mm/s² |
| Build Volume | 180 × 180 mm (axis travel X 182 mm, Y 195 mm, Z 145 mm) |
| Mainboard | Bigtreetech Manta M8P V2.0 |
| Toolboard | Bigtreetech EBB36 CAN V1.2.1 |
| Toolhead | A4T |
| Toolhead Fan | 3-Pin Delta 15000 RPM (tachometer monitored) |
| Extruder | WristWatch-G2 (LDO-36STH20-1004AHG-9T · 1.00 A Peak) |
| Hotend | Phaetus Rapido 2F Plus UHF |
| Hotend Sensor | PT1000 via MAX31865 on EBB36 (2-wire, 50 Hz filter) |
| Probe | Cartographer V4 (CAN Mode · Scan meshing · Physical Touch Probing) |
| Accelerometers | ADXL345 in Cartographer (used for resonance testing) and ADXL345 on EBB36 |
| Motors A/B | Moons MS14HS5P4150 (1.5 A Peak) |
| Motors Z | 3× Moons LE174S-T0808-200-AR3-S-065 (0.65 A Peak · Z-Tilt capable) |
| Filament Sensor | BTT SFS V2.0 (Combined Switch & Motion Sensor) |
| Chamber | Generic 3950 chamber thermistor · Nevermore VOC filter |
| Lighting | 3× Neopixel on toolhead (status colors) · PWM case light |

Documentation:
- [Salad Fork (PrintersForAnts)](https://github.com/PrintersForAnts/Salad_Fork)
- [Salad Fork (Stuhl-im-Orbit)](https://github.com/Stuhl-im-Orbit/Salad-Fork)

---

## Software Dependencies

The configuration requires the following components in addition to Klipper:

| Component | Purpose |
|---|---|
| [mainsail-config](https://github.com/mainsail-crew/mainsail-config) | Included via `[include mainsail.cfg]`. Provides `PAUSE`, `RESUME`, `CANCEL_PRINT` and `_TOOLHEAD_PARK_PAUSE_CANCEL`. |
| [Cartographer Klipper plugin](https://docs.cartographer3d.com/cartographer-probe/reference/configuration-reference) | Probe, scan mesh and touch Z offset (`CARTOGRAPHER_TOUCH_HOME`). |
| [klipper_tmc_autotune](https://github.com/andrewmcgr/klipper_tmc_autotune) | Driver tuning. X/Y/E use `performance`, Z uses `silent`. |
| [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) | Input shaper and belt analysis macros (`[shaketune]`). |

Further enabled Klipper modules: `exclude_object`, `force_move`, `gcode_arcs`, `respond`, `skew_correction`, `firmware_retraction`, `input_shaper`, `axis_twist_compensation`.

---

## Core Features

**Centralized Variable Management**
All crucial speeds, positions, and parameters are defined in a single master macro (`_MY_VARS`). At boot, `_INIT_MY_VARS` calculates the X/Y and Z travel speeds from `max_velocity` and `max_z_velocity` multiplied by the configured travel factors (default 0.8). The bed center is fixed at X 90 / Y 90.

**Safe Sensorless Homing (X/Y)**
Reduces the current of both CoreXY motors to 0.50 A and limits velocity and acceleration (50 mm/s, 1000 mm/s²) before tapping the physical limits to protect the hardware and prevent false StallGuard triggers.

**Smart Z-Leveling**
Uses the Cartographer in Touch Mode for exact Z-offset calibration. Pre-print routines include a 3-point `Z_TILT_ADJUST`, a Z re-home and an adaptive `BED_MESH_CALIBRATE`.

**Intelligent Filament Management**
Load and unload routines select the hotend temperature by priority: explicit `TEMP` parameter, temperature saved by `M600`, current hotend target, fallback of 220 °C.

**Hardware Safety Guard**
A background loop (`_FAN_GUARD`) checks the hotend fan RPM every 5 seconds while the extruder target is at least 50 °C and the temperature at least 60 °C. Below 7000 RPM it pauses a running print and switches off the hotend heater.

**Automated VOC Filtration**
The Nevermore filter is switched on in `PRINT_START` when the bed target is 90 °C or higher (ABS/ASA). After the print ends or is canceled, it runs for another 10 minutes, then stops.

**Chamber Heat Soak**
`PRINT_START` waits until the chamber sensor reaches the `CHAMBER` target. Without a slicer value the default is 45 °C for high-temperature prints and 15 °C otherwise.

**Dynamic Prime Blob**
Scales the extrusion volume for the prime blob by the nozzle cross-section relative to a 0.4 mm nozzle, based on the configured `nozzle_diameter`.

**Status LEDs**
The toolhead Neopixels indicate the current state (ready, heating, homing, leveling, meshing, cleaning, busy, printing).

---

## Slicer Configuration Guide

To ensure the macros function correctly, parameters must be passed exactly as configured in your slicer. No manual temperature commands are required in the start G-code.

Filament sensors are disabled at the end of `PRINT_START` and must be re-enabled by the slicer at layer 2. If this step is missing, runout detection stays inactive for the whole print.

### Orca Slicer, PrusaSlicer & SuperSlicer

**Machine Start G-Code**
```gcode
; Prevents slicer hardcoded heating sequences
M190 S0
M109 S0
; Send total layer count to Mainsail UI
SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]
; Pass parameters to Klipper and let the macro handle everything
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature] CHAMBER=[chamber_temperature]
```

**Machine End G-Code**
```gcode
PRINT_END
```

**Before Layer Change G-Code**
```gcode
; Reset extruder position at every layer change
G92 E0
; Safely enable filament sensor at layer 2
{if layer_num == 2}_TOGGLE_FILAMENT_SENSORS ENABLE=1{endif}
```

**After Layer Change G-Code**
```gcode
; Update current layer count in Mainsail UI and push message to display
SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num + 1}
SET_DISPLAY_TEXT MSG="Layer {layer_num + 1}/[total_layer_count]"
```

---

### Ultimaker Cura

**Start G-Code**
```gcode
PRINT_START BED_TEMP={material_bed_temperature_layer_0} EXTRUDER_TEMP={material_print_temperature_layer_0} CHAMBER={build_volume_temperature}
```

**End G-Code**
```gcode
PRINT_END
```

**Layer Change Handling**

Cura does not natively feature a "Before Layer Change" field in Machine Settings. To reset the extruder and enable filament sensors at layer 2, use the Post Processing plugin:

1. Go to `Extensions` > `Post Processing` > `Modify G-Code`.
2. Add the script `Insert at layer change`.
3. Set **When to insert** to `Before`.
4. Set **G-code to insert** to:
   ```gcode
   G92 E0
   ```
5. Add a `Search and Replace` script to enable the sensor at layer 2 and enable **Use Regular Expressions**:
   - **Search:** `;LAYER:1\n` *(Cura starts counting at 0)*
   - **Replace:** `;LAYER:1\n_TOGGLE_FILAMENT_SENSORS ENABLE=1\n`

   The trailing `\n` is required. A plain search for `;LAYER:1` also matches `;LAYER:10` to `;LAYER:19`, `;LAYER:100` and so on, which would corrupt those layer markers.

---

## Filament Change Workflow

1. `M600` (manually or triggered by a filament sensor) pauses the print, unloads the filament and switches off the hotend heater. The previous target temperature is stored in `_PRINT_STATE`.
2. Insert the new filament and run `LOAD_FILAMENT`. It reheats to the stored temperature, loads and purges, and leaves the hotend at that temperature.
3. Run `RESUME`. The Mainsail `RESUME` macro aborts if the hotend is not hot enough to extrude.

---

## Macro Reference

### Print Lifecycle & State

| Macro | Description |
|---|---|
| `PRINT_START` | Main orchestration macro. Resets state, enables the Nevermore for high-temperature prints, preheats the hotend to 150 °C, homes, heat-soaks bed and chamber, runs Z-tilt, re-homes Z, creates an adaptive mesh, calibrates the Z offset via Cartographer touch, heats to print temperature and runs the prime blob. Nozzle cleaning is prepared but currently commented out until the wiper is installed. Parameters: `BED_TEMP` (default 60), `EXTRUDER_TEMP` (default 200), `CHAMBER`. |
| `PRINT_END` | Retracts via firmware retraction, disables heaters and part fan, parks the toolhead, disables motors, re-enables filament sensors, resets state and starts the Nevermore cooldown. |
| `_PRINT_STATE` | Hidden variable macro that stores the extruder target temperature set by `M600` for the following `LOAD_FILAMENT` / `UNLOAD_FILAMENT`. |
| `_HOOK_ON_PAUSE` | Called by Mainsail `PAUSE`. Sends a notification and sets the LEDs to ready. |
| `_HOOK_ON_RESUME` | Called by Mainsail `RESUME`. Sends a notification and sets the LEDs to printing. |
| `_HOOK_ON_CANCEL` | Called by Mainsail `CANCEL_PRINT`. Re-enables filament sensors, resets state and starts the Nevermore cooldown. |

### Extruder & Filament

| Macro | Description |
|---|---|
| `LOAD_FILAMENT` | Heats by temperature priority (`TEMP`, saved `M600` temperature, current target, 220 °C fallback), loads 100 mm fast, purges 40 mm slowly and retracts. Optional parameter `TEMP`. |
| `UNLOAD_FILAMENT` | Heats by the same priority, performs a tip-shaping sequence and unloads 100 mm. Optional parameter `TEMP`. |
| `M600` | Filament change routine. Stores the target temperature, pauses the print, unloads the filament and turns off the hotend heater for safe extended idle times. |
| `PRIME_BLOB` | Purges a blob at the purge position (X 0 / Y 0), draws the string away and snaps it off with a fast X move. Parameter `SKIP_TRAVEL=1` skips the travel move. |
| `CLEAN_NOZZLE` | Wipes the nozzle across the silicone brush at the configured position, heating to at least 150 °C first. Currently not called by `PRINT_START`. |
| `_TOGGLE_FILAMENT_SENSORS` | Enables or disables the switch and motion sensor together (`ENABLE=0/1`) to prevent false triggers during purges or macros. |

### Movement & Homing

| Macro | Description |
|---|---|
| `homing_override` | Replaces the standard `G28`. Performs a Z-hop only if Z is already homed, homes X and Y sensorless, ensures X and Y are homed before Z, moves to the bed center and homes Z with the Cartographer. |
| `_CONDITIONAL_HOME` | Runs `G28` only if not all axes are homed. |
| `_HOME_AXIS` | Helper macro that temporarily lowers motor currents and velocity/acceleration limits for safe sensorless homing, then backs off 10 mm. |
| `CENTER` | Moves the toolhead to the bed center after a Z-hop limited to the Z maximum. |

### Calibration & Tuning

| Macro | Description |
|---|---|
| `Z_TILT_ADJUST` | Wrapper for the standard Klipper command with notification and LED state. |
| `BED_MESH_CALIBRATE` | Wrapper for the standard Klipper command with notification and LED state. |
| `PID_BED` | PID tuning for the heated bed. Homes if needed and centers the toolhead. Parameter `TEMP` (default 100). |
| `PID_HOTEND` | PID tuning for the extruder. Homes if needed, centers and runs the part cooling fan for realistic conditions. Parameters `TEMP` (default 245) and `FAN` (0 to 255, default 64). |

### Variables, Boot & Safety Loops

| Macro | Description |
|---|---|
| `_CLIENT_VARIABLE` | Configuration variables for the Mainsail macros: custom park position (X 90 / Y 170 / dZ 15), park on cancel, firmware retraction, 12 h idle timeout during pause, and the pause/resume/cancel hooks. |
| `_MY_VARS` | Central configuration dictionary storing factors, hop heights, temperatures, filament distances, homing current and physical coordinates. |
| `_INIT_MY_VARS` *(delayed_gcode)* | Boot script executed 2 seconds after startup. Calculates the travel speeds from the Klipper velocity limits and sets the center coordinates. |
| `SET_LEDS_ON_BOOT` *(delayed_gcode)* | Sets the toolhead LEDs to ready 1 second after startup. |
| `_FAN_GUARD` *(delayed_gcode)* | Recursive safety loop running every 5 seconds. Checks the hotend fan RPM to prevent heat creep. |
| `_STOP_NEVERMORE` *(delayed_gcode)* | Switches off the VOC filter after the cooldown period. |

### System Helpers

| Macro | Description |
|---|---|
| `_NOTIFY` | Pushes messages to both the display and the Klipper console. |
| `_RESET_STATE` | Resets speed and flow factors, clears the bed mesh, pause state, skew correction and Z G-code offset, restores the configured velocity limits and clears the saved `M600` temperature. |
| `_SET_LED_STATE` | Sets the toolhead Neopixel color by logical state (`STATE=`). |
| `_NEVERMORE_COOLDOWN` | Starts the 10-minute filter run if the Nevermore is active, otherwise only sends the notification. |

### Compatibility Aliases & Dummies

| Macro | Description |
|---|---|
| `G27` | Parks the toolhead *(Marlin)*. |
| `G29` | Executes `BED_MESH_CALIBRATE` *(Marlin)*. |
| `G32` | Homes if needed, runs `Z_TILT_ADJUST`, re-homes Z, creates an adaptive mesh and runs `CARTOGRAPHER_TOUCH_HOME` *(RepRap)*. |
| `M108` | Dummy: prevents console errors for "Cancel Heating" *(Marlin)*. |
| `M116` | Dummy: prevents console errors for "Wait for all temperatures" *(RepRap)*. |
| `M201` | Dummy: prevents console errors for "Set Max Acceleration" *(Marlin)*. |
| `M203` | Dummy: prevents console errors for "Set Max Feedrate" *(Marlin)*. |
| `M205` | Dummy: prevents console errors for "Set Advanced Settings/Jerk" *(Marlin)*. |
| `M300` | Dummy: intercepts beep/tone commands *(Marlin/RepRap)*. |
| `M572` | Sets Pressure Advance via `S` *(RepRap)*. |
| `M900` | Sets Pressure Advance via `K` or `S`, routed to `M572` *(Marlin/Cura)*. |
| `M601` | Pauses the print *(PrusaSlicer/Marlin)*. |
| `M701` | Triggers `LOAD_FILAMENT`, temperature from `S` or `T` *(Marlin)*. |
| `M702` | Triggers `UNLOAD_FILAMENT`, temperature from `S` or `T` *(Marlin)*. |
