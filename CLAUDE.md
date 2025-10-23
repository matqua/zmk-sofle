# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK (Zephyr Mechanical Keyboard) firmware configuration repository for the Eyelash Sofle keyboard, a split ergonomic mechanical keyboard with the following features:

- **Split keyboard design**: Left and right halves communicate via Bluetooth
- **Wireless**: Uses nice!nano controllers with BLE support
- **RGB underglow**: WS2812 LED strips
- **Rotary encoder**: For volume/scroll control
- **Mouse emulation**: Pointer movement and scroll wheel via keyboard
- **ZMK Studio support**: Live keymap editing (left half only)
- **Nice!View displays**: E-ink screens on both halves
- **Soft-off mode**: Deep sleep activated by pressing Q+S+Z for 2 seconds

## Build System

### Automated Builds

Firmware builds are fully automated via GitHub Actions:

- Push any changes to trigger automatic builds
- Artifacts are generated for both left and right halves, plus ZMK Studio variant
- Download firmware UF2 files from GitHub Actions artifacts
- See `.github/workflows/build.yml`

### Build Configuration

The build targets are defined in `build.yaml`:

- `eyelash_sofle_right` with `nice_view` shield - Right half with display
- `eyelash_sofle_left` with `nice_view` shield - Left half with display
- `eyelash_sofle_left` with ZMK Studio support - Left half with live configuration editing
- `eyelash_sofle_left` with `settings_reset` shield - Reset settings to defaults

**Important**: ZMK firmware is built using GitHub Actions workflow that calls the upstream `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`. There is no local build command.

## Keymap Visualization

### Automatic Keymap Drawing

When you modify keymap files in `config/`, a GitHub Action automatically generates an SVG visualization:

- Workflow: `.github/workflows/draw.yml`
- Config: `keymap_drawer.config.yaml` (extensive custom styling and symbol mappings)
- Output: `keymap-drawer/eyelash_sofle.svg`
- Commits are auto-generated with prefix `[Draw]`

### Keymap Drawer Configuration

The `keymap_drawer.config.yaml` file contains extensive customization:

- Custom glyphs using Material Design Icons (MDI) and Tabler icons
- Special CSS classes for different key types (sym_sub_text, toggle, tap_dance, etc.)
- ZMK-specific binding mappings (behaviors like `&ltq`, `&hm`, mouse controls)
- Combo visualization settings
- Apple keyboard modifier symbols

## Code Architecture

### Directory Structure

```
boards/arm/eyelash_sofle/  - Custom board definition for Eyelash Sofle
├── eyelash_sofle.dtsi     - Main device tree include (hardware definition)
├── eyelash_sofle_left.dts / eyelash_sofle_right.dts - Split keyboard halves
├── eyelash_sofle-layouts.dtsi - Physical layout definitions
├── eyelash_sofle.keymap   - Default keymap (may be overridden by config/)
├── *.defconfig            - Kconfig defaults for each half
└── *.zmk.yml              - ZMK metadata

config/                    - User keymap and configuration
├── eyelash_sofle.keymap   - **Primary keymap file** (edit this)
├── eyelash_sofle.conf     - Kconfig overrides (power, RGB, features)
├── eyelash_sofle.json     - Keymap drawer layout config
└── west.yml               - West manifest (dependency management)

keymap-drawer/             - Generated keymap visualizations
zephyr/                    - Zephyr module metadata
```

### Key Files to Edit

**Keymap**: `config/eyelash_sofle.keymap`
- Contains all layer definitions, behaviors, combos, sensor bindings
- Defines mouse movement/scroll behaviors with custom acceleration settings
- Includes soft-off combo (Q+S+Z held for 2s)
- Currently has 4 layers: base (0), navigation/function (1), bluetooth/system (2), unused (3)

**Configuration**: `config/eyelash_sofle.conf`
- Feature flags and settings (sleep timeout, RGB, backlight, etc.)
- Key settings:
  - Sleep timeout: 1 hour (`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=3600000`)
  - Debounce: 8ms press/release
  - RGB underglow auto-off when idle/USB connected
  - Mouse pointing enabled
  - Soft-off mode enabled

**West Dependencies**: `config/west.yml`
- Pulls in ZMK main repository
- References the upstream eyelash_sofle board from `https://github.com/a741725193/zmk-sofle`
- The board definition in `boards/` is imported as a module

### Keymap Structure

The keymap file (`config/eyelash_sofle.keymap`) follows this structure:

1. **Preprocessor defines**: Mouse movement/scroll defaults
2. **Includes**: Input processors, behaviors, key definitions
3. **Input processor configuration**: Mouse acceleration, scroll scaling
4. **Behaviors section**: Custom key behaviors (currently empty, but can define tap-dances, hold-taps, etc.)
5. **Combos section**: Soft-off combo (Q+S+Z)
6. **Keymap layers**: 4 layers with 65 keys each (13 columns × 5 rows)
7. **Sensor bindings**: Rotary encoder behavior per layer

### Layer Overview

- **Layer 0** (Base): QWERTY-like layout with arrow keys in center column
- **Layer 1** (Navigation): F-keys, mouse buttons/movement, RGB controls, navigation cluster
- **Layer 2** (System): Bluetooth pairing/clear, output selection, bootloader/reset
- **Layer 3** (Unused): All transparent keys

### Mouse/Pointing Configuration

Mouse behavior is configured via devicetree properties at the top of the keymap:

```c
#define ZMK_POINTING_DEFAULT_MOVE_VAL 1200
#define ZMK_POINTING_DEFAULT_SCRL_VAL 25
```

Acceleration and timing are configured in the keymap's devicetree overrides (`&mmv`, `&msc`).

## Common Development Patterns

### Modifying the Keymap

1. Edit `config/eyelash_sofle.keymap`
2. Commit and push - GitHub Actions will build firmware and update the SVG diagram
3. Flash the resulting UF2 files to your keyboard halves

### Adding Custom Behaviors

Define new behaviors in the `behaviors` section of the keymap using ZMK's behavior system (hold-tap, tap-dance, mod-morph, etc.). See ZMK documentation for available behavior types.

### Bluetooth Pairing

Layer 2 contains Bluetooth controls:
- `BT_SEL 0-4`: Select profile slots
- `BT_CLR`: Clear current profile
- `BT_CLR_ALL`: Clear all profiles
- `OUT_USB`/`OUT_BLE`: Switch between USB and Bluetooth output

### Power Management

- **Idle sleep**: 1 hour of inactivity (configurable in `.conf`)
- **Soft-off**: Q+S+Z combo held for 2 seconds - deep sleep, wake with reset button
- **RGB auto-off**: Disables when idle or USB connected to save power

## Important Notes

- The board definition in `boards/arm/eyelash_sofle/` comes from the upstream repository specified in `config/west.yml`. Local modifications should be rare.
- ZMK Studio support is only on the left half (defined in `build.yaml`)
- The rotary encoder behavior changes per layer via `sensor-bindings`
- RGB and backlight are configurable but disabled on start to conserve battery
- Debounce is set to 8ms for both press and release to balance responsiveness and reliability

## Flashing Firmware

1. Download UF2 files from GitHub Actions artifacts after a successful build
2. Put keyboard into bootloader mode (double-tap reset button or use bootloader key in Layer 2)
3. Drag and drop the appropriate UF2 file to the USB drive that appears
4. Flash left and right halves separately with their respective firmware files
