# Backtest results

Two separate validation passes were run against real historical sensor data
from InfluxDB before the model replaced the previous fixed-threshold version.

## Data sources

All data pulled from InfluxDB (`ha_data` bucket) covering approximately 110
days (late February – mid July 2026):

- **PV generation:** `powmr_6_2kw_pv_input_power` (W, hourly mean)
- **Battery SOC:** `f1_battery` (%, hourly mean from BMS)
- **Household load:** `2w_pwm_power_b` (W, hourly mean — inverter output to loads)

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
