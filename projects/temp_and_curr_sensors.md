---
layout: default
title: ESP32 Temperature & Current Sensors with Grafana container
permalink: /projects/temp-and-curr-sensors/
---

# Temperature and Current Sensors Project

## Introduction

This project uses M5Stack Atom Lite ESP32 modules running Tasmota to collect accurate temperature and current measurements. Data is published over MQTT to a Mosquitto broker and placed in an InfluxDB database for visualization in Grafana.

## Hardware Setup

### Temperature Sensors

**DS18B20**

For the temperature sensors, we used DS18B20 sensors that came with a board with the required resistor for the data line. These sensors are connected using the 1-Wire protocol to a GPIO pin on the ESP32. With the M5Stack Atom Lite, we can plug this terminal in directly as shown below. In our case, we used three sensors in a single terminal block, so we had to solder the wires together first with a little bit of solder to make it easier to get them into the terminal block. This is the only hardware setup needed.

The setup is shown in the image below:

![alt text](/static/images/temp_sens.png)

A closeup of the sensor:

![alt text](/static/images/temp_sens_closeup.png)

### Current Sensors

**ACS712 with ADS1115**

For the current sensors, we used ACS712 sensors with an ADS1115 ADC. The ACS712 sensors output an analog voltage proportional to the current flowing through them. The ADS1115 ADC is used to convert this analog voltage to a digital value that can be read by the ESP32.

In this setup, we soldered the connectors to the ADS1115 board and used a breadboard to connect everything together. The ACS712 sensors are connected to the ADS1115, which is then connected to the ESP32 via I2C. We built three setups, one with two 5A sensors, one with two 20A sensors, and one with two 30A sensors.

Below is a picture of the setup with the ADS1115 and two ACS712 sensors connected to the M5Stack Atom Lite. We connected the A3 input pin of the ADS1115 to the Vcc rail to correctly calculate the current later on.

The setup is shown below:

![alt text](/static/images/curr_sens.png)

## Mosquitto, Grafana and InfluxDB Setup with Docker

To set up the Mosquitto broker, InfluxDB database, and Grafana visualization tool, we used Docker Compose. This allows us to easily manage and deploy the different services needed for our project. Below is the Docker Compose file we used to set up everything.

### compose.yaml

```yaml
services:
  influxdb:
    image: influxdb:1.11
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb:/var/lib/influxdb
    environment:
      INFLUXDB_DB: ${INFLUXDB_DB:-mydb}
      INFLUXDB_USER: ${INFLUXDB_USER:-myuser}
      INFLUXDB_USER_PASSWORD: ${INFLUXDB_USER_PASSWORD:-mypassword}

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/grafana.ini:/etc/grafana/grafana.ini
      - grafana:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_USER: ${GF_SECURITY_ADMIN_USER:-admin}
      GF_SECURITY_ADMIN_PASSWORD: ${GF_SECURITY_ADMIN_PASSWORD:-admin}
    depends_on:
      - influxdb

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - ./nodered:/data
    environment:
      - NODE_RED_ENABLE_SAFE_MODE=true
      - NODE_RED_ENABLE_PROJECTS=true
    depends_on:
      - influxdb
      - grafana

  mqtt:
    image: eclipse-mosquitto:latest
    container_name: mqtt
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "8884:8884"
      - "9001:9001"
    volumes:
      - ./mqtt/config:/mosquitto/config
      - ./mqtt/data:/mosquitto/data
      - ./mqtt/log:/mosquitto/log
      - ./mqtt/certs:/mosquitto/certs:ro

volumes:
  influxdb:
  grafana:
  nodered:
  mqtt:
```

### SMTP relay

You can also add an microsoft-graph-smtp-relay Docker container to connect Grafana reporting to an enterprise Microsoft Graph API for emailing.

Add the following to compose.yaml under the grafana environment variables:

```yaml
      GF_SMTP_ENABLED: true
      GF_SMTP_HOST: smtp_relay:25
      GF_SMTP_FROM_ADDRESS: "example@example.com"
      GF_SMTP_SKIP_VERIFY: true
```

And add the following to the bottom of compose.yaml, but before the volumes declaration:
```yaml
  smtp_relay:
    image: ggpwnkthx/microsoft-graph-smtp-relay
    container_name: logging_smtp_relay
    environment:
      AUTHORITY: "https://login.microsoftonline.com/<tentant_id>"
      CLIENT_ID: "<client_id>"
      CLIENT_SECRET: "<client_secret>"
    restart: unless-stopped
    ports:
    - 2525:25
```

In the Microsoft Graph API, the permissions Mail.Send, Mail.ReadWrite and User.Read are needed.

## Tasmota Setup

For the ESP32 firmware, we used Tasmota, an open-source firmware for ESP8266/ESP32 devices. Tasmota provides a web interface for easy configuration and supports MQTT out of the box.

### Temperature Setup

The temperature sensors require minimal configuration in Tasmota. Here's the step-by-step setup:

