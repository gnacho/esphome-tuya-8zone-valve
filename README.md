# Controlador de Riego 8 Zonas Tuya → ESPHome

## La historia

Recibí un **controlador de riego inteligente de 8 zonas** de AliExpress. Un cacharro chino genérico con WiFi, controlable por la app Tuya Smart / Smart Life. El modelo en la PCB es **TY-W-8L-AC-DZAK** y lleva un SoC **Beken BK7231N** (módulo CBU).

Lo primero que hice fue integrarlo en Home Assistant vía **localtuya** ([guía y herramientas aquí](https://github.com/gnacho/tuya-keys-extractor)). Funcionaba, pero con limitaciones: dependía de la cloud de Tuya para algunas zonas, el polling daba errores cada 60 segundos, y no tenía control fino del hardware.

Así que decidí ir más allá: **flashear ESPHome** y tener control total, local y sin dependencia de servidores chinos.

## El hardware

- **SoC**: Beken BK7231N (módulo CBU) — NO es un ESP32
- **PCB**: TY-W-8L-AC-DZAK rev 24.9.19
- **Shift registers**: 2x 74HC595 en cascada (16 salidas)
  - Primer chip (bits 0-7): 8 triacs BT134 → bornas Z1..Z8 (válvulas de riego)
  - Segundo chip (bits 8-15): 8 LEDs de estado D1..D8
- **Pines confirmados del shift register**:
  - P6 = data (SER)
  - P15 = clock (SRCLK)
  - P17 = latch (RCLK)
  - P8 = Output Enable (OE, activo en LOW)
- **Buzzer**: P14
- **LED WiFi**: P28
- **Alimentación**: 24 VAC
- **Botones físicos**: SET, UP, DOWN (pines por identificar)

## El flasheo

El BK7231N se flashea por UART con [ltchiptool](https://github.com/libretiny-eu/ltchiptool). El proceso:

1. Abrir el dispositivo y localizar los pines UART (TX, RX, GND, 3.3V)
2. Poner en modo descarga (puentear CEN a GND durante el arranque)
3. Alimentar por 24VAC (el regulador interno genera los 3.3V)
4. Flashear con `ltchiptool flash`

Una vez flasheado, las actualizaciones son **OTA** (por WiFi, sin cables).

## Estado actual

El firmware ESPHome funciona: WiFi conectado, web UI accesible, API de Home Assistant operativa, OTA funcionando. Las 8 zonas de riego se controlan correctamente vía shift registers.

**En progreso** (método ensayo-error, sin ser ingeniero electrónico):

- Identificar los pines exactos de los 3 botones físicos (SET, UP, DOWN)
- Verificar el mapeo correcto de LEDs a zonas
- Ajustar lógica invertida donde sea necesario

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `irrigador-8z.yaml` | Firmware principal con sprinkler (8 zonas, duraciones ajustables) |
| `irrigador-8z-diag.yaml` | Firmware de diagnóstico (bits individuales, sweep de pines) |
| `secrets.yaml.example` | Plantilla de secretos (WiFi, API key, OTA password) |

## Requisitos

- ESPHome 2026.5.3+
- LibreTiny framework (integrado en ESPHome moderno)
- Home Assistant (opcional, para integración)

## Licencia

MIT
