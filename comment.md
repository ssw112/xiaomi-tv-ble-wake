For anyone trying to wake a MediaTek-based Xiaomi TV (mine is a Mi TV P1, MT7668 Bluetooth) rather than a Mi Box: a Power keycode is not the problem. In standby the TV's Bluetooth chip is in a "wake over BLE" mode and only wakes the SoC for an advertisement that matches a content filter and comes from the paired remote's address. No HID report ever reaches Android.

What worked: capture the remote's power-press advertisement with an ESP32 running esp32_ble_tracker, replay those 31 bytes from the ESP32 with the remote's BT address set via esp_iface_mac_addr_set, non-connectable. Wakes the TV from standby in about 3 s and from cold-off after a power cut in about 30 s.

Component (built on adesanto84's ESP-IDF port), example config and the notes on reading the TV's filter from dmesg: https://github.com/ssw112/xiaomi-tv-ble-wake
