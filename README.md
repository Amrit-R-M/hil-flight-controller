# hil-flight-controller

A hardware-in-the-loop (HIL) flight controller. A real microcontroller runs the flight software, and my laptop pretends to be the rocket and the sky.

**Status:** early stage. Design and physics notes are written, hardware is planned, and implementation is starting with the serial protocol and the simulator.

## What it is

- **Laptop (simulator):** calculates the rocket's physics, then sends fake sensor readings (rotation, acceleration, air pressure) to the MCU over USB serial.
- **MCU (flight computer):** estimates where the rocket is and which way it's pointing, then sends back commands like "move the fins 2 degrees."
- **Loop:** the laptop applies those commands to the fake rocket, calculates what happens next, and sends new readings. This repeats about 500 times a second.

The MCU can't tell the difference between this and a real flight. That's what makes the test meaningful.

## Why HIL

You can't test flight software by launching and hoping. Aerospace teams run the real computer and real code against a simulated flight first. Some bugs only show up on the real chip: code that's too slow, dropped serial data, number-format differences, freezes. Testing this way also lets me inject failures (a dead sensor, corrupted data) safely.

## Hardware (V1)

| Part | Why |
|---|---|
| NUCLEO-F446RE (Cortex-M4F, FPU) | The flight computer. Hardware FPU for filter math, timers for a fixed-rate loop, and a built-in debugger and USB serial. |
| USB cable | Power, flashing, debugging, and the serial link to the laptop. |
| *(optional)* USB logic analyzer, breadboard, wires | Measure real loop timing by toggling a GPIO pin each cycle. |

## Pieces

1. **Serial protocol:** framed binary messages with a CRC check.
2. **Simulator (Python):** 6-DOF rigid-body physics integrated with RK4.
3. **Sensor models:** turn true state into noisy, biased gyro, accelerometer, and barometer readings.
4. **State estimator (MCU):** complementary filter first, Kalman filter later.
5. **Flight state machine (MCU):** pad, boost, coast, apogee, descent, landed.
6. **Controller (MCU):** PID attitude control.
7. **Logging and plots:** compare true state to the MCU's estimate.

See [`physics.md`](physics.md) for the math behind each piece.

## Roadmap

**V1: HIL on a laptop (current)**
- [x] Design, README, and physics notes
- [x] Hardware plan
- [ ] Serial protocol with CRC and loopback test
- [ ] Simulator checked against free fall and constant torque
- [ ] Sensor models
- [ ] Complementary filter
- [ ] Flight state machine
- [ ] PID controller
- [ ] Closed-loop run with live plots
- [ ] Fault injection (dropped frames, stuck sensor)
- [ ] Loop timing measured with a logic analyzer

**V2: real hardware on the bench**
- [ ] Real servo with potentiometer feedback
- [ ] Real IMU and baro, compared against the sensor models
- [ ] 1-axis gimbal rig holding level

**V3: a vehicle**
- [ ] Same flight software on a small physical vehicle, with HIL kept as the regression test bench

## What I want to learn

PID control, sensor fusion (complementary filter and Kalman filter), 6-DOF dynamics and quaternions, real-time embedded loops, and how aerospace teams validate flight software.

## Repo layout (planned)

```
firmware/   MCU flight code (C/C++)
sim/        laptop simulator (Python)
scripts/    plotting and log analysis
```
