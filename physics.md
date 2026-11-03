# Physics of the project

The short version of the physics behind the simulator and the flight software. It's written for understanding, not rigor.

## 1. What gets simulated

A rocket is a rigid body. Its state is:

- **Position** and **velocity** (3 numbers each)
- **Attitude**, the direction it points (a quaternion, 4 numbers)
- **Angular velocity**, how fast it's rotating (3 numbers)

The simulator takes forces and torques and works out how that state changes each step.

## 2. Coordinate frames

- **World frame:** fixed to the ground. Gravity is easy here.
- **Body frame:** attached to the rocket, with x along the nose. Thrust and sensors are easy here.

A rotation (from the attitude) converts between them. Mixing frames up is the most common bug in flight code.

## 3. Motion (where it is)

Newton's second law:

```
m * dv/dt = thrust + drag + m * g
dr/dt = v
```

- **Gravity:** about 9.81 m/s² downward.
- **Thrust:** along the body x axis, taken from a thrust curve.
- **Drag:** `F = 0.5 * rho * v^2 * Cd * A`. Air density `rho` falls with altitude, and drag grows with the square of speed.
- Mass drops as fuel burns.

## 4. Rotation (which way it points)

Euler's equation, the rotational version of F = ma:

```
I * dw/dt + w x (I * w) = torque
```

`I` is the moment of inertia. A long thin rocket is easy to spin along its length and hard to tumble.

**Why quaternions:** a rocket pointing straight up hits "gimbal lock" with Euler angles. Quaternions have no such problem. Re-normalize them each step to keep their length at 1.

## 5. Stability and control authority

- If the **center of pressure is behind the center of mass**, any tilt creates a torque that straightens the rocket out. That's why rockets and arrows have fins at the back.
- **Fins** make a force proportional to deflection and airspeed squared. At low speed they do almost nothing, so the controller needs different gains at different speeds (gain scheduling).
- **Thrust vectoring** steers by tilting the engine, and it works even at zero airspeed.

## 6. What the sensors actually measure

- **Gyro:** rotation rate. It's smooth, but it has a small constant error (bias), so integrating it drifts over time.
- **Accelerometer:** measures *specific force*, not acceleration. At rest it reads +1 g up. In free fall it reads about 0. You can't just integrate it twice to get position.
- **Barometer:** air pressure, converted to altitude. It's noisy and slow, but it doesn't drift.

The simulator adds noise, bias, delay, and rounding to the true values, because real flight software never sees the truth.

## 7. Estimating the state

Each sensor is good at something different:

| Sensor | Good at | Bad at |
|---|---|---|
| Gyro | Short-term attitude change | Drifts over time |
| Accelerometer | Long-term "which way is down" | Corrupted when the rocket accelerates |
| Barometer | Absolute altitude, no drift | Noisy and slow |

**Complementary filter** (start here): mostly trust the gyro, and slowly nudge the result toward the accelerometer's gravity direction.

```
angle = 0.98 * (angle + gyro * dt) + 0.02 * accel_angle
```

**Kalman filter** (upgrade): keeps an estimate and its uncertainty, and weights each sensor by how much it can be trusted. For altitude, the accelerometer drives the prediction and the barometer corrects it.

## 8. Control (PID)

```
error  = target - estimate
output = Kp * error + Ki * integral(error) + Kd * rate
```

- **P:** push back in proportion to the error. Too much causes oscillation.
- **I:** removes steady offsets. Too much causes overshoot.
- **D:** damping. Use the gyro rate directly, since it's already measured and low-noise.
- **Anti-windup:** stop the integral from growing when the actuator is maxed out.

**Why timing matters:** any delay in the loop (sensor, compute, actuator) eats stability margin. A missed deadline on the MCU is a real stability problem, not just a software one.

## 9. Simulating in steps

The physics are continuous equations, but the computer steps through them in small time slices (e.g. 2 ms). **RK4** is a standard, accurate way to take each step. Use a fixed step size, so results are repeatable.

## 10. Checking the simulator is right

| Test | Expected |
|---|---|
| Free fall, no drag | `h = h0 - 0.5*g*t^2` |
| Free fall, with drag | Speed levels off at terminal velocity |
| Constant torque | `w = (torque / I) * t` |
| Accelerometer at rest | Reads +1 g up |
| Accelerometer in free fall | Reads about 0 |
| Halve the step size | RK4 error drops about 16 times |

Test one block at a time: dynamics, then sensors, then estimator, then controller, then everything together.

## 11. Units

Always SI: meters, kilograms, seconds, radians (gyro in rad/s), Pascals, Newtons. Write down your frame and quaternion conventions once, and never mix them.

## Further reading

- Beard & McLain, *Small Unmanned Aircraft: Theory and Practice*
- Roger Labbe, *Kalman and Bayesian Filters in Python* (free online)
- Åström & Murray, *Feedback Systems* (free online)
- PX4 / ArduPilot source code, as examples of real flight stacks
