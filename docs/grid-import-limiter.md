# Grid import limiter (automation G) and meter calibration

New in v4.0.

## The constraint

The grid connection has a **5 kW limit enforced by the utility's smart
meter**: exceed it for about a minute and the meter disconnects the house for
~10 minutes. The inverter (11 kW) can carry far more than that from the
battery, so the limit only matters while the house is drawing from the grid —
i.e. while grid-charging at night.

Background load at night is ~0.5 kW. At the old fixed 30 A grid-charge current
the total was ~2.3 kW; at 60 A it is ~4 kW; add a kettle (~2 kW) or a
dishwasher heating cycle and it crosses 5 kW.

## What automation G does

See [`../automations/automation_g_grid_import_limiter.yaml`](../automations/automation_g_grid_import_limiter.yaml).
While grid-charging:

1. Compute the current that fits: `(budget − household load) / (V_batt / 0.9)`,
   rounded down to 5 A, 0…max ([`sensor_grid_charge_current_target`](../helpers/sensor_grid_charge_current_target.yaml)).
2. Write it to the inverter: lower immediately, raise only after 2 minutes of
   stable load (a kettle doesn't cause two writes per cycle).
3. If less than 10 A fits, or metered import stays above 4.8 kW for 20 s,
   put the house on battery (the inverter carries the spike) and resume
   when the load drops.

One amp into the battery cost ~59 W from the grid here (measured: import was
~1.78 kW above household load while charging at 30 A, battery at 53 V), i.e.
`V_batt / 0.9`.

The inverter takes ~13 s to accept a setting; load readings arrive every
~5 s. End-to-end reaction is about half a minute — inside the meter's
tolerance, with the 500 W budget margin covering the gap.

## Which sensor to trust — and why the budget is in "meter watts"

Three devices measure grid import here: the inverter's own grid-power
reading, a smart breaker on the grid input, and a two-channel clamp meter
(channel A on the grid, channel B on the inverter output).

### Instantaneous power (2-minute windows where all three reported)

| | Smart breaker | Inverter | Clamp meter ch. A |
|---|---|---|---|
| 728 windows with grid, 14 days | 2224 W | 2215 W | 2130 W (−4 %) |
| 253 windows while charging, 7 days | 2315 W | 2340 W | 2145 W (−8 %) |

The breaker and the inverter agree within ~1 %. Clamp channel A's *reported*
power reads low (it reports on change, and averages of those reports are
biased) — but its own energy counter is fine, see below. Clamp channel B
(inverter output) matches the inverter's load reading within 1 %.

The smart breaker freezes at its last value when the protection relay cuts
the grid (its radio dies with the line), so it is unsuitable for control.

### Daily grid energy (kWh)

| Day | Smart breaker | Clamp ch. A | Inverter estimate | Clamp A × 1.155 |
|---|---|---|---|---|
| 1 | 9.67 | 9.53 | 9.00 | 11.01 |
| 2 | 9.23 | 9.01 | 8.60 | 10.42 |
| 3 | 10.40 | 10.17 | 9.54 | 11.75 |
| 4 | 7.88 | 7.77 | 2.50 | 9.00 |
| 5 | 4.53 | 4.46 | 4.20 | 5.15 |
| 6 | 16.44 | 16.06 | 15.33 | 18.55 |

- The breaker's and the clamp meter's energy counters agree within 1.5–2.5 %.
- The inverter integration's daily *estimates* are 5–7 % low and lose data
  (day 4: 2.5 instead of 7.9 kWh). Don't use them for accounting.
- **The 1.155 factor** is the household's own calibration: months of comparing
  the clamp meter's channel A against the utility meter's readings.

Chaining it: utility meter ≈ clamp A × 1.155 ≈ breaker × 1.13 ≈ inverter
instantaneous power × 1.12–1.14. The package uses
`input_number.utility_meter_correction = 1.14` and expresses the budget in
utility-meter watts (`input_number.grid_import_budget = 4500`). Without the
correction, "4500 W" by the inverter would be ~5.1–5.2 kW on the meter.

**Calibrate your own factor.** Read the utility meter in the evening and the
morning; compare the night's kWh with your sensor's counter. Repeat over a
few weeks.

## Interaction with the other automations

- G only acts while grid-charging, or while its own overload flag is on.
- The overload hold (`AUTONOMOUS` + `SOLAR_AND_GRID`) is seen by A–F as "not
  grid-charging". If C or E decide to start charging during a hold, G will
  move the house back to battery if the load is still too high.
- When B, C, D or E switch grid charging off, G clears its flag.
- At 06:58 D resets everything; G does not act in daytime unless grid
  charging was started manually.
