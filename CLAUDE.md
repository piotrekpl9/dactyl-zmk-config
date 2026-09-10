# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

ZMK firmware user-config for a **Dactyl Manuform 6x6** split keyboard running on two **nice!nano v2** controllers (nRF52840). Lewa połowa = CENTRAL (USB + BLE host, ZMK Studio), prawa = PERIPHERAL (BLE only).

Code/comments are written in Polish — keep new comments in Polish to match.

## Build

Builds run in GitHub Actions via [.github/workflows/build.yml](.github/workflows/build.yml), which delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`. The matrix is defined in [build.yaml](build.yaml):

- `dactyl_manuform_6x6_left` + `studio-rpc-usb-uart` snippet + `-DCONFIG_ZMK_STUDIO=y` → central with ZMK Studio
- `dactyl_manuform_6x6_right` → peripheral
- `settings_reset` → emergency BLE-pairing wipe firmware (flash to both halves to clear stale pairing)

Artifacts are `.uf2` files; flash by dragging onto the nice!nano in bootloader mode.

There is no local build; do not invent `west build` commands unless the user has set up a local Zephyr workspace.

## Architecture: how the split is wired together

The non-obvious part of this config is the **matrix transformation**, which maps the two halves' physical wiring into one 7×12 logical matrix. Read [boards/shields/dactyl_manuform_6x6/dactyl_manuform_6x6.dtsi](boards/shields/dactyl_manuform_6x6/dactyl_manuform_6x6.dtsi) end-to-end before editing anything matrix-related.

Key invariants:

- **Logical matrix is 7 rows × 12 cols = 84 keys.** Cols 0–5 are the left half, cols 6–11 are the right half.
- **Right half uses `col-offset = 6`**, applied by overriding `&default_transform` at the end of [dactyl_manuform_6x6_right.overlay](boards/shields/dactyl_manuform_6x6/dactyl_manuform_6x6_right.overlay). Do **not** define a separate transform node for the peripheral — `col-offset` belongs on `zmk,matrix-transform`, not on `kscan-gpio-matrix`.
- **Column ordering differs between halves** because the wiring is mirrored. Left col-gpios go `D4, D6, D5, D7, D8, D9` (inner→outer); right col-gpios go `D9, D8, D7, D5, D4, D6` (outer→inner, with D4/D6 swapped). The transform's `map = <…>` block compensates so the keymap can be written left-to-right physically.
- **Keymap row order is reversed vs. physical position.** In [config/dactyl_manuform_6x6.keymap](config/dactyl_manuform_6x6.keymap), Row 0 is the bottom thumb row and Row 6 is the top F-key row. Each bindings row has 12 keys: first 6 = right half (inner→outer), last 6 = left half (inner→outer). The ASCII diagrams in the keymap show this layout.
- **Only the central (left) has a keymap.** The peripheral just scans its matrix and forwards events over BLE.

If keys on the right half map to the wrong positions, the bug is almost always in the right overlay's `col-gpios` order or the `default_transform` `map` — not in the keymap.

## File layout

- [boards/shields/dactyl_manuform_6x6/dactyl_manuform_6x6.dtsi](boards/shields/dactyl_manuform_6x6/dactyl_manuform_6x6.dtsi) — shared: matrix transform + physical layout for ZMK Studio
- `*_left.overlay` / `*_right.overlay` — kscan GPIO matrix per half; right also overrides the transform's `col-offset`
- `boards/shields/.../*.conf` — shield-level Kconfig (per-half)
- `config/dactyl_manuform_6x6_{left,right}.conf` — user-config Kconfig (BLE, split role, sleep, debounce)
- `boards/shields/.../Kconfig.defconfig` — sets `ZMK_SPLIT=y` and assigns the CENTRAL role to the left shield
- [config/west.yml](config/west.yml) — pins ZMK to `main`
- [config/dactyl_manuform_6x6.keymap](config/dactyl_manuform_6x6.keymap) — 5 layers: BASE (QWERTY), LOWER, RAISE, ADJUST, BT

## Recovering from broken BLE pairing

If the halves won't pair (common after swapping which side is central, or after firmware role changes), flash the `settings_reset` shield from `build.yaml` onto **both** halves, then reflash the normal left/right firmware. `CONFIG_ZMK_SETTINGS_RESET_ON_START` disables BT during the reset boot — that is expected behavior, not a failure.
