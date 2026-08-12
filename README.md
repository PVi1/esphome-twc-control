# TWC3 Charge Control (ESP32-S3-RS485-CAN → Tesla Wall Connector 3)

Local control of Tesla Wall Connector Gen 3 charging current over RS485, by
emulating a Neurio/Generac CT-clamp meter (TWC3's Home Load Management
input), driven by real per-phase current/power data mirrored in from Home
Assistant (originally from a Shelly Pro 3EM on the main incomer). A cloud
integration (Tesla Fleet API, Tessie, etc.) is only needed to start/stop
the charging session — power control itself never touches the cloud.

**Home Assistant is a hard dependency**: real-time power control runs over
ESPHome's native API connection to HA (both the real current/power data and
`charge_from_grid`), not a cloud service — if that connection or the
mirrored HA entities go stale, the firmware fails safe (0A available).

Verified in practice: with **live, correlated** data, TWC3 keeps responding
continuously throughout the whole charging session (not just at its start).

## How it works

```
Shelly Pro 3EM (main incomer, L1/L2/L3, includes the TWC branch)
   → HA (Shelly integration): your ha_current_a/b/c_entity + ha_power_a/b/c_entity (secrets.yaml)
   → ESP32 (sensor: platform: homeassistant, real-time push over the API connection)
   → globals shelly_a/b/c_current + shelly_a/b/c_power (signed, for flow direction)

HA input_boolean (entity ID set by ha_charge_from_grid_entity in secrets.yaml,
mirrored via ESPHome's homeassistant: binary_sensor)
   → switches between two modes:

  GRID mode (ON):  car gets the max safe current (BOTH breakers protected)
  FVE mode (OFF):  self-balancing loop toward zero grid exchange
                   (export -> car takes more, import -> car backs off)

ESP32 computation (see recompute_ct in twc-control.yaml):
  avail_mode = per-phase or per-mode availability (see below)
  avail      = clamp(min(avail_mode, twc_breaker_limit_a, main_breaker_limit_a - real), 0, ...)
  reported   = twc_breaker_limit_a - avail
   → RS485 Modbus RTU (registers 0xF4-0xFC)
   → TWC3 applies the limit to the car

Cloud API (Tessie/Fleet, etc.): start/stop session only, independent of power control
```

Two independent breakers protected at the same time, for every mode:
- `twc_breaker_limit_a` — TWC's own sub-circuit breaker / internal Home Load
  Management limit set in the Tesla installer menu (must match exactly!)
- `main_breaker_limit_a` — the main incomer breaker, measured by Shelly

`avail` is clamped on **both** sides: from the top against `twc_breaker_limit_a`
and `main_breaker_limit_a - real` (per-phase, always active regardless of
mode), and from the bottom at `0` — so `reported` always stays within
`[0, twc_breaker_limit_a]`, never a flat "0 available" for longer than
actually needed and never an out-of-range value either.

Fail-safe: if any of the 6 HA-mirrored current/power entities has been
reporting `unavailable`/`unknown` continuously for
`shelly_unavailable_debounce_ms` (default 10s), OR the HA API client isn't
connected, full consumption is reported on all phases → TWC3 gets 0A
available for the car.

Availability is driven by **HA's own reported state**, not a fixed
no-update timeout — a timeout can't tell "Shelly went offline" apart from
"the load just isn't changing" (HA/Shelly only push a new value when one
actually occurs), which caused false staleness during genuinely stable
load. ESPHome's `homeassistant` sensor publishes `NAN` whenever the
entity's HA state fails to parse as a number (i.e. exactly on
`unavailable`/`unknown`), which is what's checked. The debounce only
absorbs a brief HA-reported outage (e.g. Shelly on a flaky powerline/mesh
link) — HA gets a chance to reconnect on its own before the fail-safe
reacts. A periodic safety-net recompute (`recompute_interval`, independent
of any HA push) keeps the debounce timer itself re-evaluated even if HA
stops sending updates without dropping the API connection.

### GRID mode

