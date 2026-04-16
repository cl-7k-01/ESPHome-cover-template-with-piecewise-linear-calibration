# ESPHome Cover with Piecewise Linear Calibration

Precise position control for roller shutters and blinds in ESPHome, compensating for the non-linear behavior caused by varying drum diameter.

## The Problem

Standard `time_based` covers assume constant speed throughout the travel. In reality, roller shutters wrap around a drum — as more material accumulates, the effective diameter increases, making the shutter move faster per rotation.

This causes position errors: asking for 50% might give you 40% or 60% depending on direction and starting point.

## The Solution

This template uses **4-point piecewise linear interpolation** to map physical position to actual travel time. You measure real times at 25%, 50%, and 75%, and the code interpolates between these calibration points.

Calibration values are exposed as **Home Assistant number entities**, so you can fine-tune without reflashing.

## Features

- Accurate positioning across the entire range
- Separate calibration for open/close directions
- Position restored after reboot
- Intermediate position calculated on stop
- All parameters adjustable from Home Assistant

## Requirements

- ESPHome
- Two relays (or equivalent) for motor control
- Home Assistant (for calibration UI)

## Installation

1. Copy `cover-piecewise-linear.yaml` to your ESPHome config
2. Define your relay switches with ids `relay_open` and `relay_close`
3. Customize device name and cover name
4. Set initial `total` times (rough estimate is fine)
5. Flash and add to Home Assistant
6. Calibrate (see below)

## Hardware Setup

You must define two switches for your relay/motor control. Example:

```yaml
switch:
  - platform: gpio
    id: relay_open
    pin: GPIO12
    restore_mode: ALWAYS_OFF
    
  - platform: gpio
    id: relay_close
    pin: GPIO14
    restore_mode: ALWAYS_OFF
```

Adjust pins and platform according to your hardware (GPIO, PCF8574, MCP23017, etc.).

**Important:** If using interlocks, define them in your switch config. The script includes a 125ms delay between direction changes for safety.

## Calibration

### Step 1: Measure Opening Times

1. Fully close the cover
2. Start a stopwatch
3. Press **Open** in Home Assistant
4. Note the seconds when the cover physically reaches:
   - 25% open
   - 50% open
   - 75% open
   - 100% open (total time)
5. Enter these values in the corresponding HA number entities

### Step 2: Measure Closing Times

1. Fully open the cover
2. Start a stopwatch
3. Press **Close** in Home Assistant
4. Note the seconds when the cover physically reaches:
   - 75% open (= 25% closed)
   - 50% open (= 50% closed)
   - 25% open (= 75% closed)
   - 0% open (= fully closed, total time)
5. Enter these values in the corresponding HA number entities

### Tips

- Use a consistent reference point (e.g., window frame marks)
- Measure multiple times and average
- If open/close speeds are identical, you can copy the values

## Adding Multiple Covers

For each additional cover, duplicate and rename:

| Component | Cover 1 | Cover 2 | Cover N |
|-----------|---------|---------|---------|
| Globals | `c1_pos`, `c1_target`, ... | `c2_pos`, `c2_target`, ... | `cN_...` |
| Numbers | `c1_open_total`, ... | `c2_open_total`, ... | `cN_...` |
| Script | `sc1` | `sc2` | `scN` |
| Cover | `cover1` | `cover2` | `coverN` |
| Relays | `relay_open`, `relay_close` | `relay2_open`, `relay2_close` | ... |

Also add each cover's position restore to `on_boot`.

## How It Works

The core is the `p2t()` function (position-to-time), which converts a physical position (0.0–1.0) to a time fraction using linear interpolation across four segments:

```
Position:  0% -------- 25% -------- 50% -------- 75% -------- 100%
Time:      0 -------- t25 -------- t50 -------- t75 --------  1.0
```

When you request a position, the script:
1. Determines direction (open/close)
2. Loads the appropriate calibration values
3. Converts current and target positions to time fractions
4. Calculates the required movement duration
5. Activates the relay for that duration
6. Updates the stored position

On stop, it calculates the intermediate position based on elapsed time.

## **WARNING / DISCLAIMER**
Use this code at your own risk. Controlling AC motors involves high voltage and mechanical risks.
The author is not responsible for any damage to your hardware or property.
This software is provided "as is", without warranty of any kind, express or implied.
In no event shall the author be held liable for any damages, including but not limited to property damage, personal injury, or hardware failure arising from the use of this configuration. Controlling high-voltage loads involves significant risks of electric shock and fire.
Always verify that both hardware and software interlocks are correctly configured before operation. Use this code at your own risk

## License
License: GNU GPL v3
This project is licensed under the GNU GPL v3.
You are free to copy, modify, and distribute this code, provided that any derivative work remains open-source
and is released under the same license. For further details, please refer to the LICENSE file included
in this repository.

## Contributing

Issues and PRs welcome.
