# CC/CV Lithium-Ion Battery Charger and Battery Model

This LTspice project simulates charging a 3000 mAh lithium-ion cell with a simplified CC/CV charging model.

The goal was to model the main parts of the charging process:

- supply about 1 A while the battery is still low,
- reduce charging current as the battery approaches 4.2 V,
- keep track of state of charge,
- stop charging when the battery is nearly full,
- and verify the result with LTspice measurements.

This is a behavioral simulation. It focuses on how the charger and battery should act, rather than modeling every transistor, regulator, or chemical effect inside a real charger and cell.

<img width="1538" height="964" alt="image" src="https://github.com/user-attachments/assets/796d1283-c1e4-465b-a7de-e1c9a189cf32" />

## Power Source

The charger is powered from a 5 V source.

A 0.10 ohm resistance is included on the powerbank/charger side before the charge controller.

The charge controller then decides how much current should be sent into the battery.

## Charge Controller

The charging current is controlled by this LTspice behavioral expression:

```text
I = IF(V(done)>0.5, 0, MAX(0, MIN(1, 1000*(4.2-V(bat)))))
```

The important parts are:

```text
MIN(1, ...)
```

This keeps the charging current from going above 1 A.

```text
4.2 - V(bat)
```

This checks how far the battery terminal voltage is from the 4.2 V limit.

When the battery is well below 4.2 V, the result is large enough that the current stays at the 1 A limit.

As the battery gets close to 4.2 V, the difference becomes smaller, so the charging current starts to decrease.

```text
MAX(0, ...)
```

This keeps the current from becoming negative.

```text
IF(V(done)>0.5, 0, ...)
```

Once the charge-complete signal goes high, the charging current is forced to 0 A.

This gives the model two main charging stages:

```text
Constant-current stage:
Current is held near 1 A.

Voltage-limited / taper stage:
Current decreases as V(bat) approaches 4.2 V.
```

## Battery Model

The battery's internal voltage is modeled with:

```text
V = min(4.2, 3.6 + 0.6*V(ncharge))
```

`V(ncharge)` is used as a simple value for how full the battery is.

If:

```text
V(ncharge) = 0
```

then:

```text
V = 3.6 + 0.6(0)
V = 3.6 V
```

If:

```text
V(ncharge) = 1
```

then:

```text
V = 3.6 + 0.6(1)
V = 4.2 V
```

So the modeled battery voltage rises from about 3.6 V when empty to about 4.2 V when full.

The `min(4.2, ...)` part prevents the calculated internal battery voltage from going above 4.2 V.

The battery model also includes a 0.10 ohm series resistance.

A simple way to see its effect while charging is:

```text
Vterminal ≈ Vbattery + I × R
```

At 1 A:

```text
I × R = 1 A × 0.10 ohm
      = 0.10 V
```

This means the measured battery terminal voltage is slightly higher while charging current is flowing.

When the charger turns off, the current drops to zero, so the voltage across this resistance disappears. This causes the small drop in `V(bat)` after charging finishes.

## State of Charge

The simulated battery capacity is 3000 mAh.

First, convert that to amp-hours:

```text
3000 mAh = 3 Ah
```

Then convert amp-hours to coulombs:

```text
3 Ah × 3600 seconds/hour = 10,800 coulombs
```

That is why the state-of-charge capacitor has a value of:

```text
C1 = 10800
```

The simulation measures the battery charging current with `V_SENSE`.

The behavioral current source uses:

```text
I = -I(V_SENSE)
```

This feeds the measured charging current into the state-of-charge capacitor.

For a capacitor:

```text
Q = C × V
```

Since:

```text
C = 10800
```

putting 10,800 coulombs into the capacitor gives:

```text
V(ncharge) = 1
```

That makes the capacitor voltage work as a simple normalized state-of-charge value:

```text
V(ncharge) = 0.0  -> 0% charged
V(ncharge) = 0.5  -> 50% charged
V(ncharge) = 1.0  -> 100% charged
```

This is a simple form of Coulomb counting. The simulation keeps adding up the charging current over time to estimate how much charge has entered the battery.

## Charge Termination

The `done` section decides when charging is finished and keeps that state stored.

The model waits for both of these conditions:

```text
Battery terminal voltage > 4.1 V
Charging current < 0.1 A
```

The 4.1 V value is not the charger voltage target.

The charger still uses 4.2 V as its voltage limit.

The 4.1 V condition is only used as a qualifier so that a low-current condition earlier in the simulation does not accidentally end charging.

The current dropping below 0.1 A means the charger is already near the end of the taper stage.

Once both conditions are true, the `done` capacitor is charged so that:

```text
V(done) = 1
```

The charge controller then sees the `done` signal and changes the charging current to:

```text
0 A
```

The capacitor keeps the signal stored, so the charger stays off instead of turning back on immediately.

## Simulation Results

The transient simulation shows the battery voltage, state of charge, and charge-complete signal over time.

<img width="1916" height="941" alt="image" src="https://github.com/user-attachments/assets/2209d229-df20-46f8-903b-0d2bfc751887" />

The plotted signals are:

```text
V(bat)     = battery terminal voltage
V(ncharge) = normalized state of charge
V(done)    = charge-complete signal
```

At the start, the battery is near its low state-of-charge voltage and `V(ncharge)` begins at 0.

During the first part of charging, the battery is charged at about 1 A.

As the battery terminal voltage approaches 4.2 V, the charging current begins to taper.

Charging finishes at about:

```text
13,170 seconds
```

which is about:

```text
3.7 hours
```

The final state-of-charge value is about:

```text
V(ncharge) = 0.983
```

or about:

```text
98.3%
```

At that point, `V(done)` goes high and the charge controller shuts off.

The battery terminal voltage then drops slightly because charging current is no longer flowing through the modeled series resistance.

## Automatic Measurements

I added LTspice `.meas` commands so the main results can be checked automatically instead of reading everything from the graph by hand.

The measurements check:

- battery voltage after charging stops,
- when the 1 A charging stage begins to taper,
- when the charger shuts off,
- the highest battery voltage reached,
- and the final state of charge.

The simulation uses:

```text
.tran 25000 startup
```

This gives enough time for the full charging process and the final resting behavior to be seen.

## Model Limitations

This project is meant to show the main behavior of a CC/CV charging system.

It does not model every part of a real lithium-ion battery or charger.

For example, it does not include:

- detailed battery chemistry,
- temperature effects,
- battery aging,
- changing internal resistance,
- a transistor-level charger circuit,
- or a real feedback-control IC.

The battery voltage is also represented with a simple relationship between voltage and state of charge.

The goal was to understand and simulate the main charging behavior rather than create a complete production battery model.
