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
- **Buzzer**: P14 (pitido al pulsar botones)
- **LED WiFi**: P28
- **Alimentación**: 24 VAC (transformador externo)
- **Botones táctiles**: UP (P7), DOWN (P6), SET/Círculo (P8)

## Cómo flashear (paso a paso)

### Qué necesitas

1. **Adaptador USB-TTL 3.3V** (también llamado "adaptador serial USB" o "CP2102/CH340/FT232"). Cuesta ~3-5€ en AliExpress o Amazon. **IMPORTANTE**: debe ser de **3.3V**, NO de 5V (puedes freír la placa).
2. **4 cables dupont** (cables con conectores hembra-hembra). Vienen con el adaptador o se compran aparte.
3. **Transformador 24VAC** (el que viene con el irrigador).
4. **Ordenador** con Python 3 instalado.

### Instalar ltchiptool

Abre una terminal (CMD en Windows, Terminal en Mac/Linux) y ejecuta:

```bash
pip install ltchiptool
```

Si da error de permisos:

```bash
pip install --user ltchiptool
```

### Conexiones (sin soldar)

**⚠️ ADVERTENCIA CRÍTICA**: **NUNCA conectes 3.3V del adaptador USB y 24VAC al mismo tiempo**. Puedes dañar la placa.

**Opción A: Solo con 3.3V (preferible si funciona)**

Conecta los 4 cables dupont así:

```
  Adaptador USB-TTL          Placa TY-W-8L-AC-DZAK
  ─────────────────          ─────────────────────
       3.3V  ───────────────────►  3.3V
        GND  ───────────────────►  GND
         TX  ───────────────────►  RX
         RX  ◄───────────────────  TX
```

**Truco**: Los cables se cruzan: TX del adaptador va a RX de la placa, y RX del adaptador va a TX de la placa.

Prueba primero así. Si el flasheo funciona, perfecto.

**Opción B: Con 24VAC (si 3.3V no funciona)**

Si la Opción A falla (la placa no responde, timeout, etc.), desconecta el cable de 3.3V y usa el transformador 24VAC:

```
  Adaptador USB-TTL          Placa TY-W-8L-AC-DZAK
  ─────────────────          ─────────────────────
        GND  ───────────────────►  GND
         TX  ───────────────────►  RX
         RX  ◄───────────────────  TX

  Transformador 24VAC        Placa TY-W-8L-AC-DZAK
  ─────────────────          ─────────────────────
       24VAC ───────────────────►  AC IN (bornas)
```

**⚠️ IMPORTANTE**: En la Opción B, **NO conectes el cable 3.3V del adaptador USB**. Solo GND, TX y RX. La placa genera sus propios 3.3V internamente desde el 24VAC.

### Paso 1: Backup del firmware original (recomendado)

Antes de flashear nada, guarda el firmware de fábrica por si algún día quieres volver atrás:

```bash
ltchiptool flash read bk7231n backup_original.bin
```

Usa el mismo cableado y modo descarga que el flasheo (ver abajo), así que merece la pena hacerlo ya que estás conectado.

### Paso 2: Flashear ESPHome

1. **Conecta los 4 cables** dupont a la placa (puedes sujetarlos con los dedos, no hace falta soldar).
2. **Enchufa el transformador 24VAC** a la placa.
3. **Pon en modo descarga**: con un cable suelto, toca brevemente el pin **RST** con **GND** (un par de toques rápidos). Esto reinicia la placa en modo de programación.
4. **Inmediatamente después**, ejecuta en la terminal:

```bash
ltchiptool flash write bk7231n sprinkler-8z.bin
```

