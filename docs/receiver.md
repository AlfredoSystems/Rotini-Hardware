# The Receiver (ESP8285)

Rotini's ESP8285 and SX1280 form a complete ExpressLRS 2.4 GHz receiver. It ships blank, so you flash ExpressLRS onto it once over UART. After that you can update it over WiFi.

## What you need

- A 3.3 V USB-to-UART adapter (CP2102, CH340, FT232 or similar). Set it to 3.3 V logic. 5 V will damage the ESP8285.
- A 6-pin JST-SH 1.0 mm cable for the J1 header, or a way to probe the test pads.
- The [ExpressLRS Configurator](https://github.com/ExpressLRS/ExpressLRS-Configurator/releases).
- A USB-C cable to power the board.

## 1. Flash the receiver before the brain

The ESP32-S2 shares the receiver's UART. Once SimpleMelt is running, the S2 drives the receiver's RX line and your adapter will fight with it. Do one of the following:

- Flash ExpressLRS while the ESP32-S2 is still blank or only running a blink sketch (easiest).
- Hold the ESP32-S2 in reset by jumpering TP5 (S2_EN) to GND while you flash.

## 2. Wire it up

| J1 pin | Net | Connect to |
|---|---|---|
| 1 | GND | Adapter GND, and one side of your boot jumper |
| 2 | ELRS_BOOT | Other side of the boot jumper (short to pin 1) |
| 3 | ELRS_RX | Adapter **TX** |
| 4 | ELRS_TX | Adapter **RX** |
| 5 | S2_BOOT | Leave open |
| 6 | 3V3 | Leave open when powering from USB-C |

1. Short ELRS_BOOT (pin 2) to GND (pin 1). A bent header pin or a wire in the JST cable works.
2. Connect adapter GND, TX and RX as in the table. TX and RX cross over.
3. Power the board by plugging in USB-C. The blue LED should light.

!!! tip "Powering from the adapter instead"
    You can skip USB-C and feed the adapter's 3.3 V output into J1 pin 6. Many adapters only supply 100 to 200 mA, which is marginal for the S2, ESP8285 and SX1280 together. USB-C is more reliable.

The ESP8285 samples GPIO0 only at power-up. Once it is powered with BOOT shorted it stays in download mode, and you can remove the jumper if it is in the way.

## 3. Build and flash

In the ExpressLRS Configurator:

| Setting | Value |
|---|---|
| Releases | Latest stable (4.x at time of writing) |
| Device category | DIY 2.4 GHz |
| Device | DIY 2400 RX ESP8285 SX1280 |
| Flashing method | UART |
| Regulatory domain | ISM 2400 for most of the world, EU CE 2400 in the EU |
| Binding phrase | The same phrase as your transmitter module |
| WiFi SSID / password | Optional. Fill in to enable WiFi updates on your network |
| AUTO_WIFI_ON_INTERVAL | Optional. The receiver enters WiFi mode after this many seconds without a link. Leave it on if you want WiFi updates; turn it off if it annoys you during testing |

Then:

1. Select your adapter's COM port under Serial Device.
2. Click **Build & Flash**.
3. Wait for the log to end with **SUCCESS**. The build step downloads a toolchain the first time and can take a few minutes.

!!! info "Why this target"
    The SX1280 is wired to ESP8285 GPIO2 (reset), GPIO4 (DIO1), GPIO5 (busy), GPIO12 to GPIO14 (SPI) and GPIO15 (chip select), with the status LED on GPIO16. That is the pinout of the ExpressLRS DIY 2400 RX ESP8285 SX1280 target, so no custom hardware definition is needed.

## 4. Verify

1. Unplug the board, remove the boot jumper and adapter, and plug USB-C back in.
2. The yellow LED should blink slowly. That means the receiver is running and waiting for a link.
3. Power your transmitter with the same binding phrase. The yellow LED goes solid when connected.

If you set no binding phrase, put the receiver into bind mode by power cycling it three times, then bind from your transmitter's ExpressLRS Lua script.

## Updating later over WiFi

With the receiver powered and no transmitter link, wait for the WiFi timeout (60 seconds by default). The yellow LED blinks fast. Either:

- Join the `ExpressLRS RX` access point (password `expresslrs`) and open `http://10.0.0.1`, or
- If you entered your home WiFi credentials, open `http://elrs_rx.local`.

Upload the `.bin` built by the configurator with Flashing method set to WiFi. UART flashing always works as a fallback.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Configurator cannot connect, or times out | BOOT was not grounded at power-up, or TX and RX are swapped |
| Flash succeeds but the yellow LED never blinks | Power cycle without the boot jumper. If still dark, the wrong target was flashed |
| Flashing fails partway through | The ESP32-S2 is driving the UART. Ground TP5 to hold it in reset |
| Adapter shows garbage in the log | Adapter is set to 5 V, or the baud rate is wrong. Use 3.3 V and let the configurator pick the baud |
