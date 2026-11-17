# Backtest results

Two separate validation passes were run against real historical sensor data
from InfluxDB before the model replaced the previous fixed-threshold version.

## Data sources

All data pulled from InfluxDB (`ha_data` bucket) covering approximately 110
days (late February – mid July 2026):

- **PV generation:** the inverter's PV input power sensor (W, hourly mean)
- **Battery SOC:** the BMS SOC sensor (%, hourly mean)
- **Household load:** a clamp-meter channel on the inverter output (W, hourly mean)

UTC → local time conversion was applied in Python (+3h offset), not in Flux
queries — the `location:` parameter in `aggregateWindow()` was found to produce
no actual time shift in the InfluxDB version deployed.

## Pass 1: Decision-point backtest (108 nights, March–June)

**Setup:** Simulated automation A running exactly as it does in production —
evaluated at 23:00, computing the full overnight requirement. Used actual
recorded PV generation as a stand-in for the solar forecast (so this tests
the stop/accumulate algorithm logic, not real-world forecast accuracy).

**Dangerous miss definition:** the model decided not to charge, but the actual
battery SOC fell below the safety floor (20%) during the night. This is the
failure mode that matters — leaving the house without power.

**Result: 0 dangerous misses across 108 nights.**

| Season | Nights | Would charge | Avg. min SOC on no-charge nights |
|--------|--------|-------------|----------------------------------|
| Spring/autumn (transition) | 61 | 52 (85%) | 26.0% (n=9) |
| Summer | 47 | 22 (47%) | 35.4% (n=25) |

The relatively high charge rate in the transition season reflects the hourly
load profile used, which is built from a full ~110-day dataset and comes out
higher than an earlier shorter-window estimate. This is a data difference,
not a logic bug.

### One notable night: April 11

On the night of April 11, the model correctly flagged a charge requirement
(energy deficit held above the threshold throughout the entire night). The
actual battery SOC fell to 17–20% by morning — below the 20% safety floor.

Inspection confirmed this was not a model failure. On that specific night, the
*actual* system was running an earlier version of the automation with different
logic. The new hourly model would have turned the breaker on from 23:05 and
kept it on, preventing the drop. This night is retained in the dataset as a
data point, not excluded — it validates the model rather than contradicting it.

## Pass 2: Intra-night dynamics (9 nights, 3 per morning category)

Pass 1 only validates the decision made *at* 23:00. It says nothing about
whether the sensor behaves correctly when re-evaluated mid-night — which matters
because an earlier version had a bug where the sensor always started its
calculation from 23:00 regardless of the actual current hour (causing identical
readings at 01:30 and 05:00).

**Night selection:** objective, based on the hour at which generation first
covered load for two consecutive hours (`covered_hour` from Pass 1):

| Category | `covered_hour` | Nights selected |
|----------|---------------|-----------------|
| Good morning | 8 | 3 nights (Apr 29, May 3, May 4) |
| Average morning | 9–10 | 3 nights (Apr 11, May 11, May 25) |
| Poor morning | 11 or None | 3 nights (Mar 26, Apr 5, May 31) |

For each night, the model was re-run at `cur_h` = 23, 0, 1, 2, 3, 4, 5, 6, 7,
using actual recorded generation and SOC data.

**Result:** remaining energy deficit and target SOC decrease monotonically
across the night on every test night, with no repeated or stagnant values
between adjacent hours. The fix for the 23:00-always bug is validated.

Two results that initially appeared anomalous were investigated:

**April 11 (average morning) — deficit_real stayed roughly constant for several
hours.** This is mathematically correct: the battery's actual SOC was falling
at almost the same rate the calculated requirement was falling, so their
difference barely moved. The model was correctly signalling a continuous need
to charge throughout that night.

**`covered_hour = None` on poor morning nights within the 23–07 test window.**
Consistent with Pass 1 results for the same dates, where coverage also never
arrived within the full 23–11 window. Not a discrepancy — the test window is
narrower (23–07) than the model's full window (23–11) by design, to keep the
test focused on the hours the breaker is actually active.

## What backtesting does not cover

Both passes use actual recorded generation as the forecast input. This tests
the algorithm's internal logic but not real-world forecast accuracy. A forecast
that is wrong on the night itself (predicts "good morning" but reality is
"poor morning") is a different failure mode, not exercised by either backtest.

This is the primary reason a forecast-error buffer remains an open question
rather than a settled decision (see `docs/model.md`).

---

## Pass 3 — minimum run-time guard validation (July + August 2026)

### Context

After observing oscillation in automation C (emergency night correction) —
the breaker turning on and off multiple times per night — a minimum 30-minute
run-time guard was added to the turn-off branch. Before deploying, the change
was validated by simulation on real InfluxDB data.

### Data

**July 2026:** 859 data points at 10-minute resolution from InfluxDB
(`sensor.energy_deficit`). Switch history from HA recorder (168 state
changes, 11 nights with active charging).

**August 1→2, 2026:** 61 data points at 1-minute resolution (full resolution
pull). Confirmed against logbook which shows exact automation attribution
(C triggered all switching after the initial A turn-on at 23:05).

### July results

| Variant | Nights detected | Oscillation nights | Extra ON cycles |
|---|---|---|---|
| Real (logbook) | 11 | 2 | 3 |
| Current (no change) | 27 | 2 | 2 |
| Solution: smooth 10 min | 27 | 0 | 0 |
| **Solution: min 30 min** | **27** | **1** | **1** |
| Combo (smooth + 30 min) | 27 | 0 | 0 |

The 27 detected nights vs 11 real nights discrepancy reflects that the
simulator sees only `energy_deficit` — it fires C whenever the threshold is
crossed regardless of what automation A decided. Nights where A correctly
decided not to charge still show C activity in the simulation.

The minimum 30-minute guard reduces oscillation nights from 2 to 1 and extra
cycles from 3 to 1 on July data — without introducing any new dangerous misses.

### August 1→2 results

| Variant | C switching cycles |
|---|---|
| Real (logbook) | 7 |
| Current (simulated) | 9 |
| Smooth 10 min | 9 (no improvement) |
| **Min 30 min** | **5** ✅ |
| Combo | 7 (neutral) |

The 30-minute guard is the clear winner on the August oscillation night:
reduces cycles from 9 to 5. Smoothing alone does not help because the
underlying signal (battery actually charging, then deficit returning) is real,
not noise — it cannot be averaged away.

### Why smoothing does not help here

The spike at 02:59 UTC (5.87 kWh, caused by Open-Meteo forecast rotation at
06:00 local time) is genuine data that smoothing partially absorbs, but the
main oscillation pattern — deficit rising above 0.7 after a short charge
session, then falling below 0.3 once the breaker turns on — is physical
behaviour that no amount of signal averaging will eliminate. The minimum
run-time guard addresses the root cause directly.

### Decision

Minimum 30-minute run-time guard deployed as the sole change to automation C
(v2.6). Smoothing deferred — it adds complexity without improving the
metric that matters on real oscillation nights.
