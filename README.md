# 8-Zone Tuya Irrigation Controller → ESPHome

*[Versión en español](README.es.md)*

## The story

I received an **8-zone smart irrigation controller** from AliExpress. A generic Chinese device with WiFi, controllable via the Tuya Smart / Smart Life app. The PCB model is **TY-W-8L-AC-DZAK** and it uses a **Beken BK7231N** SoC (CBU module).

The first thing I did was integrate it into Home Assistant via **localtuya** ([guide and tools here](https://github.com/gnacho/tuya-8zone-valve-3.5)). It worked, but with limitations: it depended on Tuya cloud for some zones, polling gave errors every 60 seconds, and I didn't have fine control over the hardware.

So I decided to go further: **flash ESPHome** and have total, local control without dependency on Chinese servers.

## The hardware

- **SoC**: Beken BK7231N (CBU module) — NOT an ESP32
- **PCB**: TY-W-8L-AC-DZAK rev 24.9.19
- **Shift registers**: 2x 74HC595 **independent** (not cascaded)
  - **LED SR** (U2): data P9, clock P15, latch P17 → 8 D1..D8 status LEDs
  - **Valve SR** (U3): data P16, clock P22, latch P20 → 8 BT14 triacs → Z1..Z8 terminals
- **Buzzer**: P14
- **WiFi LED**: P28
- **Power**: 24 VAC
- **Touch buttons**: P6, P7, P8, P26 (prev/next/start/stop)
- **Rain sensor**: P14 (input, for external sensor)

## Current status

**100% FUNCTIONAL!** All hardware features operate correctly.

- ✅ WiFi connected, web UI accessible, Home Assistant API operational
- ✅ OTA working (wireless updates)
- ✅ 8 valves individually controlled
- ✅ 8 LEDs synced with their valves (turn on when valve opens)
- ✅ Sprinkler component active (scheduling, adjustable durations from web, auto-advance)
- ✅ Buzzer P14: confirmation beep on button press
- ✅ Touch buttons functional:
  - **UP** (P7): next zone
  - **DOWN** (P6): previous zone
  - **SET/Circle** (P8): smart toggle — if navigated with arrows, waters only that zone; otherwise full 8-zone cycle; if watering, stops everything
- ✅ Irrigation durations persist in flash (adjustable from web without Home Assistant)
- ⚠️ WiFi LED (P28): under testing

## Files

| File | Description |
|------|-------------|
| `irrigador-8z-sep.yaml` | **Main firmware** — 2 independent SRs, full sprinkler, auto LEDs |
| `irrigador-8z.yaml` | Previous firmware (2 SRs cascaded, doesn't work on this PCB) |
| `irrigador-8z-diag.yaml` | Diagnostic firmware (individual bits, pin sweep) |
| `secrets.yaml.example` | Secrets template (WiFi, API key, OTA password) |

## Requirements

- ESPHome 2026.5.3+
- LibreTiny framework (integrated in modern ESPHome)
- Home Assistant (optional, for integration)

## License

AGPL-3.0
