# BLDC ESC in Simulink

A Simulink / Simscape Electrical model of a PWM-controlled brushless DC (BLDC) electronic speed controller (ESC). The controller takes an RC-style PWM throttle (1-2 ms pulse, 50 Hz) and regulates the speed of a Crouzet BLDC motor through a 3-phase MOSFET inverter. It is built so the controller maps onto STM32 firmware later.

> Status: simulation only. Current limiting is not implemented yet (see Roadmap).

## Overview

```
Throttle PWM -> Input Capture -> Rate Limiter -> Speed PI + Feedforward -> Duty
                                                        ^                    |
                                              RPM (filtered)                 v
                                   Rotor angle -> Commutation + Complementary PWM -> 6 gate signals
                                                                                         |
                                                       Gate drivers -> MOSFET bridge -> BLDC motor -> Load
```

## Model details

| Part | Details |
|---|---|
| Motor | Crouzet 80120301 BLDC from the Simscape Electrical part selector: 24 V, 4 pole pairs, 205 W, 1.526 N·m and 1400 rpm rated, 11.9 A rated |
| MOSFETs | N-channel, parameterised from the Infineon IRFB7430PBF (40 V) datasheet, including parasitic capacitances and body diode |
| Gate drivers | Ideal: a controlled voltage source per gate (12 V on, 0 V off) with input filtering. No bootstrap, under-voltage lockout or driver delays are modelled |
| DC bus | 24 V source with bus capacitor |
| Commutation | 6-step (trapezoidal), sector chosen from rotor angle, electrical offset 2π/6 |
| PWM | 10 kHz carrier, low side chopped with complementary high side, 1 µs dead time |
| Speed loop | Discrete PI at 1 kHz plus feedforward (ref / 1720 rpm), output limits ±0.5, clamping anti-windup, final duty clamp 0 to 1 |
| Throttle decode | Input-capture style pulse-width measurement, maps 1-2 ms to 260-1400 rpm, below 3 % = stop |
| Feedback | RPM through a 50 Hz first-order low-pass at 1 kHz |
| Solver | Fixed step 1e-6 s, Simscape local solver (Backward Euler) |

### Controller limits found during tuning

- Duty is limited to about 0.05-0.85 when chopping, with full-on above 0.85. With the Infineon device model, a complementary window shorter than about 10-15 µs gave wrong speeds.
- Duty below about 0.15 is unstable open loop, so the speed reference is floored at 260 rpm.
- With the datasheet motor at 24 V, no-load speed is about 1720 rpm.

## Requirements

- MATLAB R2026b
- Simulink
- Simscape
- Simscape Electrical

## How to run

1. Open `ESC.slx`.
2. Set the stop time to 1 s.
3. Run. The scopes show throttle, reference rpm, duty, speed and phase A current.
4. The load subsystem applies a torque step. Edit its Step block to try 0.5 N·m or 1.5 N·m.


## Roadmap

- [ ] Phase current limit (reduce duty above about 20 A)
- [ ] Soft-start tuning to cut the startup current spike
- [ ] Rated load test (1.5 N·m)
- [ ] Hall-sensor based commutation instead of the angle signal
- [ ] Code generation for STM32 (Embedded Coder)

## Mapping to STM32

| Simulation | Firmware |
|---|---|
| Input Capture subsystem | Timer input capture on the throttle pin |
| Speed PI at 1 kHz | Periodic timer interrupt |
| Commutation and PWM | Advanced timer (TIM1) with dead-time and break |
| Rate Limiter | Software ramp |
