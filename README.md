# 🥗 Salad Fork 180 - Klipper Configuration

> ⚠️ **DISCLAIMER: UNTESTED & WORK IN PROGRESS**
> The physical printer is currently under construction. While this configuration has been thoroughly structured and logically verified, it has **not yet been physically tested on the actual hardware**. Use extreme caution, double-check all pin assignments, and verify kinematics before homing!

## 📑 Table of Contents
1. [Hardware Specifications](#hardware-specifications)
2. [Dependencies & Includes](#dependencies--includes)
3. [Key Features & Smart Logic](#key-features--smart-logic)
4. [Slicer Configuration Guide](#slicer-configuration-guide)
5. [Reuse & Porting Guide](#reuse--porting-guide)

---

## 🛠️ Hardware Specifications

| Component | Hardware / Model |
| :--- | :--- |
| **Mainboard** | Bigtreetech Manta M8P V2.0 |
| **Toolboard** | Bigtreetech EBB36 CAN V1.2.1 |
| **Toolhead** | A4T |
| **Toolhead Fan** | 3Pin Delta 15000 RPM |
| **Probe** | Cartographer V4 (CAN Mode) |
| **Extruder** | WristWatch-G2 (LDO-36STH20-1004AHG-9T @ 1.00A Peak) |
| **Hotend** | Phaetus Rapido 2F Plus UHF |
| **Motors X/Y (A/B)** | Moons MS14HS5P4150 (1.5A Peak) |
| **Motors Z** | Moons LE174S-T0808-200-AR3-S-065 (0.65A Peak) |
| **Filament Sensor** | BTT SFS V2.0 |

---

## 🔗 Dependencies & Includes

This configuration relies on external configuration files. Ensure the following file is present in your Klipper configuration directory to prevent startup errors:
* `mainsail.cfg`: Required for Mainsail integration, web interface states, and standard variables.

---

## 🧠 Key Features & Smart Logic

This configuration leverages advanced Klipper features and communication protocols to fully automate and secure the printing process.

### Architecture & Code Structure
* **Dual Variable Management:** The configuration strictly separates parameters. `_CLIENT_VARIABLE` hooks directly into the official Mainsail standard macros. `_MY_VARS` manages all custom logic for this machine (kinematics, purge distances, wiper coordinates).

### Safety & Mechanical Protection
* **Safe Sensorless Homing:** The TMC current for the X and Y motors is automatically reduced to 0.50A prior to homing to minimize mechanical stress on the drivetrain.
* **Active Heat Creep Protection:** A background loop (`_FAN_GUARD`) continuously monitors the hotend fan RPM. If the RPM drops below 7000 while the heater is active, the system triggers an emergency heater shutdown and pauses the print.
* **Automated Nozzle Cleaning:** The `CLEAN_NOZZLE` macro safely wipes the nozzle on a physical silicone comb, automatically heating the hotend to 150°C if it is too cold, protecting the silicone from tearing.

### Commissioning & Calibration
* **Safe PID Tuning Macros:** Automated macros (`PID_BED`, `PID_HOTEND`) safely home the printer, move the toolhead to the exact center of the bed, and position the nozzle at a realistic Z-height before executing the native calibration.
* **TMC Autotune Integration:** Incorporates optimized motor registers automatically, tuning the Z-axis for silent operation and the XY/Extruder axes for maximum performance.

### Thermal Management & Filtration
* **Advanced Thermal Soak:** The `PRINT_START` macro enforces a mandatory mechanical heat soak of the print bed before executing Z-tilt or bed meshing to account for thermal expansion. 
* **Dynamic Chamber Control:** The chamber temperature target is dynamically calculated. If the target bed temperature is 90°C or higher, the chamber defaults to 45°C; otherwise, it defaults to a safe 15°C.
* **Automated VOC Filtration:** The Nevermore filter activates automatically when printing high-temperature filaments and runs for an additional 10 minutes post-print to clear residual VOCs.

### Print Lifecycle & Automation
* **Slicer Compatibility:** Integrated translation macros catch hardcoded commands sent by the slicer, mapping limits safely to Klipper native commands to prevent console spam.
* **Smart Filament Sensor Logic:** To prevent false runout triggers caused by abrupt extrusion changes during the first layer, the dual BTT SFS V2.0 sensors are explicitly disabled at the end of the `PRINT_START` sequence. They must be safely re-enabled by the slicer starting at layer 2.

---

## 💻 Slicer Configuration Guide

To ensure the macros function correctly, parameters must be passed exactly as configured in your slicer. No manual temperature commands are required in the start G-code.

### Orca Slicer & PrusaSlicer

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
; Reset extruder and enable filament sensor safely after the first layer
G92 E0
{if layer_num == 2}_TOGGLE_FILAMENT_SENSORS ENABLE=1{endif}
```

### Ultimaker Cura

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

**Layer Change & Sensor Toggle:**
Use the Post-Processing Script: "Insert at layer change"
1. When: "Before"
2. Layer: "3" (Note: Cura starts counting differently than Prusa/Orca)
3. G-code: `_TOGGLE_FILAMENT_SENSORS ENABLE=1`

---

## 🔄 Reuse & Porting Guide

This configuration can serve as a robust blueprint. Because the macros calculate positions automatically based on your axis limits, porting this setup to another CoreXY printer requires only a few manual adjustments.

### 1. Mainsail Variables `[gcode_macro _CLIENT_VARIABLE]`
* `variable_custom_park_x`: Update to the new X-axis bed center.
* `variable_custom_park_y`: Update to the new Y-axis maximum.

### 2. Custom Variables `[gcode_macro _MY_VARS]`
* Check `variable_wipe_x` and `variable_wipe_y` to ensure the silicone comb coordinates match your physical hardware.
* Ensure a smaller bed still provides enough physical space for the `PRIME_BLOB` sequence.

### 3. Kinematics & Limits
* `[stepper_x]` / `[stepper_y]`: Adjust `position_max` and `position_endstop`.
* Add or remove Z-stepper definitions based on your frame configuration.

### 4. Leveling & Resonance
* **Kinematics:** Update `z_positions` (exact physical motor/pivot points) and `points` (probe locations) to match your new bed geometry.
* `[bed_mesh]`: Set the new `zero_reference_position`, `mesh_min`, & `mesh_max`.
* `[resonance_tester]`: Change `probe_points` to the new bed center.

### 5. Hotend Fan Protection
* If you are not using a 3-pin hotend fan with a tachometer wire, you must comment out the `tachometer_pin` definition in the `[heater_fan hotend_fan]` section and delete the `[delayed_gcode _FAN_GUARD]` macro.

### 6. Firmware Retraction
* The `[firmware_retraction]` block must be defined. Macros like `PRINT_END` and the filament management routines rely heavily on native `G10` / `G11` commands.
