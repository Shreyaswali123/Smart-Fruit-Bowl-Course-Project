# Smart Fruit Bowl

An ESP32-based smart fruit monitoring prototype that uses the Bosch BME68x environmental sensor to collect temperature, humidity, and gas-resistance data, classify fruit condition using rule-based thresholds, publish measurements through MQTT, and log the data on a Python subscriber.

## Overview

The system is designed around the idea that changes in volatile compounds and humidity around fruit can be used as indicators of ripening. The firmware periodically measures the environment, calculates a baseline, evaluates changes in gas resistance and humidity, drives status LEDs, publishes JSON messages over MQTT, and enters deep sleep between measurement cycles.

## System Architecture

```text
                 +----------------------+
                 |   BME680 / BME68x   |
                 | Temperature          |
                 | Humidity             |
                 | Gas resistance       |
                 +----------+-----------+
                            |
                           I2C
                            |
                 +----------v-----------+
                 |     ESP32 Firmware    |
                 |-----------------------|
                 | Sensor initialization |
                 | Sampling @ 1 Hz       |
                 | Baseline calculation  |
                 | Ripeness classification|
                 | LED status output     |
                 | Deep sleep            |
                 +----+-------------+----+
                      |             |
                   Wi-Fi          GPIO
                      |             |
                   MQTT       Status LEDs
                      |
             +--------v---------+
             |   MQTT Broker    |
             | broker.emqx.io   |
             +--------+---------+
                      |
             +--------v---------+
             |   Python Logger   |
             | logger/logger.py  |
             +--------+---------+
                      |
             +--------v---------+
             | CSV data files    |
             +-------------------+
```

## Features

- BME68x sensor communication over I2C
- Temperature, humidity, and gas-resistance acquisition
- Two-minute sensor warm-up phase
- 1 Hz sampling rate
- Validation of gas measurement and heater stability
- RTC-retained sampling state across deep-sleep cycles
- Gas-resistance and humidity baselining
- Rule-based fruit-condition classification
- Four GPIO status outputs for fruit condition indication
- Wi-Fi connectivity
- MQTT telemetry publishing
- SNTP time synchronization with IST formatting
- Twelve-minute deep-sleep interval for low-power operation
- Python MQTT subscriber for CSV logging

## Repository Structure

```text
Smart-Fruit-Bowl-Course-Project/
│
├── README.md
├── .gitignore
├── CMakeLists.txt
│
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml
│   ├── bme_isr.c
│   ├── bme68x.c
│   ├── bme68x.h
│   └── bme68x_defs.h
│
└── logger/
    └── logger.py
```

## Hardware

### Core components

- ESP32-series MCU running ESP-IDF firmware
- Bosch BME680/BME68x environmental sensor
- Four status LEDs
- Wi-Fi network for MQTT connectivity

### Current GPIO configuration

| Function | GPIO |
|---|---:|
| I2C SDA | 5 |
| I2C SCL | 6 |
| Empty LED | 1 |
| Fresh LED | 2 |
| Ripening LED | 3 |
| Overripe LED | 4 |

The BME68x is configured for the lower I2C address (`0x76`).

## Sensor Configuration

The current firmware uses:

- Temperature oversampling: 2x
- Humidity oversampling: 1x
- Pressure measurement: disabled
- IIR filter: size 3
- Gas heater: 320°C for 150 ms
- Forced-mode measurements
- Ambient temperature parameter: 25°C

## Sampling and Power-Cycle Strategy

The firmware operates in repeating measurement cycles:

1. Connect to Wi-Fi.
2. Synchronize time using SNTP.
3. Start MQTT communication.
4. Initialize the BME68x sensor.
5. Warm up the sensor for 2 minutes.
6. Collect 60 valid samples at 1 Hz.
7. Store samples in RTC-retained memory.
8. Publish each valid sample through MQTT.
9. Evaluate a 120-sample decision window when available.
10. Enter deep sleep for 12 minutes.
11. Resume after wake-up while retaining selected state in RTC memory.

## Ripeness Classification Logic

The current firmware uses gas-resistance change relative to a stored baseline together with humidity rise as a confirmation signal.

### Thresholds in the current implementation

| Condition | Gas-resistance change | Humidity condition |
|---|---:|---:|
| `FRESH` | < 2% drop | Not required |
| `EARLY_RIPENING` | 2% to < 5% drop | > 1.5 rise |
| `FRESH` | 2% to < 5% drop | <= 1.5 rise |
| `RIPENING` | 5% to < 10% drop | Not required |
| `OVERRIPE` | >= 10% drop | Not required |