`avail_mode` is simply `twc_breaker_limit_a` on all 3 phases — the car gets
the max current either breaker allows, ignoring the direction of household
flow.

### FVE mode — per-phase (default, aggregate metering OFF)

`avail_mode` reacts only to that phase's own export/import
(`shelly_x_power` sign): exporting → `floor(real_x)` (rounded **down** to a
whole amp, so surplus is never overshot into an actual import); importing →
`-real_x` (precise, no rounding — back off exactly as much as needed).

### FVE mode — aggregate/net balance metering (optional, per-switch)

Some utilities net import/export **across all 3 phases together** for
billing, instead of settling each phase separately. In that case, blocking
or under-using charging just because one phase individually imports (while
others export a lot more) is overly conservative — the `switch.*_aggregate_balance_metering`
entity (default OFF) fixes that.

**Design history / why the final algorithm looks the way it does** — TWC3
runs a live correlation check: it compares its own actual ramping current
against the reported value, and stops charging within seconds if they don't
track together (confirmed by direct testing, several iterations):

1. First attempt: report the **same phase-averaged** value on all 3
   registers. Broke correlation — while a phase ramps up alone (TWC3 engages
   L1 → L2 → L3 sequentially, not all at once), averaging diluted that
   phase's own signal to ~1/3 of its real slope. TWC3 saw the mismatch and
   stopped charging within seconds of starting.
2. Second attempt: keep each phase's own real current as the base (fixes
   correlation), rescue an importing phase only up to breakeven (`avail = 0`)
   using surplus borrowed from exporting phases. Correlation-safe, but since
   TWC3 applies `min(avail_a, avail_b, avail_c)` as the actual charging
   current, a single weak/importing phase capped at breakeven still
   bottlenecked the whole session to ~0A even with a large net export.
3. Third attempt: **water-filling** — raise the weakest phase(s) all the way
   toward the strongest, to maximize the minimum. This turned out to be an
   actual bug, not just an over-optimization: since TWC3 applies
   `I = min(avail_a, avail_b, avail_c)` as the *same* current on all 3
   phases, total car power is `3*I`. Leaving every phase's own value
   untouched while *also* spending the whole surplus pool again to raise the
   minimum double-counted it — e.g. a real 23A/~5.3kW pool got water-filled
   to `min(avail) = 15.33A`, implying `3*15.33 ≈ 46A`/~10.6kW of charging
   power from surplus that didn't exist.
4. Fourth: **`avail_mode_x = max(strict_x, pool/3)`**. For the whole house to
   stay net-export/zero in aggregate, `3*I <= pool` must hold, i.e.
   `I <= pool/3` — that's the exact, no-more-no-less ceiling. A phase
   already above `pool/3` is left untouched (1:1 correlation, no dilution);
   a phase below it is raised exactly up to `pool/3` (never higher). The
   resulting `min(avail)` lands exactly on `pool/3`, so the real surplus is
   used in full with no double-counting and no correlation loss for
   whichever phase doesn't need help.

The per-phase main-breaker safety check (`main_breaker_limit_a - real`)
still applies underneath all 3 variants, unconditionally.

**Self-balancing loop gain (`self_balance_gain`)** — the formula above still
feeds the car's own current straight back into its own availability
calculation with full 1:1 weight, which is a textbook marginally-stable
discrete feedback loop (multiplier -1): with a constant household load and
constant PV output, this produces a *sustained bang-bang oscillation*
instead of converging (confirmed live — reported cycling ~13-16A while TWC3
hunted between ~8-12A, indefinitely).

- A temporal fix (EMA/low-pass smoothing of `reported`) was tried and
  reverted: any such filter necessarily lags the reported value behind
  TWC3's actual current, and TWC3's live correlation check distrusts that
  mismatch and stops charging (confirmed live — charging stopped/restarted
  exactly when a smoothed value was still catching up to an already-changed
  real current).
