# 🥗 Salad Fork 180 - Klipper Configuration

This document provides a comprehensive overview of the Klipper configuration for the Salad Fork 180 3D printer. The setup is engineered for maximum reliability, hardware safety, and dynamic adaptability, utilizing a full CAN-bus ecosystem.

---

## 🖨️ Hardware Specifications

* 🏎️ **Kinematics:** CoreXY (Max Velocity: 400 mm/s, Max Accel: 7500 mm/s²)
* 📏 **Build Volume:** 180 x 180 mm
* 🧠 **Mainboard:** Bigtreetech Manta M8P V2.0
* 🔧 **Toolboard:** Bigtreetech EBB36 CAN V1.2.1
* 🚀 **Toolhead:** A4T
* 🌬️ **Toolhead Fan:** 3Pin Delta 15000 RPM
* ⚙️ **Extruder:** WristWatch-G2 (LDO-36STH20-1004AHG-9T, 1.00A Peak)
* 🔥 **Hotend:** Phaetus Rapido 2F Plus UHF
* 📡 **Probe:** Cartographer V4 (CAN Mode, Physical Touch Probing)
* 🏃 **Motors A/B:** Moons MS14HS5P4150 (1.5A Peak)
* ↕️ **Motors Z:** 3x Moons LE174S-T0808-200-AR3-S-065 (0.65A Peak, Z-Tilt capable)
* 🧵 **Filament Sensor:** BTT SFS V2.0 (Combined Switch & Motion Sensor)
* 📖 **Documentation:** [Salad Fork GitHub](https://github.com/Stuhl-im-Orbit/Salad-Fork)

---

## ✨ Core Features

* 🧠 **Centralized Variable Management:** All crucial speeds, positions, and parameters are defined in a single master macro (`_MY_VARS`) and dynamically calculated at system boot based on global printer limits.
* 🛑 **Safe Sensorless Homing (X/Y):** Automatically reduces motor current and acceleration before tapping the physical limits to protect the hardware and prevent false StallGuard triggers.
* 🎚️ **Smart Z-Leveling:** Utilizes the Cartographer in Touch Mode for exact Z-offset calibration. Pre-print routines include a 3-point `Z_TILT_ADJUST` and adaptive `BED_MESH_CALIBRATE`.
* 🌡️ **Intelligent Filament Management:** Load and unload routines evaluate current thermal states. They maintain existing hotend temperatures or safely heat up to a fallback threshold if the hotend is cold.
* 🛡️ **Hardware Safety Guard:** A continuous background loop (`_FAN_GUARD`) monitors the hotend fan RPM. In case of failure or heat creep danger, it forces a pause and shuts down the heater.
* 💨 **Automated VOC Filtration:** The Nevermore filter automatically kicks in based on bed temperature thresholds (e.g., for ABS/ASA) and runs a 10-minute cooldown cycle after the print finishes.
* 💧 **Dynamic Prime Blob:** Calculates the extrusion volume for the prime blob based on the configured nozzle diameter, making nozzle swaps hassle-free.
* 💡 **Dynamic RGB LED States:** The toolhead Neopixels visually indicate the current printer status (heating, homing, meshing, printing, ready, etc.).

---

## 🛠️ Macro Reference Guide

### 🎬 Print Lifecycle & State
* `PRINT_START`: The main orchestration macro. Handles heat-soaking, homing, nozzle cleaning, Z-tilt, adaptive meshing, and the prime blob. Requires slicer parameters.
* `PRINT_END`: Safely terminates the print. Retracts filament, disables heaters, parks the toolhead, and triggers the Nevermore cooldown timer.
* `_PRINT_STATE`: A hidden variable macro that memorizes extruder temperatures across pauses and filament swaps.
* `_HOOK_ON_PAUSE`: System hook that triggers LED states and notifications when a print is paused.
* `_HOOK_ON_RESUME`: System hook that resets LED states to printing when resuming.
* `_HOOK_ON_CANCEL`: System hook that ensures background loops are reset and the filter cooldown begins on cancel.

### 🧵 Extruder & Filament
* `LOAD_FILAMENT`: Safely loads filament with predefined speeds, preserving the current hotend target temperature.
* `UNLOAD_FILAMENT`: Ejects filament and includes a tip-shaping sequence to prevent stringing inside the extruder.
* `M600`: Standard filament change routine. Pauses the print, memorizes the temperature, unloads the filament, and turns off the heater for safe extended idle times.
* `PRIME_BLOB`: Purges the nozzle on the edge of the bed and cleanly snaps the filament tail before starting the print.
* `CLEAN_NOZZLE`: Executes a precise wiping pattern across a fixed silicone brush to clean the nozzle before physical probing.
* `_TOGGLE_FILAMENT_SENSORS`: Globally enables or disables the runout and motion sensors to prevent false triggers during purges or macros.

### 🧭 Movement & Homing
* `homing_override`: Replaces the standard G28 command with a safe sequence. Performs Z-hops and ensures X and Y are homed sensorless before centering and homing Z with the Cartographer.
* `_CONDITIONAL_HOME`: Checks if the printer is already homed. If yes, it skips homing to save time; if no, it triggers G28.
* `_HOME_AXIS`: The helper macro that temporarily lowers motor currents and kinematic limits for safe sensorless homing.
* `CENTER`: Moves the toolhead to the exact center of the build volume, ensuring a safe Z-hop is performed first.

### 📐 Calibration & Tuning
* `Z_TILT_ADJUST`: Wrapper for the standard Klipper command that adds LED state changes (yellow) and notifications.
* `BED_MESH_CALIBRATE`: Wrapper for the standard Klipper command that adds LED state changes (green) and notifications.
* `PID_BED`: Convenience macro for PID tuning the heated bed. Automatically homes and centers the toolhead.
* `PID_HOTEND`: Convenience macro for PID tuning the extruder. Automatically homes, centers, and activates the part cooling fan for realistic thermal simulation.

### ⚙️ Variables, Boot & Safety Loops
* `_CLIENT_VARIABLE`: Defines specific configuration variables utilized by the Mainsail web interface (e.g., custom park positions and idle timeouts).
* `_MY_VARS`: Central user-defined configuration dictionary storing factors, hop heights, temperatures, and physical coordinates.
* `_INIT_MY_VARS` *(delayed_gcode)*: Boot script executed 2 seconds after startup. Calculates dynamic speeds and center coordinates based on Klipper's hardware limits.
* `_FAN_GUARD` *(delayed_gcode)*: A recursive safety loop running every 5 seconds. Checks the hotend fan RPM against temperature thresholds to prevent heat creep.
* `SET_LEDS_ON_BOOT` *(delayed_gcode)*: Sets the initial toolhead LED state to 'ready' after the Klipper boot sequence finishes.
* `_STOP_NEVERMORE` *(delayed_gcode)*: Timed macro to cleanly shut down the VOC filter after a defined cooldown period.

### 🛠️ System Helpers
* `_NOTIFY`: Pushes formatted messages to both the Mainsail UI and the Klipper console simultaneously.
* `_RESET_STATE`: Clears speed factors, extrusion multipliers, bed meshes, and background loops to reset the printer to a clean state.
* `DEBUG_VARS`: Prints all dynamically calculated boot variables (speeds, centers) to the console for verification.
* `_SET_LED_STATE`: Controls the Neopixel colors based on predefined logical printer states.

### 🔌 Compatibility Aliases & Dummies
* `G27`: Parks the toolhead (Marlin compatibility).
* `G29`: Executes `BED_MESH_CALIBRATE` (Marlin compatibility).
* `G32`: Homes the printer and executes `CARTOGRAPHER_TOUCH_HOME` (RepRap compatibility).
* `M108`: Dummy macro to prevent console spam for "Cancel Heating" (Marlin compatibility).
* `M116`: Dummy macro to prevent console spam for "Wait for all temperatures" (RepRap compatibility).
* `M201`: Dummy macro to prevent console spam for "Set Max Acceleration" (Marlin compatibility).
* `M203`: Dummy macro to prevent console spam for "Set Max Feedrate" (Marlin compatibility).
* `M205`: Dummy macro to prevent console spam for "Set Advanced Settings/Jerk" (Marlin compatibility).
* `M300`: Placeholder to intercept standard beep/play tone commands (Marlin/RepRap compatibility).
* `M572`: Sets Pressure Advance (RepRap compatibility).
* `M900`: Sets Pressure Advance, routing to `M572` (Marlin/Cura compatibility).
* `M601`: Pauses the print (PrusaSlicer/Marlin compatibility).
* `M701`: Triggers the `LOAD_FILAMENT` macro (Marlin compatibility).
* `M702`: Triggers the `UNLOAD_FILAMENT` macro (Marlin compatibility).

---

## 💻 Slicer Configuration Guide

To ensure the macros function correctly, parameters must be passed exactly as configured in your slicer. No manual temperature commands are required in the start G-code.

### 🔪 Orca Slicer & PrusaSlicer

**Machine Start G-Code:**
```gcode
; Pass parameters to Klipper and let the macro handle everything
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature] CHAMBER=[chamber_temperature]
```

**Machine End G-Code:**
```gcode
; Trigger the end sequence
PRINT_END
```

**Before Layer Change G-Code:**
```gcode
; Reset extruder position at every layer change
G92 E0
; Safely enable filament sensor at layer 2
{if layer_num == 2}_TOGGLE_FILAMENT_SENSORS ENABLE=1{endif}
```

### 🔪 Ultimaker Cura

**Start G-Code:**
```gcode
; Pass parameters to Klipper and let the macro handle everything
PRINT_START BED_TEMP={material_bed_temperature_layer_0} EXTRUDER_TEMP={material_print_temperature_layer_0} CHAMBER={build_volume_temperature}
```

**End G-Code:**
```gcode
; Trigger the end sequence
PRINT_END
```

**Layer Change Handling in Cura:**
Cura does not natively feature a "Before Layer Change" text box in the standard Machine Settings. To trigger the filament sensors at layer 2 and reset the extruder, you have two options:

**Option 1: Using Post Processing Scripts (Recommended)**
1. Go to `Extensions` > `Post Processing` > `Modify G-Code`.
2. Add the script `Insert at layer change`.
3. Set **When to insert** to `Before`.
4. Set **G-code to insert** to:
   ```gcode
   ; Reset extruder position
   G92 E0
   ```
5. Add another `Search and Replace` script to enable the sensor specifically at Layer 2:
   * **Search:** `;LAYER:1` *(Note: Cura starts counting at 0)*
   * **Replace:** `;LAYER:1\n_TOGGLE_FILAMENT_SENSORS ENABLE=1`

**Option 2: Simplify via Start G-Code**
If you prefer not to use Post-Processing scripts, simply let the `PRIME_BLOB` macro enable the sensor at the very end of your pre-print routine (already configured in your `printer.cfg`), and skip the layer 2 conditional entirely.
