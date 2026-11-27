# Dronaware Optical Flow Documentation

Integration guides for [Dronaware Optical Flow](https://dronaware.com/opticalflow/), licensed firmware that turns off-the-shelf ESP32 and ESP32-S3 camera boards into a MAVLink `OPTICAL_FLOW_RAD` optical flow sensor.

## Guides

- [PX4 Integration Guide](docs/px4.md): wiring, mounting, PX4 parameters, MAVLink output and verification.

ArduPilot is not supported in the current firmware release, because it does not read `OPTICAL_FLOW_RAD`.

## Support

- Questions about these guides, or problems integrating the sensor with your flight controller: open an [issue](../../issues) in this repository.
- Do not post licence keys, board MAC addresses or account details in a public issue. For anything about a specific device, licence or purchase, contact support through [dronaware.com/opticalflow](https://dronaware.com/opticalflow/).

Purchase, activation, flashing and the Testbench are documented on the [product page](https://dronaware.com/opticalflow/#documentation).

## About this repository

This repository holds documentation only. The firmware is proprietary and is not distributed here.
Dronaware maintains this guide and is responsible for its accuracy; it is not part of the PX4 documentation.
