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

- `GPIO32` -> `1`
- `GPIO33` -> `2`
- `GPIO25` -> `3`
- `GPIO26` -> `4`
- `GPIO27` -> `5`
- `GPIO14` -> `6`
- `GPIO12` -> `7`
- `GPIO13` -> `8`
- `GPIO15` -> `9`
- `GPIO16` -> `10`

The first 8 GPIOs are the actual physical relays. The last 2 are virtual relays: one to enable/disable the schedule (`Schedule Active`) and another to manually start the schedule (`Manual Schedule Start`).

To give the buttons mnemonic names in the UI use the `WebButtonX` command like so:

```bash
WebButton1 Sprenger1
```

### Matter

Go to `Configuration > Matter > Add to Configuration` and create new endpoints for all the 10 relays (both physical and virtual). You may need to check `Matter enable` first.

- Name: Whatever you like
- Type: Relay
- Parameter: The respective relay number (so `1` for `Relay 1`, `2` for `Relay 2`, etc.)

Once done, scan the Matter QR-Code as you normally would.

### Runtimes

Set default runtimes after which each relay turns off automatically using `PulseTimeX TIME`. Note: [`PulseTime`](https://tasmota.github.io/docs/Commands/#pulsetime) works with 1/10 of a second for the values from 1 to 111. From 112 onwards you need to add 100 to the desired seconds:

```bash
PulseTime1 50
```

runs for 50/10 = 5 seconds

```bash
PulseTime2 1300
```

runs for (1300 - 100) / 60 = 20 minutes

Set your own values accordingly.

### Timers

Create a timer, either via `Configuration > Timer` or command

```bash
Timer1 {"Enable":1,"Mode":0,"Time":"19:00","Window":0,"Repeat":1,"Days":"1111111","Action":1,"Output":10}
```

Adjust the `Time` to your needs.

If you add more than one `Timer`, make sure to add them in `Rule1` accordingly

### Rules

This is where the magic happens.

Apply the rules via `Tools > Console`.

**Rule 1:**

- Store whether the schedule is enabled or disabled in `mem1` (to be correctly restored after restart)
- Enable/disable the timer according to the state of `Schedule Active`
- When the schedule gets disabled turn off everything else, too

```bash
Rule1
ON Power9#State DO Backlog Mem1 %value%; Timer1 {"Enable":%value%} ENDON
ON Power9#State=0 DO Backlog Power1 0; Power2 0; Power3 0; Power4 0; Power5 0; Power6 0; Power7 0; Power8 0; Power10 0 ENDON
Rule1 1
```

**Rule 2:**

- Store the state of `Manual Schedule Start` in `var1`
- When the schedule is started manually turn on `Relay 1` if the schedule is enabled, otherwise turn `Manual Schedule Start` back off
- When `Manual Schedule Start` is turned off also turn off all physical relays

```bash
Rule2
ON Power10#State DO Var1 %value% ENDON
ON Power10#State=1 DO IF (mem1 == 1) Power1 1 ELSE Power10 0 ENDIF ENDON
ON Power10#State=0 DO Backlog Power1 0; Power2 0; Power3 0; Power4 0; Power5 0; Power6 0; Power7 0; Power8 0 ENDON
Rule2 1
```

**Rule 3:**

- When `Relay 1` turns off and it was started by the schedule (automatically or manually) start `Relay 2` and so on
- After the last relay has finished turn `Manual Schedule Start` back off

```bash
Rule3
ON Power1#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power2 1 ENDIF ENDON
ON Power2#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power3 1 ENDIF ENDON
ON Power3#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power4 1 ENDIF ENDON
ON Power4#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power5 1 ENDIF ENDON
ON Power5#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power6 1 ENDIF ENDON
ON Power6#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power7 1 ENDIF ENDON
ON Power7#State=0 DO IF ((mem1 == 1) AND (var1 == 1)) Power8 1 ENDIF ENDON
ON Power8#State=0 DO Power10 0 ENDON
Rule3 1
```

Note: Not checking both `mem1` and `var1` would lead to strange behavior when turning off `Schedule Active` mid-run: The current relay will turn off but then all following will turn on and off quickly sequentially.

**Rule 3 - Alternative with pauses:**

- Works just like above but pauses for 5 seconds after each relay has turned off

```bash
Rule3
ON Power1#State=0 DO RuleTimer1 5 ENDON
ON Rules#Timer=1 DO IF ((mem1 == 1) AND (var1 == 1)) Power2 1 ENDIF ENDON
ON Power2#State=0 DO RuleTimer2 5 ENDON
ON Rules#Timer=2 DO IF ((mem1 == 1) AND (var1 == 1)) Power3 1 ENDIF ENDON
ON Power3#State=0 DO RuleTimer3 5 ENDON
ON Rules#Timer=3 DO IF ((mem1 == 1) AND (var1 == 1)) Power4 1 ENDIF ENDON
ON Power4#State=0 DO RuleTimer4 5 ENDON
ON Rules#Timer=4 DO IF ((mem1 == 1) AND (var1 == 1)) Power5 1 ENDIF ENDON
ON Power5#State=0 DO RuleTimer5 5 ENDON
ON Rules#Timer=5 DO IF ((mem1 == 1) AND (var1 == 1)) Power6 1 ENDIF ENDON
ON Power7#State=0 DO RuleTimer6 5 ENDON
ON Rules#Timer=6 DO IF ((mem1 == 1) AND (var1 == 1)) Power7 1 ENDIF ENDON
ON Power8#State=0 DO RuleTimer7 5 ENDON
ON Rules#Timer=7 DO IF ((mem1 == 1) AND (var1 == 1)) Power8 1 ENDIF ENDON
ON Power8#State=0 DO Power10 0 ENDON
Rule3 1
```

**Rule 3 - Alternative with groups:**

- When `Relay 1` turns on also turn on `Relay 2`
- When `Relay 2` turns off turn on `Relay 3` and `Relay 4` and so on...
- Adjust the grouping to your needs
- The example below assumes that the `PulseTime` for each relay in one group is the same or at least that the relay checked in `PowerX#State=0` is the one with the longer runtime. Otherwise the next group starts before the former one completely finished

```bash
Rule3
ON Power1#State=1 DO IF ((mem1 == 1) AND (var1 == 1)) Power2 1 ENDIF ENDON
ON Power2#State=0 DO RuleTimer1 5 ENDON
ON Rules#Timer=1 DO IF ((mem1 == 1) AND (var1 == 1)) Power3 1; Power4 1 ENDIF ENDON
ON Power4#State=0 DO RuleTimer2 5 ENDON
ON Rules#Timer=2 DO IF ((mem1 == 1) AND (var1 == 1)) Power5 1; Power6 1 ENDIF ENDON
ON Power6#State=0 DO RuleTimer3 5 ENDON
ON Rules#Timer=3 DO IF ((mem1 == 1) AND (var1 == 1)) Power7 1; Power8 1 ENDIF ENDON
ON Power8#State=0 DO Power10 0 ENDON
Rule3 1
```

## Troubleshooting

### Connection issues

Using the default settings and despite having 100% signal strength, the ESP would disappear from my network only minutes after being powered up - after being reachable initially just fine.

The solution for me was to switch from [Dynamic Sleep to Normal Sleep](https://tasmota.github.io/docs/Dynamic-Sleep/):

```bash
SetOption60 1
```

Also I set the [Sleep](https://tasmota.github.io/docs/Energy-Saving/#device-power-consumption-and-measurement) value to 50, however this should be the default anyway:

```bash
Sleep 50
```
