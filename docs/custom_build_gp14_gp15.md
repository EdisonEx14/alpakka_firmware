# Custom build: GP15 = stick push, GP14 = L4

This fork is based on the pristine upstream `inputlabs/alpakka_firmware` 1.2.0 release.
Only the changes listed below are applied.

## 1. Pin remap (v0 / wired)

| Function | Upstream pin | This build |
| --- | --- | --- |
| `PIN_I2C_SDA` | GP14 | unused (I2C disabled) |
| `PIN_I2C_SCL` | GP15 | unused (I2C disabled) |
| `PIN_L3` (stick push) | IO expander 1, pin 12 (`112`) | **GP15** |
| `PIN_L4` | IO expander 1, pin 9 (`109`) | **GP14** |

Everything else is untouched: A/B/X/Y and dpad stay on the IO expanders, HOME stays
on GP20, the four LEDs stay on GP2-GP5, touch stays on GP6/GP7, the thumbstick
axes stay on GP26/GP27.

Because GP14/GP15 were the I2C bus, the expander path is compiled out:
`PIN_I2C_DISABLED` makes `bus_i2c_init()`, `bus_i2c_io_init()` and
`bus_i2c_io_cache_update()` skip the I2C peripheral entirely, so those pins keep
their plain GPIO function and no expander button reports anything (the cache stays
at zero). `bus_i2c_io_init()` also seeds the PCB generation as 0, which the touch
code requires.

Hardware: wire the stick push switch to GP15 and the extra button to GP14, each
switched to GND (internal pull-ups are enabled by `Button_()`).

## 2. Stuck input when switching profile

Bug: with a thumbstick direction mapped to a keyboard key (or mouse button), holding
that direction and pressing a button mapped to `PROC_PROFILE_n` left the key
pressed forever. Reason: the keys live in `state_matrix` as counters, and the
profile that pressed them is no longer reported after the switch, so the release
branch never runs.

Fix: `loop_controller_task()` tracks `profile_get_active_index(false)`; when it
changes it calls `profile_reset_all()`, which clears the HID matrix and resets every
button runtime before the next report, so the previous profile releases what it
held. Covers `PROC_PROFILE_n`, HOME profile selection and home-gamepad transitions.

`profile_reset_all()` is now declared in `profile.h`.

## 3. CI

`.github/workflows/build.yml` gained `permissions: contents: write` and a final step
that copies the compiled v0 firmware to `bin/alpakka_pico_<version>.uf2`, so it can
be downloaded without authenticating to the Actions artifact (marketing the commit
with `[skip ci]` prevents a build loop).
