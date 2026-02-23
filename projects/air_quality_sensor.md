---
layout: default
title: ESP32 air quality sensor with Home Assistant
permalink: /projects/air-quality-sensor/
---

# Pond & Indoor Air Quality Monitor (ESP32 + BME680 + DS18B20)

## Introduction

This project uses an ESP32 flashed with Tasmota to measure indoor air quality using a BME680 and pond water temperature using a 10 m DS18B20 probe. Sensor data is forwarded over MQTT and visualized in Home Assistant, running inside a Proxmox VM on a repurposed PC.

## Hardware Setup

### BME680 — Indoor Air Quality

The BME680 sensor measures:

- Temperature
- Humidity
- Barometric pressure
- Gas resistance (basis for estimating indoor air quality, VOC buildup, ventilation effectiveness)

It communicates over I²C, making it ideal for ESP32 + Tasmota setups. The sensor is placed indoors near the ESP32 to monitor room conditions and detect changes such as:

- Poor ventilation
- Humidity increase (cooking, showering, drying clothes)
- Air stagnation
- VOC spikes from candles, cleaning sprays, paints, etc.

### DS18B20 — Pond Temperature

A fully waterproof DS18B20 probe with a 10 m cable measures pond water temperature.
Why DS18B20 works well here:

- 1‑Wire allows long cable runs
- Waterproof stainless steel housing is reliable outdoors

## ESPHome Configuration

In ESPHome, click Install to flash the ESP32.
Once it connects to WiFi, ESPHome will notify Home Assistant.

Below is an example yaml configuration:

```yaml
esphome:
  name: esp32-sensors
  friendly_name: ESP32 Sensors

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable WiFi
wifi:
  ssid: "YOUR_WIFI"
  password: "YOUR_PASSWORD"

# Logging
logger:

# API for Home Assistant
api:
  encryption:
    key: "YOUR_API_KEY"

ota:

# I2C for BME688
i2c:
  sda: 21
  scl: 22
  scan: true
  id: bus_a

# BME688 Sensor
bme680_bsec:
  address: 0x76
  i2c_id: bus_a

sensor:
  - platform: bme680_bsec
    temperature:
      name: "BME688 Temperature"
    pressure:
      name: "BME688 Pressure"
    humidity:
      name: "BME688 Humidity"
    gas_resistance:
      name: "BME688 Gas Resistance"
    iaq:
      name: "BME688 IAQ"
    co2_equivalent:
      name: "BME688 CO2 Equivalent"
    breath_voc_equivalent:
      name: "BME688 Breath VOC"

# DS18B20 on GPIO4
  - platform: ds18b20
    pin: 4
    update_interval: 10s
    address: AUTO
    name: "DS18B20 Temperature"
```

The Bosch library for using the BME688 is included automatically and allows us to use the estimated IAQ, CO2 equivalent and VOC equivalent.

## Home Assistant on Proxmox

Home Assistant runs inside a Proxmox virtual machine hosted on an old repurposed PC.
This setup is lightweight, efficient, and ideal for 24/7 monitoring.

### Proxmox

The Proxmox installation is really straightforward, you create a bootable USB drive with the Proxmox installer, boot the 'donor' pc from this stick and you follow the installer. Afterwards there is some configuration in Proxmox and you can use a community script to install a Home Assistant virtual machine.

### Home Assistant

Install ESPHome
Go to Settings → Devices & Services
You will see a popup: “New ESPHome device found”
Click Configure
Approve and finish setup

All sensors (BME688 data + DS18B20 temperature) will automatically appear as entities.

With this data, you are able to create some nice dashboards.

## Result

Together, the ESP32 + Tasmota + Home Assistant stack results in a simple, low‑maintenance, high‑accuracy sensor network.