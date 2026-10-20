# Wake a Xiaomi (MediaTek) Android TV over Bluetooth with an ESP32, even after a power cut

Home Assistant turns my Xiaomi Mi TV P1 on with Wake-on-LAN. That works from standby only.
After a power cut the TV sits in a cold-off state: no network, WoL packets do nothing, and
the only thing that brings it back is the power button on the Bluetooth remote. I wanted
an automation that works from that state too, so I made an ESP32-C3 pretend to be the remote.

It works. Standby to awake in about 3 seconds, cold-off to booted in about 30 seconds.
This repo has the ESPHome component, an example config, and the notes on how the TV decides
to wake, because your TV will almost certainly need a different packet.

## What the TV actually listens for

Everything below was read from the TV itself over SSH with root (SimpleSSHD + Magisk).

The TV's Bluetooth chip is a MediaTek MT7668. When Android suspends, the `btmtk_usb` driver
puts the chip into "wake over BLE" mode. `dmesg` shows it:

```
btmtk_usb_handle_entering_WoBLE_state: begin
btmtk_usb_set_Woble_APCF: ... rc list is 54:A4:93:4D:1A:4B;
btmtk_usb_set_Woble_APCF_filter_parameter: rc_num = 1
```

Two filters are armed:

1. An address list. It comes from the Android property `persist.bluetooth.wakeuprcaddr`,
   which `/init.rc` copies into `/sys/class/BT_chrdev/stpbt/rclist`. By default it holds the
   paired remote's address.
2. An Android APCF packet filter on the advertisement content. The chip only wakes the SoC
   when a packet from a listed address also matches the content filter.

Things that do not wake it, all tested: Wake-on-LAN from cold-off, directed advertising to the
TV's address (high and low duty cycle), plain undirected advertising from a listed address, and
the manufacturer-data template found in `/vendor/firmware/woble_setting_7668.bin`. That file
looks like the answer and is not; the driver logs `woble_setting_apcf[0].length 0` and uses its
own defaults.

What does wake it is the exact packet the remote sends. I captured it with a second ESP32
running ESPHome's `esp32_ble_tracker` and a lambda that dumps `get_scan_result()`. The remote
("MI RC", address 54:A4:93:4D:1A:4B) sends this on a power press:

```
02 01 05              flags
03 FF 00 01           manufacturer data, company 0x0100, byte 01 = power (a Settings press sends 00 00)
06 08 4D 49 20 52 43  short name "MI RC"
03 02 12 18           16-bit UUID 0x1812 (HID)
04 0D 04 05 00        class of device
02 0A 00              TX power
04 FE E9 FA C2        vendor field with the last three bytes of the TV's own address (60:23:A4:C2:FA:E9)
```

Replaying those 31 bytes from the ESP32 wakes the TV from standby, as long as the ESP32's
address is in `rclist`. From cold-off it does not, because Android never ran after the power
cut and `rclist` was never loaded; the chip only knows the remote's address from before.
So the ESP32 has to use the remote's address. ESP-IDF allows that with
`esp_iface_mac_addr_set(mac, ESP_MAC_BT)` before the controller starts.

## The component

`components/ble_keyboard` is a fork of
[adesanto84/esphome-blekeyboard](https://github.com/adesanto84/esphome-blekeyboard), itself the
ESP-IDF/NimBLE port of [dmamontov/esphome-blekeyboard](https://github.com/dmamontov/esphome-blekeyboard).
Added:

- `ble_keyboard.mtk_wake`: sends a raw advertisement for N ms. The payload is the packet above,
  hardcoded in `start_mtk_wake()` in `ble_keyboard.cpp`; edit it for your TV.
- `bt_mac`: override the BT public address.
- `advertise_on_boot: false`: beacon mode. The board never advertises on its own and stays
  silent after a burst. Without this the board falls back to normal connectable HID advertising
  from the remote's address, the TV connects to it, re-bonds, and your real remote stops working.
  I learned that the hard way; a re-pair fixed it.
- `ble_keyboard.start_directed`: directed advertising. Kept for experiments, useless for this TV.
- `play(const Ts &...x)` signature fix for ESPHome 2026.5.

Wake bursts are non-connectable, so the TV can never connect to the beacon and the real remote's
bond stays intact.

## Setup

1. Root access to the TV. I used SimpleSSHD from F-Droid plus Magisk; adb works too while it works.
2. Read the wake list and the driver log:
   ```
   getprop persist.bluetooth.wakeuprcaddr
   dmesg | grep -i woble
   ```
   If you see `WoBLE` and an `rc list`, this approach applies.
3. Capture your remote's advertisement. Any ESP32 with ESPHome: add an `on_ble_advertise`
   handler filtered to the remote's address and log `format_hex_pretty(r.ble_adv, r.adv_data_len)`
   from `x.get_scan_result()`. Press the power button on the remote while the TV is off.
4. Put the bytes into `start_mtk_wake()`, set `bt_mac` to the remote's address, flash
   `example/tv-remote.yaml` (ESP32-C3, ESP-IDF).
5. Call the `tv_wake_mtk` service from Home Assistant. Test from standby first, then pull the plug.

## Limits

- One TV model, one firmware (Mi TV P1, Android 9, MT7668). The packet format will differ on
  other TVs. The method of finding it should not.
- The beacon only turns the TV on. Off is a separate path (I send KEYCODE_SLEEP over SSH).
- Two devices now share the remote's BT address. It has caused no trouble in a day of use because
  the beacon transmits for 10 seconds on request and is never connectable, but keep it in mind.
