# ha-battery-night-charge

Home Assistant automation package that decides once per night whether to charge
a home battery from the grid — and automatically manages the grid breaker
throughout the night based on an hourly solar generation forecast.

> **AI-assisted project.** The logic, backtest methodology and bug fixes in this
> repository were developed iteratively in partnership with
> [Claude](https://claude.ai) (Anthropic). The AI helped analyse real historical
> sensor data from InfluxDB, identify edge cases, write and validate Jinja2
> templates, and run backtests against ~110 days of recorded data before any
> automation went live.

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

## How the decision is made

At 23:05, a template sensor walks hour-by-hour from the current hour through
the overnight window (23:00–11:00 the next morning), comparing a conservative
solar forecast against a seasonal hourly load profile. It stops accumulating
deficit as soon as generation covers load for two consecutive hours. The result
is a kWh deficit figure — if it exceeds 0.5 kWh and real grid voltage is
confirmed (>210 V for 2 minutes), the breaker turns on.

**Forecast source:** `min(Solcast hourly, Open-Meteo hourly)` per hour — the
more conservative of two independent sources, chosen to reduce the chance of a
single bad forecast leaving the battery flat.

**Load profile:** a static hourly table built from ~110 days of real household
consumption data, split into three seasons (winter/transition/summer). Replace
with your own data.

Full design rationale, two bugs found and fixed during development, and
accepted known risks are documented in [`docs/model.md`](docs/model.md).

---

## Automations

| ID | Alias | Trigger | What it does |
|----|-------|---------|--------------|
| **A** | Lock target & start charge | Time 23:05 | Snapshots the current target SOC into a helper (so it doesn't drift overnight), then turns the breaker on if `energy_deficit > 0.5 kWh` AND real grid voltage confirmed (`>210 V`). Sends Telegram summary either way. |
| **B** | Off — target reached | SOC state change, `for: 30s` | Watches SOC all night; turns breaker off when SOC ≥ `min(target+2%, 95%)` stable for 30 seconds. The 30 s debounce is intentional — BMS units commonly flap to `unknown` during charging, making longer windows unreliable. |
| **C** | Emergency night correction | `energy_deficit` crosses 0.7 / 0.3 kWh | Reacts to reality diverging from forecast mid-night. Turns breaker on if deficit climbs above 0.7 kWh for 2 minutes; turns it off if deficit drops below 0.3 kWh. Hysteresis prevents oscillation. `mode: restart` so a fresh trigger always wins. |
| **D** | Morning safety off | Time 06:58 | Unconditional `switch.turn_off`, no conditions, no notification. This is the layer that *must* work even if everything else is broken. |
| **E** | Grid recovery | Breaker voltage > 210 V for 2 min | Handles the case where the grid disappeared and came back. Four branches: resume charging (night + deficit), confirm target already reached (night + SOC ok), force off (daytime — avoid paying peak tariff), emergency on (battery critically low regardless of time). |
| **F** | Recovery after HA restart | Home Assistant `start` event | If HA itself restarts mid-night, none of A/C/E's triggers fire retroactively. After a 120 s delay (for the BMS and load-history helper to come up), turns the breaker on if it's off despite `energy_deficit > 0.5 kWh`. Local notification only (persistent_notification, not Telegram). |

---

## The phantom voltage problem

Cheap Zigbee/Tuya smart breakers power their radio chip from the same line they
switch. When a voltage protection relay upstream cuts the grid, the breaker
loses power — but before it fully dies it sends garbage readings (10–175 V) to
Home Assistant instead of going `unavailable`. A naive `> 0 V` check would
trigger the breaker to "turn on" against a dead chip.

**Solution used here:** all automations that turn the breaker *on* require
`breaker_voltage > 210 V` held for 2 minutes as a precondition. Based on 30
days of historical data, phantom readings never exceeded 175 V or held a stable
value for more than a few minutes.

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

## Design patterns worth stealing

**Snapshot the decision, don't chase a moving target.** The target SOC is
written to `input_number.locked_target_soc` once at 23:05 and never re-read
during the night. A live-recalculating target would cause the breaker to
oscillate as the forecast shifts overnight.

**Dumb safety layers are more reliable than smart ones.** Automation D
deliberately has zero conditions. It turns the breaker off at 06:58 every
morning, full stop. This is the layer that works even if every other automation
is broken, the BMS is unavailable, or the forecast API is down.

**Debounce real hardware.** BMS units commonly report SOC as `unknown` for
~35 seconds during active charging. A `for: 2 minutes` trigger window will
almost never fire. `for: 30s` is the right balance — long enough to ignore
transient flaps, short enough to catch the actual target.

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

- Home Assistant with a Zigbee/WiFi smart breaker entity
- LiFePO4 battery with a BMS that exposes SOC — must resolve to
  `unavailable`/`unknown` (not a stale number) when the BMS is unreachable
- [Solcast](https://github.com/BJReplay/ha-solcast-solar) integration
- [Open-Meteo Solar Forecast](https://www.home-assistant.io/integrations/open_meteo_solar_forecast/) integration
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
| `switch.grid_switch` | Your smart breaker switch entity |
| `sensor.breaker_voltage` | Voltage sensor on the breaker |
| `sensor.battery_soc_effective` | Your BMS SOC sensor (must go `unavailable`, not stale) |
| `sensor.pv_forecast_today/tomorrow` | Solcast entities |
| `sensor.energy_production_today/tomorrow` | Open-Meteo Solar Forecast entities |
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
