# Voron 2.4 StealthChanger

Klipper configuration for my Voron 2.4 (300 × 300 mm), converted into a toolchanger with [StealthChanger](https://github.com/DraftShift/StealthChanger) and 4 toolheads.

## Hardware

| Area | Component |
|---|---|
| Frame | Voron 2.4, 300 × 300 mm |
| Mainboard | BTT Octopus (STM32F446) |
| Toolchanger | StealthChanger, 4 docks |
| Toolhead boards | LDO Nitehawk 36 StealthChanger Edition (one per tool, USB) |
| XY | 0.9° steppers, X sensorless homing, Y with endstop |
| Z | 4 × Z motors (Quad Gantry Level), homing via tool probe |
| Heated bed | 230 V AC |
| Tool calibration | Sexball probe, evaluated with Axiscope |
| Chamber | "The Filter" mod with 4 bed fans, chamber temperature sensor |

### Toolheads

| Tool | Extruder | Hotend | Use |
|---|---|---|---|
| T0 | Orbiter 2.5 | Rapido 2 HF | Standard material, reference tool |
| T1 | Orbiter 2.5 | Rapido 2 HF | Standard material |
| T2 | Orbiter 2.0 | Rapido 2 HF | Support |
| T3 | Galileo 2 Standalone | Dragon HF Voron Edition | Abrasive materials |

Each tool has its own ADXL345, Tap probe, input shaper values and offsets relative to T0.

## Software

- [Klipper](https://github.com/Klipper3d/klipper) with [klipper-toolchanger-easy](https://github.com/jwellman80/klipper-toolchanger-easy)
- Mainsail / Fluidd, Moonraker, Crowsnest
- [Axiscope](https://github.com/nic335/Axiscope) for XY and Z tool offsets
- [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) for input shaping
- Spoolman with per-tool spool tracking

## Config structure

```
Printer_Config_Backup/
├── printer.cfg               # Main config, includes everything + SAVE_CONFIG block
├── moonraker.conf            # Moonraker, update manager, Spoolman
├── saved_variables.cfg       # Spoolman spool assignment per tool
├── config/
│   ├── machine.cfg           # General settings (idle_timeout, exclude_object, skew …)
│   ├── mcus/                 # BTT Octopus + pin aliases
│   ├── kinematics/           # CoreXY, velocity and acceleration limits
│   ├── hardware/             # XY, Z, heated bed, fans, filter, chamber sensor
│   ├── software/             # QGL, bed mesh, axis twist, input shaper, Axiscope, Spoolman
│   └── macros/               # PRINT_START/END, load/unload filament, dock tuner, ooze test
└── toolchanger/
    ├── toolchanger-config.cfg  # Own overrides (dock paths, homing)
    ├── readonly-configs/       # Stock klipper-toolchanger-easy files, do not edit
    └── tools/                  # T0–T3: MCU, extruder, fans, probe, LEDs, offsets
```

Per-tool values (PID, offsets, input shaper) are maintained directly in `tools/Tx.cfg`, not via `SAVE_CONFIG`.
