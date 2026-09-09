# CC/CV Lithium-Ion Battery Charger Simulation

This is an LTspice project that simulates charging a **3000 mAh lithium-ion battery**.

The main goal was to model a simple charger that:

- charges at about 1 A while the battery is low,
- reduces the current as the battery gets close to 4.2 V,
- keeps track of the battery's state of charge,
- and shuts the charger off when charging is finished.

The project is split into four main parts: the power source, charge controller, battery model, and state-of-charge/termination logic.

## Power Source

The charger starts with a 5 V source. A small 0.10 ohm output resistance is included before the charge controller.

This is the simulated power source feeding the battery charger.

## Charge Controller

The main charging current is controlled with this LTspice behavioral formula:

```text
I = IF(V(done)>0.5, 0, MAX(0, MIN(1, 1000*(4.2-V(bat)))))
```

The idea is simpler than the formula looks.

- If `done` is on, charging current becomes 0 A.
- If the battery is well below 4.2 V, the charger supplies the full 1 A.
- As the battery gets close to 4.2 V, the current starts decreasing.
- `MIN(1, ...)` keeps the current from going above 1 A.
- `MAX(0, ...)` keeps the current from becoming negative.

So the charger starts with a constant-current stage and then tapers the current near the voltage limit.

## Battery Model

The battery voltage is modeled with:

```text
V = min(4.2, 3.6 + 0.6*V(ncharge))
```

`ncharge` represents how full the battery is.

- `ncharge = 0` gives:

```text
3.6 + 0.6(0) = 3.6 V
```

- `ncharge = 1` gives:

```text
3.6 + 0.6(1) = 4.2 V
```

This makes the battery voltage rise from about 3.6 V to 4.2 V as the battery charges.

The `min(4.2, ...)` part keeps the modeled battery voltage from going above 4.2 V.

## State of Charge

The battery capacity is 3000 mAh, or 3 Ah.

To convert that to coulombs:

```text
3 Ah × 3600 s/hour = 10,800 C
```

So the state-of-charge section uses a capacitor with a value of 10,800.

The charging current is measured using `V_SENSE`, and this current is fed into the capacitor:

```text
I = -I(V_SENSE)
```

A capacitor naturally adds up current over time, so its voltage can be used like a charge meter.

The basic idea comes from:

```text
Q = I × t
```

where:

- `Q` is charge in coulombs,
- `I` is current in amps,
- `t` is time in seconds.

Because the capacitor value is 10,800, `V(ncharge)` works as a simple battery-charge value.

```text
0 = empty
1 = full
```

## Charge Termination

The `done` section remembers when charging has finished.

It waits for both of these conditions:

```text
Battery voltage > 4.1 V
Charging current < 0.1 A
```

The 4.1 V check is only a qualifier. It prevents a low-current condition earlier in the simulation from being treated as a finished charge.

The charger actually shuts off once the battery is already in the high-voltage region and the charging current has tapered below 100 mA.

When that happens, the small `C_done` capacitor is charged so that:

```text
V(done) = 1
```

The charge controller sees `done = 1` and changes its output current to 0 A.

The capacitor keeps the `done` signal stored, so the charger stays off instead of immediately turning back on.

## Automatic Measurements

I added LTspice `.meas` commands to check the main results automatically.

They measure:

- battery voltage after charging stops,
- when the 1 A charging stage ends,
- when the charger shuts off,
- the highest battery voltage reached,
- and the final state of charge.

The simulation runs for:

```text
.tran 25000 startup
```

In the model, charging takes about **13,170 seconds**, or about **3.7 hours**, and ends at about **98.3% state of charge**.

## What I Learned

This project helped me understand how a basic CC/CV-style charging model can be built in LTspice.

I also practiced using:

- behavioral voltage and current sources,
- current sensing,
- simple state-of-charge tracking,
- charge termination logic,
- and automatic LTspice measurements.

This is a simulation model made for learning and testing the charging logic, not a complete real-world battery charger circuit.
