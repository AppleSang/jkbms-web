# JK-BMS Monitor

A single-page, install-free web app to monitor and control **JK-BMS** battery management systems over **Bluetooth Low Energy**, built on the browser's **Web Bluetooth API**.

![Screens](https://img.shields.io/badge/platform-Chrome%20%2F%20Edge-blue) ![Protocol](https://img.shields.io/badge/protocol-JK02%20BLE-green) ![Firmware](https://img.shields.io/badge/firmware-%3C%2011%20and%20%E2%89%A5%2011%20supported-success)

## Features

- **Real-time battery data**: state of charge (SOC, with sub-percent precision on fw ≥ 11), remaining / configured / aged capacity (Ah), SOH, cycle count, total voltage, current, power
- **Per-cell voltages** with lowest/highest highlighting and delta (mV)
- **30-second sparkline** on every metric tile (voltage, current, power, cell min/max, delta, temperatures, balance current)
- **Temperatures**: battery sensor 1 & 2, MOSFET temperature (invalid sensor values are shown as `--`)
- **BMS controls**: charge / discharge / balancer switches, and **Emergency mode** (fw ≥ 11 only, with a danger confirmation dialog)
- **Automatic firmware-version detection** (SW ≥ 11.x → new 32-byte-offset frame layout, older → legacy 24S layout), with a manual override selector
- **Robust BLE framing**: resyncs on frame header mid-stream, tolerates junk bytes between frames (e.g. `AT\r\n` floods from JK-PB inverters), handles notifications split at arbitrary byte boundaries
- **Bilingual UI (English / Vietnamese)** — auto-detected from your IP address (GeoJS, with fallbacks), switchable with one click
- **Dark / light theme**, persisted in `localStorage`
- Desktop: logs go to the browser DevTools console; mobile: on-screen log panel with raw frame hex dumps

## Requirements

| Requirement | Details |
|---|---|
| Browser | Chrome or Edge (desktop, or Android). **Not** iOS Safari / Firefox — no Web Bluetooth there |
| Connection | Page must be served over **HTTPS** or **http://localhost** (Web Bluetooth requirement) |
| BMS | JK-BMS with a **BLE** module (JK02 protocol). Classic-Bluetooth-only (SPP) modules will not appear |
| Firmware | Both legacy (< 11) and new (≥ 11) firmwares are supported |

## Usage

1. Serve the folder over localhost (or any HTTPS host):

   ```bash
   python -m http.server 8080
   # then open http://localhost:8080
   ```

2. A connection popup appears immediately — press **Connect Bluetooth** and pick your BMS from the chooser. The page lists **all** nearby BLE devices (many JK modules advertise names that do not start with "JK", so no name filter is applied); the page validates the JK service (FFE0) after you pick.
3. Watch the data: the SOC ring, tiles with 30s sparklines, and the cell-voltage grid update live (the BMS pushes a status frame roughly once per second after the first poll).
4. Use **BMS Controls** to toggle charge / discharge / balancer, or trigger Emergency mode (a danger confirmation is shown; the switch state re-syncs from the BMS ~0.4 s after each command).
5. Header tools: frame-version override (`Auto-detect` recommended), **EN/VI** language toggle, dark/light theme toggle.
6. Troubleshooting:
   - *"No JK-BMS service (FFE0)"* — you picked a non-JK device, or the BMS is occupied by the vendor app (close the app / phone connection first).
   - Empty device list — ensure the BMS BLE module is powered and not connected elsewhere; Bluetooth adapter enabled.
   - Values look wrong — try the frame-version selector manually (`fw < 11` vs `fw ≥ 11`).
   - Page doesn't change after an update — hard refresh (`Ctrl+F5`), static servers cache aggressively.

> ⚠️ **Safety disclaimer**: the control buttons write directly to the BMS. Emergency mode bypasses cell protections and can damage the battery or create safety hazards. Use at your own risk.

## How it works (technical)

### BLE layer

- GATT service `0xFFE0`, characteristic `0xFFE1` (some modules expose two FFE1 characteristics: one write-only, one notify — the page picks them by declared properties).
- `requestDevice({ acceptAllDevices: true, optionalServices: [0xFFE0] })` — no name filter, because many JK modules advertise other/empty names.
- The notify characteristic is a **raw UART bridge**: packets do not respect frame boundaries, so the app accumulates a buffer, resyncs on the `55 AA EB 90` header, and consumes exactly 300-byte frames.

### Request frames (20 bytes)

```
AA 55 90 EB | ADDR | LEN | VALUE(4, LE) | zero-pad to 19 | CHECKSUM
```

- `CHECKSUM` = simple 8-bit sum of bytes 0..18.
- Commands used: `0x96` (request cell/status data), `0x97` (request device info).
- Switch writes: `ADDR` = `0x1D` charge, `0x1E` discharge, `0x1F` balancer, `0x6B` emergency (fw ≥ 11 only); `LEN` = 4, value `0x00000001` / `0x00000000`.
- Polling: `0x97` once after connect, `0x96` every 2 s until data flows, then a keep-alive if the stream goes silent > 9 s (the BMS auto-pushes afterwards).

### Response frames (300 bytes)

```
55 AA EB 90 | TYPE | COUNTER | payload... | CHECKSUM(last byte = sum of bytes 0..298)
```

Accepted types: `0x01` settings, `0x02` cell info, `0x03` device info. A valid checksum *and* a known type are required (an 8-bit sum gives ~1/256 false-positive odds for junk that happens to look like a header).

### Version auto-detection

The device-info frame (`0x03`) carries the firmware version at offset 30. `major ≥ 11` → the "new" frame layout (every field after the cell/resistance blocks shifts by **+32** bytes); otherwise the legacy 24S layout. This mirrors the approach proven in batmon-ha and avoids the mis-decoding you get from guessing via the enabled-cells bitmask.

### Cell-info frame offsets (`0x02`)

| Offset | Field | Coefficient |
|---|---|---|
| 6 + 2·i | cell voltage i (24 or 32 slots) | mV |
| 54..57 | enabled-cells bitmask | — |
| 118+off | total voltage | 0.001 V |
| 126+off | current (signed; **positive = charging**) | 0.001 A |
| 130+off / 132+off | temperature sensor 1 / 2 (−2000 = absent) | 0.1 °C |
| 112+off (new) / 134+off (old) | MOS temperature | 0.1 °C |
| 134+off (new) / 136+off (old) | error bitmask | — |
| 138+off | balance current | 0.001 A |
| 140+off | balancer action (0 idle / 1 charge-bal / 2 discharge-bal) | — |
| 141+off | SOC byte (1 % resolution) | % |
| 142+off | remaining capacity | 0.001 Ah |
| 146+off | aged capacity (fw ≥ 11) / nominal (legacy) | 0.001 Ah |
| 150+off | cycle count | — |
| 158+off | SOH (fw ≥ 11) | % |
| 162+off | uptime | s |
| 166+off / 167+off | charge / discharge FET enabled | bool |
| 186+off | emergency timer | s |

`off` = 32 for the new layout, 0 for legacy.

### Settings frame (`0x01`)

Cell count @114, charge/discharge/balancer switches @118/122/126, user-configured capacity @130 (0.001 Ah). The **configured capacity is taken from this frame** — on fw ≥ 11 the cell-info capacity field holds a BMS-internal aged value that drifts from the user setting.

### SOC precision

The SOC byte is integer-percent only. On fw ≥ 11 the app recomputes `remaining / aged × 100` (both BMS-internal values), recovering the sub-percent precision the vendor app displays (e.g. `65.87 %` instead of `66 %`).

## Credits

This project stands on the shoulders of these excellent open-source projects:

- **[syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms)** — the annotated JK02/JK04 protocol documentation, per-byte frame maps, switch holding-register addresses (0x1D/0x1E/0x1F/0x6B) and write-frame format.
- **[fl4p/batmon-ha](https://github.com/fl4p/batmon-ha)** (and its bundled `bmslib`) — the robust `feed_frames` resync strategy, firmware-version-based frame-layout detection (SW ≥ 11 → offset 32), the capacity-from-settings-frame fix (issue #365), sub-percent SOC recomputation (issue #369), and real captured frames used as test fixtures.
- **[sstallion/go-jk-bms](https://github.com/sstallion/go-jk-bms)** and **[jblance/mpp-solar](https://github.com/mpp-solar)** — additional protocol references cited by the projects above.
- **[Web Bluetooth API](https://developer.mozilla.org/docs/Web/API/Web_Bluetooth_API)** — MDN documentation and Chrome implementation notes.
- Theme variables in `style.css` follow the shadcn/ui CSS-variable convention.

Licensed for personal and educational use; no warranty. Battery work is dangerous — verify anything safety-critical independently.