**Web Configuration:**
- **Logging** → Set Telemetry period to 10 seconds (controls how often data is sent)
- **Other** → Device name: TempX (where X is 1, 2, or 3 for each sensor)
- **Module** → Assign GPIO25 to DS18X20 (this is the 1-Wire protocol pin for temperature sensors)
- **MQTT** → Configure:
  - Host: IP address of the Mosquitto broker
  - Topic: TempSens/TempX (where X matches the device number)

The temperature readings will be automatically published to the MQTT broker at the configured interval.

### Current Setup

The current sensors require more configuration due to the need to read multiple ADC channels and aggregate the data into a single JSON payload.

**Web Configuration:**
- **Logging** → Set Telemetry period to 10 seconds
- **Other** → Device name: Current XA (where X is 5, 20, or 30 for the sensor range in amps)
- **Module** → Configure I2C pins:
  - GPIO21: I2C SDA (data line for ADS1115 communication)
  - GPIO25: I2C SCL (clock line for ADS1115 communication)
- **MQTT** → Configure:
  - Host: IP address of the Mosquitto broker
  - Topic: CurrSens/XA (where X matches the sensor range)

**Console Rules:**

In the Tasmota console, you need to create rules that capture the raw ADC readings from all four channels and combine them into a single JSON message. The A3 channel contains the Vcc reference voltage, which will be used later for current calculation.

For each current sensor range, add the corresponding rule:

```
Rule1 ON ADS1115#A0 DO Var1 %value% ENDON ON ADS1115#A1 DO Var2 %value% ENDON ON ADS1115#A2 DO Var3 %value% ENDON ON ADS1115#A3 DO Publish tele/CurrSens/5A {"A0": %Var1%, "A1": %Var2%, "A2": %Var3%, "A3": %value%} ENDON
```

For the 20A sensor:

```
Rule1 ON ADS1115#A0 DO Var1 %value% ENDON ON ADS1115#A1 DO Var2 %value% ENDON ON ADS1115#A2 DO Var3 %value% ENDON ON ADS1115#A3 DO Publish tele/CurrSens/20A {"A0": %Var1%, "A1": %Var2%, "A2": %Var3%, "A3": %value%} ENDON
```

For the 30A sensor:

```
Rule1 ON ADS1115#A0 DO Var1 %value% ENDON ON ADS1115#A1 DO Var2 %value% ENDON ON ADS1115#A2 DO Var3 %value% ENDON ON ADS1115#A3 DO Publish tele/CurrSens/30A {"A0": %Var1%, "A1": %Var2%, "A2": %Var3%, "A3": %value%} ENDON
```

Then enable the rules by executing:

```
Rule1 1
```

These rules work by:
1. Capturing each ADC channel value from the ADS1115 into variables (A0, A1, A2)
2. When the A3 channel is read (the Vcc reference), publishing all values as a JSON object
3. The topic destination identifies which sensor range the data came from

## Node-RED Setup

Node-RED acts as the data processing layer between the MQTT broker and InfluxDB. It subscribes to the MQTT topics, transforms the raw ADC readings into meaningful current values, and stores them in the database.

### Current Sensor Data Processing

For the current sensors, Node-RED performs the following operations:

**MQTT Subscription:**
- Subscribe to the MQTT topics: `tele/CurrSens/5A`, `tele/CurrSens/20A`, and `tele/CurrSens/30A`
- Each message contains raw ADC readings from the four ADS1115 channels in JSON format

**Data Transformation:**
- Extract the individual channel values (A0, A1, A2) which represent the analog voltage readings from the ACS712 sensors
- Extract the A3 value, which is the Vcc reference voltage reading
- Convert the raw ADC values to actual current readings using the formula: `Current = ((Channel Value - (A3 Value / 2)) / A3 Value) * (Sensitivity * 2)`
- The A3 value is critical for accurate current calculation because it normalizes the readings against the reference voltage

**InfluxDB Storage:**
- Create measurement points with the calculated current values
- Tag the data with the sensor range (5A, 20A, or 30A) for easy filtering and visualization
- Write the data points to the InfluxDB database with appropriate timestamps

This approach ensures that even if the power supply voltage fluctuates, the current readings remain accurate because they are normalized against the actual Vcc reference voltage captured in channel A3.

### Temperature Sensor Data Processing

For the temperature sensors, Node-RED performs simpler operations:

**MQTT Subscription:**
- Subscribe to the MQTT topics: `tele/TempSens/Temp1`, `tele/TempSens/Temp2`, and `tele/TempSens/Temp3`
- Each message contains the temperature reading in degrees Celsius

**InfluxDB Storage:**
- Extract the temperature value from the MQTT payload
- Create measurement points tagged with the sensor identifier
- Write the data points to InfluxDB for visualization in Grafana

The temperature data requires minimal processing since the DS18B20 sensors provide direct temperature readings that don't need conversion or normalization.


## Grafana dashboard

After this, you can create Grafana dashboards with the available data. An example is shown below.

![alt text](/static/images/grafana_dashboard.png)

---
