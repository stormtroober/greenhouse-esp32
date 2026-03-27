# Greenhouse ESP32 — IoT Greenhouse Monitoring System

Greenhouse ESP32 is an embedded IoT system running on an ESP32 microcontroller that monitors environmental conditions inside a greenhouse and allows remote control of its lighting. It connects to an MQTT broker to publish sensor readings and receive actuation commands in real time.

---

## Platform & Language

**Platform:** ESP32 (AZ-Delivery DevKit V4)  
**Language:** C++ (Arduino framework, via PlatformIO)  
**Main libraries:** `ArduinoJson`, `PubSubClient`, `ESP32Ping`, `DHT sensor library`, `Ticker`

---

## Hardware Components

| Component | Pin | Role |
|---|---|---|
| DHT11 sensor | GPIO 4 | Temperature and humidity measurement |
| Photoresistor | GPIO 34 (ADC) | Ambient light / brightness measurement |
| LED | GPIO 2 | Controllable light actuator |

---

## How It Works

On startup, the ESP32 initialises the sensors and actuator, connects to a WiFi network, pings the MQTT broker to verify reachability, and then enters the main loop. A `Ticker` interrupt fires every second, driving a synchronous step function that keeps the system ticking at a fixed rate.

### State Machine

The firmware operates a two-state machine:

| State | Condition | Behaviour |
|---|---|---|
| `S_CONNOK` | MQTT broker connected | Read sensors, publish data, handle incoming commands |
| `S_CONNERROR` | MQTT connection lost | Attempt reconnection; restart ESP if too many failures |

### Sensor Reading & Publishing

Every tick, the firmware reads all sensors and publishes a JSON `update` message to the outbound MQTT topic:

```json
{
  "thingId": "com.project.thesis:greenhouse01",
  "type": "update",
  "temperature": 23.5,
  "humidity": 60,
  "brightness": 145,
  "light": "off"
}
```

If the temperature exceeds the configured threshold (`TEMP_TRESHOLD = 24.3 °C`), an additional `event` message is published immediately to signal a high-temperature alert.

### Incoming Commands

The firmware subscribes to an inbound MQTT topic and listens for JSON commands. The only supported command is `switchLight`, which turns the LED on or off:

```json
{ "path": ".../switchLight", "value": "on" }
```

---

## MQTT Topics

| Direction | Topic |
|---|---|
| Publish (outbound) | `com.greenhouse/com.project.thesis:greenhouse01` |
| Subscribe (inbound) | `com.greenhouse.notification/com.project.thesis:greenhouse01` |

The public broker `test.mosquitto.org` (port `1883`) is used by default.

---

## Brightness Calculation

The photoresistor reading is converted from a raw ADC value to a lux approximation using a power-law formula:

```
lux = 6207.987 × resistance^(−0.522)
```

The output is capped at **260 lux** to stay within the expected indoor greenhouse range.
