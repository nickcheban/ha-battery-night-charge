# Backtest: "until stable generation" vs "until next 23:00" (v4.0)

Why the target-SOC model was changed for winter, and how the new rule behaves
over a full spring–autumn season. Data: InfluxDB, hourly means of household
load and PV generation, March 2 – November 15, 2026 (old 6.2 kW single-array
inverter until July 31, new 11 kW dual-array inverter from August 2). Local
time with DST handled. 249 complete nights (incomplete nights and DST-change
days dropped).

## Part 1 — what November actually needed (26.10 – 16.11, 22 days)

For each day: how much energy the battery must hold at 07:00 (above the
floor) to reach 23:00 without buying daytime-tariff energy, versus what the
v3.1 model sees after 07:00.

- **"Needed"** = the maximum of the running sum (load − PV) from 07:00 to
  23:00.
- **"v3.1 sees"** = sum of hourly deficits until two consecutive covered hours
  or 11:00 (the old stop rule).

| Day type (PV that day) | Days | Needed, kWh | v3.1 sees, kWh |
|---|---|---|---|
| Sunny (> 12 kWh) | 7 | 0.4–1.3 | 0.4–1.1 |
| Medium (7–12 kWh) | 6 | 1.7–4.5 | 0.5–2.0 |
| Cloudy (< 6 kWh) | 9 | 5.8–10.7 | 1.4–3.1 |

Average need 4.8 kWh; the old model saw 1.45. Even with the 1.5 kWh reserve,
it under-provisioned on 14 of 22 days, by ~2.2 kWh/day — bought back in the
evening at the daytime tariff.

The shortfall is in the **evening (16:00–23:00)**, not the late morning.
Moving the old window's end from 11:00 to later while keeping "two covered
hours" would only change 4 of 22 days (those where cover first arrived at
12:00 or never). Simply summing all deficits until 23:00 overshoots
(average 7.6 kWh — no credit for midday surplus). The right quantity is the
**peak of the running balance**.

Household load stayed at ~16.8 kWh/day (13.8–20.5), 12.7 kWh of it between
07:00 and 23:00 — the static profile (17.4 kWh/day) is fine. PV ranged
1.6–18.6 kWh/day, average 8.9.

A fixed "always charge to 95 %" is also wrong: on 7 of those 22 days it would
have filled the battery with grid energy and left no room for free solar.

## Part 2 — 249 nights, old rule vs new rule

For every night, both rules evaluated at 23:00 on actual load and PV
(i.e. a perfect forecast). Values: kWh needed above the floor, no reserve.

| Month | Nights | Load kWh/day | PV kWh/day | Old avg | New avg | Same (< 0.3) | Diff 0.3–3 | Diff ≥ 3 | Max diff |
|---|---|---|---|---|---|---|---|---|---|
| March | 29 | 15.9 | 13.6 | 4.97 | 5.89 | 21 | 5 | 3 | 7.0 |
| April | 26 | 17.1 | 16.3 | 5.23 | 5.86 | 20 | 4 | 2 | 6.0 |
| May | 30 | 16.3 | 18.1 | 4.26 | 4.24 | 30 | 0 | 0 | 0 |
| June | 30 | 18.6 | 18.9 | 3.95 | 4.18 | 28 | 1 | 1 | 6.6 |
| July | 30 | 19.4 | 19.4 | 4.11 | 4.18 | 27 | 3 | 0 | 1.1 |
| August | 30 | 22.8 | 25.1 | 4.33 | 4.32 | 29 | 1 | 0 | 0.3 |
| September | 30 | 16.1 | 19.1 | 4.06 | 4.06 | 30 | 0 | 0 | 0 |
| October | 30 | 15.7 | 12.0 | 4.93 | 6.84 | 18 | 3 | 9 | 7.6 |
| November (1–15) | 14 | 17.1 | 10.3 | 5.29 | 8.00 | 5 | 2 | 7 | 8.1 |

- **May–September: identical on 144 of 150 nights.** The one large summer
  difference (June, 6.6 kWh) is a cloudy day where the extra is justified.
- **The new rule never asks for less than the old one.** It only adds where
  the evening isn't covered.
- **Shoulder months and winter diverge**: a quarter of nights in March–April,
  12 of 30 in October, 9 of 14 in the first half of November.
- **Capacity cap.** Nights where the new rule needs more than fits under 95 %
  of a 13.3 kWh setting (8.4 kWh above floor + reserve): 27 (October 10,
  November 6, March–April 9, June 1, August 1). Old rule: 1. This is the
  main argument for the larger battery bank (see README → Hardware context).
- **Calendar switching would be wrong.** "Winter" days occur in March and
  June, "summer" days in November. The rule needs no season switch.

Caveats: actual-as-forecast (forecast error excluded); PV on sunny days is
under-recorded (the inverter curtails when the battery is full), which only
makes summer agreement more likely; before August there was one PV array and a
smaller inverter, so spring 2027 will diverge less.

## Part 3 — forecast accuracy (value at 23:00 for the next day, 26.10–16.11)

Actual ÷ forecast: Solcast 0.73, `min(Solcast, Open-Meteo)` 0.81. Worst
misses of the `min()`: 13.0 forecast vs 5.7 actual; 5.7 vs 1.6; 6.7 vs 2.0;
9.5 vs 5.7 kWh. Hence `input_number.forecast_derate = 0.8`. Note that during
this period the "+03:00" key bug had removed Open-Meteo from the live
calculation, so the system was running on Solcast alone.

## Part 4 — grid availability at night

Hours per night with grid voltage > 200 V between 23:00 and 07:00:

- 26.10–03.11: 4.6–7.9 h (mostly ~7.9).
- 04.11–17.11: 1.8–7.9 h, average 5.2 h; under 5 h on 8 of 14 nights.

Most of the cuts are the household's own over-voltage relay (opens at 255 V
while the street sits at 250–260 V at night), not utility outages. Charging
pulls the voltage down by ~2.8 V per kW (share of minutes ≥ 250 V: 30 %
without charging, 4 % while charging ~1.6 kW).

At the old fixed 30 A (~1.5 kW into the battery) a 2-hour grid window adds
~3 kWh. Hence v4.0 raises the grid-charge current (up to 60 A) under the
import limiter — see [`grid-import-limiter.md`](grid-import-limiter.md).
