# Voron 2.4 StealthChanger

Klipper configuration for my Voron 2.4 (300 × 300 mm), converted into a toolchanger with [StealthChanger](https://github.com/DraftShift/StealthChanger) and 4 toolheads.

This repository is a backup of `~/printer_data/config` on the printer, pushed manually with [Klipper-Backup](https://github.com/Staubgeborener/klipper-backup) via the `UPDATE_GIT` macro (no automatic or on-boot backups, only at known-good states).

---

## Hardware

| Area | Component |
|---|---|
| Frame | Voron 2.4, 300 × 300 mm |
| Mainboard | BTT Octopus (STM32F446) |
| Toolchanger | StealthChanger, 4 docks |
| Toolhead boards | LDO Nitehawk 36 StealthChanger Edition (one per tool, USB) |
| XY | 0.9° steppers, X sensorless homing, Y with endstop |
| Z | 4 × Z motors (Quad Gantry Level), homing via tool probe (Tap) |
| Heated bed | 230 V AC |
| Tool calibration | Sexball probe, evaluated with Axiscope |
| Chamber | "The Filter" mod with 4 bed fans, chamber temperature sensor |

### Toolheads

| Tool | Extruder | Hotend | Use |
|---|---|---|---|
| T0 | Orbiter 2.5 | Rapido 2 HF | Standard material, reference tool |
| T1 | Orbiter 2.5 | Rapido 2 HF | Standard material |
| T2 | Orbiter 2.0 | Rapido 2 HF | Support |
| T3 | Galileo 2 Standalone (9:1) | Dragon HF Voron Edition | Abrasive materials |

Each tool has its own ADXL345, Tap probe, hotend fan with tachometer, input shaper values and XYZ offsets relative to T0.

---

## Software

- [Klipper](https://github.com/Klipper3d/klipper) with [klipper-toolchanger-easy](https://github.com/jwellman80/klipper-toolchanger-easy)
- Mainsail / Fluidd, Moonraker, Crowsnest, Sonar
- [Axiscope](https://github.com/nic335/Axiscope) for XY and Z tool offsets
- [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) for input shaping
- Spoolman with per-tool spool tracking
- OrcaSlicer (pressure advance is set per filament profile in Orca, `0.025` in the config is only the fallback)
- [Klipper-Backup](https://github.com/Staubgeborener/klipper-backup) + `gcode_shell_command` (installed via KIAUH)

---

## Config structure

```
printer_data/config/
├── printer.cfg               # Main config, includes everything + SAVE_CONFIG block (skew, axis twist)
├── moonraker.conf            # Moonraker, update manager, Spoolman
├── crowsnest.conf            # Camera
├── sonar.conf                # WiFi keepalive
├── saved_variables.cfg       # Spoolman spool assignment per tool
├── config/
│   ├── machine.cfg           # idle_timeout, respond, exclude_object, arcs, skew, save_variables
│   ├── mcus/                 # BTT Octopus + pin aliases
│   ├── kinematics/           # CoreXY, velocity and acceleration limits
│   ├── hardware/             # XY, Z, heated bed, fans, filter, chamber sensor
│   ├── software/             # QGL, bed mesh, axis twist, resonance tester, Shake&Tune, Axiscope, Spoolman
│   └── macros/               # see "Macros" below
└── toolchanger/
    ├── toolchanger-config.cfg  # Own overrides (dock paths, homing, M109 deadband)
    ├── readonly-configs/       # Stock klipper-toolchanger-easy files (not in this backup)
    └── tools/                  # T0–T3: MCU, extruder, fans, probe, LEDs, offsets, tool macro
```

Per-tool values (PID, offsets, input shaper) are maintained directly in `tools/Tx.cfg`, not via `SAVE_CONFIG`.

**Not included in the backup:** `mainsail.cfg` and `toolchanger/readonly-configs/` are stock files from mainsail-config and klipper-toolchanger-easy. For a restore, install both first, then copy this config.

---

## Macros

| File | Macro(s) | Purpose |
|---|---|---|
| `print_start.cfg` | `PRINT_START`, `_TOOL_FIRST_USE` | Heating, homing, calibration, material Z offset, prime line |
| `print_end.cfg` | `PRINT_END` | Retract, return to T0, park, reset everything |
| `load_filament.cfg` | `LOAD_FILAMENT` | Load and purge into the active tool |
| `unload_filament.cfg` | `UNLOAD_FILAMENT` | Tip shaping and unload from the active tool |
| `hotend_fan_monitor.cfg` | `_HOTEND_FAN_CHECK`, `_HOTEND_FAN_FAULT` | Watches the hotend fan RPM of all tools |
| `filament_ooze_calibration.cfg` | `OOZE_TEST` | Extrude at a set flow, then retract (to find the retract length) |
| `dock_tuner.cfg` | `DOCK_TUNER` | Step through pickup/dropoff paths to tune the docks |
| `git_backup.cfg` | `UPDATE_GIT` | Push the config to this repository |

### PRINT_START

Called from the OrcaSlicer machine start G-code with these parameters:

| Parameter | Meaning |
|---|---|
| `TOOL` | First tool of the print |
| `TOOL_TEMP` | First layer temperature of that tool |
| `BED_TEMP` | Bed temperature |
| `MATERIAL` | Filament type of the first tool, e.g. `MATERIAL={filament_type[initial_extruder]}` |
| `T<n>_TEMP` | Passed for every other tool used in the print, triggers standby preheat |

Sequence:

1. Reset the first-use list for `_TOOL_FIRST_USE` (the first tool counts as used).
2. Pick the material Z offset from `MATERIAL` (substring match, most specific first). ASA is the reference with `0.0`, PETG `+0.03` (nozzle higher, less squish).
3. Disable crash detection, heat the bed and wait.
4. `G28` → `T0` → `QUAD_GANTRY_LEVEL` → `G28 Z` → adaptive bed mesh → load skew profile `Califlower`. All calibration is done with T0 as reference tool.
5. If the first tool is not T0: turn off T0 and pick up the first tool.
6. Preheat all other tools that appear in the print to 150 °C standby (no wait).
7. Wait for the first tool to reach print temperature, re-enable crash detection.
8. Apply the material Z offset as an **absolute** `SET_GCODE_OFFSET Z=`.
9. Prime line at the front (X100 → X200), starting with a 16 mm deretract.

Heat soak is not part of PRINT_START. It is done beforehand with the material preheat buttons.

### _TOOL_FIRST_USE

Called from the **filament start G-code** of every filament profile in Orca, so it runs on every tool change. On the first use of a tool within a print it waits for temperature and deretracts 16 mm. On later changes it does nothing.

Why: the tools sit in the dock 16 mm retracted, but the slicer only deretracts what it retracted itself during this print.

### PRINT_END

1. Calculate a safe move away from the print (20 mm in X/Y towards the side with more room, Z +2 mm).
2. Reset feed rate, flow rate and pressure advance (fallback `0.025`).
3. Retract 16 mm.
4. Switch to T0, so the next print starts with T0 on the shuttle (intentionally before the Z lift).
5. Move away from the print, park at the rear center.
6. Reset the G-code offset to 0 (removes the material Z offset).
7. Turn off heaters, part fans and extruder motors of all tools, turn off the bed.
8. Clear the skew profile and restore velocity/accel limits from `printer.cfg`.

### LOAD_FILAMENT

Parameter: `TEMP` (default 245 °C — works for all materials and melts leftover ASA/ABS when switching to PLA/PETG).

1. `INITIALIZE_TOOLCHANGER` to detect the mounted tool (works after a restart without homing).
2. Heat and wait.
3. Feed 90 mm to just above the nozzle, then 50 mm slowly through the nozzle.
4. Wait 10 s so trapped air can escape.
5. Extrude 40 mm at 15 mm³/s, then retract 16 mm at 45 mm/s — the same conditions as `OOZE_TEST` and the tool change.
6. Drop to 150 °C standby (`idle_timeout` turns it off if nothing follows).

### UNLOAD_FILAMENT

Parameters: `TEMP` (default 245 °C), `DISTANCE` (default 150 mm).

1. Detect the mounted tool, heat and wait.
2. Deretract 16 mm (the tool rests retracted), so the filament is back in the melt zone.
3. Tip shaping: short push/pull moves of 2, 3 and 4 mm.
4. Fast retract 20 mm, wait 3 s for the tip to cool, then unload the rest.
5. Drop to 150 °C standby.

### Hotend fan monitor

Runs every 3 s for every `heater_fan` (all tools, also docked ones). If a fan should be running, its hotend is above `heater_temp` + 10 °C and the RPM is below `min_rpm` (1500, measured), a counter goes up. After 3 checks in a row the heater of that tool is turned off and a running print is paused. Cold, unused tools are not checked because their fan is off.

Based on the `fan_tach_monitor` from Ellis' Print Tuning Guide, extended to multiple tools.

### Tool macros (T0–T3)

Defined in `tools/Tx.cfg`. They set the Spoolman spool assigned to the tool (or clear the active spool if none is assigned) and then call `SELECT_TOOL`. The spool assignment is stored in `saved_variables.cfg` and restored on startup by `RESTORE_SELECTED_SPOOLS`.

---

## How it all works together

### The 16 mm retract

The Rapido 2 HF cools down slowly and oozes in the dock while it is still hot. Every tool therefore rests **16 mm retracted** whenever it is not printing. The value was found with `OOZE_TEST` (PETG at 15 mm³/s: nothing oozes at 16 mm).

| Where | Retract | Deretract |
|---|---|---|
| Tool change (slicer) | 16 mm before dropoff | — |
| `PRINT_END` | 16 mm | — |
| `LOAD_FILAMENT` | 16 mm at the end | — |
| `PRINT_START` prime line | — | 16 mm for the first tool |
| `_TOOL_FIRST_USE` | — | 16 mm on the first use of every other tool |
| `UNLOAD_FILAMENT` | — | 16 mm before tip shaping |

If one of these is changed, all others have to be changed with it. Otherwise the first lines after a tool change will be under- or over-extruded.

### Temperatures

| State | Temperature |
|---|---|
| Print temperature | From the slicer (ASA 275 °C) |
| Standby (other tools in the print, after load/unload) | 150 °C — no ooze, short reheat on the first change |
| Load/unload | 245 °C |
| Not used | Off, `idle_timeout` (30 min) turns off forgotten heaters |

Tools never wait in the dock at full print temperature. Reheating between changes is handled by the slicer's preheat and standby temperatures.

### Z offsets — three layers

1. **Tool probe `z_offset`** (`[tool_probe Tx]`): Tap trigger point to nozzle. Only T0's value is used, since homing is always done with T0.
2. **`gcode_z_offset`** (`[tool Tx]`): height of T1–T3 relative to T0, measured with the Sexball probe via Axiscope. Applied automatically by the toolchanger on every tool change.
3. **Material Z offset**: global offset set in `PRINT_START` from the material of the first tool, kept across tool changes, reset in `PRINT_END`. It only matters for the first layer, so mixed materials higher up (e.g. PETG as support interface for ASA) need nothing special.

Axis twist compensation (X and Y) and the skew profile are in the `SAVE_CONFIG` block of `printer.cfg`.

### Print flow

```
Preheat button (heat soak)
        │
PRINT_START ── home/QGL/mesh with T0 ── first tool ── standby other tools ── material offset ── prime line
        │
   Print (Orca)
        │   tool change: retract 16 → dropoff → pickup → filament start G-code → _TOOL_FIRST_USE (deretract 16 once)
        │   hotend fan monitor runs in the background
        │
PRINT_END ── retract 16 ── T0 ── park ── offsets 0 ── everything off
        │
UPDATE_GIT (manually, when the config is in a good state)
```

---

## Slicer (OrcaSlicer)

- **Machine start G-code:** `PRINT_START` with `TOOL`, `TOOL_TEMP`, `BED_TEMP`, `MATERIAL` and `T<n>_TEMP` for each used tool.
- **Machine end G-code:** `PRINT_END`.
- **Filament start G-code (every profile):** `_TOOL_FIRST_USE`.
- **Retraction on tool change:** 16 mm.
- **Pressure advance:** per filament profile.

---

## Backup

Push the current config from Mainsail/Fluidd:

```
UPDATE_GIT
UPDATE_GIT MESSAGE="Short description"
```