- Fix instead: a plain multiplicative gain `k` (`self_balance_gain`,
  currently `0.65`), recomputed fresh every cycle from the *current* real
  reading only — no history, no lag, so it still moves in lockstep with
  TWC3's own current changes (same cycle), just with reduced slope.
  Iterating `I_(n+1) = k*E - k*I_n` has multiplier `-k` instead of `-1`,
  which converges for `k < 1`.
- Trade-off: the equilibrium settles at a fraction of the theoretical max
  surplus usage, not the full amount — confirmed live, a few hundred W of
  real surplus goes unused at steady state. This is the price of stability
  without a full PID controller.
- Gain has a floor: TWC3's own minimum charge current is 5A, and too low a
  `k` shrinks the aggregate `pool/3` target (which absorbs the same gain as
  every `strict_x` feeding into it) below that floor — confirmed live,
  `k=0.5` produced a target of 3.5A in a real scenario that would've
  gotten 7A ungained, and charging never started at all. `k=0.65` keeps
  enough headroom above 5A in the same class of scenario while still
  meaningfully damping the oscillation.

**Escalation (`escalation_timeout_ms`, all modes):** TWC3 was observed to
ramp down very slowly toward a sustained `reported == twc_breaker_limit_a`
(0A available) on all 3 phases, and could keep drawing a small residual
current indefinitely despite continued import instead of stopping outright.
If all 3 phases stay pinned at the full breaker limit for
`escalation_timeout_ms` straight, `reported` is nudged 0.1A past the limit
(`twc_breaker_limit_a + 0.1`) to force a hard stop.

Confirmed live: the gain-damped self-balancing loop still briefly touches
the `reported == twc_breaker_limit_a` ceiling during a session's *startup*
transient (before it settles) — the original 30s threshold was short enough
for that transient alone to trigger this escalation, forcing a hard stop
right after every session start, which then restarted, repeating
indefinitely. Raised to 120s, which gives the loop enough time to settle
past the startup transient before escalation can trigger.

### Disabling external control (`switch.*_twc_control_enabled`)

Master switch for the whole loop, default **ON**. When turned OFF,
`recompute_ct` reports a constant `0A` (max availability) on all 3 phases
instead of computing anything live. TWC3's own live correlation check then
distrusts that static value within seconds of a session starting and falls
back to its own internal ceiling — i.e. it ends up ignoring this firmware
entirely and charges at whatever it decides on its own (e.g. the Tesla
app's own current slider). Useful for temporarily handing a session back to
TWC3's stock behavior without touching the RS485 wiring or reflashing.

## Before the first flash

1. Copy `secrets.yaml.example` → `secrets.yaml` (it's in `.gitignore`, never
   pushed) and fill in:
   - WiFi SSID/password, fallback AP password
   - API encryption key (`python3 -c "import os,base64;print(base64.b64encode(os.urandom(32)).decode())"`)
   - OTA password
   - `twc_breaker_limit_a` — TWC's own sub-circuit breaker / internal Home
     Load Management limit, set in the Tesla installer menu for TWC3 —
     **must match exactly**, otherwise the limit will be offset
   - `main_breaker_limit_a` — your main incomer breaker (measured by the
     Shelly Pro 3EM). Can be higher than `twc_breaker_limit_a` — the car is
     limited by whichever of the two is lower
   - `ha_current_a_entity` / `ha_current_b_entity` / `ha_current_c_entity` —
     your Shelly Pro 3EM's per-phase current entities in HA (Settings →
     Devices & Services → Shelly → the device's entities, or Developer
     Tools → States to find the exact IDs)
   - `ha_power_a_entity` / `ha_power_b_entity` / `ha_power_c_entity` — the
     matching per-phase **signed** active power entities (negative = export,
     positive = import)
   - `ha_charge_from_grid_entity` — the `input_boolean` helper you create in
     step 4 below (defaults to `input_boolean.charge_from_grid`)
2. The Shelly CT clamps must be on the **main incomer** (measuring the TWC
   branch too) — otherwise the formula in `recompute_ct` doesn't hold.
3. In Home Assistant, confirm the 6 entities from step 1 exist and update
   with the Shelly's own refresh cadence — no further config needed, the
   firmware reads them directly by entity ID.
