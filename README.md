# Salad Fork 180 — Klipper Configuration

> **Work in Progress**
> This printer and its configuration are currently under construction. Use these files entirely at your own risk — I assume no liability for any damage to hardware, property, or personal injury resulting from their use. Please review and verify all settings carefully before applying them to your machine.

---

## Contents

- [Hardware Specifications](#hardware-specifications)
- [Core Features](#core-features)
- [Slicer Configuration Guide](#slicer-configuration-guide)
  - [Orca Slicer, PrusaSlicer & SuperSlicer](#orca-slicer-prusaslicer--superslicer)
  - [Ultimaker Cura](#ultimaker-cura)
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
| Kinematics | CoreXY · Max 400 mm/s · Max 7500 mm/s² |
| Build Volume | 180 × 180 mm |
| Mainboard | Bigtreetech Manta M8P V2.0 |
| Toolboard | Bigtreetech EBB36 CAN V1.2.1 |
| Toolhead | A4T |
| Toolhead Fan | 3-Pin Delta 15000 RPM |
| Extruder | WristWatch-G2 (LDO-36STH20-1004AHG-9T · 1.00 A Peak) |
| Hotend | Phaetus Rapido 2F Plus UHF |
| Probe | Cartographer V4 (CAN Mode · Physical Touch Probing) |
| Motors A/B | Moons MS14HS5P4150 (1.5 A Peak) |
| Motors Z | 3× Moons LE174S-T0808-200-AR3-S-065 (0.65 A Peak · Z-Tilt capable) |
| Filament Sensor | BTT SFS V2.0 (Combined Switch & Motion Sensor) |

Documentation: [Salad Fork GitHub](https://github.com/Stuhl-im-Orbit/Salad-Fork)

---

## Core Features

**Centralized Variable Management**
All crucial speeds, positions, and parameters are defined in a single master macro (`_MY_VARS`) and dynamically calculated at system boot based on global printer limits.

**Safe Sensorless Homing (X/Y)**
Automatically reduces motor current and acceleration before tapping the physical limits to protect the hardware and prevent false StallGuard triggers.

**Smart Z-Leveling**
Uses the Cartographer in Touch Mode for exact Z-offset calibration. Pre-print routines include a 3-point `Z_TILT_ADJUST` and adaptive `BED_MESH_CALIBRATE`.

**Intelligent Filament Management**
Load and unload routines evaluate current thermal states. They maintain existing hotend temperatures or safely heat up to a fallback threshold if the hotend is cold.

**Hardware Safety Guard**
A continuous background loop (`_FAN_GUARD`) monitors the hotend fan RPM. In case of failure or heat creep danger, it forces a pause and shuts down the heater.

**Automated VOC Filtration**
The Nevermore filter automatically kicks in based on bed temperature thresholds (e.g., for ABS/ASA) and runs a 10-minute cooldown cycle after the print finishes.

**Dynamic Prime Blob**
Calculates the extrusion volume for the prime blob based on the configured nozzle diameter, making nozzle swaps hassle-free.

---

## Slicer Configuration Guide

To ensure the macros function correctly, parameters must be passed exactly as configured in your slicer. No manual temperature commands are required in the start G-code.

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
5. Add a `Search and Replace` script to enable the sensor at Layer 2:
   - **Search:** `;LAYER:1` *(Cura starts counting at 0)*
   - **Replace:** `;LAYER:1\n_TOGGLE_FILAMENT_SENSORS ENABLE=1`

---

## Macro Reference

### Print Lifecycle & State

| Macro | Description |
|---|---|
| `PRINT_START` | Main orchestration macro. Handles heat-soaking, homing, nozzle cleaning, Z-tilt, adaptive meshing, and the prime blob. Requires slicer parameters. |
| `PRINT_END` | Safely terminates the print. Retracts filament, disables heaters, parks the toolhead, and triggers the Nevermore cooldown timer. |
| `_PRINT_STATE` | Hidden variable macro that memorizes extruder temperatures across pauses and filament swaps. |
| `_HOOK_ON_PAUSE` | System hook that triggers notifications when a print is paused. |
| `_HOOK_ON_RESUME` | System hook that resets the printer state when resuming. |
| `_HOOK_ON_CANCEL` | System hook that resets background loops and begins the filter cooldown on cancel. |

### Extruder & Filament

| Macro | Description |
|---|---|
| `LOAD_FILAMENT` | Safely loads filament with predefined speeds, preserving the current hotend target temperature. |
| `UNLOAD_FILAMENT` | Ejects filament and includes a tip-shaping sequence to prevent stringing inside the extruder. |
| `M600` | Filament change routine. Pauses the print, memorizes the temperature, unloads the filament, and turns off the heater for safe extended idle times. |
| `PRIME_BLOB` | Purges the nozzle on the edge of the bed and cleanly snaps the filament tail before starting the print. |
| `CLEAN_NOZZLE` | Executes a precise wiping pattern across a fixed silicone brush to clean the nozzle before physical probing. |
| `_TOGGLE_FILAMENT_SENSORS` | Globally enables or disables the runout and motion sensors to prevent false triggers during purges or macros. |

### Movement & Homing

| Macro | Description |
|---|---|
| `homing_override` | Replaces the standard `G28` with a safe sequence. Performs Z-hops and ensures X and Y are homed sensorless before centering and homing Z with the Cartographer. |
| `_CONDITIONAL_HOME` | Checks if the printer is already homed. If yes, skips homing to save time; if no, triggers `G28`. |
| `_HOME_AXIS` | Helper macro that temporarily lowers motor currents and kinematic limits for safe sensorless homing. |
| `CENTER` | Moves the toolhead to the exact center of the build volume, ensuring a safe Z-hop is performed first. |

### Calibration & Tuning

| Macro | Description |
|---|---|
| `Z_TILT_ADJUST` | Wrapper for the standard Klipper command with added notifications. |
| `BED_MESH_CALIBRATE` | Wrapper for the standard Klipper command with added notifications. |
| `PID_BED` | Convenience macro for PID tuning the heated bed. Automatically homes and centers the toolhead. |
| `PID_HOTEND` | Convenience macro for PID tuning the extruder. Automatically homes, centers, and activates the part cooling fan for realistic thermal simulation. |

### Variables, Boot & Safety Loops

| Macro | Description |
|---|---|
| `_CLIENT_VARIABLE` | Configuration variables for the Mainsail web interface (e.g., custom park positions and idle timeouts). |
| `_MY_VARS` | Central configuration dictionary storing factors, hop heights, temperatures, and physical coordinates. |
| `_INIT_MY_VARS` *(delayed_gcode)* | Boot script executed 2 seconds after startup. Calculates dynamic speeds and center coordinates based on Klipper's hardware limits. |
| `_FAN_GUARD` *(delayed_gcode)* | Recursive safety loop running every 5 seconds. Checks the hotend fan RPM against temperature thresholds to prevent heat creep. |
| `_STOP_NEVERMORE` *(delayed_gcode)* | Timed macro to cleanly shut down the VOC filter after a defined cooldown period. |

### System Helpers

| Macro | Description |
|---|---|
| `_NOTIFY` | Pushes formatted messages to both the Mainsail UI and the Klipper console simultaneously. |
| `_RESET_STATE` | Clears speed factors, extrusion multipliers, bed meshes, and background loops to reset the printer to a clean state. |
| `DEBUG_VARS` | Prints all dynamically calculated boot variables (speeds, centers) to the console for verification. |

### Compatibility Aliases & Dummies

| Macro | Description |
|---|---|
| `G27` | Parks the toolhead *(Marlin)*. |
| `G29` | Executes `BED_MESH_CALIBRATE` *(Marlin)*. |
| `G32` | Homes the printer and executes `CARTOGRAPHER_TOUCH_HOME` *(RepRap)*. |
| `M108` | Dummy — prevents console spam for "Cancel Heating" *(Marlin)*. |
| `M116` | Dummy — prevents console spam for "Wait for all temperatures" *(RepRap)*. |
| `M201` | Dummy — prevents console spam for "Set Max Acceleration" *(Marlin)*. |
| `M203` | Dummy — prevents console spam for "Set Max Feedrate" *(Marlin)*. |
| `M205` | Dummy — prevents console spam for "Set Advanced Settings/Jerk" *(Marlin)*. |
| `M300` | Placeholder to intercept beep/tone commands *(Marlin/RepRap)*. |
| `M572` | Sets Pressure Advance *(RepRap)*. |
| `M900` | Sets Pressure Advance, routing to `M572` *(Marlin/Cura)*. |
| `M601` | Pauses the print *(PrusaSlicer/Marlin)*. |
| `M701` | Triggers `LOAD_FILAMENT` *(Marlin)*. |
| `M702` | Triggers `UNLOAD_FILAMENT` *(Marlin)*. |
