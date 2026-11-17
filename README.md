# ha-battery-night-charge

Home Assistant automation package that decides once per night whether to charge
a home battery from the grid — and automatically manages the inverter's own
charge-mode selects throughout the night based on an hourly solar generation
forecast.

> **v4.0 — winter mode.** The target is now "enough to reach the next 23:00"
> (the start of the next cheap night window) instead of "enough to reach the
> first two hours of stable generation". In summer the two give the same
> answer; from October to March the old rule left the battery flat every
> cloudy evening. v4.0 also adds a grid-import limiter (automation G) for a
> 5 kW utility connection, fixes a time-zone bug that silently removed one of
> the two forecast sources after the switch to winter time, and adds a
> forecast derating factor. Tuned for the current ~13.3 kWh battery; a
> 2 × 16 kWh bank is next (see [Hardware context](#hardware-context)).
> Details: [`docs/model.md#v40-winter-horizon`](docs/model.md#v40-winter-horizon),
> [`docs/backtest-horizon.md`](docs/backtest-horizon.md),
> [`docs/grid-import-limiter.md`](docs/grid-import-limiter.md).

> **v3.0 architecture note:** earlier versions of this package (documented in
> [`docs/anti-flapping-case-study.md`](docs/anti-flapping-case-study.md))
> made this decision by toggling a physical grid-input breaker on and off.
> As of v3.0, the breaker is left permanently energised and the decision is
> instead applied by switching the inverter's own output/charge-priority
> mode — see [`docs/model.md#inverter-mode-switching`](docs/model.md#inverter-mode-switching)
> for why. If your hardware only exposes a breaker and no inverter-side mode
> select, the breaker-toggling shape is still valid — see the pre-v3.0 diffs
> referenced in the automation YAML comments.

> **v3.1 sync note:** for a while the running system and this repo drifted
> apart. v3.1 brings the repo back in line with what actually runs: all mode
> changes go through one retry script, grid presence is read from the
> inverter's own input voltage, automation E also handles grid *loss*, and the
> target SOC is live rather than locked at 23:05. Two refinements documented
> earlier (Solcast P10 blending, live load correction) are not part of the
> running system and are removed. See [Known issues](#known-issues) for what
> the running system still gets wrong.

> **Who did what.** The project — the system and its hardware, the goals,
> every design decision, and running it in production — is led by the author.
> Analysis and calculations were done with [Claude](https://claude.ai)
> (Anthropic): querying and analysing historical sensor data in InfluxDB,
> backtests (including the 249-night horizon comparison for v4.0), drafting
> Jinja2 templates and automations and checking them against the live Home
> Assistant, and diagnosing the flapping incident from history and traces
> (see [`docs/anti-flapping-case-study.md`](docs/anti-flapping-case-study.md)).
> Claude's proposals went into production only after the author reviewed them;
> some were rejected or changed — for example, locking the target at 23:05 and
> a "house on grid after charging" hold mode.

---

## The problem it solves

A solar + LiFePO4 battery + hybrid inverter system needs to decide each evening:
*"Will the battery last until reliable morning generation, or do I need to top it
up from the cheap night tariff?"*

A fixed threshold ("charge if SOC < X%") wastes money on grid electricity that
solar would have covered for free a few hours later. This package instead asks:
**starting from right now, how much energy does the battery need to hold to
reach the next 23:00 — the start of the next cheap night window — given the
hourly forecast?** — and recalculates that answer throughout the night. (Up to
v3.1 the question stopped at "the first two hours where generation covers
load"; that is the same answer in summer and the wrong one in winter.)

---

## Hardware context

- Hybrid inverter, 11 kW, dual independent MPPT inputs (5.5 kW each)
- Two PV arrays, one per MPPT tracker, on deliberately different azimuths so
  each catches a different part of the day instead of one compromise-angle
  array — roughly 180° apart (e.g. one biased toward morning sun, one toward
  afternoon sun), same tilt
- LiFePO4 battery with a BMS exposing SOC. **Currently ~13.3 kWh (full
  capacity as reported by the BMS) — all parameters in this release are tuned
  for it.** Two new packs are already on
  hand: 2 × 16 kWh (each 16S, 314 Ah LiFePO4 cells, ANENJI ANJ-314AH-S),
  ~32 kWh in total. The package will be re-tuned for that bank (capacity,
  reserve, charge currents); with ~13.3 kWh, 27 of 249 backtest nights needed
  more than fits under the 95 % cap
- A 5 kW grid connection limit enforced by the utility's smart meter
  (exceeding it disconnects the house for ~10 minutes) — handled by
  automation G
- An over/under-voltage protection relay on the grid input (opens above
  255 V) — the street supply often sits right at that limit at night, so the
  grid "disappears" and "comes back" many times a night
- A Zigbee/WiFi smart breaker on the grid input — left permanently on and, as
  of v3.1, not read by this package at all (see
  [the phantom voltage problem](#the-phantom-voltage-problem))
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

A template sensor walks hour by hour from now to the next 23:00, keeping a
running balance of seasonal hourly load minus the conservative solar
forecast × a derating factor (0.8). The **peak** of that running balance is
what the battery must hold on top of the safety floor and a 1.5 kWh reserve
(capped at 95 %). On a sunny day the peak is in the morning — the same answer
as the old "until stable generation" rule; on a cloudy day it is in the
evening — enough for the whole day. At 23:05 the resulting kWh deficit — if it exceeds 0.5 kWh, the SOC reading is valid and
the grid is present, the inverter is switched into grid-charge mode. The same
sensors keep recalculating all night; B stops charging when SOC reaches the
*current* target, C reacts if the forecast changes, E reacts to the grid
coming and going.

**Forecast source:** `min(Solcast hourly, Open-Meteo hourly)` per hour × 0.8 —
the more conservative of two independent sources, then derated (over
late October – mid November the actual/forecast ratio of that `min()` was
0.81). On the Open-Meteo side, if you
have two PV arrays on separate azimuths, their forecasts are summed before
this comparison (see Hardware context above).

**Load profile:** a static hourly table built from ~110 days of real household
consumption data, split into three seasons (winter/transition/summer). Replace
it with your own data.

Full design rationale, bugs found and fixed during development, and
accepted known risks are documented in [`docs/model.md`](docs/model.md).

---

## Automations

All automations act on `select.inverter_output_priority` /
`select.inverter_charge_priority` (inverter mode), and — except where noted —
through one script, `script.inverter_set_mode_with_retry`
([`scripts/set_mode_with_retry.yaml`](scripts/set_mode_with_retry.yaml)),
which verifies the write, retries once after 30 s and alerts with the caller's
tag if it still fails. Grid presence everywhere means
`sensor.inverter_grid_voltage > 200 V`.

| ID | Alias | Trigger | What it does |
|----|-------|---------|--------------|
| **A** | Night charge decision | Time 23:05 | Waits up to 5 min for a valid SOC if the sensor is flapping. Deficit > 0.5 kWh + grid present → grid-charge mode. Deficit > 0.5 kWh + no grid → warning only (E/C take over when the grid returns). Otherwise → autonomous mode. Records the starting target in a helper for the notification only — nothing locks it. |
| **B** | Stop at target | SOC change, `for: 30s` | While grid-charging at night: SOC ≥ `min(live target + 2 %, 95 %)` → autonomous mode. The target falls through the night, so B stops as soon as the battery holds enough for what is *left*. |
| **C** | Emergency night correction | `energy_deficit` crosses 0.7 / 0.3 kWh | Deficit > 0.7 for 2 min while autonomous and grid present → grid-charge. Deficit < 0.3 after ≥ 30 min of grid-charging and grid present → autonomous. `mode: restart`. |
| **D** | Morning safety reset | Time 06:58 | Unconditional autonomous mode via the retry script — the layer that must work even if everything else is broken. |
| **E** | Grid loss / recovery | Grid voltage < 50 V (30 s), > 200 V (2 min), every 15 min | Grid lost while grid-charging → autonomous (keeps HA in sync with the hardware). Grid back at night with deficit > 0.5 → grid-charge (the 15-min tick is the safety net for a missed edge). Target already met → autonomous. Daytime → autonomous. Battery < 49 V → grid-charge. |
| **F** | Recovery after HA restart | HA `start` + 120 s | Night, deficit > 0.5 kWh, inverter autonomous, grid present → grid-charge. Local notification only. |
| **G** | Grid import limiter (v4.0) | Target charge current changes / mode changes / metered import > 4.8 kW for 20 s | While grid-charging: sets the inverter's grid-charge current to what fits under the import budget (down at once, up after 2 min stable). Less than 10 A fits or import too high → house on battery, resumes when load drops. See [`docs/grid-import-limiter.md`](docs/grid-import-limiter.md). |

---

## The phantom voltage problem

Cheap Zigbee/Tuya smart breakers power their radio chip from the same line they
switch. When a voltage protection relay upstream cuts the grid, the breaker
loses power — but before it fully dies it sends garbage readings (10–175 V) to
Home Assistant instead of going `unavailable`. Up to v3.0 this package worked
around that with a "> 210 V held for 2 minutes" rule on the breaker's voltage.

**As of v3.1 the breaker isn't read at all.** Grid presence comes from the
inverter's own grid-input voltage, measured after the protection relay. The
inverter runs from the battery, so its reading stays valid through an outage.
The 2-minute hold on E's recovery trigger is kept anyway.

See [`docs/phantom_voltage.md`](docs/phantom_voltage.md) for the original
analysis.

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

**Pass 4 — horizon change (v4.0, 249 nights, March–November):** old "until
stable generation" rule vs new "peak balance until next 23:00", on actual
data. Identical on 144 of 150 nights May–September; the new rule never asks
for less, and adds where the evening isn't covered — 12 of 30 nights in
October, 9 of 14 in early November. Over 22 November days the old rule saw
1.45 kWh of need on average where 4.8 kWh were needed. Details:
[`docs/backtest-horizon.md`](docs/backtest-horizon.md).

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

**Don't snapshot the target — buy only what is still needed.** An earlier
version locked the target SOC at 23:05. It was dropped: the goal is to buy
exactly as much grid energy as the rest of the night needs, and a target
frozen at 23:05 ignores a forecast that improves overnight and overpays every
time it does. The target is recalculated continuously; what keeps the mode
from oscillating is hysteresis and debounce (B's +2 % buffer, C's 0.7/0.3 band
and 30-minute guard), not freezing the input.

**One actuator path, verified.** Every mode change goes through one queued
script that checks the select actually changed, retries once and says which
automation's command was lost. A Wi-Fi data collector drops writes now and
then; without the check the house can sit in the wrong mode until 06:58.

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

**Look at the peak of the balance, not at a calendar.** "Until stable
generation" and "until tonight" are the same question on a sunny day and very
different ones on a cloudy day. Taking the peak of the running
load-minus-generation balance up to the next cheap window answers both without
a summer/winter switch — and a calendar switch would have been wrong anyway
(winter-like days happen in June, summer-like days in November).

**Express a limit in the units of whoever enforces it.** The 5 kW limit is
enforced by the utility's meter, which reads ~13 % higher than the inverter.
The budget is set in meter watts and converted, so "4500" means 4500 on the
meter, not 5100.

**Never build a timestamp by hand.** A literal `+03:00` in a lookup key worked
all summer and then silently removed half of the forecast the night the
clocks changed. Build the key from an actual datetime.

**Two independent forecast sources, combined pessimistically.** Relying on one
weather API is a single point of failure for the most important input. Taking
`min()` of two sources per hour costs almost nothing and removes an entire
failure class.

**Confirm real grid presence from a device that survives the outage.** A
breaker reporting `on` with 50 V across it is not charging anything, and a
breaker's own radio dies with the grid. Read grid voltage from something that
stays powered (here the battery-backed inverter) and check it in every branch
that starts charging.

**Recovery from grid loss is not optional.** A system that only works when
nothing restarts and the grid never blinks is essentially unfinished. Automation
E exists because real-world grids do both.

**Recovery from *your own* restart is not optional either.** A grid outage
isn't the only thing that can silently break the night's automations — HA
itself restarting mid-night (update, crash, add-on reload) makes every
time-based and threshold-based trigger miss its window just as thoroughly.
Automation F re-checks state on `homeassistant: start` for the same reason
automation E re-checks it on grid recovery.

---

## Known issues

Documented here because the repo mirrors what actually runs:

- **C's debounce is asymmetric.** `stop_charge` fires without `for:` and the
  start branch has no 30-minute guard. [The case study](docs/anti-flapping-case-study.md)
  explains why symmetric is better; the running system hasn't been changed.
- **E daytime vs emergency.** In daytime with the battery below 49 V, the
  15-minute tick alternates between the emergency branch (grid-charge) and
  the daytime branch (back to autonomous). The inverter's own low-battery
  cut-over (52 V) acts first in practice.
- **B has a dead trigger** (`deficit_below_0_5`) with no matching branch.
- **A's grid check at 23:05 is an instantaneous reading**, no debounce.

---

## Installation

### Prerequisites

- Home Assistant with an inverter integration exposing output-source-priority
  and charge-source-priority `select` entities (v3.0 actuator)
- An inverter grid-input voltage sensor (grid presence) and a battery voltage
  sensor (automation E's emergency branch)
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
| `input_number.locked_target_soc` | Number | Written by automation A at 23:05 for the notification text only |
| `input_number.forecast_derate` | Number (0.5–1.0, step 0.05) | Multiplier on the forecast — default 0.8 |
| `input_number.grid_import_budget` | Number (W) | Import budget in **utility-meter** watts — default 4500 for a 5 kW limit |
| `input_number.utility_meter_correction` | Number (1.00–1.30) | Utility meter ÷ inverter grid power — default 1.14; calibrate your own (see `docs/grid-import-limiter.md`) |
| `input_number.grid_charge_current_max` | Number (A) | Upper limit for the grid-charge current — default 60 |
| `input_boolean.grid_import_overload` | Toggle | Set by automation G while the house is held on battery |

Template sensors (each file in `helpers/` says how to create it):
`sensor.target_soc_tonight`, `sensor.energy_deficit`,
`sensor.grid_charge_current_target`, `sensor.grid_import_metered`.

Also create the script from [`scripts/set_mode_with_retry.yaml`](scripts/set_mode_with_retry.yaml)
with the id `inverter_set_mode_with_retry`.

### Entity ID mapping

All entity IDs in this repo use generic names. Map them to your actual entities:

| Placeholder | What it should be |
|-------------|------------------|
| `select.inverter_output_priority` | Your inverter's output-source-priority select. Options used in this repo: `AUTONOMOUS` (your inverter's "prioritize battery/solar over grid" mode, e.g. "PV-Battery-Utility"/"SBU") and `GRID_CHARGE` (your inverter's "prioritize grid for charging" mode, e.g. "PV-Utility-Battery"/"SUB") — substitute your integration's actual option strings |
| `select.inverter_charge_priority` | Your inverter's charge-source-priority select. Options used in this repo: `SOLAR_ONLY` (e.g. "PV Priority") and `SOLAR_AND_GRID` (e.g. "PV and Utility") — substitute your integration's actual option strings |
| `sensor.inverter_grid_voltage` | Grid voltage as seen by the inverter's input (after any protection relay) — the grid-presence signal |
| `sensor.battery_voltage` | Battery voltage (automation E's emergency branch, automation G's W-per-amp) |
| `sensor.inverter_grid_power` | Inverter's grid input power, W (automation G) |
| `sensor.inverter_load_power` | Inverter's output (household load) power, W (automation G) |
| `number.inverter_max_grid_charge_current` | Inverter's max AC/grid charge current setting, A (written by automation G) |
| `sensor.battery_soc_effective` | Your SOC sensor (must go `unavailable`, not stale) |
| `sensor.pv_forecast_today/tomorrow` | Solcast entities |
| `sensor.energy_production_today/tomorrow` | Open-Meteo Solar Forecast entities — PV array 1 |
| `sensor.energy_production_today/tomorrow_array2` | Open-Meteo Solar Forecast entities — PV array 2 (omit if you only have one array; see `docs/model.md`) |
| `sensor.avg_load_7d` | A 7-day rolling average daily load — used only by automation F as a "statistics helper has data" check |
| `notify.YOUR_TELEGRAM_BOT` | Your Telegram notify entity |

### Load profile

The `load_profile_by_season` dict in the sensor templates is built from one
specific household's data. **It will not match your house.** Export your own
hourly consumption averages from your energy monitoring tool and replace the
values before deploying, in both sensor templates.

---

## Changelog

- **v4.0 (2026-11)** — winter mode. Target horizon "until next 23:00" (peak of
  running balance) replaces "until two covered hours / 11:00"; forecast
  derate 0.8; Open-Meteo hour key built from a real datetime (fixes the
  `+03:00` bug that dropped Open-Meteo after the DST change);
  `energy_deficit` derived from `target_soc_tonight`; new automation G (grid
  import limiter, meter-calibrated budget, grid-charge current up to 60 A).
  Tuned for ~13.3 kWh; 2 × 16 kWh re-tune pending.
- **v3.1 (2026-11)** — repo synced with the running system: retry script,
  grid presence from inverter input voltage, E handles grid loss + 15-min
  tick, live target, P10 blend and load correction removed, known issues
  listed.
- **v3.0 (2026-08)** — inverter mode switching replaces breaker toggling.
- **v2.7 (2026-08)** — symmetric debounce fix after a relay-flapping incident;
  dual-array PV support.
- **v2.x (2026-07)** — hourly deficit model, dynamic recalculation, restart
  recovery.

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
