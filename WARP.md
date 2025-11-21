# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is a ZMK (Zephyr Mechanical Keyboard) module for the Forager split keyboard, a 34-key ergonomic keyboard that uses the Seeeduino XIAO BLE board. The module integrates with the ZMK firmware ecosystem and includes support for ZMK Studio (runtime keyboard configuration) and RGB LED features via the zmk-rgbled-widget dependency.

## Build System

### GitHub Actions Build
Firmware builds automatically via GitHub Actions on push, pull request, or manual workflow dispatch:
- Workflow uses `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`
- Build configuration defined in `build.yaml` at repository root

### Local Build Commands
This is a ZMK module, not a standalone build target. To build locally:
1. Create a separate zmk-config repository with this module as a dependency
2. Use west build commands from the zmk-config directory:
   ```bash
   west build -d build/left -b seeeduino_xiao_ble -- -DSHIELD="forager_left rgbled_adapter" -DSNIPPET="studio-rpc-usb-uart"
   west build -d build/right -b seeeduino_xiao_ble -- -DSHIELD="forager_right rgbled_adapter"
   ```
3. Flash with: `west flash -d build/left` or `west flash -d build/right`

## Architecture

### Module Structure
- **`boards/shields/forager/`** - Shield definitions for the Forager keyboard
  - `forager.dtsi` - Base devicetree include with physical layout (34 keys: 30 main + 4 thumb), matrix transform, and kscan configuration
  - `forager_left.overlay` / `forager_right.overlay` - Split-specific GPIO pin mappings for Seeeduino XIAO BLE
  - `forager.keymap` - Default keymap with 4 layers (BASE, NAV, SYM, ADJ) and custom behaviors
  - `forager.zmk.yml` - ZMK Studio metadata
  - `Kconfig.shield` / `Kconfig.defconfig` - Build configuration enabling split keyboard and ZMK Studio on left half
- **`config/west.yml`** - West manifest declaring dependencies (zmk, zmk-rgbled-widget)
- **`zephyr/module.yml`** - Declares this as a Zephyr module with board_root at repository root
- **`build.yaml`** - GitHub Actions build matrix for left/right shields plus settings_reset

### Key Technical Details

**Split Keyboard Architecture:**
- Left half is central (master), right half is peripheral
- Uses col2row diode direction, 5 columns × 4 rows per half
- 10 total columns when combined (col-offset = 5 for right half)
- GPIO pull-down resistors on row pins
- Serial communication between halves via XIAO pins (xiao_serial disabled in overlays)

**Keymap Behaviors:**
- Custom auto-shift behavior (AS macro) for shifted key access
- Home row mods (HRM) with positional hold-tap on both hands (280ms tapping term, 175ms quick-tap)
- Smart shift: single tap = sticky shift, double tap = caps word
- Shift-backspace morphs to Alt-backspace
- Conditional layer: NAV + SYM = ADJ layer
- Layer-tap on thumbs with tap-preferred flavor

**ZMK Studio Support:**
- Enabled by default on left half (Kconfig.defconfig sets CONFIG_ZMK_STUDIO=y)
- Requires `studio-rpc-usb-uart` snippet in build configuration
- Allows runtime keymap editing without reflashing firmware

## Dependencies

### External Modules
- **zmk** (zmkfirmware/zmk@main) - Core ZMK firmware
- **zmk-rgbled-widget** (caksoylar/zmk-rgbled-widget@main) - RGB LED status widget
  - Requires `rgbled_adapter` shield in build.yaml
  - See https://github.com/caksoylar/zmk-rgbled-widget for configuration

### Hardware Requirements
- Seeeduino XIAO BLE board (nRF52840-based)
- Forager PCB with split design

## Common Modifications

### Editing the Keymap
Modify `boards/shields/forager/forager.keymap`:
- Layer definitions are in the `keymap` node (lines 81-119)
- Add/modify behaviors in the `behaviors` node (lines 15-71)
- Layers: 0=BASE (Colemak-DH), 1=NAV, 2=SYM, 3=ADJ
- Use devicetree syntax: `&kp KEY`, `&lt LAYER KEY`, `&mo LAYER`, etc.

### Adding GPIO Pins
Edit pin mappings in overlay files:
- `forager_left.overlay` - Left half col-gpios (GPIO0/GPIO1 pins) and row-gpios
- `forager_right.overlay` - Right half col-gpios and row-gpios
- Pin numbers correspond to nRF52840 GPIO on Seeeduino XIAO BLE

### Changing Build Targets
Edit `build.yaml` to modify:
- Board selection (currently `seeeduino_xiao_ble`)
- Shields (currently `forager_left/right` with `rgbled_adapter`)
- Snippets (currently `studio-rpc-usb-uart` for ZMK Studio)
- Add cmake-args for debugging: `-DCONFIG_ZMK_USB_LOGGING=y`

### Integration in zmk-config
To use this module in a user config repository, add to `config/west.yml`:
```yaml
manifest:
  remotes:
    - name: carrefinho
      url-base: https://github.com/carrefinho
    - name: caksoylar
      url-base: https://github.com/caksoylar
  projects:
    - name: forager-zmk-module
      remote: carrefinho
      revision: main
    - name: zmk-rgbled-widget
      remote: caksoylar
      revision: main
```

## File Organization Patterns

- Devicetree files use `.dtsi` extension for includes, `.overlay` for board-specific overrides
- Keymaps always named `<shield>.keymap` in shield directory
- Matrix transform and physical layout defined in base `.dtsi`, GPIO pins in split-specific `.overlay` files
- Kconfig files follow ZMK naming: `Kconfig.shield` for shield selection, `Kconfig.defconfig` for default configs
