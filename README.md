# ha-battery-night-charge

Home Assistant automation package that decides once per night whether to charge
a home battery from the grid — and automatically manages the inverter's own
charge-mode selects throughout the night based on an hourly solar generation
forecast.

> **v3.0 architecture note:** earlier versions of this package (documented in
> [`docs/anti-flapping-case-study.md`](docs/anti-flapping-case-study.md))
> made this decision by toggling a physical grid-input breaker on and off.
> As of v3.0, the breaker is left permanently energised and the decision is
> instead applied by switching the inverter's own output/charge-priority
> mode — see [`docs/model.md#inverter-mode-switching`](docs/model.md#inverter-mode-switching)
> for why. If your hardware only exposes a breaker and no inverter-side mode
> select, the breaker-toggling shape is still valid — see the pre-v3.0 diffs
> referenced in the automation YAML comments.

> **AI-assisted project.** The logic, backtest methodology and bug fixes in this
> repository were developed iteratively in partnership with
> [Claude](https://claude.ai) (Anthropic). The AI helped analyse real historical
> sensor data from InfluxDB, identify edge cases, write and validate Jinja2
> templates, run backtests against ~110 days of recorded data before any
> automation went live, and later diagnose a real production flapping
> incident from history and trace data (see
> [`docs/anti-flapping-case-study.md`](docs/anti-flapping-case-study.md)).

---

## The problem it solves

A solar + LiFePO4 battery + hybrid inverter system needs to decide each evening:
*"Will the battery last until reliable morning generation, or do I need to top it
up from the cheap night tariff?"*

A fixed threshold ("charge if SOC < X%") wastes money on grid electricity that
solar would have covered for free a few hours later. This package instead asks:
**starting from right now, how much energy is actually needed to make it to
the first two consecutive hours where generation covers load?** — and
recalculates that answer dynamically throughout the night.

---

## Hardware context

- Hybrid inverter, 11 kW, dual independent MPPT inputs (5.5 kW each)
- Two PV arrays, one per MPPT tracker, on deliberately different azimuths so
  each catches a different part of the day instead of one compromise-angle
  array — roughly 180° apart (e.g. one biased toward morning sun, one toward
  afternoon sun), same tilt
- LiFePO4 battery pack with a BMS exposing SOC
- A Zigbee/WiFi smart relay as the grid breaker — left permanently on as of
  v3.0, its voltage sensor is still used as a grid-presence signal; see
  [the phantom voltage problem](#the-phantom-voltage-problem) below for why
  a cheap relay like this needs special handling even when only used as a
  sensor
- Inverter-side output-source-priority and charge-source-priority selects —
  the actual actuator this package drives as of v3.0 (see
  [`docs/model.md#inverter-mode-switching`](docs/model.md#inverter-mode-switching))

The dual-array setup is not load-bearing for the model's *logic* — see
[`docs/model.md`](docs/model.md#two-pv-arrays-on-independent-mppt-inputs) for
exactly what changes (forecast summing) and what doesn't (everything else).
If you only have one array, the model collapses back to its original
single-source form.

---

## How the decision is made

At 23:05, a template sensor walks hour-by-hour from the current hour through
the overnight window (23:00–11:00 the next morning), comparing a conservative
solar forecast against a seasonal hourly load profile. It stops accumulating
deficit as soon as generation covers load for two consecutive hours. The result
is a kWh deficit figure — if it exceeds 0.5 kWh and the SOC reading is valid,
the inverter is switched into grid-charge mode.

**Forecast source:** `min(Solcast hourly, Open-Meteo hourly)` per hour — the
more conservative of two independent sources, chosen to reduce the chance of a
single bad forecast leaving the battery flat. On the Open-Meteo side, if you
have two PV arrays on separate azimuths, their forecasts are summed before
this comparison (see Hardware context above).

**Load profile:** a static hourly table built from ~110 days of real household
consumption data, split into three seasons (winter/transition/summer), scaled
by a live 7-day-average correction factor so it doesn't go stale. Replace the
base table with your own data.

Full design rationale, bugs found and fixed during development, and
accepted known risks are documented in [`docs/model.md`](docs/model.md).

---

## Automations

All six automations now act on `select.inverter_output_priority` /
`select.inverter_charge_priority` (inverter mode) rather than a
`switch.grid_switch` (breaker) — see
[the v3.0 architecture note](#ha-battery-night-charge) above.

| ID | Alias | Trigger | What it does |
|----|-------|---------|--------------|
| **A** | Lock target & set night charge mode | Time 23:05 | Snapshots the current target SOC into a helper (so it doesn't drift overnight), then switches the inverter into grid-charge mode if `energy_deficit > 0.5 kWh` AND SOC reading is valid — otherwise explicitly switches it into autonomous (solar-only) mode. Sends Telegram summary either way. |
| **B** | Autonomous — target reached | SOC state change, `for: 30s` | Watches SOC all night; switches the inverter back to autonomous mode when SOC ≥ `min(target+2%, 95%)` stable for 30 seconds. The 30 s debounce is intentional — BMS units commonly flap to `unknown` during charging, making longer windows unreliable. |
| **C** | Emergency night correction | `energy_deficit` crosses 0.7 / 0.3 kWh | Reacts to reality diverging from forecast mid-night. **v2.7** logic (symmetric debounce + symmetric 30-min minimum-runtime guard), **v3.0** actuator — switches to grid-charge mode if deficit climbs above 0.7 kWh for 2 minutes (and inverter's been autonomous ≥30 min); switches back to autonomous if deficit drops below 0.3 kWh for 2 minutes (and inverter's been grid-charging ≥30 min). Fixes a real multi-cycle relay-flapping incident from the breaker era — see [`docs/anti-flapping-case-study.md`](docs/anti-flapping-case-study.md). `mode: restart` so a fresh trigger always wins. |
| **D** | Morning safety reset | Time 06:58 | Unconditional switch to autonomous mode, no conditions, no notification. This is the layer that *must* work even if everything else is broken. |
| **E** | Grid recovery | Breaker voltage > 210 V for 2 min | Handles the case where the grid disappeared and came back. The breaker itself stays always-on; its voltage sensor is still read as the "grid actually present" signal. Four branches: resume grid-charge (night + deficit), confirm target already reached → autonomous (night + SOC ok), force autonomous (daytime — avoid paying peak tariff), emergency grid-charge (battery critically low regardless of time). |
| **F** | Recovery after HA restart | Home Assistant `start` event | If HA itself restarts mid-night, none of A/C/E's triggers fire retroactively. After a 120 s delay (for the BMS and load-history helper to come up), switches the inverter to grid-charge mode if it's in autonomous mode despite `energy_deficit > 0.5 kWh`. Local notification only (persistent_notification, not Telegram). |

---

## The phantom voltage problem

Cheap Zigbee/Tuya smart breakers power their radio chip from the same line they
switch. When a voltage protection relay upstream cuts the grid, the breaker
loses power — but before it fully dies it sends garbage readings (10–175 V) to
Home Assistant instead of going `unavailable`. A naive `> 0 V` check would
trigger the breaker to "turn on" against a dead chip.

**Solution used here:** automation E, which reacts to the grid coming back,
requires `breaker_voltage > 210 V` held for 2 minutes as a precondition
before acting. Based on 30 days of historical data, phantom readings never
exceeded 175 V or held a stable value for more than a few minutes. (Pre-v3.0,
every automation that turned the breaker itself *on* also carried this
check directly — as of v3.0 the breaker is never actuated, only sensed, so
the check now lives solely in automation E's trigger.)

See [`docs/phantom_voltage.md`](docs/phantom_voltage.md) for full analysis.

---

## Backtest results

Before going live, the model was validated against real recorded data:

**Pass 1 — decision-point backtest (108 nights, March–June):**
Using actual PV generation as a stand-in for forecast (tests algorithm logic,
not forecast accuracy). **Result: 0 dangerous misses** — no night where the
model decided not to charge and the battery actually fell below the safety floor.

**Pass 2 — intra-night dynamics (9 nights, 3 per morning category):**
Simulated the sensor re-evaluating at `cur_h` = 23, 0, 1 … 7 on each test
night. Confirmed that the remaining deficit decreases monotonically as the night
progresses — validating the fix for a bug where the sensor always re-calculated
from 23:00 regardless of the current hour (causing identical readings at 01:30
and 05:00).

Full methodology and results: [`docs/backtest.md`](docs/backtest.md).

---

## A real production incident, diagnosed and fixed

After the inverter/dual-array upgrade, one night produced 5 breaker on/off
cycles, including a 3-second flap. This is documented end-to-end — symptom,
diagnosis method (including where automation trace retention got in the way),
root cause, and the v2.7 fix — in
[`docs/anti-flapping-case-study.md`](docs/anti-flapping-case-study.md). It's
a fairly generic short-cycling problem and the fix follows standard Home
Assistant community patterns (symmetric debounce + a `min_cycle_duration`-
style guard), so it's included here as a worked example even if your setup
never hits this exact bug. **v3.0 goes a step further** and removes the
mechanical relay from the nightly decision loop entirely (see
[`docs/model.md#inverter-mode-switching`](docs/model.md#inverter-mode-switching))
— the v2.7 debounce/guard logic is preserved and ported onto the new
actuator, but the class of relay-wear risk the incident surfaced no longer
applies to this package's normal operation.

---

## Design patterns worth stealing

**Snapshot the decision, don't chase a moving target.** The target SOC is
written to `input_number.locked_target_soc` once at 23:05 and never re-read
during the night. A live-recalculating target would cause the breaker to
oscillate as the forecast shifts overnight.

**Dumb safety layers are more reliable than smart ones.** Automation D
deliberately has zero conditions. It resets the inverter to autonomous mode
at 06:58 every morning, full stop. This is the layer that works even if
every other automation is broken, the BMS is unavailable, or the forecast
API is down.

**Remove the risky actuator, don't just guard it better.** v2.7 fixed a
relay-flapping incident by debouncing symmetrically — a textbook fix. v3.0
went further: once the hardware offered an alternative (inverter-side mode
selects with no mechanical wear), the physical relay was taken out of the
nightly decision loop entirely rather than just guarded more carefully. Not
every system will have this option, but when it does, it's worth asking
whether the actuator itself is the actual risk, not just the logic driving it.

**Debounce real hardware — symmetrically.** BMS units commonly report SOC as
`unknown` for ~35 seconds during active charging. A `for: 2 minutes` trigger
window will almost never fire; `for: 30s` is the right balance. And when a
value oscillates around a hysteresis band (not just a single threshold),
debounce *both* sides of the band the same way — an asymmetric debounce is
exactly what caused the incident above.

**Two independent forecast sources, combined pessimistically.** Relying on one
weather API is a single point of failure for the most important input. Taking
`min()` of two sources per hour costs almost nothing and removes an entire
failure class.

**Confirm real grid presence, not just switch state.** A breaker reporting `on`
with 50 V across it is not charging anything. Require `voltage > 210 V` for 2
minutes before acting — not just a non-zero reading.

**Recovery from grid loss is not optional.** A system that only works when
nothing restarts and the grid never blinks is essentially unfinished. Automation
E exists because real-world grids do both.

**Recovery from *your own* restart is not optional either.** A grid outage
isn't the only thing that can silently break the night's automations — HA
itself restarting mid-night (update, crash, add-on reload) makes every
time-based and threshold-based trigger miss its window just as thoroughly.
Automation F re-checks state on `homeassistant: start` for the same reason
automation E re-checks it on grid recovery.

**Scale a static profile with a live signal instead of hand-editing it.** The
load profile table is a frozen snapshot that goes stale. Rather than
periodically re-exporting and replacing it, a single live ratio (7-day actual
average ÷ season's static total, clamped to a sane range) keeps the whole
table roughly current without turning it into something that needs ongoing
manual maintenance.

**A percentile forecast is a cheap safety margin, if the source offers one.**
Solcast's P10 (10th-percentile, pessimistic) estimate costs nothing extra to
read but is normally ignored in favor of the median (P50) forecast. Blending
in a configurable fraction of P10 buys protection against cloudy-day
underperformance for free — the same idea behind Predbat's
`weather_forecast_pv_quantile_bias`.

---

## Installation

### Prerequisites

- Home Assistant with an inverter integration exposing output-source-priority
  and charge-source-priority `select` entities (v3.0 actuator)
- A Zigbee/WiFi smart breaker entity — not actuated, but its voltage sensor is
  used as the grid-presence signal (automation E); if your inverter has no
  mode selects, the breaker can still be used as the primary actuator — see
  the pre-v3.0 shape referenced in the automation YAML comments
- LiFePO4 battery with a BMS that exposes SOC — must resolve to
  `unavailable`/`unknown` (not a stale number) when the BMS is unreachable
- [Solcast](https://github.com/BJReplay/ha-solcast-solar) integration
- [Open-Meteo Solar Forecast](https://www.home-assistant.io/integrations/open_meteo_solar_forecast/) integration (one entry per PV array, if you have more than one)
- Telegram bot configured in Home Assistant (`notify.send_message` service)

### Helpers to create

Before importing automations, create these in
*Settings → Devices & Services → Helpers*:

| Helper | Type | Purpose |
|--------|------|---------|
| `input_number.battery_capacity_kwh` | Number | Usable battery capacity in kWh |
| `input_number.minimum_soc_floor` | Number | Minimum SOC % you want at dawn |
| `input_number.locked_target_soc` | Number | Written by automation A — do not edit manually |
| `input_number.solcast_p10_bias` | Number (0–1) | Blend weight toward Solcast's pessimistic P10 forecast — default 0.3 |

### Entity ID mapping

All entity IDs in this repo use generic names. Map them to your actual entities:

| Placeholder | What it should be |
|-------------|------------------|
| `select.inverter_output_priority` | Your inverter's output-source-priority select. Options used in this repo: `AUTONOMOUS` (your inverter's "prioritize battery/solar over grid" mode, e.g. "PV-Battery-Utility"/"SBU") and `GRID_CHARGE` (your inverter's "prioritize grid for charging" mode, e.g. "PV-Utility-Battery"/"SUB") — substitute your integration's actual option strings |
| `select.inverter_charge_priority` | Your inverter's charge-source-priority select. Options used in this repo: `SOLAR_ONLY` (e.g. "PV Priority") and `SOLAR_AND_GRID` (e.g. "PV and Utility") — substitute your integration's actual option strings |
| `switch.grid_switch` | Your smart breaker switch entity — left permanently on, not actuated by this package as of v3.0 |
| `sensor.breaker_voltage` | Voltage sensor on the breaker — still read by automation E as the grid-presence signal |
| `sensor.battery_soc_effective` | Your BMS SOC sensor (must go `unavailable`, not stale) |
| `sensor.pv_forecast_today/tomorrow` | Solcast entities |
| `sensor.energy_production_today/tomorrow` | Open-Meteo Solar Forecast entities — PV array 1 |
| `sensor.energy_production_today/tomorrow_array2` | Open-Meteo Solar Forecast entities — PV array 2 (omit if you only have one array; see `docs/model.md`) |
| `sensor.avg_load_7d` | A 7-day rolling average daily load — e.g. a Statistics helper (`state_characteristic: change`, `max_age: 7 days`) on your inverter-output energy sensor. Drives the live load correction (see `docs/model.md`) |
| `notify.YOUR_TELEGRAM_BOT` | Your Telegram notify entity |

### Load profile

The `load_profile_by_season` dict in the sensor templates is built from one
specific household's data. **It will not match your house.** Export your own
hourly consumption averages from your energy monitoring tool and replace the
values before deploying. If you replace it, also update `season_daily_totals`
(the daily kWh sum per season) — it feeds the live load-correction factor
described in `docs/model.md`, and the two will silently disagree if only one
is updated.

---

## More mature alternative

If this looks like a hand-rolled battery charge scheduler — it is. For a more
comprehensive, actively maintained solution to the same problem (with real
optimisation, tariff awareness, and multi-inverter support), see
[**Predbat**](https://github.com/springfall2008/batpred). This project is
narrower in scope and tuned to one specific system's quirks (phantom voltage,
dual forecast blending, seasonal load profile), but solves the same core
problem that Predbat solves — just without the generalisation.

---

## License

MIT — use, adapt and redistribute freely.
