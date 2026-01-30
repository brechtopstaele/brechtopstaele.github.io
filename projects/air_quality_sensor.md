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
- Excellent digital noise resilience
- Tasmota natively supports multi‑sensor setups
- Waterproof stainless steel housing is reliable outdoors

## Tasmota Configuration

Once Tasmota is flashed onto the ESP32:

### I²C (BME680)

Assign the ESP32 pins used for I²C, such as:

- SDA → GPIO21
- SCL → GPIO22

Tasmota automatically detects the BME680 on the bus after a restart.
In the Tasmota web interface, the environmental readings appear under the SENSOR page and are published automatically via MQTT.
You typically get values like:

- BME680.Temperature
- BME680.Humidity
- BME680.Pressure
- BME680.Gas

The Gas value is in ohms and reflects VOC concentration indirectly — lower resistance generally indicates poorer air quality. Tasmota does not compute IAQ directly but provides raw values that Home Assistant can use for trends and alerting.

### DS18B20

Set the pin used for the 1‑Wire bus (for example GPIO25).
Tasmota identifies each DS18B20 by its unique ROM ID, ensuring stable identification even with long‑cable probes outdoors.

Both sensors are automatically included in the standard Tasmota telemetry messages.

## Home Assistant on Proxmox

Home Assistant runs inside a Proxmox virtual machine hosted on an old repurposed PC.
This setup is lightweight, efficient, and ideal for 24/7 monitoring.

### Proxmox

The Proxmox installation is really straightforward, you create a bootable USB drive with the Proxmox installer, boot the 'donor' pc from this stick and you follow the installer. Afterwards there is some configuration in Proxmox and you can use a community script to install a Home Assistant virtual machine.

### Home Assistant

Installing Home Assistant is also a hassle-free process. After adding the Tasmota integration and creating a user for Tasmota, you only need to enter the IP address of the Home Assistant VM and the user credentials and the ESP32 shows up automatically with the BME680 and DS18B20 data as sensors. 

With this data, you are able to create some nice dashboards.

## Result

Together, the ESP32 + Tasmota + Home Assistant stack results in a simple, low‑maintenance, high‑accuracy sensor network.