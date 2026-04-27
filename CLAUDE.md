# CLAUDE.md

Owner-specific notes for the `BANANASJIM/zmk-for-charybdis` fork. This repo
hosts ZMK shield definitions for Charybdis split keyboard variants. It is
consumed by external build pipelines (e.g. `BANANASJIM/miryoku_zmk`) via
their outboard mechanism.

## Branches

| Branch | Shield | Hardware | Consumers |
| --- | --- | --- | --- |
| `main` | tracks Vzhao-L upstream | reference | none |
| `Cygnus` | `charybdis` | Plain Charybdis (no trackball, EC11 RGB) | `miryoku_zmk:outboards/shields/charybdis` |
| `Cygnus-Nano35` | `charybdis_nano35` | Charybdis + PMW3610 trackball + ZMK Studio physical layout | `miryoku_zmk:outboards/shields/charybdis_nano35` |

`Cygnus-Nano35` is based on Vzhao-L `charybdis-Nano35-MOD` upstream (which
adds the trackball) plus owner-specific changes:

1. `wakeup-source;` on the kscan node so any key wakes the SoC from deep sleep
2. Shield renamed from `charybdis` → `charybdis_nano35` so it can coexist
   with the plain-Charybdis shield in downstream consumers
3. `EXT_POWER` removed (see below)
4. `ZMK_KEYBOARD_NAME` shortened to fit 16-char Zephyr BT name limit

## Locked-in design decisions (do not regress)

- **No `CONFIG_ZMK_EXT_POWER=y`** in `charybdis_nano35_right.conf`. The
  Bastardkb sensor PCB (`Bastardkb/charybdis-pmw-3360-sensor-pcb`) has no
  power-gating MOSFET — verified against the KiCad BOM. Enabling EXT_POWER
  without an `ext_power` GPIO node in the dtsi only causes ~3.8 mA deep
  sleep leak. If a future hardware revision adds a MOSFET, add the GPIO
  node *first*, then re-enable.
- **`wakeup-source;` only on kscan, not on PMW3610.** The
  `DoctorWangWang/zmk-pmw3610-driver@main` driver has zero `pm_device`
  hooks (`src/pmw3610.c` confirmed). Adding `wakeup-source;` to the
  trackball@0 node compiles but does nothing at runtime.
- **`ZMK_KEYBOARD_NAME = "Cygnus-Nano35"` (13 chars)**. Must stay
  `< CONFIG_BT_DEVICE_NAME_MAX` (default 16). Zephyr build asserts this in
  `subsys/bluetooth/host/hci_core.c:4112`. Earlier attempt with
  `"Charybdis-Nano35"` (16 chars) failed CI for this reason.
- **3-wire SPI on PMW3610**: `MOSI` and `MISO` both pinned to `P0.17` in
  `charybdis_nano35_right.overlay` — this is intentional (PMW3610 is a
  3-wire SDIO part) and the DoctorWangWang driver supports it. Do not
  "fix" by separating to two pins.

## Naming conventions across files

For `charybdis_nano35` shield:

| File | Field | Value |
| --- | --- | --- |
| `Kconfig.shield` | symbol | `SHIELD_CHARYBDIS_NANO35_LEFT/RIGHT` |
| `Kconfig.shield` | shields_list_contains key | `charybdis_nano35_left/right` |
| `Kconfig.defconfig` | `ZMK_KEYBOARD_NAME` default | `"Cygnus-Nano35"` |
| `charybdis_nano35.zmk.yml` | `id` | `charybdis_nano35` |
| `charybdis_nano35.zmk.yml` | `name` | `Charybdis Nano35` (hardware family label) |
| `config/charybdis_nano35.conf` | `ZMK_KEYBOARD_NAME` | `"Cygnus-Nano35"` |
| `config/charybdis_nano35.json` | `id` / `name` | `charybdis_nano35` / `Cygnus-Nano35` |
| `build.yaml` | shield matrix | `charybdis_nano35_left/right` |

Keep all of these in sync if renaming.

## Standalone vs. consumed builds

This repo's own `build.yaml` produces a standalone firmware (with EC11 +
ZMK Studio + own keymap) for testing the shield in isolation. External
consumers (miryoku_zmk) only symlink `config/boards/shields/<shield>/` and
provide their own keymap + kconfig. The top-level `config/<shield>.{conf,
json,keymap}` only matter for the standalone build path.

## Adding a new shield variant in the future

1. New branch off `main` (or current Vzhao upstream)
2. Rename shield directory + Kconfig symbols + zmk.yml id consistently
   (see naming conventions table above)
3. Update `build.yaml` shield names
4. Add `wakeup-source;` on kscan if not already present
5. Verify `ZMK_KEYBOARD_NAME` length < 16
6. Push, then add a matching outboard file in the consumer repo
