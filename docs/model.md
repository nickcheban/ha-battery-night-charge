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

## Two PV arrays on independent MPPT inputs

The system originally had a single PV array feeding one inverter's single
MPPT tracker. After a hardware upgrade to a larger, dual-MPPT inverter, a
second array was added on a different azimuth — deliberately close to the
opposite orientation of the first (one biased toward morning sun, the other
toward afternoon sun), so each MPPT tracks its own array independently
instead of one compromise-angle array trying to do both.

This changes the shape of the generation curve (flatter, wider peak instead
of one sharp midday peak) but not the model's logic: Open-Meteo has no
native concept of "multiple orientations per entry," so the second array is
registered as a second integration entry, and its `wh_period` forecast is
simply summed with the first array's, hour by hour, before the
`min(Open-Meteo, Solcast)` comparison. Solcast needs no equivalent change —
it natively aggregates multiple rooftop resources into one forecast per
integration entry, so a second array there is just a second resource in the
same integration config, not a second set of entities.

If you only have one array, this collapses back to the original single-source
form — see the comments in `helpers/sensor_energy_deficit.yaml` for the exact
lines to remove.

## Inverter mode switching

**v1.x–v2.7** of this package decided "charge from grid tonight, yes/no" by
turning a physical Zigbee/Tuya grid-input breaker on and off. That breaker
was a mechanical relay being cycled on a decision that can legitimately
change several times a night (forecast correction, BMS noise, grid blips)
— see [`docs/anti-flapping-case-study.md`](anti-flapping-case-study.md) for
a real incident where an asymmetric debounce let that relay flap 5 times in
one night, including a 3-second on→off flip. v2.7 fixed the *symptom*
(symmetric debounce + minimum-runtime guard, the standard HA community
fix for short-cycling).

**v3.0** removes the underlying exposure instead of just guarding around it.
After a hardware upgrade, the new inverter exposes its own
output-source-priority and charge-source-priority selects (e.g.
`PV-Battery-Utility` / `PV-Utility-Battery` and `PV Priority` / `PV and
Utility` — exact option strings are vendor/model-specific). The breaker is
now left permanently energised, and every automation that used to call
`switch.turn_on` / `switch.turn_off` now calls `select.select_option`
against those two selects instead:

| Old actuator | New actuator |
|---|---|
| `switch.turn_on` (breaker) | `select.select_option` → `GRID_CHARGE` on output priority + `SOLAR_AND_GRID` on charge priority |
| `switch.turn_off` (breaker) | `select.select_option` → `AUTONOMOUS` on output priority + `SOLAR_ONLY` on charge priority |

Up to v3.0 the breaker's voltage sensor was still read as the grid-presence
signal. **As of v3.1 it isn't:** every automation checks the inverter's own
grid-input voltage instead (measured after the household's voltage-protection
relay, threshold 200 V). The inverter is battery-powered, so that reading stays
valid through an outage, unlike the breaker's radio chip (see
[`phantom_voltage.md`](phantom_voltage.md)).

**Why this is a better fix, not just a different one:**
- A relay has a finite mechanical cycle life; an inverter-firmware mode
  select does not wear out the same way.
- The v2.7 debounce/guard logic didn't disappear — it was ported as-is onto
  the new actuator (same `for:` windows, same 30-minute minimum-runtime
  guard, same hysteresis band). Removing the breaker from the loop doesn't
  remove the need for debouncing a noisy deficit sensor; it just removes the
  mechanical-wear consequence of getting it wrong.
- If your inverter has no equivalent mode select and a breaker is genuinely
  your only actuator, the pre-v3.0 logic (available in the git history / the
  original `switch.turn_on`/`switch.turn_off` shape shown in the diffs above)
  is still valid — this repo just no longer documents it as the primary path.

## Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Round-trip efficiency | 0.92 | Charge efficiency from grid to battery |
| Safety floor | `input_number.minimum_soc_floor` % | Default 20% — minimum SOC at dawn |
| Reserve | 1.5 kWh | Fixed buffer on top of calculated deficit |
| Battery capacity | `input_number.battery_capacity_kwh` | Set to your usable kWh |
| Charge threshold | 0.5 kWh | `energy_deficit > 0.5` triggers automation A |
| Night window | 23:00–11:00 | Wide enough for slow winter sunrise |
| Forecast source filter | > 10 Wh | A source counts for an hour only if it forecasts more than 10 Wh |

## Load profile

The `load_profile_by_season` dict is a static table of mean Watts per hour,
built from ~110 days of real household consumption data pulled from InfluxDB
and processed in Python with proper UTC → local time conversion (+3h offset
applied in Python, not in Flux — the `location:` parameter in
`aggregateWindow()` was found to be non-functional in the InfluxDB version
used).

### Removed in v3.1: live load correction and Solcast P10 blending

Earlier versions of this repo documented two refinements — scaling the
hourly profile by a live 7-day-average ratio, and blending a fraction of
Solcast's pessimistic P10 estimate into the Solcast side of the `min()`. They
are not part of the system that actually runs, so v3.1 removes them from the
templates and from this document. The idea behind P10 blending is taken up
differently in v4.0 (a single derating factor applied to the combined
forecast).

The forecast-source rule in production is also simpler than the earlier
"count a source if it has a data point" version: a source counts for an hour
only if it forecasts more than 10 Wh.

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

- **Forecast error buffer.** The model takes the forecast at face value
  (pessimistic `min()` of two sources plus a fixed 1.5 kWh reserve). A
  derating factor is added in v4.0.
- **Full kWh-based comparison throughout.** Internal calculations are energy-
  based (kWh), but the threshold comparisons in automations B and E still
  compare SOC percentages.
- **Load profile drift.** The hourly table is a static snapshot; the winter
  profile is still a copy of the transition profile.