(Descarga `sprinkler-8z.bin` de [Releases](https://github.com/gnacho/esphome-tuya-8zone-valve/releases), o compila tú mismo `sprinkler-8z.yaml` con ESPHome.)

Si todo va bien, verás una barra de progreso y al final "Flash complete".

### Después del flasheo

1. **Desenchufa el transformador** y desconecta los cables dupont.
2. **Vuelve a enchufar el transformador** (solo 24VAC, sin cables USB).
3. La placa arrancará y creará una red WiFi llamada `Sprinkler-8Z Fallback` (contraseña: `12345678`).
4. **Conéctate a esa red** con tu móvil/ordenador.
5. Se abrirá automáticamente una página web (o ve a `http://192.168.4.1`).
6. **Introduce el nombre y contraseña de tu WiFi** y guarda.
7. La placa se conectará a tu WiFi. Desde ahora, podrás acceder a ella en `http://sprinkler-8z.local` (o por su IP si conoces).

**¡Listo!** Ya puedes controlar el riego desde la web o desde Home Assistant.

### Servidor web integrado

El firmware incluye un **servidor web** sin contraseña. Entra en `http://sprinkler-8z.local` para ver la configuración por defecto nada más arrancar: los 8 interruptores de zona, los deslizadores de duración de riego (5 min por zona por defecto), los controles del controlador de riego (inicio/parada, auto-avance) y los logs en vivo. Todo funciona de forma autónoma — Home Assistant es opcional.

### Actualizaciones OTA (tras el primer flasheo)

El cableado USB-TTL solo hace falta **una vez**. Después del primer flasheo, todas las actualizaciones de firmware se hacen **por OTA** vía WiFi: edita `sprinkler-8z.yaml` si quieres y ejecuta `esphome run sprinkler-8z.yaml` (o usa el dashboard de ESPHome). No hace falta abrir la caja ni tocar RST/GND otra vez.

## Uso de los botones

### Navegación y selección de zona

- **UP** (flecha arriba): selecciona la zona siguiente (el LED parpadea)
- **DOWN** (flecha abajo): selecciona la zona anterior (el LED parpadea)
- **SET/Círculo**: confirma la selección y activa la zona (el LED queda fijo)

Si no pulsas nada durante **8 segundos**, la selección se cancela automáticamente y los LEDs vuelven a su estado normal.

### Activar una zona individual

1. Pulsa UP o DOWN hasta que parpadee el LED de la zona deseada
2. Pulsa círculo → la zona se activa (LED fijo)
3. Para apagarla: navega con flechas hasta esa zona (parpadea sobre el LED fijo) y pulsa círculo

### Activar todas las zonas (ciclo completo)

1. Mantén **UP** presionado **4 segundos** hasta que todos los LEDs parpadeen
2. Pulsa círculo → se activan las 8 zonas en ciclo completo (3 pitidos de confirmación)

### Parar todo el riego

1. Mantén **DOWN** presionado **4 segundos** hasta que todos los LEDs parpadeen
2. Pulsa círculo → se cierran todas las zonas (3 pitidos de confirmación)

### Notas

- Cada pulsación de botón produce un pitido corto de confirmación
- Los tiempos de riego se ajustan desde la web (por defecto 5 min por zona)
- Con "Auto Avance" activado, las zonas se ejecutan en orden con 5s de solapamiento
- El pin P26 está reservado para un posible cuarto botón (no presente en esta PCB)

## Ajustar tiempos de riego

Desde la web (`http://sprinkler-8z.local`), cada zona tiene un control deslizante para ajustar la duración (por defecto 5 minutos). El valor se guarda en la placa y no se pierde aunque se vaya la luz.

## Estado actual

**¡100% FUNCIONAL!** Todas las características del hardware operan correctamente.

- ✅ WiFi conectado, web UI accesible, API de Home Assistant operativa
- ✅ OTA funcionando (actualizaciones inalámbricas)
- ✅ 8 válvulas controladas individualmente
- ✅ 8 LEDs sincronizados con sus válvulas (encienden al abrir)
- ✅ Componente Sprinkler activo (programación, duraciones ajustables desde web, auto-avance)
- ✅ Buzzer P14: pitido de confirmación al pulsar botones
- ✅ Botones táctiles funcionales:
  - **UP** (P7): zona siguiente
  - **DOWN** (P6): zona anterior
  - **SET/Círculo** (P8): toggle inteligente — si se navegó con flechas, riega solo esa zona; si no, ciclo completo de 8 zonas; si está regando, para todo
- ✅ Tiempos de riego persistentes en flash (ajustables desde web sin Home Assistant)
- ✅ LED WiFi (P28): encendido en rojo

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `sprinkler-8z.yaml` | Código fuente ESPHome (compílalo o modifícalo a tu gusto) |
| `sprinkler-8z.bin` | **Firmware compilado listo para flashear** (descargar de [Releases](https://github.com/gnacho/esphome-tuya-8zone-valve/releases)) |
| `secrets.yaml.example` | Plantilla de secretos (WiFi, API key, OTA password) |

*Las configs antiguas con nombre en español (`irrigador-8z*.yaml`) se eliminaron en v1.2.0 — `sprinkler-8z.yaml` es la traducción directa al inglés del antiguo `irrigador-8z-sep.yaml`, idéntica en funcionalidad.*

## Requisitos

- ESPHome 2026.5.3+
- LibreTiny framework (integrado en ESPHome moderno)
- Home Assistant (opcional, para integración)

## Licencia

AGPL-3.0
