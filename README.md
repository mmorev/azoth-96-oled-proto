# ASUS ROG Azoth 96 HE — Touchscreen OLED Protocol

Reverse-engineered documentation of the USB HID protocol that drives the touchscreen
OLED of the **ASUS ROG Azoth 96 HE** wireless keyboard (USB `VID 0x0B05` / `PID 0x1C10`,
internal codename **"M901"**). On Windows the display is normally fed by the proprietary
**GearLink** companion software; this project documents the wire protocol so it can be
implemented in third-party host software.

Everything here was obtained independently, from:

- USB traffic captures (USBPcap + Wireshark) during live GearLink sessions,
- decompilation of the GearLink host components,
- static analysis of the device firmware (Nordic nRF54H20, Zephyr RTOS),

and verified with live device tests. Findings are cross-referenced between all three
sources; open questions are marked as such in the text.

## Contents (`docs/`)

| File | What it covers |
|---|---|
| [`report.md`](docs/report.md) | Primary analysis: device/HID map, 64-byte vendor channel, opcode table, bitmap upload protocol |
| [`PROTOCOL.md`](docs/PROTOCOL.md) | Firmware file layout (SUIT DFU package), USB/HID transport, vendor protocol from the firmware side |
| [`PROTOCOL_OLED.md`](docs/PROTOCOL_OLED.md) | OLED internals: widget registry in the system controller, 0x62/0x67 bitmap protocols, RGB565 format, commit semantics |
| [`PROTOCOL_STATUSBAR.md`](docs/PROTOCOL_STATUSBAR.md) | Widget slots, slot mask (0x24/0x6A), status bar rendering path |
| [`PROTOCOL_VOLUME.md`](docs/PROTOCOL_VOLUME.md) | Volume rocker events and OSD: the full 0x51/0x25 chain |
| [`PROTOCOL_GESTURES.md`](docs/PROTOCOL_GESTURES.md) | Touchscreen gesture decoder/dispatcher; why swipe-up is invisible to the host |
| [`music-mode.md`](docs/music-mode.md) | Music mode: opcode 0x67 frame format, spectrum data, track-info bitmaps, GearLink pipeline |

## Notes

- The docs are written as working RE notes (in Russian) with absolute firmware addresses;
  references to local workspace files (`analysis/...`, capture PCAPs) describe the research
  environment and are not part of this repository.
- A standalone Python daemon that replaces GearLink (clock, PC battery, CPU/RAM metrics,
  slideshows) is planned as a separate release in this repository.

## License

Documentation is licensed under the [MIT License](LICENSE).

> Not affiliated with or endorsed by ASUS. Provided for interoperability and research
> purposes. Use at your own risk.
