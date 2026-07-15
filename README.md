<img width="960" height="960" alt="tcsepzt9qcdh1" src="https://github.com/user-attachments/assets/f7352d06-76f5-4397-9b43-53ddc21dea8f" />
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
  - **Valve SR** (U3): data P16, clock P22, latch P20 → 8 BT134 triacs → Z1..Z8 terminals
- **Buzzer**: P14 (beep on button press)
- **WiFi LED**: P28
- **Power**: 24 VAC (external transformer)
- **Touch buttons**: UP (P7), DOWN (P6), SET/Circle (P8)

## How to flash (step by step)

### What you need

1. **3.3V USB-TTL adapter** (also called "USB serial adapter" or "CP2102/CH340/FT232"). Costs ~$3-5 on AliExpress or Amazon. **IMPORTANT**: must be **3.3V**, NOT 5V (you can fry the board).
2. **4 dupont cables** (female-female jumper wires). Usually come with the adapter or can be bought separately.
3. **24VAC transformer** (the one that comes with the irrigator).
4. **Computer** with Python 3 installed.

### Install ltchiptool

Open a terminal (CMD on Windows, Terminal on Mac/Linux) and run:

```bash
pip install ltchiptool
```

If you get permission errors:

```bash
pip install --user ltchiptool
```

### Connections (no soldering)

**⚠️ CRITICAL WARNING**: **NEVER connect 3.3V from USB adapter and 24VAC at the same time**. You can damage the board.

**Option A: 3.3V only (preferable if it works)**

Connect the 4 dupont cables like this:

```
  USB-TTL Adapter            TY-W-8L-AC-DZAK Board
  ───────────────            ─────────────────────
       3.3V  ───────────────────►  3.3V
        GND  ───────────────────►  GND
         TX  ───────────────────►  RX
         RX  ◄───────────────────  TX
```

**Tip**: The cables cross: TX from adapter goes to RX on board, and RX from adapter goes to TX on board.

Try this first. If flashing works, great.

**Option B: With 24VAC (if 3.3V doesn't work)**

If Option A fails (board doesn't respond, timeout, etc.), disconnect the 3.3V cable and use the 24VAC transformer:

```
  USB-TTL Adapter            TY-W-8L-AC-DZAK Board
  ───────────────            ─────────────────────
        GND  ───────────────────►  GND
         TX  ───────────────────►  RX
         RX  ◄───────────────────  TX

  24VAC Transformer          TY-W-8L-AC-DZAK Board
  ─────────────────          ─────────────────────
       24VAC ───────────────────►  AC IN (terminals)
```

**⚠️ IMPORTANT**: In Option B, **DO NOT connect the 3.3V cable from the USB adapter**. Only GND, TX and RX. The board generates its own 3.3V internally from the 24VAC.

### Flashing process

1. **Connect the 4 dupont cables** to the board (you can hold them with your fingers, no soldering needed).
2. **Plug in the 24VAC transformer** to the board.
3. **Enter download mode**: with a loose wire, briefly touch the **RST** pin to **GND** (a couple of quick taps). This restarts the board in programming mode.
4. **Immediately after**, run in the terminal:

```bash
ltchiptool flash write bk7231n irrigador-8z-sep.bin
```

If everything goes well, you'll see a progress bar and at the end "Flash complete".

### Backup original firmware (optional but recommended)

Before flashing, you can save the original firmware in case you want to revert:

```bash
ltchiptool flash read bk7231n backup_original.bin
```

### After flashing

1. **Unplug the transformer** and disconnect the dupont cables.
2. **Plug the transformer back in** (only 24VAC, no USB cables).
3. The board will boot and create an **open WiFi network** called `Irrigador-8Z-SEP Fallback`.
4. **Connect to that network** with your phone/computer.
5. A web page will open automatically (or go to `http://192.168.4.1`).
6. **Enter your WiFi name and password** and save.
7. The board will connect to your WiFi. From now on, you can access it at `http://irrigador-8z-sep.local` (or by its IP if you know it).

**Done!** You can now control irrigation from the web or from Home Assistant.

## Using the buttons

- **UP** (up arrow): advance to next zone
- **DOWN** (down arrow): go back to previous zone
- **SET/Circle**: 
  - If you used arrows to choose a zone → waters **only that zone**
  - If you didn't touch anything → waters **all 8 zones in full cycle**
  - If it's watering → **stops everything**

Each time you press a button, a **beep** sounds for confirmation.

## Adjusting irrigation times

From the web (`http://irrigador-8z-sep.local`), each zone has a slider to adjust duration (default 5 minutes). The value is saved on the board and won't be lost even if power goes out.

## Current status

**100% FUNCTIONAL!** All hardware features operate correctly.

- ✅ WiFi connected, web UI accessible, Home Assistant API operational
- ✅ OTA working (wireless updates)
- ✅ 8 valves individually controlled
- ✅ 8 LEDs synced with their valves (turn on when valve opens)
- ✅ Sprinkler component active (scheduling, adjustable durations from web, auto-advance)
- ✅ Buzzer P14: confirmation beep on button press
- ✅ Touch buttons functional with smart toggle logic
- ✅ Irrigation durations persist in flash (adjustable from web without Home Assistant)
- ✅ WiFi LED (P28): lit in red

## Files

| File | Description |
|------|-------------|
| `irrigador-8z-sep.bin` | **Compiled firmware ready to flash** (download from Releases) |
| `irrigador-8z-sep.yaml` | ESPHome source code (for those who want to compile or modify) |
| `irrigador-8z.yaml` | Previous firmware (2 SRs cascaded, doesn't work on this PCB) |
| `irrigador-8z-diag.yaml` | Diagnostic firmware (individual bits, pin sweep) |
| `secrets.yaml.example` | Secrets template (WiFi, API key, OTA password) |

## Requirements

- ESPHome 2026.5.3+
- LibreTiny framework (integrated in modern ESPHome)
- Home Assistant (optional, for integration)

## License

AGPL-3.0
<img width="4000" height="3000" alt="completo" src="https://github.com/user-attachments/assets/bcee812b-427a-4c48-b915-1d799bee2054" />
