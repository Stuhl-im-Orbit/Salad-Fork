# 🥗 Salad Fork 180 - Klipper Configuration

> ⚠️ **DISCLAIMER: UNTESTED & WORK IN PROGRESS**
> The physical printer is currently under construction. While this configuration has been thoroughly structured and logically verified, it has **not yet been physically tested on the actual hardware**. Use extreme caution, double-check all pin assignments, and verify kinematics before homing!

## 📑 Table of Contents
1. [Hardware Specifications](#%EF%B8%8F-hardware-specifications)
2. [Dependencies & Includes](#-dependencies--includes)
3. [Key Features & Smart Logic](#-key-features--smart-logic)
4. [Slicer Configuration Guide](#-slicer-configuration-guide)
5. [Reuse & Porting Guide](#-reuse--porting-guide)

---

## 🛠️ Hardware Specifications

| Component | Hardware / Model |
| :--- | :--- |
| **Mainboard** | Bigtreetech Manta M8P V2.0 |
| **Toolboard** | Bigtreetech EBB36 CAN V1.2.1 |
| **Toolhead** | A4T |
| **Probe** | Cartographer V4 (CAN Mode) |
| **Extruder** | WristWatch-G2 (LDO-36STH20-1004AHG-9T @ 1.00A Peak) |
| **Hotend** | Phaetus Rapido 2F Plus UHF |
| **Motors X/Y (A/B)** | Moons MS14HS5P4150 (1.5A Peak) |
| **Motors Z** | Moons LE174S-T0808-200-AR3-S-065 (0.65A Peak) |
| **Filament Sensor** | BTT SFS V2.0 |

---

## 🔗 Dependencies & Includes

This configuration relies on external configuration files. Ensure the following files are present in your Klipper configuration directory to prevent startup errors:
* `mainsail.cfg`: Required for Mainsail integration, web interface states, and standard variables.
* `toolhead_leds.cfg`: Contains the Neopixel definitions and macro logic required for the visual status feedback.

---

## 🧠 Key Features & Smart Logic

This configuration leverages advanced Klipper features and communication protocols to fully automate and secure the printing process. A core design principle is the strict separation of logic and execution.

### Architecture & Code Structure
* **Strict Jinja2 Namespace Isolation**
  All complex macros separate data retrieval, math, and logic from G-Code execution. Variables (`ref`), parameters (`param`), logic checks (`logic`), and kinematics (`pos`, `speed`) are defined upfront using strictly typed namespaces. This ensures predictable, crash-free execution.
* **Dual Variable Management (`_CLIENT_VARIABLE` & `_MY_VARS`)**
  The configuration strictly separates parameters into two dedicated macro blocks. `_CLIENT_VARIABLE` hooks directly into the official Mainsail standard macros (handling variables like pause/resume/cancel park positions and idle timeouts), while `_MY_VARS` specifically manages all custom logic for this machine (kinematics, custom temperatures, purge distances, timers). This prevents conflicts and keeps Mainsail updates seamless.

### Safety & Mechanical Protection
* **Inherent Boundary Checks**
  Movement macros inherently check physical limits before execution. For example, Z-hops utilize `[ref.th.position.z + param.z_hop, ref.th.axis_maximum.z]|min` to mathematically prevent crashes into the upper frame limits.
* **Safe Sensorless Homing**
  The TMC current for the X and Y motors is automatically reduced to 0.50A prior to homing to minimize mechanical stress on the drivetrain. It also includes relative back-off moves to clear the StallGuard registers safely.
* **Active Heat Creep Protection (`_FAN_GUARD`)**
  A background loop continuously monitors the hotend fan RPM (via the tachometer pin). If the RPM drops below a critical threshold while the heater is active, the system triggers an emergency heater shutdown and pauses the print to prevent hotend clogs.

### Commissioning & Calibration
* **Safe PID Tuning Macros (`PID_BED`, `PID_HOTEND`)**
  Inspired by the official Voron Commissioning guidelines. These automated macros safely home the printer (if required), move the toolhead to the exact center of the bed, and position the nozzle at a realistic Z-height (e.g., 5mm above the bed with the part cooling fan active at 25% for the hotend) before executing the native calibration. This ensures the PID values are tuned under realistic thermal airflow conditions.

### Thermal Management & Filtration
* **Advanced Thermal Soak**
  The `PRINT_START` macro enforces a mandatory mechanical heat soak of the print bed before executing Z-tilt or bed meshing to account for thermal expansion. 
* **Dynamic Chamber Control**
  The chamber temperature target is dynamically calculated based on the bed temperature if no explicit slicer parameter is provided (> 0°C). If the target bed temperature is 90°C or higher, the chamber defaults to 45°C; otherwise, it defaults to a safe 15°C to avoid endless waiting for cold prints.
* **Automated VOC Filtration**
  The Nevermore filter activates automatically when printing high-temperature filaments (triggered by the bed temperature threshold) and runs for an additional 10 minutes post-print to clear residual VOCs.

### Print Lifecycle & Automation
* **Slicer Compatibility & Dummy Macros**
  Integrated translation macros (M201, M203, M205, M900, G27, G32, etc.) catch hardcoded Marlin or RepRap commands sent by the slicer. They map limits safely to Klipper native commands or ignore them, preventing console spam and print aborts.
* **Smart Filament Sensor Logic**
  To prevent false runout triggers caused by abrupt extrusion changes, the BTT SFS V2.0 is temporarily disabled during the start sequence and the prime line. It is safely re-enabled by the slicer starting at layer 2.
* **Advanced Purge Sequence**
  Features a Prime Blob sequence (inspired by RatOS) to ensure optimal nozzle priming and clean breakaways before the print starts.
* **Comprehensive Idle Management**
  Handles automated toolhead parking, safe power-down of heaters and steppers, and the cancellation of pending timers (like filter cooldowns) during idle timeouts or print cancellations.
* **TMC Autotune Integration**
  Incorporates optimized motor registers automatically, tuning the Z-axis for silent operation and the XY/Extruder axes for maximum performance.

---

## 💻 Slicer Configuration Guide

To ensure the macros function correctly, parameters must be passed exactly as configured in your slicer.

### Orca Slicer & PrusaSlicer

**Machine Start G-Code:**
```gcode
M190 S0 ; Prevents slicer hardcoded heating
M109 S0 ; Prevents slicer hardcoded heating
SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature] CHAMBER=[chamber_temperature]
```

**Machine End G-Code:**
```gcode
PRINT_END
```

**Before Layer Change G-Code:**
```gcode
G92 E0
{if layer_num == 2}_TOGGLE_FILAMENT_SENSORS ENABLE=1{endif} ; Enable sensor after layer 2
```

**After Layer Change G-Code:**
```gcode
SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num + 1}
SET_DISPLAY_TEXT MSG="Layer {layer_num + 1}/[total_layer_count]"
```

### Ultimaker Cura

**Start G-Code:**
```gcode
PRINT_START BED_TEMP={material_bed_temperature_layer_0} EXTRUDER_TEMP={material_print_temperature_layer_0} CHAMBER={build_volume_temperature}
```

**End G-Code:**
```gcode
PRINT_END
```

**Layer Change & Sensor Toggle:**
Use the Post-Processing Script: "Insert at layer change"
1. When: "Before"
2. Layer: "3"
3. G-code: `_TOGGLE_FILAMENT_SENSORS ENABLE=1`

---

## 🔄 Reuse & Porting Guide

While not explicitly generated as a template, this configuration can easily serve as a robust blueprint or inspiration. Because the macros calculate positions automatically based on your axis limits, porting this setup to another CoreXY printer (like a Voron 2.4, Trident, or Micron) or resizing the build volume requires only a few manual adjustments.

> **DISCLAIMER:** Use at your own risk. No warranty or guarantee is provided. Always verify physical limits and kinematics. I assume you know what you are doing.

### 1. Mainsail Variables `[gcode_macro _CLIENT_VARIABLE]`
* `variable_custom_park_x`: Update to the new X-axis bed center.
* `variable_custom_park_y`: Update to the new Y-axis maximum.

### 2. Custom Variables `[gcode_macro _MY_VARS]`
* Check `variable_blob_feedrate` and purge distances. Make sure a smaller bed still provides enough physical space for the purge sequence.

### 3. Kinematics & Limits
* `[stepper_x]` / `[stepper_y]`: Adjust `position_max` and `position_endstop`.
* **Z-Steppers:** Add or remove stepper definitions based on your frame configuration (e.g., add `stepper_z3` for a Voron 2.4).

### 4. Leveling & Resonance
* **Kinematics:** Use `[z_tilt]` for 3 Z-motors or `[quad_gantry_level]` for 4.
* **Coordinates:** Update `z_positions` (exact physical motor/pivot points) and `points` (probe locations) to match your new bed geometry.
* `[bed_mesh]`: Set the new `zero_reference_position`, `mesh_min`, & `mesh_max`.
* `[resonance_tester]`: Change `probe_points` to the new bed center.

### 5. Hotend Fan Protection
* **No Tachometer Pin:** If you are not using a 3-pin hotend fan with a tachometer wire, you must remove or comment out the `tachometer_pin` definition in the `[heater_fan hotend_fan]` section. You can then simply delete the `[delayed_gcode _FAN_GUARD]` macro entirely, as the hardware monitoring will not function without that pin.

### 6. Firmware Retraction
* **Mandatory Requirement:** The `[firmware_retraction]` block must be defined and enabled in your configuration. Macros like `PRINT_END` and the filament management routines rely heavily on native `G10` / `G11` commands. Plausible default values (e.g., length: 0.8, speed: 60) are completely sufficient to avoid execution errors; extensive tuning is not required just for the macros to work safely.
