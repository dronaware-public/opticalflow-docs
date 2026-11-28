# Dronaware Optical Flow: PX4 Integration Guide

[Dronaware Optical Flow](https://dronaware.com/opticalflow/) is proprietary (closed-source) firmware that can be flashed on an off-the-shelf ESP32 or ESP32-S3 camera board, turning it into a MAVLink [optical flow](https://docs.px4.io/main/en/sensor/optical_flow.html) sensor for PX4.
The board computes optical flow from a downward facing camera and sends [OPTICAL_FLOW_RAD](https://mavlink.io/en/messages/common.html#OPTICAL_FLOW_RAD) messages to the flight controller over a UART, at about 89 to 132 Hz depending on the camera (see [MAVLink Messages](#mavlink-messages)).

The firmware does not measure distance.
A separate downward facing [distance sensor](https://docs.px4.io/main/en/sensor/rangefinders.html) is required for optical flow navigation.

> [!NOTE]
> Dronaware Optical Flow is sold as a per-device license (US$29), under the [Optical Flow End User License Agreement](https://dronaware.com/legal/eula).
> The firmware is flashed and activated from a web browser at [dronaware.com/opticalflow](https://dronaware.com/opticalflow/).
> This guide describes firmware version 1.1.08; update older boards before following it.

> [!WARNING]
> Activation permanently secures the ESP32 board, so afterwards it can only run Dronaware firmware. It can still be updated, re-flashed and switched to another supported camera from the browser.

## Supported Boards

The following boards are currently available for activation.
The UART column lists the default ESP32 pins that carry MAVLink to the flight controller.
They can be customised during the flash session, as far as the board's free pins allow.

| Board                                                                                         | SoC      | Camera | Default UART TX / RX (GPIO) | Rated flow limit (rad/s per axis) |
| --------------------------------------------------------------------------------------------- | -------- | ------ | --------------------------- | --------------------------------- |
| [AI-Thinker ESP32-CAM](https://randomnerdtutorials.com/esp32-cam-ai-thinker-pinout/)          | ESP32    | OV2640 | 1 / 3                       | 5.21                              |
| [AI-Thinker ESP32-CAM](https://randomnerdtutorials.com/esp32-cam-ai-thinker-pinout/)          | ESP32    | OV3660 | 1 / 3                       | 5.94                              |
| [LILYGO TTGO T-Journal](https://lilygo.cc/products/t-journal)                                 | ESP32    | OV2640 | 14 / 13                     | 5.24                              |
| [Waveshare ESP32-S3-CAM](https://docs.waveshare.com/ESP32-S3-CAM-OVxxxx#interface-definition) | ESP32-S3 | GC0308 | 43 / 44                     | 6.49                              |
| [Waveshare ESP32-S3-CAM](https://docs.waveshare.com/ESP32-S3-CAM-OVxxxx#interface-definition) | ESP32-S3 | OV3660 | 43 / 44                     | 5.86                              |

The complete list of boards, with their test status, is maintained in the _Device compatibility_ section of the [Dronaware documentation](https://dronaware.com/opticalflow/#documentation).

The firmware applies a lens calibration for each camera module, so the flow output is already in radians and [SENS_FLOW_SCALE](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#SENS_FLOW_SCALE) should normally be left at `1.0`.

### Flow Rate Limits

The rated flow limit is the highest flow rate, per image axis, that each board and camera measures reliably on firmware 1.1.08.
It was rated in the Dronaware firmware emulator, running each profile at its measured board timing, over a random ground texture of medium and high contrast (signal-to-noise ratio 5 and 8), which gave the same limit.
A speed counts as measured when at least 60 % of measurement intervals produce a message and at most 1 % of messages are wrong by more than 1 pixel or 15 %, whichever is larger.
Where the two axes differ, the lower value is listed.
On low-contrast ground (signal-to-noise ratio 3) the limit is about 40 % lower.

The camera sees translation and rotation together, so the usable ground speed at height _h_ is about _h_ × (limit − rotation rate).
For example, an OV3660 board on medium contrast ground at 1 m, with the vehicle not rotating, measures up to about 5.9 m/s.

> [!IMPORTANT]
> These limits are emulator ratings. On the boards, firmware 1.1.08 has been confirmed only at low flow rates (Testbench), and flight has been tested only within the `SENS_FLOW_MAXR=1.0` limit (see [Estimator](#estimator)).
> Do not raise `SENS_FLOW_MAXR` based on this table.

## Wiring

Connect the board to a spare flight controller UART (for example `TELEM2`):

| ESP32 board | Flight controller             |
| ----------- | ----------------------------- |
| TX          | RX                            |
| RX          | TX                            |
| GND         | GND                           |
| `V_IN`      | Regulated power supply (note) |

The firmware only transmits, so the flight controller TX to board RX line is unused (see [MAVLink Messages](#mavlink-messages)).
Both sides use 3.3 V logic, so no level shifter is needed.

> [!NOTE]
> `V_IN` is the board's power input pin, for example the 5V pin on the AI-Thinker ESP32-CAM.
> Connect it to a regulated power supply on the flight controller whose voltage and current meet the board manufacturer's specification for that pin.
> Check each board's requirements before powering it from the flight controller.

> [!TIP]
> ESP32 camera boards draw current in short bursts while capturing frames.
> A low-ESR electrolytic capacitor (for example 470 µF, 10 V or higher) across the board's `V_IN` and GND pins prevents brownout resets, particularly on the AI-Thinker ESP32-CAM.

> [!WARNING]
> On the AI-Thinker ESP32-CAM, the default MAVLink pins (1 / 3) are on UART0, which is also used by the USB-serial adapter.
> When using the default pins, disconnect USB before connecting the board to the flight controller.
> Do not power the board from both USB and the flight controller at the same time.

## Mounting

Mount the board rigidly with the camera facing straight down and an unobstructed view of the ground.

Before installing the board, confirm the direction of its flow axes with the _Dronaware Testbench_ (opened automatically at the end of activation).
The Testbench reports which board edge must face the vehicle's front, and whether any flow axis is mirrored.
If it reports a mirrored axis, or cannot determine the direction, do not fly the board: `SENS_FLOW_ROT` corrects only yaw rotation, not a mirrored axis.
Contact [Dronaware support](https://dronaware.com/opticalflow/) instead.

Mount the board with that edge facing forwards and set [SENS_FLOW_ROT](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#SENS_FLOW_ROT) to `0` (no rotation).
If the board must be mounted in a different orientation, set `SENS_FLOW_ROT` to the matching yaw rotation.

The output must follow the direction convention in [Optical Flow > Setup](https://docs.px4.io/main/en/sensor/optical_flow.html#setup) (forward movement gives + Y, right movement gives - X).

## PX4 Configuration

### MAVLink Port

Configure a dedicated [MAVLink instance](https://docs.px4.io/main/en/peripherals/mavlink_peripherals.html) on the port the board is wired to:

| Parameter                                                                                              | Value                                                       |
| ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| [MAV_1_CONFIG](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#MAV_1_CONFIG)       | The port the board is connected to (for example `TELEM2`).  |
| [MAV_1_MODE](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#MAV_1_MODE)           | `Custom` (no default streams).                              |
| [MAV_1_FORWARD](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#MAV_1_FORWARD)     | `0` (disabled).                                             |
| [MAV_1_FLOW_CTRL](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#MAV_1_FLOW_CTRL) | `0` (no hardware flow control).                             |
| [SER_TEL2_BAUD](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#SER_TEL2_BAUD)     | `115200` (use the baud parameter of the port you selected). |

`MAV_1_*` is used as an example; use whichever MAVLink instance is free on your vehicle.
Reboot after setting `MAV_1_CONFIG` so that the port's baud rate parameter appears.

### Estimator

Set the following parameters to enable optical flow fusion in [EKF2](https://docs.px4.io/main/en/sensor/optical_flow.html#ekf2):

| Parameter                                                                                                                                                                                                                                                                                       | Value                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| [EKF2_OF_CTRL](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#EKF2_OF_CTRL)                                                                                                                                                                                                | `1` (enabled).                                  |
| [EKF2_OF_GYR_SRC](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#EKF2_OF_GYR_SRC)                                                                                                                                                                                          | `1` (Internal).                                 |
| [EKF2_OF_DELAY](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#EKF2_OF_DELAY)                                                                                                                                                                                              | `20` (ms).                                      |
| [SENS_FLOW_MAXR](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#SENS_FLOW_MAXR)                                                                                                                                                                                            | `1.0`.                                          |
| [EKF2_OF_POS_X](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#EKF2_OF_POS_X), [EKF2_OF_POS_Y](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#EKF2_OF_POS_Y), [EKF2_OF_POS_Z](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#EKF2_OF_POS_Z) | Camera position relative to the vehicle centre. |

The firmware leaves the gyro fields of `OPTICAL_FLOW_RAD` unset (`NaN`), so PX4 compensates rotation with its own gyro.
With `EKF2_OF_GYR_SRC=1`, EKF2 uses its IMU at the time of each flow measurement; this is the configuration the sensor was flight tested with.
`EKF2_OF_DELAY=20` is the delay the sensor was flight tested with.
PX4's default (7 ms) suits DroneCAN flow nodes, so set `EKF2_OF_DELAY` explicitly and reboot.

`SENS_FLOW_MAXR=1.0` is the lowest value in the parameter's range.
PX4 then limits horizontal speed to about half the height above ground per second (about 0.5 m/s at 1 m), which keeps the flow rate within what has been flight tested.
This limit applies only while optical flow is the only source of horizontal position.
Flow samples with a rate above 1 rad/s are not fused, including rate caused by rotating the vehicle, so fast attitude changes briefly interrupt optical flow fusion.

Configure the downward [distance sensor](https://docs.px4.io/main/en/sensor/rangefinders.html) as described in its own documentation.

## MAVLink Messages

The firmware uses MAVLink 2 at 115200 baud, 8 data bits, no parity, 1 stop bit.

- **Outgoing:** [OPTICAL_FLOW_RAD](https://mavlink.io/en/messages/common.html#OPTICAL_FLOW_RAD) from system ID `42`, component ID `197` (`MAV_COMP_ID_VISUAL_INERTIAL_ODOMETRY`).
  - Each message states its own measurement interval in `integration_time_us`. Typical values over textured ground, measured on firmware 1.1.08:

    | Camera | Message rate | `integration_time_us` |
    | ------ | ------------ | --------------------- |
    | OV3660 | about 113 Hz | about `8800`          |
    | OV2640 | about 89 Hz  | about `11100`         |
    | GC0308 | about 132 Hz | about `7600`          |

  - A measurement interval without a reliable result sends no message, so over textureless or poorly lit ground the message rate drops below these values.
  - `distance` is always `-1` (unknown); PX4 ignores it and uses the separate distance sensor.
  - `integrated_xgyro`, `integrated_ygyro` and `integrated_zgyro` are always `NaN`; PX4 compensates rotation with its own gyro (see [Estimator](#estimator)).
- **Incoming:** none. The firmware needs no data from the flight controller.

Each `OPTICAL_FLOW_RAD` message is 56 bytes.
At 132 Hz it uses about 7.4 kB/s of the roughly 11.5 kB/s available at 115200 baud, and the firmware never sends more messages than the link can carry.

## Verification

1. In the [MAVLink Shell](https://docs.px4.io/main/en/debug/mavlink_shell.html), run `uorb top sensor_optical_flow` with the board over a textured surface, and check that the topic updates at about the rate listed for your camera in [MAVLink Messages](#mavlink-messages).
2. In the [MAVLink Shell](https://docs.px4.io/main/en/debug/mavlink_shell.html), run `listener sensor_optical_flow` and check that `quality` is above zero over a textured surface.
   Over a plain, textureless surface the firmware sends fewer or no messages, so the topic slows or stops updating.
3. With the propellers removed, run `listener estimator_optical_flow_vel` in the MAVLink Shell, hold the vehicle over a textured surface and rotate it about its roll and pitch axes.
   `flow_rate_uncompensated` should follow `gyro_rate`, and `flow_rate_compensated` should stay near zero.
   Because the message gyro fields are `NaN`, the check in [Optical Flow > Scale Factor](https://docs.px4.io/main/en/sensor/optical_flow.html#scale-factor) cannot be used.
   Do not adjust `SENS_FLOW_SCALE` based on `vehicle_optical_flow`, `vehicle_optical_flow_vel` or plots built from them: with the message gyro fields unset, their gyro values come from PX4's fallback integration, which reads below the true rotation.
4. Fly the [MC_06 - Optical Flow](https://docs.px4.io/main/en/test_cards/mc_06_optical_flow.html) test card.

## See Also

- [Optical Flow](https://docs.px4.io/main/en/sensor/optical_flow.html) (PX4 documentation)
- [Distance Sensors (Rangefinders)](https://docs.px4.io/main/en/sensor/rangefinders.html) (PX4 documentation)
- [MAVLink Peripherals](https://docs.px4.io/main/en/peripherals/mavlink_peripherals.html) (PX4 documentation)
- [Dronaware Optical Flow](https://dronaware.com/opticalflow/) (product page, setup guide and Testbench)
