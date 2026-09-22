# The Brain (ESP32-S2)

The ESP32-S2 runs the meltybrain control loop. You program it over the USB-C port with the Arduino IDE. The first upload needs a boot jumper. After that, uploads are one click.

## 1. Install the tools

1. Install [Arduino IDE 2](https://www.arduino.cc/en/software).
2. Open **File → Preferences** and add this to Additional boards manager URLs:

    ```
    https://espressif.github.io/arduino-esp32/package_esp32_index.json
    ```

3. Open **Tools → Board → Boards Manager**, search for `esp32`, and install **esp32 by Espressif Systems**.

## 2. Board settings

Select **Tools → Board → esp32 → ESP32S2 Dev Module**, then set:

| Tools menu item | Value |
|---|---|
| USB CDC On Boot | Enabled |
| Upload Mode | Internal USB |
| USB DFU On Boot | Disabled |
| USB Firmware MSC On Boot | Disabled |
| Flash Size | 4MB (32Mb) |
| Partition Scheme | Default 4MB with spiffs |
| PSRAM | Disabled |
| CPU Frequency | 240MHz |
| Upload Speed | 921600 |

**USB CDC On Boot** is what makes `Serial` print over the USB port and lets the IDE reset the board into the bootloader for you. Leave it enabled in every sketch you upload.

## 3. First flash

A fresh ESP32-S2 has no firmware that knows how to reset itself, so you force it into download mode by hand once.

1. Short **S2_BOOT** (J1 pin 5, or test pad TP4) to **GND** (J1 pin 1, or TP3). A jumper wire in the JST-SH cable works, as does a pair of tweezers held across the pads while you plug in.
2. Plug in USB-C. The board enumerates as a new serial port.
3. In the IDE, pick that port under **Tools → Port**.
4. Open **File → Examples → 01.Basics → Blink** and change the LED pin to the green status LED:

    ```cpp
    #define LED_BUILTIN 41
    ```

5. Click **Upload**.
6. Unplug USB, remove the boot jumper, and plug USB back in.
7. The green LED blinks.

From now on you can upload without the jumper. The running sketch's USB CDC port lets the IDE reboot the chip into the bootloader.

!!! tip "When you need the jumper again"
    Use the boot jumper any time the port disappears or an upload fails to connect. Typical causes are a sketch that crashes before USB comes up, or a sketch built with USB CDC On Boot disabled. The jumper always works because it bypasses whatever is in flash.

!!! note "Reset after the first flash"
    After a jumper-assisted upload the board does not restart on its own. Unplug and replug USB. Uploads done without the jumper reset automatically.

## 4. Install SimpleMelt

1. Open **Tools → Manage Libraries**, search for `SimpleMelt`, and install it. Accept the prompt to install its dependencies, **AlfredoCRSF** and **SparkFun LIS331 Accelerometers**.
2. Open **File → Examples → SimpleMelt → Rotini-Example**.

If SimpleMelt is not in the Library Manager yet, clone [AlfredoSystems/SimpleMelt](https://github.com/AlfredoSystems/SimpleMelt) into your `Arduino/libraries` folder and install the two dependencies from the Library Manager.

## 5. Configure and upload the example

The pin constants at the top of `Rotini-Example.ino` already match Rotini V3. The values you tune for your robot are in `setup()`:

| Field | Meaning |
|---|---|
| `melty_led_offset_CW` / `_CCW` | Angle in radians between the accelerometer and the heading LED, for each spin direction |
| `turn_speed` | Heading rotation rate in rotations per second at full stick |
| `accelerometer_radius` | Distance from the spin axis to the accelerometer, in metres |
| `radius_trim` | Fine adjustment to the radius, also adjustable from the transmitter |

Upload it exactly as you uploaded Blink. Open **Tools → Serial Monitor** at 115200 baud to see output.

The example expects the receiver to already be running ExpressLRS. See [The Receiver](receiver.md). Without a link, the robot stays in the disconnected state and the motors are held off.

!!! danger "Motors off while programming"
    Disconnect ESC power or remove the weapon and drive motors before uploading anything. The FOO and BAR pins float while the bootloader runs, and a badly behaved ESC can read that as a command.

## Transmitter mapping used by the example

The example was written for a RadioMaster Zorro. Channels are CRSF channel numbers.

| Control | Channel | Action |
|---|---|---|
| Right stick X | 1 | Rotation |
| Left stick Y | 3 | Throttle |
| SWA | 6 | Back: stop, middle: arcade, forward: melty |
| SWD | 5 | Spin direction |
| SWB | 12 | Selects what the left arrows trim: radius, CW LED offset, CCW LED offset |
| Bumpers | 11 | Spin power down / up in 2 % steps |
| Right arrows | 7, 8 | Spin power presets: 0 %, 18 %, 100 % |
| Left arrows | 9, 10 | Trim the value selected by SWB |

## Troubleshooting

| Symptom | Fix |
|---|---|
| No serial port appears | Try another cable, some are power only. Then use the boot jumper |
| Port appears only with the jumper | The sketch in flash is not bringing up USB. Check USB CDC On Boot is enabled and re-upload |
| Upload fails with `No serial data received` | Board is not in download mode. Use the boot jumper |
| `Serial.print` shows nothing | USB CDC On Boot is disabled, or the Serial Monitor is on the wrong port |
| Board resets when the ESC arms | Pack voltage spike. See the power limits on the [Overview](index.md#power) |