The thresholds are implementation parameters and should be calibrated experimentally for the target fruit, enclosure, sensor placement, and environmental conditions.

## MQTT Interface

### Broker

```text
Host: broker.emqx.io
Port: 1883
Protocol: MQTT
QoS: 1
Retain: 0
```

The broker is a public MQTT endpoint used by the project for development/testing. Do not use it for sensitive or production telemetry without appropriate security controls.

### Topics

#### Sensor data

```text
bme680/data
```

Example payload:

```json
{
  "timestamp": 123,
  "ist": "2026-09-27 21:30:00",
  "temp": 25.40,
  "hum": 61.20,
  "gas": 125430
}
```

#### Decision data

```text
bme680/decision
```

Example payload:

```json
{
  "timestamp": 240,
  "ist": "2026-09-27 23:30:00",
  "samples": 120,
  "gas_drop_pct": 6.25,
  "hum_rise": 2.10,
  "condition": "RIPENING"
}
```

## Python Data Logger

`logger/logger.py` subscribes to both MQTT topics and appends received data to CSV files.

Output files:

```text
bme680_data.csv
bme680_decision_v3.csv
```

The CSV files are intentionally ignored by Git so that generated runtime data is not automatically committed to the source repository.

### Install dependency

```bash
pip install paho-mqtt
```

### Run the logger

From the repository root:

```bash
python logger/logger.py
```

The logger prints incoming MQTT messages and stores them in the corresponding CSV file.

## ESP-IDF Setup

### 1. Install ESP-IDF

Install a compatible ESP-IDF release and configure the environment according to Espressif's official documentation.

### 2. Open the project

```bash
cd Smart-Fruit-Bowl-Course-Project
```

### 3. Select the target

For an ESP32-S3 based setup, for example:

```bash
idf.py set-target esp32s3
```

Use the target that matches your actual hardware.

### 4. Build

```bash
idf.py build
```

### 5. Flash

```bash
idf.py flash
```

### 6. Monitor serial output

```bash
idf.py monitor
```

Or flash and monitor together:

```bash
idf.py flash monitor
```

## Wi-Fi Configuration

Do not commit real Wi-Fi credentials to a public repository.

The firmware expects Wi-Fi credentials to be supplied through its configuration. Replace the placeholder values in your local working copy before building, and keep private credentials out of Git-tracked source files.

## Important Configuration Parameters

The main firmware exposes parameters for tuning the sensing and power strategy, including:

```c
#define SAMPLE_RATE_HZ          1
#define SAMPLES_PER_CYCLE       60
#define DECISION_WINDOW_SAMPLES 120
#define WARMUP_MS               (2UL * 60UL * 1000UL)
#define SLEEP_DURATION_US       (12ULL * 60ULL * 1000000ULL)
#define HEATER_TEMP_C           320
#define HEATER_DUR_MS           150
```

These values can be adjusted for experiments and calibration.

## Data Flow

```text
BME68x measurement
        |
        v
Validation
        |
        v
RTC ring buffer
        |
        +----> MQTT bme680/data ----> logger.py ----> bme680_data.csv
        |
        v
120-sample mean
        |
        v
Baseline comparison
        |
        +----> MQTT bme680/decision --> logger.py --> bme680_decision_v3.csv
        |
        v
LED status
        |
        v
Deep sleep
```

## Development Notes

- Sensor readings are only accepted when the gas measurement is valid and the heater is stable.
- The baseline is retained in RTC memory so it survives ESP32 deep-sleep resets.
- The application uses SNTP to produce human-readable IST timestamps.
- The current classification algorithm is heuristic and should be validated against controlled fruit-ripening experiments before being treated as a calibrated measurement system.
- Generated build directories, Python virtual environments, and runtime CSV files are excluded through `.gitignore`.

## Third-Party Software

The repository includes Bosch Sensortec BME68x Sensor API source files. Those files retain their original BSD-3-Clause license and copyright notices.

See:

- `main/bme68x.c`
- `main/bme68x.h`
- `main/bme68x_defs.h`

## Future Improvements

- Move Wi-Fi credentials into a secure local configuration mechanism.
- Add TLS-secured MQTT and authentication for non-development deployments.
- Calibrate ripeness thresholds using controlled datasets for specific fruit types.
- Add persistent configuration for thresholds and sampling intervals.
- Add a dashboard for live sensor and decision visualization.
- Add fault/reconnect handling and clearer offline behavior.
- Add automated tests for the classification logic and data parser.

## Project Status

Course project / development prototype.

The current implementation demonstrates end-to-end sensing, local classification, MQTT telemetry, data logging, status indication, and low-power sleep/wake operation.
