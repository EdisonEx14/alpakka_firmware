# Left thumbstick push on GP1 (Alpakka v0 patch)

Personal fork of [inputlabs/alpakka_firmware](https://github.com/inputlabs/alpakka_firmware).
**Only `DEVICE=alpakka_v0` (Raspberry Pi Pico build) is affected; `alpakka_v1` (Marmota) is untouched.**

## What changed

| Signal | Upstream | This fork |
| --- | --- | --- |
| Left thumbstick push (`SECTION_LSTICK_PUSH`, HID `L3` by default) | IO expander #1, bit 12 (`PIN_L3 = 112`, port `P14`) | **RP2040 GP1** (`PIN_L3 = 1`, Pico header pin 2) |
| Right thumbstick / DHat push (`SECTION_RSTICK_PUSH`, HID `R3` by default) | `PIN_R3 = 204` | unchanged |

One line in `src/headers/pin.h` (v0 block):

```diff
-    #define PIN_L3 112
+    // Left thumbstick push switch.
+    // CUSTOM PATCH: read from Pico GP1 instead of IO expander #1 bit 12 (P14).
+    #define PIN_L3 1
```

`PIN_L3` is referenced in exactly one place in the whole code base,
`thumbstick_from_ctrl()` in `src/thumbstick.c`:

```c
const uint8_t PIN_PUSH = index ? PIN_R3 : PIN_L3;
/* ... */
Button_from_ctrl(PIN_PUSH, ctrl->sections[SECTION_STICK_PUSH])
```

so the left push becomes `Button_(1, ...)`, which lands in the "board GPIO"
branch of `Button__is_pressed()`:

```c
else if (is_between(self->pin, PIN_GROUP_BOARD, PIN_GROUP_BOARD_END)) {
    return !gpio_get(self->pin);   // active low, internal pull-up
}
```

`is_between()` is inclusive on both ends (`value >= a && value <= b` when
`a < b`), so pin `1` is inside the `PIN_GROUP_BOARD (1) .. PIN_GROUP_BOARD_END (99)`
group and is read as a real GPIO.

No other file needs to change.

## Wiring

The stick module push switch is a switch to GND. Tap the non-GND side of the
switch (the node that upstream routed to expander `P14`) and wire it to the Pico
header **pin 2 (GP1)**. The original expander trace can stay in place — it is
simply no longer read by the firmware.

## Build / flash

```sh
DEVICE=alpakka_v0 make
make load
```

`make test` (or `T` over the serial session) now asks for `Thumbstick Push`,
which accepts the press on GP1.

## Changing what the press does (web configurator)

No firmware change and no recompilation are needed: the press is a first-class
profile section (`SECTION_LSTICK_PUSH`, index 36, labelled `LStick Push`) in the
existing control protocol.

1. Open <https://ctrl.inputlabs.io> and connect the controller.
2. Edit a profile and find the `LStick Push` cell (left stick block, centre).
3. Bind any action: gamepad button, keyboard key, mouse button, macro, etc.
4. Save the profile to the controller.

## Caveats

- **UART stdio RX is lost on v0.** `CMakeLists.txt` calls
  `pico_enable_stdio_uart(...)`, whose default RX pin is GP1. Once
  `gpio_init(1)` runs, GP1 is plain SIO input, so log output on GP0 still works
  but single-character devkit commands (`R` restart, `B` bootsel, `C` calibrate,
  `F` format, `T` test) no longer reach the controller. If you need them back,
  move stdio to USB (`pico_enable_stdio_usb`) in `CMakeLists.txt`.
- **`PIN_L3` no longer means "expander pin"** on v0. It now names the left stick
  push wherever it lives; see the comment in `pin.h`.
- **`push_auto_toggle`** (profile setting) still applies: press at full
  deflection, release to latch the button. That is derived from the profile, not
  from the pin.
- The patch is not compiled in this repository; upstream builds with
  `-Wall -Werror`, so run a local `DEVICE=alpakka_v0 make` before flashing.
