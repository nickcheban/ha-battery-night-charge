# Phantom voltage: the Zigbee breaker problem

## What happens

Cheap Zigbee/Tuya smart breakers (the kind used as a grid input switch in
solar systems) power their radio chip from the same line they switch. The chip
draws a small continuous current through the measurement circuit even when the
relay contact is open.

When a voltage protection relay upstream cuts the grid:

1. The breaker loses its power source
2. Before the chip fully dies, it drains residual charge from its measurement
   circuit capacitors
3. During this drain (~seconds to minutes), it transmits garbage voltage
   readings to Home Assistant — typically 10–175 V, fluctuating, unstable
4. The chip then goes fully dead — but Home Assistant may not register this as
   `unavailable` if the last-seen timeout hasn't elapsed yet

The result: Home Assistant believes the breaker is alive and reporting ~50–100 V,
when in reality it is physically de-energised and unable to execute any command.

## What goes wrong without mitigation

**At 23:05:** Automation A fires and sends `switch.turn_on`. The command goes
to a dead chip. Nothing happens. Home Assistant reports the breaker as `on`
(because that was the last state it received). The system believes charging
is in progress. No charging actually occurs.

**Mid-night:** The grid comes back. The breaker chip wakes up and restores its
last saved state (`restore previous state` firmware option). If the last state
was `off`, the breaker stays off — the recovery automation (E) must detect
this and re-enable it.

**At 06:58:** Automation D fires and sends `switch.turn_off`. If the breaker
is still dead (grid still absent), the command is ignored. When the grid
eventually returns, the breaker may come back as `on` (last saved state was
`on` from the 23:05 command). The system now has a live breaker pulling from
the grid during daytime peak tariff — with Home Assistant believing it is `off`.

## Evidence from 30 days of recorded data

Analysis of `sensor.breaker_voltage` history across 30 days revealed:

- Phantom readings ranged from **10 V to 175 V**
- Phantom readings were **unstable** — values changed every 10–30 seconds
- Phantom readings **never exceeded 175 V**
- Phantom readings **never held a stable value for more than a few minutes**
- Real grid voltage consistently read **225–260 V**
- Real grid voltage was **stable** across multiple consecutive readings

This gives a clear separation: anything below ~200 V or fluctuating rapidly
is phantom. Anything above 210 V held for 2 minutes is real.

## The 210 V / 2-minute rule

All automations in this package that turn the breaker **on** require:

```yaml
- condition: numeric_state
  entity_id: sensor.breaker_voltage
  above: 210
  alias: "Real grid present (>210V, not phantom)"
```

Automation E (grid recovery) uses a trigger with `for: 2 minutes`:

```yaml
triggers:
  - trigger: numeric_state
    entity_id: sensor.breaker_voltage
    above: 210
    for:
      minutes: 2
```

The 2-minute hold prevents reacting to a phantom spike that briefly crosses
210 V — which was not observed in 30 days of data, but is a theoretical risk
given the unstable nature of phantom readings.

## What this does not fix

The phantom voltage problem is rooted in the hardware architecture of
these breakers. No firmware update can make a chip that has lost its power
source report a correct state.

The 210 V / 2-minute rule is a detection heuristic, not a hardware fix.
It correctly handles the most common scenarios but has limits:

- If the grid returns and immediately has a voltage spike that trips the
  protection relay again within 2 minutes, the recovery automation will not
  fire (the 2-minute window never completes). This is the correct behaviour —
  better to miss one charge cycle than to command a breaker into an unstable
  grid.
- If a phantom reading somehow sustained itself above 210 V for 2 minutes
  (not observed in data, but theoretically possible on different hardware),
  an automation could attempt to turn on the breaker against a dead chip.
  The command would simply be ignored.

## Long-term hardware recommendation

A breaker whose radio chip is powered from a separate supply (not from the
switched line) would not exhibit this problem. The chip would remain alive
and reachable throughout grid outages, and would correctly report its relay
state and voltage at all times.

Shelly Pro DIN-rail relays are one example of a device with separate control
and load power inputs — though they lack built-in voltage measurement,
requiring an external sensor for the 210 V check logic.
