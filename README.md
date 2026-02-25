# 🥗 Salad Fork 180 - Klipper Configuration

> ⚠️ **Work in Progress:** This configuration is actively being developed and refined.

This documentation outlines the Klipper configuration for a **Salad Fork 180**. The configuration features highly dynamic macros, making it flexible, easy to maintain, and fail-safe.

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

## 🧠 Key Features & Smart Logic

This configuration leverages advanced Klipper features to fully automate and secure the printing process:

* **Centralized Variable Management (`_MY_VARS`)**
  All relevant parameters (temperatures, park positions, speeds) are managed in a single macro block. This prevents redundancy and simplifies future adjustments.
* **Advanced Thermal Management**
  The `PRINT_START` macro enforces a mechanical heat soak of the print bed before executing Z-tilt or bed meshing. The chamber temperature is controlled dynamically based on the bed temperature.
* **Active Heat Creep Protection (`_FAN_GUARD`)**
  A background loop continuously monitors the hotend fan RPM (via the tachometer pin). If the RPM drops below 7000 while the heater is active, the system triggers an emergency heater shutdown and pauses the print.
* **Safe Sensorless Homing**
  The TMC current for the X and Y motors is reduced to 0.50A prior to homing to minimize mechanical stress. An intelligent "blind Z-lift" protects the print surface if the Z-axis has not yet been homed.
* **Smart Filament Sensor Logic**
  To prevent false runout triggers caused by abrupt extrusion changes, the BTT SFS V2.0 is temporarily disabled during the start sequence and the prime line. It is safely re-enabled by the slicer starting at layer 2.
* **Automated VOC Filtration**
  The Nevermore filter activates automatically when printing high-temperature filaments (triggered by the bed temperature) and runs for an additional 10 minutes after the print finishes to clear residual VOCs.
* **Slicer Compatibility**
  Integrated dummy macros and command translations (G27, G29, M108, etc.) catch and prevent console spam or print aborts caused by hardcoded Marlin or RepRap commands sent by the slicer.

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
