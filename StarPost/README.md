# LiRover, Curated by Alex, Anish, Om, Yash

## Overview #
Welcome to LiRover! This hardware for Hackclub's Stardance event will be centered around a hardware initiative by 4 high schoolers. 

After a failed rocket idea, we settled on an RC car with a LiDAR sensor attatched that can create a comprehensive map of its surroundings. This is done
from scratch by creating a 3D printed chassis, custom PCB, and an embedded control system. This project is controlled through a Raspberry Pi Zero 2 W and an arduino. The robot will combine LiDAR sensing, an IMU (which includes an accelerometer for linear acceleration, gyroscope for measuring angular rotation, and magnetometer for magnetic field and acting as a compass), motor drivers, and wheel odometry to estimate its position and generate a real-time 2D "minimap" of its surrounding area. To control it, we implemented a Bluetooth module for SLAM development (Simaltaneous Localization and Mapping, essentialy enabling the robot to construct a map and track its position). 

---

## Features #

- Custom CAD and 3D-printed chassis for the RC car
- Custom PCB
- Bluetooth remote control
- Real time LiDAR mapping
- IMU-based tracking
    - Accelerometer for linear acceleration
    - 3-plane gyroscope for angular rotation
    - Magnetometer for magnetic field detection

---

## Hardware

### Computing
- Raspberry Pi Zero 2 W
- Raspberry Pi Pico

### Sensors
- 2D/3D Lidar
- IMU
- Wheel Encoders (??)

### Electronics
- Motor driver
- Lithium Battery Pack
- Voltage regulation circuitry
- Bluetooth module

### Mechanical
- Fully custom 3D-Printed chassis
- Custom motor mounts

---

## Software

### Raspberry Pi Zero 2 W
- Sensor fusion
- Mapping
- Localization
- Bluetooth comms
- Dashboard server
- Robot logic / Robot comms

### Arduino / Raspberry Pi Pico (*Communicates with the Raspberry Pi Zero 2 W*)
- Sensor reading
- Motor Pulse Width Modulation (PWM)
- Encoder Feedback

---

## Project Goals

- Design and develop a complete robitics platform
- Develop an embedded control system
- Generate a live 2D map using LiDAR and motion estimation (Main goal!!!)

---

## Repository Structure

_Coming Soon!!_

---

## License

This project is licensed with the MIT License.