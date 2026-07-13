# How the model works

## The problem with a fixed threshold

The first version of this system compared the battery's state of charge against
a fixed nightly energy requirement — a single kWh constant per season
(winter / spring-autumn / summer). This worked, but it conflated two separate
questions:

1. How much energy will the house consume before generation takes over?
2. *When* will generation actually start covering that load tomorrow morning?

A fixed seasonal constant answers neither well. On a June morning with good
sun it might be 07:00. On a cloudy October morning it could be 11:00. The same
threshold treats both nights identically.

## The hourly deficit model

The current model asks one narrower question: **starting from the current hour,
how much energy is needed to keep the battery above a safety floor until
generation reliably covers the household load?**

It walks forward hour by hour through the night window (23:00 → 11:00 the next
morning — wide enough to cover a slow winter sunrise), comparing a conservative
generation forecast against an hourly load profile:

```
for each hour h in [23, 0, 1, 2, ..., 11]:
    gen_wh  = min(solcast_hourly[h], open_meteo_hourly[h])
    load_wh = load_profile[season][h]

    if gen_wh >= load_wh:
        if previous_hour_also_covered:
            STOP — generation has taken over
    else:
        deficit += load_wh - gen_wh

target_energy = deficit / round_trip_efficiency + safety_floor + reserve_kwh
target_soc    = target_energy / battery_capacity_kwh
```

The **two-consecutive-hours** rule avoids stopping early on a single lucky
hour immediately followed by clouds.

## Why it recalculates dynamically, not just once at 23:05

An earlier version of the sensor always started its loop from 23:00, even when
evaluated at 05:00. This meant it was answering "what would the deficit have
been *if we were starting the night now*" rather than "what is *left* of the
night from where we are."

**Bug symptom:** the sensor showed nearly identical values at 01:30 and 05:00,
because it kept re-deriving the same answer to a question no longer being asked.

**Fix:** the loop now starts at `now().hour` and only sums the hours still
ahead. This also required correctly determining which calendar date's forecast
data to use for each hour — since when the sensor re-evaluates at 05:00, hours
5–11 belong to *today's* forecast data, not *tomorrow's*.

## Forecast source selection: today vs. tomorrow

Weather integrations (Solcast, Open-Meteo) rotate their "today"/"tomorrow"
entities at midnight. The model applies this rule:

