# ⌚ WatchWinder 4x — ESP32-C3 + ESPHome + Home Assistant

Control inteligente para enrollador automático de 4 relojes mecánicos, basado en **ESP32-C3** con firmware **ESPHome**, integración nativa con **Home Assistant** y soporte para comunicación **ESP-NOW**.

---

## 📋 Tabla de contenidos

- [Características](#-características)
- [Hardware](#-hardware)
- [Arquitectura del sistema](#-arquitectura-del-sistema)
- [Esquema de conexiones](#-esquema-de-conexiones)
- [Firmware ESPHome](#-firmware-espHome)
- [Integración con Home Assistant](#-integración-con-home-assistant)
- [ESP-NOW](#-esp-now)
- [Modos de giro](#-modos-de-giro)
- [Instalación y configuración](#-instalación-y-configuración)
- [Estructura del proyecto](#-estructura-del-proyecto)
- [Roadmap](#-roadmap)
- [Licencia](#-licencia)

---

## ✨ Características

- Control independiente de **4 motores** (un motor por reloj)
- **4 modos de giro** configurables por reloj: horario, antihorario, bidireccional, parado
- **Turnos por día (TPD)** configurables individualmente (300–1200 TPD)
- Integración nativa con **Home Assistant** vía ESPHome API
- Comunicación **ESP-NOW** para entornos sin WiFi o como canal secundario
- Panel de control en **Home Assistant** con tarjetas por reloj
- **Temporizadores y planificación** horaria desde HA
- Indicador de estado con LED RGB WS2812
- Modo de **bajo consumo** cuando todos los relojes están en pausa
- Configuración persistente en flash (NVS)
- OTA (actualización firmware en remoto)

---

## 🔧 Hardware

### Lista de materiales (BOM)

| Componente | Modelo | Cantidad | Notas |
|---|---|---|---|
| Microcontrolador | ESP32-C3 (DevKitM-1 o similar) | 1 | Con antena integrada o externa |
| Driver de motores | A4988 o DRV8825 | 4 | Uno por motor |
| Motor paso a paso | NEMA 17 o 28BYJ-48 | 4 | Según diseño mecánico |
| LED indicador | WS2812B (Neopixel) | 1 | Estado del sistema |
| Fuente de alimentación | 12V 3A (motores paso a paso) | 1 | O 5V para 28BYJ-48 |
| Regulador 3.3V | AMS1117-3.3 o LDO similar | 1 | Alimentación ESP32-C3 |
| Condensadores | 100µF electrolítico + 100nF cerámico | varios | Desacoplo por driver |
| Resistencias pull-up | 10kΩ | 4 | ENABLE de drivers |
| Caja/estructura | Diseño propio (ver `/hardware/3d`) | 1 | Archivos STL incluidos |
| PCB | Ver `/hardware/pcb` | 1 | KiCad 8.x |

### Herramientas necesarias

- Soldador y estaño
- Multímetro
- Programador USB (USB-C o UART según módulo)
- Software: ESPHome CLI / Home Assistant Add-on

---

## 🏗️ Arquitectura del sistema

```
┌─────────────────────────────────────────────────────┐
│                  HOME ASSISTANT                     │
│  Dashboard  ·  Automatizaciones  ·  Historial       │
└──────────────────────┬──────────────────────────────┘
                       │ ESPHome API (WiFi)
                       │ ESP-NOW (fallback / broadcast)
┌──────────────────────▼──────────────────────────────┐
│              ESP32-C3 — WatchWinder FW              │
│                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐  │
│  │ Motor 1 │  │ Motor 2 │  │ Motor 3 │  │Motor 4│  │
│  │ A4988   │  │ A4988   │  │ A4988   │  │A4988  │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └───┬───┘  │
└───────┼────────────┼────────────┼────────────┼──────┘
        │            │            │            │
      NEMA17       NEMA17       NEMA17       NEMA17
      Reloj 1      Reloj 2      Reloj 3      Reloj 4
```

---

## 🔌 Esquema de conexiones

### ESP32-C3 → Drivers A4988

| ESP32-C3 GPIO | Señal | Driver (×4) |
|---|---|---|
| GPIO0 | STEP — Motor 1 | A4988 #1 STEP |
| GPIO1 | DIR — Motor 1 | A4988 #1 DIR |
| GPIO2 | STEP — Motor 2 | A4988 #2 STEP |
| GPIO3 | DIR — Motor 2 | A4988 #2 DIR |
| GPIO4 | STEP — Motor 3 | A4988 #3 STEP |
| GPIO5 | DIR — Motor 3 | A4988 #3 DIR |
| GPIO6 | STEP — Motor 4 | A4988 #4 STEP |
| GPIO7 | DIR — Motor 4 | A4988 #4 DIR |
| GPIO8 | ENABLE (todos) | A4988 #1–4 ENABLE |
| GPIO10 | DATA WS2812 | LED RGB |
| GND | Común | — |

> ⚠️ **ENABLE** activo en LOW. Usar pull-up a 3.3V para que los motores arranquen deshabilitados al inicio.

> ⚠️ Los drivers A4988/DRV8825 necesitan **alimentación separada** para los motores (12V) y lógica (3.3V/5V).

---

## 📦 Firmware ESPHome

### Requisitos

- ESPHome ≥ 2024.6
- Home Assistant (recomendado) o ESPHome CLI standalone

### Archivo de configuración principal

```yaml
# watchwinder.yaml
substitutions:
  device_name: watchwinder
  friendly_name: "WatchWinder 4x"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  platform: ESP32
  board: esp32-c3-devkitm-1

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${friendly_name} Fallback"
    password: !secret ap_password

api:
  encryption:
    key: !secret api_key

ota:
  password: !secret ota_password

logger:
  level: INFO

# LED de estado
light:
  - platform: neopixelbus
    type: GRB
    variant: WS2812
    pin: GPIO10
    num_leds: 1
    name: "Estado"
    id: status_led

# Control de motores (via stepper o custom component)
# Ver /firmware/components/watchwinder/

number:
  - platform: template
    name: "TPD Reloj 1"
    id: tpd_watch1
    min_value: 300
    max_value: 1200
    step: 50
    initial_value: 650
    optimistic: true
    restore_value: true

select:
  - platform: template
    name: "Modo Reloj 1"
    id: mode_watch1
    options:
      - "Parado"
      - "Horario"
      - "Antihorario"
      - "Bidireccional"
    initial_option: "Bidireccional"
    optimistic: true
    restore_value: true

switch:
  - platform: template
    name: "Activar Reloj 1"
    id: enable_watch1
    optimistic: true
    restore_state: true
```

> El archivo completo y los componentes custom están en `/firmware/`.

---

## 🏠 Integración con Home Assistant

### Dashboard — Tarjeta por reloj

```yaml
type: entities
title: WatchWinder — Reloj 1
entities:
  - entity: switch.watchwinder_activar_reloj_1
    name: Activado
  - entity: select.watchwinder_modo_reloj_1
    name: Modo de giro
  - entity: number.watchwinder_tpd_reloj_1
    name: Turnos por día
```

### Automatización de ejemplo — Iniciar al llegar a casa

```yaml
alias: WatchWinder — Arrancar al llegar
trigger:
  - platform: state
    entity_id: person.manuel
    to: home
action:
  - service: switch.turn_on
    target:
      entity_id:
        - switch.watchwinder_activar_reloj_1
        - switch.watchwinder_activar_reloj_2
        - switch.watchwinder_activar_reloj_3
        - switch.watchwinder_activar_reloj_4
```

---

## 📡 ESP-NOW

El firmware implementa **ESP-NOW** como canal de comunicación alternativo o complementario al WiFi:

- **Modo broadcast**: permite recibir comandos desde otros dispositivos ESP (botón físico, mando, etc.) sin necesidad de router.
- **Modo directo**: comunicación punto a punto con otro ESP32 actuando como gateway.
- **Fallback automático**: si el WiFi no está disponible, el dispositivo sigue operativo y acepta comandos ESP-NOW.

### Estructura del mensaje ESP-NOW

```c
typedef struct {
    uint8_t  watch_id;    // 0–3: número de reloj
    uint8_t  command;     // 0=stop, 1=CW, 2=CCW, 3=BI
    uint16_t tpd;         // Turnos por día (300–1200)
    uint8_t  enabled;     // 0=off, 1=on
} watchwinder_msg_t;
```

### Configuración del peer (ESP-NOW)

La MAC del gateway o del mando remoto se configura en `secrets.yaml`:

```yaml
espnow_peer_mac: "AA:BB:CC:DD:EE:FF"
```

---

## 🔄 Modos de giro

| Modo | Descripción | Recomendado para |
|---|---|---|
| **Parado** | Motor desactivado (bajo consumo) | Relojes en uso |
| **Horario (CW)** | Giro continuo en sentido horario | Relojes sólo CW |
| **Antihorario (CCW)** | Giro continuo en sentido antihorario | Relojes sólo CCW |
| **Bidireccional** | Alterna CW/CCW según programa TPD | La mayoría de automáticos |

El algoritmo bidireccional reparte los TPD en ciclos de 30 min alternando dirección, con pausas configurables entre ciclos.

---

## 🚀 Instalación y configuración

### 1. Clonar el repositorio

```bash
git clone https://github.com/tu-usuario/watchwinder-esp32c3.git
cd watchwinder-esp32c3
```

### 2. Configurar secrets

```bash
cp firmware/secrets.yaml.example firmware/secrets.yaml
# Editar con tus credenciales WiFi, API key, etc.
```

### 3. Flashear con ESPHome CLI

```bash
cd firmware
esphome run watchwinder.yaml
```

### 4. Adoptar en Home Assistant

Una vez en la misma red, el dispositivo aparecerá automáticamente en **Ajustes → Integraciones → ESPHome**.

### 5. Importar dashboard

Importar el archivo `/homeassistant/dashboard_watchwinder.yaml` en una vista nueva de Lovelace.

---

## 📁 Estructura del proyecto

```
watchwinder-esp32c3/
├── firmware/
│   ├── watchwinder.yaml          # Configuración ESPHome principal
│   ├── secrets.yaml.example      # Plantilla de credenciales
│   ├── packages/
│   │   ├── motors.yaml           # Lógica de control de motores
│   │   ├── espnow.yaml           # Configuración ESP-NOW
│   │   └── status_led.yaml       # LED WS2812
│   └── components/
│       └── watchwinder/          # Componente custom ESPHome (C++)
│           ├── __init__.py
│           ├── watchwinder.h
│           └── watchwinder.cpp
├── hardware/
│   ├── pcb/                      # Proyecto KiCad 8.x
│   │   ├── watchwinder.kicad_pro
│   │   ├── watchwinder.kicad_sch
│   │   └── watchwinder.kicad_pcb
│   ├── 3d/                       # Modelos STL para impresión 3D (ver sección 3D)
│   │   ├── caja/
│   │   │   ├── caja_base.stl         # Base de la caja electrónica
│   │   │   ├── caja_tapa.stl         # Tapa con rejillas de ventilación
│   │   │   └── panel_frontal.stl     # Panel frontal con hueco para LED
│   │   ├── mecanica/
│   │   │   ├── soporte_motor_x4.stl  # Soporte de motor (imprimir ×4)
│   │   │   ├── tambor_reloj_x4.stl   # Tambor porta-reloj (imprimir ×4)
│   │   │   ├── eje_tambor.stl        # Eje de transmisión motor→tambor
│   │   │   └── tapa_tambor.stl       # Tapa frontal del tambor
│   │   └── ensamblaje/
│   │       └── watchwinder_v1.step   # Ensamblaje completo (STEP)
│   └── bom/
│       └── bom.csv
├── homeassistant/
│   ├── dashboard_watchwinder.yaml
│   └── automations_examples.yaml
├── media/                        # Fotografías y vídeos del proyecto
│   ├── photos/
│   │   ├── 01_pcb_assembled.jpg      # PCB montada
│   │   ├── 02_caja_3d_print.jpg      # Caja impresa en 3D
│   │   ├── 03_mecanica_motores.jpg   # Detalle mecánica y motores
│   │   ├── 04_relojes_montados.jpg   # 4 relojes en los tambores
│   │   ├── 05_ha_dashboard.jpg       # Captura del dashboard HA
│   │   └── 06_dispositivo_final.jpg  # Resultado final completo
│   ├── videos/
│   │   ├── demo_funcionamiento.mp4   # Demo de giro de los 4 relojes
│   │   └── ha_control_live.mp4       # Control en tiempo real desde HA
│   └── README.md                     # Índice visual del contenido multimedia
├── docs/
│   ├── esquema_conexiones.pdf
│   └── imagenes/
├── LICENSE
└── README.md
```

---

## 🖨️ Diseño 3D

Los modelos están diseñados para impresión FDM estándar. Todos los STL están en `/hardware/3d/`.

### Parámetros de impresión recomendados

| Parámetro | Valor recomendado |
|---|---|
| Material | PLA o PETG |
| Altura de capa | 0.2 mm |
| Relleno | 25% (caja) / 40% (piezas mecánicas) |
| Perímetros | 3 |
| Soporte | Solo en `caja_tapa.stl` y `panel_frontal.stl` |
| Temperatura cama | 60°C (PLA) / 80°C (PETG) |

### Piezas y cantidades

| Archivo STL | Cant. | Descripción |
|---|---|---|
| `caja/caja_base.stl` | 1 | Base donde se aloja la PCB y los drivers |
| `caja/caja_tapa.stl` | 1 | Tapa con rejillas de ventilación pasiva |
| `caja/panel_frontal.stl` | 1 | Panel con ventana para el LED de estado |
| `mecanica/soporte_motor_x4.stl` | 4 | Fijación del motor NEMA17 al chasis |
| `mecanica/tambor_reloj_x4.stl` | 4 | Tambor giratorio donde se coloca el reloj |
| `mecanica/eje_tambor.stl` | 4 | Acopla el eje del motor al tambor |
| `mecanica/tapa_tambor.stl` | 4 | Retención frontal del reloj en el tambor |

> El archivo `ensamblaje/watchwinder_v1.step` permite abrir el conjunto completo en FreeCAD, Fusion 360 u OnShape para modificaciones.

---

## 🗺️ Roadmap

- [x] Control de 4 motores independientes
- [x] Integración ESPHome + Home Assistant
- [x] Comunicación ESP-NOW
- [x] Modos CW / CCW / bidireccional / parado
- [x] TPD configurable por reloj
- [ ] Display OLED local (SSD1306) con estado en tiempo real
- [ ] Sensor de corriente por motor (detección de bloqueo)
- [ ] App móvil de control directo vía ESP-NOW (ESP32 + BLE)
- [ ] PCB v2 con todo integrado (ESP32-C3 embebido)
- [ ] Integración con calendario HA (no winding cuando el reloj está en la muñeca)

---

## 📄 Licencia

Este proyecto está publicado bajo la licencia **MIT**. Ver [LICENSE](LICENSE) para más detalles.

---

> Proyecto desarrollado con ❤️ y ESPHome.  
> Si te resulta útil, no olvides dejar una ⭐ en GitHub.
