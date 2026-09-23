# The Receiver (ESP8285)

Rotini's ESP8285 and SX1280 form a complete ExpressLRS 2.4 GHz receiver. You flash and update it through the ESP32-S2 over the board's own USB-C port. No UART adapter, no wires, no boot jumper.

The one exception is a brand new board. The ESP8285 arrives blank, and the passthrough method needs ExpressLRS already running on it. For that first flash, see [Flashing a blank receiver](#flashing-a-blank-receiver) at the bottom.

## What you need

- The Arduino setup from [The Brain](brain.md), with the **AlfredoCRSF** library installed (it comes with SimpleMelt).
- The [ExpressLRS Configurator](https://github.com/ExpressLRS/ExpressLRS-Configurator/releases).
- A USB-C cable.

## 1. Load the passthrough sketch on the ESP32-S2

The ESP32-S2 shares a UART with the receiver. The `elrsPassthrough` example turns the S2 into a bridge: it answers the Configurator the way a Betaflight flight controller would, sends the receiver into its bootloader, then copies bytes between USB and the receiver.

1. Open **File → Examples → AlfredoCRSF → elrsPassthrough**.
2. Change the two pin defines near the top to Rotini's receiver UART:

    ```cpp
    #define PIN_RX 7
    #define PIN_TX 8
    ```

3. Upload it with the same board settings you use for SimpleMelt. USB CDC On Boot must be enabled.

!!! danger "Motors off"
    While passthrough is running the S2 does nothing else. It stops parsing CRSF, so failsafe stops with it. Unplug ESC power before you start.

## 2. Build and flash from the Configurator

| Setting | Value |
|---|---|
| Releases | Latest stable (4.x at time of writing) |
| Device category | BETAFPV 2.4 GHz |
| Device | BETAFPV 2.4GHz Lite RX |
| Flashing method | **Betaflight Passthrough** |
| Regulatory domain | ISM 2400 for most of the world, EU CE 2400 in the EU |
| Binding phrase | The same phrase as your transmitter module |
| WiFi SSID / password | Optional. Only needed if you want WiFi updates too |

Then:

1. Under Serial Device, pick Rotini's USB port. It is the same port you upload sketches to.
2. Click **Build & Flash**.
3. Wait for the log to end with **SUCCESS**. The first build downloads a toolchain and takes a few minutes.

The Configurator drives the whole process. You should see it enter the CLI, ask for passthrough, then run esptool against the receiver.

!!! info "Why a BetaFPV target"
    Rotini has no BetaFPV parts on it. The target is chosen for its pinout. The SX1280 is wired to ESP8285 GPIO2 (reset), GPIO4 (DIO1), GPIO5 (busy), GPIO12 to GPIO14 (SPI) and GPIO15 (chip select), with the status LED on GPIO16. That is ExpressLRS's "Generic 2400" ESP8285 layout, which the BETAFPV 2.4GHz Lite RX target uses. The old DIY 2400 RX ESP8285 SX1280 target used the same layout but has been removed from the Configurator. Any other target built on the Generic 2400 layout works too, such as the Foxeer Lite or JHEMCU Lite 2.4 GHz RX.

## 3. Put SimpleMelt back

Upload your normal SimpleMelt sketch again. Power cycle the board.

- The yellow LED blinks slowly: the receiver is running and waiting for a link.
- Power your transmitter with the same binding phrase. The yellow LED goes solid when connected.

If you set no binding phrase, put the receiver into bind mode by power cycling it three times, then bind from your transmitter's ExpressLRS Lua script.

## Manual passthrough with esptool

The sketch also has a hand-driven mode. Open a serial terminal on Rotini's port at 115200 baud, type `bl` and press Enter. The S2 sends the receiver into its bootloader and starts bridging. Close the terminal and run esptool against the same port:

```
esptool --passthrough --chip esp8266 --port <port> --baud 420000 --before no_reset --after hard_reset write_flash 0x0 firmware.bin
```

Use the `.bin` the Configurator produces with the **Build** button. Reset the S2 to leave passthrough mode.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Configurator says it cannot find a Betaflight CLI | Wrong port, or the passthrough sketch is not running. Re-upload it and check `PIN_RX` / `PIN_TX` are 7 and 8 |
| CLI works but esptool cannot sync | The receiver did not reboot into its bootloader. Raise `BOOTLOADER_REBOOT_MS` in the sketch to 500 and retry. If the receiver is blank, use the UART method below |
| Flash succeeds but the yellow LED never blinks | Power cycle. If still dark, the wrong target was flashed |
| Receiver connects but SimpleMelt sees no channels | You are still running the passthrough sketch. Upload SimpleMelt |

## Flashing a blank receiver

A new board's ESP8285 has no ExpressLRS on it yet, so it cannot act on the bootloader command. Flash it once over UART. After this, use passthrough for every update.

You need a 3.3 V USB-to-UART adapter and something to reach the J1 header (a 6-pin JST-SH 1.0 mm cable) or the test pads.

| J1 pin | Net | Connect to |
|---|---|---|
| 1 | GND | Adapter GND, and one side of the boot jumper |
| 2 | ELRS_BOOT | Other side of the boot jumper (short to pin 1) |
| 3 | ELRS_RX | Adapter **TX** |
| 4 | ELRS_TX | Adapter **RX** |
| 5 | S2_BOOT | Leave open |
| 6 | 3V3 | Leave open when powering from USB-C |

1. Make sure the ESP32-S2 is not driving the UART. Either do this before SimpleMelt is loaded, or ground TP5 (S2_EN) to hold the S2 in reset.
2. Short ELRS_BOOT (J1 pin 2) to GND (J1 pin 1).
3. Connect adapter GND, TX and RX as in the table. TX and RX cross over.
4. Plug in USB-C to power the board. The ESP8285 samples GPIO0 only at power-up, so the jumper can come off once the board is powered.
5. In the Configurator use the same settings as above but with Flashing method **UART**, pick the adapter's COM port, and click **Build & Flash**.
6. Unplug everything, remove the jumper, and power cycle. The yellow LED blinks slowly.

If the adapter cannot connect, BOOT was not grounded at power-up or TX and RX are swapped. If the adapter's log shows garbage, it is set to 5 V. Use 3.3 V.
