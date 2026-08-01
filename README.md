IoT Healthcare Monitoring System

About

An integrated IoT healthcare monitoring system built on an ESP32 running FreeRTOS. A single Wokwi/Arduino project handles patient vitals monitoring, facility environment sensing, remote-controlled medication dosage, remote-controlled smart bed elevation, adaptive sensor sampling, and fault-tolerant offline data buffering — all as concurrent FreeRTOS tasks coordinated through a shared, mutex-protected data structure. Data is published over MQTT to two role-based Adafruit IO dashboards: a Medical Staff Dashboard (body temperature, heart rate, SpO2, motion) and a Facility Management Dashboard (room temperature, air quality, system link status).

How It Works

Sensor acquisition. One task reads body temperature (DS18B20), heart rate and SpO2 (simulated via onboard switches), and motion (PIR) on an adaptive interval — sampling every 5 seconds when a vital is abnormal, and backing off toward 60 seconds when stable. A second task reads the gas sensor for air quality and derives a simulated room temperature.

Shared state. All sensor readings are written into one struct protected by a FreeRTOS mutex, so every other task always reads a consistent snapshot.

Safety evaluation. A dedicated task scores the current vitals as normal, warning, or critical, and immediately signals a binary semaphore on a critical event so the alert reaches the dashboard without waiting for the next publish cycle.

Actuator control. A dosage-control task ramps a medication dosage value toward a dashboard-set target in small, rate-limited steps, requiring confirmation before crossing into the critical range. A bed-control task smoothly drives a servo toward a target angle, with automatic overrides forcing a safer posture when vitals are dangerous. An indicator task drives LED and buzzer channels for warning, critical, and motion states.

Display. An OLED cycles between a vitals screen, a dosage/bed screen, and a facility/link status screen, switching to an offline banner when connectivity is down.

Connectivity. A single task manages WiFi and MQTT, publishes to both Adafruit IO dashboards on a periodic cycle (or immediately on a critical alert), and ingests the three dashboard sliders that control dosage, bed angle, and sample rate. If the link drops, readings are buffered in a queue (and mirrored to SPIFFS when available) instead of being lost, and the system resyncs everything automatically once the connection returns, using exponential backoff to retry.
