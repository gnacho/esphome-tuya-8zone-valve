# Controlador de Riego 8 Zonas Tuya → ESPHome

*[English version](README.md)*

## La historia

Recibí un **controlador de riego inteligente de 8 zonas** de AliExpress. Un cacharro chino genérico con WiFi, controlable por la app Tuya Smart / Smart Life. El modelo en la PCB es **TY-W-8L-AC-DZAK** y lleva un SoC **Beken BK7231N** (módulo CBU).

Lo primero que hice fue integrarlo en Home Assistant vía **localtuya** ([guía y herramientas aquí](https://github.com/gnacho/tuya-8zone-valve-3.5)). Funcionaba, pero con limitaciones: dependía de la cloud de Tuya para algunas zonas, el polling daba errores cada 60 segundos, y no tenía control fino del hardware.

Así que decidí ir más allá: **flashear ESPHome** y tener control total, local y sin dependencia de servidores chinos.

## El hardware

- **SoC**: Beken BK7231N (módulo CBU) — NO es un ESP32
- **PCB**: TY-W-8L-AC-DZAK rev 24.9.19
- **Shift registers**: 2x 74HC595 **independientes** (no en cascada)
  - **SR de LEDs** (U2): data P9, clock P15, latch P17 → 8 LEDs de estado D1..D8
  - **SR de válvulas** (U3): data P16, clock P22, latch P20 → 8 triacs BT134 → bornas Z1..Z8
- **Buzzer**: P14
- **LED WiFi**: P28
- **Alimentación**: 24 VAC
- **Botones táctiles**: P6, P7, P8, P26 (anterior/next/start/stop)
- **Sensor de lluvia**: P14 (entrada, para sensor externo)

## El flasheo

El BK7231N se flashea por UART con [ltchiptool](https://github.com/libretiny-eu/ltchiptool). El proceso:

1. Abrir el dispositivo y localizar los pines UART (TX, RX, GND, 3.3V)
2. Poner en modo descarga (puentear RST a GND durante el arranque)
3. Alimentar por 24VAC (el regulador interno genera los 3.3V)
4. Flashear con `ltchiptool flash`

Una vez flasheado, las actualizaciones son **OTA** (por WiFi, sin cables).

## Estado actual

**¡FUNCIONA!** Las 8 válvulas y los 8 LEDs operan correctamente con la topología de 2 shift registers independientes.

- ✅ WiFi conectado, web UI accesible, API de Home Assistant operativa
- ✅ OTA funcionando (actualizaciones inalámbricas)
- ✅ 8 válvulas controladas individualmente
- ✅ 8 LEDs sincronizados con sus válvulas (encienden al abrir)
- ✅ Componente Sprinkler activo (programación, duraciones, auto-avance)
- ✅ Botones táctiles: anterior/siguiente/iniciar/parar
- ⚠️ LED WiFi (P28): en pruebas
- ⚠️ Sensor de lluvia (P14): sin sensor físico conectado

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `irrigador-8z-sep.yaml` | **Firmware principal** — 2 SR independientes, sprinkler completo, LEDs auto |
| `irrigador-8z.yaml` | Firmware anterior (2 SR en cascada, no funciona en esta PCB) |
| `irrigador-8z-diag.yaml` | Firmware de diagnóstico (bits individuales, sweep de pines) |
| `secrets.yaml.example` | Plantilla de secretos (WiFi, API key, OTA password) |

## Requisitos

- ESPHome 2026.5.3+
- LibreTiny framework (integrado en ESPHome moderno)
- Home Assistant (opcional, para integración)

## Licencia

AGPL-3.0