4. In Home Assistant, create the `input_boolean` helper referenced by
   `ha_charge_from_grid_entity` (Helpers → Toggle). If it doesn't exist, the
   firmware safely falls back to GRID mode.
5. If your utility nets import/export across all 3 phases together for
   billing, enable `switch.*_aggregate_balance_metering` (default OFF) once
   FVE mode is confirmed working in plain per-phase mode first.

## Physical wiring

- RS485 A/B from the Waveshare board to TWC3's internal RS485 terminals
  (inside the enclosure, near mains voltage → have an electrician handle it
  / turn off the breaker before opening it).
- Board pins: TX=GPIO17, RX=GPIO18, DE/RE (flow control)=GPIO21.
- If TWC3 is at the end of the RS485 bus (typically point-to-point), enable
  the 120Ω termination jumper on the board.
- Power the ESP32 independently (USB-C or 7-36V DC terminal) — the RS485
  side is galvanically isolated from TWC3.

## Build & flash

```bash
python3 -m venv venv
./venv/bin/pip install esphome
./venv/bin/esphome run twc-control.yaml   # first time over USB, then OTA (--device <host>.local)
```

## Diagnostic entities in HA

- `sensor.*_twc_poll_interval`, `*_twc_time_since_last_poll`,
  `binary_sensor.*_twc_polling_active` — tracks TWC3's real 0xF4 Modbus polls.
- `sensor.*_shelly_real_current_l1/l2/l3`, `*_shelly_active_power_l1/l2/l3` —
  real measured data mirrored in from HA (signed power: − export, + import).
- `sensor.*_twc_reported_current_l1/l2/l3` — what's currently being reported to TWC3.
- `binary_sensor.*_shelly_data_fresh` — is the mirrored real-current/power
  data available (HA-reported, debounced — see Fail-safe above).
- `binary_sensor.*_twc_ha_link_ok` — is the HA API connection active.
- `binary_sensor.*_charge_from_grid` — currently mirrored mode from HA.
- `switch.*_aggregate_balance_metering` — **default OFF**, FVE-mode-only, see
  "aggregate/net balance metering" above.
- `switch.*_twc_control_enabled` — **default ON**, see "Disabling external
  control" above.

## Known limitations / behavior

- **Home Assistant is a hard dependency for power control**, not just
  start/stop — if the HA API connection or the 6 mirrored real-current/power
  entities go stale, the firmware fails safe (0A available) rather than
  continuing to charge on old data. This trades away the earlier
  direct-HTTP-poll design's partial independence from HA in exchange for
  avoiding Shelly Gen2's brute-force login protection entirely (no local
  auth to manage at all).
- TWC3 applies **a single shared current to all phases** (not per-phase) and
  engages them sequentially (L1 → L2 → L3) when ramping up — this is exactly
  why the aggregate-metering algorithm had to be redesigned around
  correlation-safe, additive-only credit capped at `pool/3` (see above)
  instead of a naive phase average or an uncapped water-fill.
- FVE mode (both per-phase and aggregate) is a bang-bang controller (not
  PID) — near `signed≈0` (exactly balanced) it may pulse slightly around
  zero grid exchange.
- `self_balance_gain < 1` (see "Self-balancing loop gain" above) trades
  full surplus utilization for stability: confirmed live, a few hundred W
  of real export goes unused at steady state, and a real load transient
  (e.g. an appliance switching on) can still cause brief, self-recovering
  oscillation before the loop re-settles.
- The register map (identification block, Neurio meter MAC/model/serial
  number) is a fixed placeholder taken from the reverse-engineered project
  linked below — it's not real data from any physical device.

## Sources

- https://gist.github.com/LucaTNT/4adf01a7252386559070023612efa117 (register mapping)
- https://github.com/Klangen82/tesla-wall-connector-control
- https://community.home-assistant.io/t/tesla-wall-connector-gen-3-via-esphome-rs485-dynamic-current-control-no-wifi/985613
- https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/EM (EM.GetStatus RPC)
