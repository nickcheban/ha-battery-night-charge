# Case study: diagnosing and fixing a breaker-flapping incident

This documents a real production bug found the morning after a hardware
upgrade (inverter and PV array change — see main README), how it was
diagnosed from Home Assistant's own history and automation trace data, and
the fix that shipped as automation C v2.7.

## Symptom

Overnight, `switch.grid_switch` cycled on/off **5 times**, including one
on→off flap only **3 seconds** apart. Expected behavior is at most one or
two clean transitions per night (charge on, charge off once target is
reached).

## Diagnosis

**Step 1 — reconstruct the timeline.** Pulling significant-change history
for the breaker state, `sensor.energy_deficit`, the breaker voltage sensor,
and the BMS SOC sensor over the incident window made the full on/off
sequence visible, each event timestamped to the second.

**Step 2 — try to attribute each event to a specific automation via
traces.** This did not fully work: Home Assistant's automation trace
retention is shallow (the automations involved had already accumulated
newer traces from later, higher-frequency triggers by the time the
investigation happened — automation B alone re-evaluates on every SOC
tick). Traces from several hours earlier had already rotated out. **Lesson:
if you want to diagnose a flapping incident after the fact, pull traces
immediately, not the next day** — or increase `trace: stored_traces` for
the automations in question.

**Step 3 — reason from config + partial evidence.** Cross-referencing what
trace data *was* available against the automation configs themselves
surfaced two concrete, confirmable-from-source-code issues (see Root cause
below) — no further live evidence was needed to confirm these, since they
were structural, not timing-dependent.

**Step 4 — rule out the hardware.** For the specific 3-second flap, breaker
voltage was stable (~250 V, not phantom-voltage range) and the deficit
sensor had been flat at 0.0 for hours before and after — meaning no
automation *should* have acted at that moment on the documented logic. This
particular event is most likely a Zigbee/Tuya relay-level bounce (the same
relay had a brief `unavailable` blip earlier the same night), not an
automation logic bug. Not everything that looks like an automation bug is
one — cheap relays bounce on their own.

## Root cause (automation C)

Two structural gaps, both in the "emergency correction" automation (the one
most likely to react to noisy intermediate sensor readings, by design):

1. **Asymmetric debounce.** The turn-on trigger (`need_charge`, deficit >
   0.7 kWh) had a `for: 2 minutes` settle window. The turn-off trigger
   (`stop_charge`, deficit < 0.3 kWh) had none — a single instantaneous dip
   below the threshold fired it immediately.
2. **Asymmetric minimum-runtime guard.** A 30-minute "don't turn off unless
   it's been on this long" guard existed, but only on the turn-off branch.
   Nothing prevented the breaker from being turned back on again within
   seconds of turning off.

Both gaps point the same direction: hysteresis and debounce protections had
only ever been applied to the turn-off path, not symmetrically to both
directions. A recently larger inverter (see README) likely made this worse
in practice — a higher charge rate moves the SOC-driven deficit sensor
across the 0.7→0.3 kWh hysteresis band faster, giving the asymmetric guards
more opportunities to matter.

## Fix (v2.7)

- `for: 2 minutes` added to the `stop_charge` trigger, matching
  `need_charge`.
- The 30-minute minimum-runtime guard now applies to **both** branches
  (breaker must have been off ≥30 min before turning on, and on ≥30 min
  before turning off), and is implemented as a native `for:` on a `state`
  condition rather than template `now() - last_changed` duration math — the
  native form is schema-validated at config load; the template form fails
  silently at runtime if malformed.

See `automations/automation_c_emergency_correction.yaml` for the applied
diff and inline rationale.

**Explicit trade-off:** during the 30-minute cooldown after any flip, C will
not react to a fresh deficit spike or drop. This is accepted because
automation A (the primary 23:05 decision) and automation E's branch 4
(emergency turn-on below 49 V battery voltage, no time restriction) are not
gated by this cooldown and remain available for genuinely urgent cases.

## What the general pattern is

This is a textbook short-cycling problem, and the fix follows the two most
common community-recommended techniques for it:

- **Debounce (`for:` on a trigger)** — the standard Home Assistant
  mechanism for ignoring a signal until it's been stable for a minimum
  time. Apply it to *both* sides of any hysteresis band, not just one.
- **Minimum cycle duration** — the same protection HA's own
  `generic_thermostat` integration provides natively (`min_cycle_duration`)
  to stop a relay-driven compressor from short-cycling. There's no
  equivalent built-in helper for a bare `switch` automation, so it's
  implemented here as a native `for:` state condition — the schema-
  validated equivalent of the manual `last_changed` template math this
  project used before v2.7.

A [Threshold helper](https://www.home-assistant.io/integrations/threshold/)
is the more declarative way to get symmetric hysteresis without hand-rolling
two triggers with separate on/off values — worth considering if you're
building this from scratch rather than patching an existing automation.

## Postscript (v3.0): the actuator changed, not just the debounce

This case study describes the breaker-toggling architecture (v1.x–v2.7).
Since then, the package moved to switching the inverter's own
output/charge-priority mode instead of a physical breaker — see
[`docs/model.md#inverter-mode-switching`](model.md#inverter-mode-switching).
The v2.7 debounce and minimum-runtime-guard logic described above is still
exactly what protects the (now different) actuator from short-cycling; only
the target of `select.select_option` vs. `switch.turn_on/off` changed. This
case study is kept as-is because the diagnosis method and the general
short-cycling lesson are unchanged and still the most useful part of this
document — the specific mechanical-relay-wear risk it originally motivated
is what the v3.0 architecture change addresses at the root.

## Sources

- [Home Assistant Community: Bouncing switch](https://community.home-assistant.io/t/bouncing-switch/77119)
- [Home Assistant Community: Do not trig automation too often ("debounce")](https://community.home-assistant.io/t/do-not-trig-automation-too-often-debounce/101462)
- [Home Assistant Community: Possible hysteresis in On/Off Automation Option](https://community.home-assistant.io/t/possible-hysteresis-in-on-off-automation-option/793678)
- [Threshold integration docs](https://www.home-assistant.io/integrations/threshold/)
- [Generic Thermostat integration docs](https://www.home-assistant.io/integrations/generic_thermostat/) (`min_cycle_duration`)
