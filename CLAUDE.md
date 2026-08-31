# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [ZMK](https://zmk.dev) user config repo for a handwired **Redox** split keyboard, built around the standard `zmkfirmware/zmk-config-template` layout. It vendors ZMK via West rather than committing the firmware source itself, and defines the Redox shield as a custom board module living inside this same repo (`zephyr/module.yml` sets `board_root: .`, so `boards/shields/redox_handwire` is picked up as this repo's own Zephyr module — it is not pulled in from an external shield module).

**Naming note:** the shield is called `redox_handwire`, not `redox`, deliberately. ZMK's own upstream repo ships its own built-in shield literally named `redox` (for a different, commercially-designed PCB with a different pinout). An earlier version of this repo used the name `redox` and West silently resolved every build to ZMK's bundled shield instead of this repo's own files — every devicetree/Kconfig edit here was a no-op for months of debugging before the collision was found. Do not rename this shield back to `redox`.

There is no build tooling, package manager, or test suite here — this is Devicetree/Kconfig source and YAML manifests only. All actual compilation happens in CI (or a local `west build`), not via any command in this repo.

## Building firmware

Firmware is built exclusively by GitHub Actions on push/PR/workflow_dispatch (`.github/workflows/build.yml`), which delegates entirely to ZMK's reusable workflow:

```yaml
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

`build.yaml` at the repo root is the actual build matrix consumed by that workflow — each entry is one firmware artifact (`board` + `shield` [+ optional `snippet`]). To add/remove a build target (e.g. a new MCU board, or toggling the `zmk-usb-logging` snippet for serial debug output), edit `build.yaml` directly; there's nothing else to wire up.

To build locally instead of waiting on CI, use a standard West workflow from a full ZMK checkout with this repo as the manifest (`west init -l config`, `west update`, `west build -s zmk/app -b <board> -- -DSHIELD=<shield>`) — see ZMK's own docs, not this repo, for that flow.

## Repository layout

- `config/west.yml` — the West manifest. Pins ZMK to `v0.3` and imports `app/west.yml` from `zmkfirmware/zmk`. `self.path: config` marks this repo as the manifest repo. Add any additional external modules (extra shields/boards from other repos) here.
- `config/redox_handwire.keymap` / `config/redox_handwire.conf` — the **user-facing** keymap and Kconfig overrides for the `redox_handwire` shield build. ZMK's build system resolves these by shield name and layers them over the shield module's own defaults in `boards/shields/redox_handwire/`.
- `boards/shields/redox_handwire/` — the custom shield definition for this handwired Redox (source keyboard design: https://github.com/mattdibi/redox-keyboard):
  - `redox_handwire.zmk.yml` — shield metadata (id `redox_handwire`, siblings `redox_handwire_left`/`redox_handwire_right`, requires `pro_micro`).
  - `Kconfig.shield` / `Kconfig.defconfig` — declares the `SHIELD_REDOX_HANDWIRE_LEFT`/`SHIELD_REDOX_HANDWIRE_RIGHT` Kconfig symbols and split-keyboard defaults (left half is `ZMK_SPLIT_ROLE_CENTRAL`, both halves set `ZMK_SPLIT`).
  - `redox_handwire.dtsi` — the shared devicetree: GPIO matrix kscan (5 rows × 7 cols per half, row2col diode direction, pin assignment traced from the actual hand-wired board) and the `matrix_transform` mapping physical positions to a 14-column logical layout (left = cols 0–6, right = cols 7–13, right half auto-offset by the central's column count).
  - `redox_handwire_left.overlay` / `redox_handwire_right.overlay` — trivial per-half overlays that just `#include "redox_handwire.dtsi"`; the halves are otherwise identical and differentiated only by the `SHIELD_REDOX_HANDWIRE_LEFT`/`_RIGHT` Kconfig split above.
  - `redox_handwire.keymap` — the shield's own baseline keymap; currently kept byte-for-byte identical to `config/redox_handwire.keymap` (i.e. no divergence yet between the shield default and the user override).
- `.zmk/` — a local, gitignored West workspace checkout (used by IDE/CLI tooling for local builds); not part of the repo's source of truth and shouldn't be edited or committed to.

## Working on the keymap

- `config/redox_handwire.keymap` currently defines a single `default_layer` and is explicitly a **starting/bring-up keymap**: the file's own header comment notes the legend-to-physical-key mapping isn't expected to be correct yet for a handwire build, and keys should be rearranged in place once physical testing shows what each position actually registers.
- Each keymap row is 14 bindings: 7 for the left half followed by 7 for the right half, matching the `matrix_transform` column layout in `redox_handwire.dtsi`.
- When adding layers, behaviors (hold-taps, combos, etc.), keep `boards/shields/redox_handwire/redox_handwire.keymap` and `config/redox_handwire.keymap` in sync only if you intend the shield's built-in default to match the user's config — they are not automatically kept in sync, so update both if that invariant matters, or let them diverge deliberately once the user keymap stabilizes.
- RGB underglow and USB logging are toggled via commented-out `CONFIG_*` lines in `config/redox_handwire.conf`; uncomment there rather than adding new Kconfig fragments elsewhere.