- Hour 23: always use `_today` entities (it's still the same calendar date)
- Hours 0–11, evaluated *at* 23:00 (`started_at_23 = true`): use `_tomorrow`
  entities (the next calendar date)
- Hours 0–11, evaluated *after* midnight (`started_at_23 = false`): use
  `_today` entities (we are now on that calendar date)

### Midnight rotation edge case

If the sensor re-evaluates in the first 5 minutes after midnight (`hour == 0`
and `minute < 5`), the model treats this the same as hour 23 and uses
`_tomorrow` entities. This is a mitigation for the rotation lag — integrations
don't always update exactly at 00:00:00, and reading a freshly-rotated entity
in that window can return stale data.

**Known accepted risk:** this mitigation is a simple time-based heuristic, not
a robust fix. A complete solution would look up the correct hour by its actual
ISO timestamp key in *both* entities, rather than picking one by name. This is
deferred — the edge case window is narrow (~5 minutes) and the consequences
are bounded (one re-evaluation might use slightly stale data).

## Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Round-trip efficiency | 0.92 | Charge efficiency from grid to battery |
| Safety floor | `input_number.minimum_soc_floor` % | Default 20% — minimum SOC at dawn |
| Reserve | 1.5 kWh | Fixed buffer on top of calculated deficit |
| Battery capacity | `input_number.battery_capacity_kwh` | Set to your usable kWh |
| Charge threshold | 0.5 kWh | `energy_deficit > 0.5` triggers automation A |
| Night window | 23:00–11:00 | Wide enough for slow winter sunrise |
| Solcast P10 bias | `input_number.solcast_p10_bias`, default 0.3 | 0–1 blend weight toward Solcast's pessimistic P10 forecast (see below) |
| Load correction bounds | [0.7, 1.8] | Clamp on the live load-correction factor (see below) |

## Load profile

The `load_profile_by_season` dict is a static table of mean Watts per hour,
built from ~110 days of real household consumption data pulled from InfluxDB
and processed in Python with proper UTC → local time conversion (+3h offset
applied in Python, not in Flux — the `location:` parameter in
`aggregateWindow()` was found to be non-functional in the InfluxDB version
used).

### Live load correction

The static table is a frozen snapshot — it drifts out of date as household
consumption changes. Rather than editing the table by hand, the sensor
compares a live 7-day rolling average (`sensor.avg_load_7d`, a Statistics
helper) against the season's static daily total and scales every hourly
value in the table by the ratio:

```
load_correction_raw = live_avg_load / season_daily_total
load_correction     = clamp(load_correction_raw, 0.7, 1.8)
load_wh[h]          = load_profile[season][h] * load_correction
```

The clamp exists so a single bad reading — or the Statistics helper not
having accumulated 7 days of history yet — can't scale the whole profile to
an extreme value. `season_daily_totals` must be kept equal to the sum of
`load_profile_by_season[season]` for each season; if you replace the hourly
table with your own data, recompute this total too.

This directly addresses the "rolling load profile" item that was previously
listed under Open questions below — it is not a true rolling average of the
table itself, but a live correction factor applied on top of it, which was
judged simpler and safer to reason about than continuously rewriting the
table.

### Solcast P10/P50 blending

Solcast's `detailedHourly` forecast exposes both `pv_estimate` (P50, the
median forecast) and `pv_estimate10` (P10 — "10% chance actual generation is
this low or lower"). Using P50 alone means roughly half of all cloudy days
will underperform the forecast. Blending a configurable fraction of P10 into
the Solcast side of the calculation adds a safety margin against that:

```
sc_estimate = sc_p50 * (1 - p10_bias) + sc_p10 * p10_bias
```

`input_number.solcast_p10_bias` controls the blend (0 = pure P50, 1 = pure
P10; default 0.3). This is independent of, and happens *before*, the
existing `min(Solcast, Open-Meteo)` step — it only makes the Solcast side of
that comparison more conservative.

Three seasons:

| Season | Months |
|--------|--------|
| `winter` | Dec, Jan, Feb (note: mapped to `transition` data — no real winter data available, system deployed in late February) |
| `transition` | Mar, Apr, Oct, Nov |
| `summer` | May–Sep |

**The winter profile is currently a copy of the transition profile.** Real
winter data will be accumulated over the first full winter season and the
profile will be updated.

Replace all values with your own household's actual hourly consumption before
deploying. These numbers are specific to one home and will not transfer.

## Open questions / deliberately deferred

- **Forecast error buffer — partially addressed.** The model originally
  trusted the forecast at face value. Solcast P10/P50 blending (see above)
  now adds a configurable safety margin on the Solcast side specifically;
  Open-Meteo has no equivalent percentile data to blend, so its side of the
  `min()` is still taken at face value. A true buffer applied after the
  `min()` (inflating the deficit uniformly regardless of source) remains
  undone.
- **Full kWh-based comparison throughout.** Internal calculations are energy-
  based (kWh), but the threshold comparisons in automations B and E still
  compare SOC percentages. A full conversion to kWh everywhere is a reasonable
  next step, not yet done.
- **Rolling load profile — addressed via a correction factor, not a rewrite.**
  The hourly table itself is still a static snapshot, but it's now scaled by
  a live 7-day-average correction factor (see above) rather than left to
  drift unchecked. Replacing the table's *shape* (not just its overall scale)
  would require an actual rolling-average rebuild of the hourly table, which
  is not implemented.
