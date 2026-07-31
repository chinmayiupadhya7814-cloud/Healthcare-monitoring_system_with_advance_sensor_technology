# Smart Healthcare Monitoring System using ESP32, FreeRTOS and MQTT

## Overview

The Smart Healthcare Monitoring System is a system that uses the internet to monitor health and the environment in real time. This project uses the ESP32 microcontroller FreeRTOS for doing tasks at the same time MQTT protocol for talking to the cloud and Adafruit IO for monitoring from far away.

The system always collects data from sensors processes it using FreeRTOS tasks at the same time shows live information on a small OLED display and uploads the data to the cloud so people can see it from anywhere.

---

## Project Objectives

- Create a system that monitors healthcare in time using ESP32.

- Use FreeRTOS to do many tasks at the same time.

- Send sensor data to Adafruit IO using MQTT.

- Show real-time information on an OLED screen.

- Make alerts when sensor values are not safe.

- Show that this system can be used for healthcare applications.

---

## Features

- Monitor body temperature in real time

- Monitor heart rate

- Monitor oxygen levels in the blood

- Detect motion using a sensor

- Monitor air quality using a special sensor

- Show live system status on a small OLED display

- Control a smart bed using a servo motor

- Make alerts using LEDs and a buzzer

- Use FreeRTOS to do many tasks at the same time

- Monitor from the cloud using MQTT and Adafruit IO

- Show data in a way that is easy to understand

---

## Hardware Components

- ESP32 Development Board

- Temperature Sensor

- Motion Sensor

- Gas Sensor

- Servo Motor

- Small OLED Display

- LEDs

- Buzzers

- Breadboard and Jumper Wires

---

## Software and Tools

- Arduino IDE

- Wokwi Simulator

- FreeRTOS

- Adafruit IO

- MQTT Protocol

- GitHub

---

## FreeRTOS Task Structure

The Smart Healthcare Monitoring System is divided into tasks:

- Temperature Monitoring Task

- Heart Rate Monitoring Task

- Oxygen Level Monitoring Task

- Motion Detection Task

- Air Quality Monitoring Task

- OLED Display Task

- MQTT Communication Task

- Alert Handling Task

Each Smart Healthcare Monitoring System task does its job independently to make sure the system works well in real time.

---

## Cloud Dashboard

The Smart Healthcare Monitoring System uses Adafruit IO dashboards to show sensor readings in time.

### Medical Dashboard

- Body Temperature

- Heart Rate

- Oxygen Levels in the Blood

- Patient Status

### Facility Dashboard

- Air Quality Index

- Motion Detection

- System Status

---

## Project Structure

```

Smart-Healthcare-Monitoring-System

│

├── Smart_Healthcare.ino

├── README.md

├── Report.pdf

├── Circuit_Diagram.png

├── Architecture_Diagram.png

├── Workflow_Diagram.png

├── Dashboard_Screenshots

├── Wokwi_Link.txt

└── Dashboard_Links.txt

```

---

## Applications

- Smart Hospitals

- Remote Patient Monitoring

- Healthcare IoT Research

- Embedded Systems Learning

- Smart Medical Laboratories

---

## Future Enhancements

- Monitor heart activity and blood pressure

- Store data when the system is not connected to the internet

- Make a mobile application

- Manage medication

- Use advanced cloud analytics

- Use artificial intelligence to predict health

---

## Author

**Chinmayi Upadhya**

Bachelor of Engineering, in Electronics and Telecommunication

---

## License

The Smart Healthcare Monitoring System project was made for an IoT internship to learn and teach.
