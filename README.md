# Tasmota Irrigation Controller

The goal of this project is to build an irrigation controller for 8 24VAC magnetic valves.

## Main concerns

- Support schedules
- Is able to run those schedules autonomously, without the need of an external controller
- Supports at least 8 magnetic valves
- Is connected via Matter for independence, simplicity and future proofing

## Tools / Hardware used

- ESP32-C6 (for Matter support)
- Tasmota (for ease of use)
- 8-channel relay
- 230VAC to 24VAC transformer (for the valves)
- 230VAC to 5VDC transformer (for the ESP32)

## Wiring

tbd

## Configuring Tasmota

### Flashing

I used the [Tasmota Web Installer](https://tasmota.github.io/install/) for flashing

### GPIOs

Go to `Configuration > Module` and set:

- `GPIOXX` -> `1`
- `GPIOXX` -> `2`
- `GPIOXX` -> `3`
- `GPIOXX` -> `4`
- `GPIOXX` -> `5`
- `GPIOXX` -> `6`
- `GPIOXX` -> `7`
- `GPIOXX` -> `8`
- `GPIOXX` -> `9`
- `GPIOXX` -> `10`

The first 8 GPIOs are the actual physical relays. The last 2 are virtual relays: one to enable/disable the schedule (`Schedule Active`) and another to manually start the schedule (`Manual Schedule Start`).

### Matter

Go to `Configuration > Matter > Add to Configuration` and create new endpoints for all the 10 relays (both physical and virtual). You may need to check `Matter enable` first.

- Name: Whatever you like
- Type: Relay
- Parameter: The respective relay number (so `1` for `Relay 1`, `2` for `Relay 2`, etc.)

Once done, scan the Matter QR-Code as you normally would.

### Rules

This is where the magic happens.

Apply the rules via `Tools > Console`.

**Rule 1:**

- Store whether the schedule is enabled or disabled in `var1`
- When the schedule gets disabled turn off everything else, too

```bash
Rule1
ON Power9#State DO Var1 %value% ENDON
ON Power9#State=0 DO Backlog Power1 0; Power2 0; Power3 0; Power4 0; Power5 0; Power6 0; Power7 0; Power8 0; Power10 0 ENDON
```

**Rule 2:**

- Store the state of `Manual Schedule Start` in `var2`
- When the schedule is started manually turn on `Relay 1` if the schedule is enabled, otherwise turn `Manual Schedule Start` back off
- When `Manual Schedule Start` is turned off also turn off all physical relays

```bash
Rule2
ON Power10#State DO Var2 %value% ENDON
ON Power10#State=1 DO IF (var1 == 1) Power1 1 ELSE Power10 0 ENDIF ENDON
ON Power10#State=0 DO Backlog Power1 0; Power2 0; Power3 0; Power4 0; Power5 0; Power6 0; Power7 0 ENDON
```

**Rule 3:**

- When `Relay 1` turns off and it was started by the schedule (automatically or manually) start `Relay 2` and so on
- After the last relay has finished turn `Manual Schedule Start` back off

```bash
Rule3
ON Power1#State=0 DO IF (var2 == 1) Power2 1 ENDIF ENDON
ON Power2#State=0 DO IF (var2 == 1) Power3 1 ENDIF ENDON
ON Power3#State=0 DO IF (var2 == 1) Power4 1 ENDIF ENDON
ON Power4#State=0 DO IF (var2 == 1) Power5 1 ENDIF ENDON
ON Power5#State=0 DO IF (var2 == 1) Power6 1 ENDIF ENDON
ON Power6#State=0 DO IF (var2 == 1) Power7 1 ENDIF ENDON
ON Power7#State=0 DO IF (var2 == 1) Power8 1 ENDIF ENDON
ON Power8#State=0 DO Power10 0 ENDON
```

### Timers

tbd
