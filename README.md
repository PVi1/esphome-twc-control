# TWC3 Charge Control (ESP32-S3-RS485-CAN → Tesla Wall Connector 3)

Local control of Tesla Wall Connector Gen 3 charging current over RS485, by
emulating a Neurio/Generac CT-clamp meter (TWC3's Home Load Management
input), driven by real per-phase current/power data mirrored in from Home
Assistant (originally from a Shelly Pro 3EM on the main incomer). **No
cloud integration at all** — session start/stop isn't driven by this
project or any cloud API (Tesla Fleet API, Tessie, etc.); the car starts
charging on its own the moment it's plugged in or asked to via the Tesla
app, and stops/restarts itself in response to the current this firmware
publishes (`reported`, see "Publication law" below) — same as it would
against any other Home Load Management meter.

**Home Assistant is a hard dependency**: real-time power control runs over
ESPHome's native API connection to HA (both the real current/power data and
`charge_from_grid`), not a cloud service — if that connection or the
mirrored HA entities go stale, the firmware fails safe (0A available).

Verified in practice: with **live, correlated** data, TWC3 keeps responding
continuously throughout the whole charging session (not just at its start).

📊 **[Full decision tree](docs/decision-tree.md)** — a diagrammed,
section-by-section walkthrough of `recompute_ct`, including the
zone-steering algorithm and the measured TWC3 reaction-curve findings.

## How it works

```
Shelly Pro 3EM (main incomer, L1/L2/L3, includes the TWC branch)
   → HA (Shelly integration): your ha_current_a/b/c_entity + ha_power_a/b/c_entity (secrets.yaml)
   → ESP32 (sensor: platform: homeassistant, real-time push over the API connection)
   → globals shelly_a/b/c_current + shelly_a/b/c_power (signed, for flow direction)

TWC3's own /api/1/vitals (twc_vitals_ip, polled directly, diagnostic AND
   R1-floor/household-split input -- see "Publication law" below)

HA input_boolean (entity ID set by ha_charge_from_grid_entity in secrets.yaml,
mirrored via ESPHome's homeassistant: binary_sensor)
   → switches between two modes:

  GRID mode (ON):  car gets the max safe current (BOTH breakers protected)
  FVE mode (OFF):  self-balancing loop toward zero grid exchange
                   (export -> car takes more, import -> car backs off)

ESP32 computation (see recompute_ct in twc-control.yaml) -- a "publication
law", not a proportional availability calculation, see below for why:
  desired_avail = self-balancing target (household-only, car-free signal)
  worst         = max(signed_a, signed_b, signed_c)  (car-INCLUSIVE)
  o_raw         = worst + (twc_breaker_limit_a - desired_avail)
  reported      = o_raw <= limit ? o_raw
                    : limit + clamp(excess_gain*(o_raw-limit), law_nudge_min_a, excess_max_a)
   → RS485 Modbus RTU (registers 0xF4-0xFC), published SYMMETRICALLY on all 3
   → TWC3 applies the limit to the car

Session start/stop: entirely TWC3's own doing, not this project's — begins
   when the car is plugged in or told to charge (Tesla app or otherwise),
   ends/restarts on its own based on `reported` (see "TWC3 reaction curve"
   in the decision tree above)
```

### Publication law (why this isn't a simple availability formula)

TWC3 firmware **26.26.1** does **not** proportionally track `reported` below
its own configured breaker limit at all — confirmed live, repeatedly: it
just charges toward its own internal ceiling regardless of what's reported,
as long as that value stays under the limit. It only reacts once `reported`
crosses the limit by a margin — confirmed live, consistently **~1.1-1.2A**
above `twc_breaker_limit_a`, across independent trials and two different HA
data sources. This ruled out the entire earlier "compute a precise
available current" family of designs (see the numbered history further
below, kept for context) — none of that fine-grained math has any effect
on TWC3's actual behavior on this firmware version.

The current design is adapted from an independently-developed, live-
validated open-source solution to this identical TWC3 behavior:
[zany92/tesla-loadpilot](https://github.com/zany92/tesla-loadpilot)
(verified against its actual source, `esphome/packages/twc-core.yaml`, not
just its docs — an earlier pass here mis-implemented a "decaying tail" its
docs described that doesn't actually exist in the real code, and that bug
is described further below as an example of why this was verified against
source).

- **Below the limit**: `reported` tracks the **worst-phase, car-inclusive**
  real current 1:1 (`worst`) — no gain, no damping. TWC3 ignores the exact
  value here for control purposes, but still needs to see it move in
  lockstep with its own ramping current for its live plausibility check;
  diluting that slope (tried and reverted, see history) risks a distrust
  latch instead.
- **`desired_avail`**: the self-balancing *target* — computed from the
  **household-only** signal (TWC3's own vitals API gives the car's OWN
  per-phase current directly, subtracted out of Shelly's combined reading,
  eliminating the self-referential feedback loop at its source instead of
  damping it) — and **asymmetrically slew-rate-limited**
  (`desired_avail_slew_down_a_per_s`, 1A/s; `desired_avail_slew_up_a_per_s`,
  10A/s — recovering availability is treated as protective/instant, only
  declines are throttled). This mirrors the reference project's own architecture: its
  "budget" is a slow/external quantity, kept separate from the fast
  `worst` term used for correlation. Confirmed live: without this slew
  limit, Shelly and TWC3's own vitals API — two independently-polled
  sources — can briefly disagree by several amps during a fast multi-phase
  engagement (one lagging the other's newly-engaged-phase reading),
  spiking `reported` right at the critical startup window.
- **Above the limit**: the excess is compressed (`excess_gain`, default
  0.75) and capped (`excess_max_a`, default 1.2A — kept close to, not
  below, the confirmed reaction margin) instead of climbing further.
- **R1 hard floor**: with the contactor closed, every phase carries at
  least the vehicle's own measured current (from vitals) — applied to the
  *input* signal, not patched onto the output afterward (per the reference
  project). A lower reading is physically impossible and would latch a
  distrust state.
- **Anti-glitch firewall** (asymmetric, per the reference project): a rise
  in real current (or a gentle drop) is trusted immediately; a **sudden**
  drop (`glitch_drop_a`, default 3A) is held at the last trusted value
  until 2 consecutive samples agree on a new value (within
  `glitch_confirm_tol_a`) — confirmed live, a single stale/desynced Shelly
  sample dropped >4A in <10ms right during a fast multi-phase engagement
  (physically impossible for a real ramp).
- **Dither** (`dither_amplitude_a`, ±0.05A alternating at 1Hz, always on
  including fail-safe) — `reported` is never perfectly static for long.
- **Symmetric publication**: the final value is published identically on
  all 3 Modbus registers — TWC3's own service loop expects
  `min == mean == max` to correctly engage at the true constraint,
  regardless of which phase is actually weakest.
- **Escalation** (secondary safety net, 2-stage step per the reference
  project, not a continuous ramp): if `o_raw` stays above the limit
  continuously for `escalation_timeout_ms` (default 120s), force `reported`
  to at least `law_nudge_min_a` past the limit; if it's *still* there after
  `2×escalation_timeout_ms`, force it to `escalation_kick_a` (default
  1.5A) — safely past the confirmed reaction margin.

Both breakers (`twc_breaker_limit_a` and `main_breaker_limit_a`) still
clamp `desired_avail` before any of the above — main-breaker protection is
applied **after** slewing, so it's never delayed, regardless of mode.

**GRID mode**: `desired_avail = twc_breaker_limit_a` (max allowed, subject
to the same breaker clamps).

**FVE mode**: `desired_avail` is the self-balancing target — averaged
across the household-only signal on all 3 phases (aggregate/net billing,
`switch.*_aggregate_balance_metering`) or the weakest phase (per-phase
billing, default), plus the manual `fve_offset_kw`
(`number.*_fve_offset`, runtime-adjustable in HA).

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

`desired_avail = twc_breaker_limit_a` (the max either breaker allows),
subject to the same publication law and breaker clamps described above.

### FVE mode — per-phase (default, aggregate metering OFF)

`desired_avail` tracks the **weakest phase's household-only** signal
(`shelly_x_power` sign, car's own draw subtracted out via the TWC vitals
API): exporting → back off less / take more; importing → back off.

### FVE mode — aggregate/net balance metering (optional, per-switch)

Some utilities net import/export **across all 3 phases together** for
billing, instead of settling each phase separately. In that case, blocking
or under-using charging just because one phase individually imports (while
others export a lot more) is overly conservative — the
`switch.*_aggregate_balance_metering` entity (default OFF) switches
`desired_avail` from the weakest phase to the **average** of the
household-only signal across all 3 phases instead.

Either way, the per-phase main-breaker safety check
(`main_breaker_limit_a - real`) still applies unconditionally, after
slewing, so it's never delayed.

### TWC3 reaction curve (threshold-probe findings, `twc_breaker_limit_a`=20A)

Live-tested by publishing a fixed, manually-stepped `reported` value (a
dedicated diagnostic test mode, `test/threshold-probe` branch — not part
of production) and watching `twc_vitals_vehicle_current` for a reaction.
The reaction is **not a sharp on/off threshold** — it behaves like a
time-integrated/cumulative response: every tested value above the limit
*eventually* causes a reduction given enough time, there is no value that
holds forever, but how fast it reacts (and how far it goes before
plateauing) depends heavily on how far past the limit it is:

| excess over `twc_breaker_limit_a` | observed reaction |
|---|---|
| +0.9A | first visible reduction only after tens of seconds; can plateau for a while (e.g. 13.1A→10A over ~175s, then flat) but is **not safe indefinitely** — held long enough it resumes declining (10A→11A→10A over further ~80-175s in separate trials) |
| +1.0 / +1.05A | reduction starts after ~20s in an isolated, static test, then a gradual decline — but this ~20s figure did **not** generalize to real/dynamic conditions: one live stop happened after only ~4s at flat +1.0A when the car's actual current was already low/near the floor (less headroom to brake before hitting it), and another sustained ~8 minutes of continuous +0.9-1.0A holding (actual stabilized just above the floor, a genuine small deficit) before TWC3 aborted anyway |
| +1.1A and above (1.2/1.5/2.0A tested) | fast/cascading reduction within seconds, full stop in ~3s; reaction speed **saturates** around +1.1A — going higher doesn't react meaningfully faster |

Recovery from a cascade requires fully exiting to the natural/low
published value — nudging the published value down slightly mid-cascade
does not reliably halt it once started.

**Practical implication**: there's no single excess value that's both
"fast enough to react to a real deficit" and "gentle enough to never
overshoot into a stop" — a continuous law has to pick one operating
point and live with its trade-off. This is what motivated **zone
steering** below: instead of one value, react with an explicit sequence
of increasingly firm bands, and escalate over time only if the gentler
ones aren't working.

### Zone steering (graceful degradation for transient deficits)

The classic publication law above (a single, continuous excess value)
reacts to a sudden production drop (a cloud) either too weakly — sitting
in a real deficit for minutes before `escalation_timeout_ms` kicks in —
or, if tuned more aggressively, too strongly, cascading into a full stop
within seconds per the reaction curve above. Zone steering is an
alternative response, active only once the classic law's own
computation would already publish at or above `twc_breaker_limit_a`
while charging: instead of one excess value, it publishes one of 4
discrete bands, chosen by comparing the vehicle's own **actual** current
(`twc_vitals_vehicle_current`) against `desired_avail` (the target):

- **INCREASE** (`actual < target - tolerance`, and past a post-brake
  recovery cooldown): publish `twc_breaker_limit_a - 1.0A`, i.e. let the
  classic law's own below-limit tracking take back over.
- **HOLD** (small deadband around target): publish
  `twc_breaker_limit_a + 0.2A` — inside TWC3's ignored range, no
  reaction, but keeps `reported` moving (correlation-safe) without
  drifting toward a stop.
- **SLOW / DESCEND_TO_FLOOR** (`actual` moderately over target): publish
  `twc_breaker_limit_a + 0.9A` (confirmed-gentle per the reaction curve
  above; `+0.9A` when `actual` is already below
  `number.*_zone_steering_low_current_threshold_a`, `+1.0A` otherwise) —
  deliberately below the fast-cascade zone, giving the car time to ease
  down instead of dropping out.
- **HARD** (`actual` far over target, or the *stuck-timeout* below has
  fired): publish `twc_breaker_limit_a + 1.1A` — the confirmed-fast,
  reserved-for-a-real-persistent-excess band.

**Auto-engage / handoff (latch)**: engages the instant the classic
computation would publish `>= twc_breaker_limit_a` while charging, and
stays engaged through normal fluctuation. The moment `actual` reaches
`number.*_zone_steering_floor_a` inside the HARD band, it disengages
immediately and hands control back to the classic algorithm's own
grace-period mechanism (rather than maintaining a separate floor-hold
state with its own timer) — confirmed live that holding a separate
floor-hold state let `actual` stabilize just above the floor (a real,
small, sustained deficit) for 8+ minutes without ever escalating or
handing off, and TWC3 eventually aborted anyway. Disengages fully on
`!car_charging`, re-engaging fresh on the next threshold hit.

**Stuck-timeout escalation** (`number.*_zone_steering_stuck_timeout_s`,
default 60s, 5-900s range): if `actual` shows no real progress (no
genuine >0.3A decrease from its baseline) while braking is needed for
this long, SLOW/DESCEND_TO_FLOOR escalate to the HARD excess (+1.1A)
instead of holding the weaker brake value indefinitely — added directly
in response to the 8-minute stuck case above, for when `actual` never
quite reaches the floor at all.

**Minimum-dwell gate** (`number.*_zone_steering_min_dwell_s`, default
3s): confirmed live, `desired_avail`'s fast recovery slew combined with
a tight tolerance could make the target jitter across a band boundary
several times per second, and every such flip reset TWC3's own internal
"sustained excess" timer — so a correction never ran long enough
uninterrupted to have any effect. The gate holds the last **published**
value sticky for at least this long before accepting an escalation to a
firmer band. Asymmetric: a decrease (easing off) is always accepted
instantly, matching the "rises trusted immediately, only firming up
needs caution" pattern used throughout this design (anti-glitch
firewall, R1 floor, the release logic above) — only an *increase* in
severity is dwell-gated.

**Recovery-hold cooldown** (`number.*_zone_steering_recovery_hold_s`,
default 10s): blocks INCREASE for this long after any braking cycle,
falling through to HOLD instead — confirmed live, without it a brief dip
of `actual` just under `target` right after a correction triggered an
immediate ramp back up, undoing the correction it had just made.

Entities: `switch.*_zone_steering_mode` (default **ON**, enables the
whole mechanism), `binary_sensor.*_zone_steering_engaged` (is it
actually steering right now vs. the classic algorithm running
normally), `number.*_zone_steering_tolerance_a` (0.5A default),
`number.*_zone_steering_hard_excess_a` (3.0A default — deadband before
the HARD band, distinct from the fixed +1.1A HARD excess value itself),
`number.*_zone_steering_floor_a` (6.5A default — where steering hands
off), `number.*_zone_steering_low_current_threshold_a` (8.0A default)
and `number.*_zone_steering_slow_brake_excess_a` (0.9A default — the
gentler SLOW excess used below that threshold), `number.*_zone_steering_recovery_hold_s`,
`number.*_zone_steering_min_dwell_s`, `number.*_zone_steering_stuck_timeout_s`.

**Design history — earlier algorithm generations, kept for context.** The
generations below all predate the discovery that TWC3 FW 26.26.1 doesn't
proportionally track `reported` below its own breaker limit at all — they
were solving a real problem (TWC3's live *correlation* check, which is
still real and still relevant) with a fine-grained "compute exact available
current" model that turned out not to matter for actual control on this
firmware. Kept here because the correlation-safety lessons (1:1 slope
tracking, no dilution, no smoothing/EMA) still apply directly to the
current design's `worst` term:

1. First attempt: report the **same phase-averaged** value on all 3
   registers. Broke correlation — while a phase ramps up alone (TWC3 engages
   L1 → L2 → L3 sequentially, not all at once), averaging diluted that
   phase's own signal to ~1/3 of its real slope. TWC3 saw the mismatch and
   stopped charging within seconds of starting.
2. Second attempt: keep each phase's own real current as the base (fixes
   correlation), rescue an importing phase only up to breakeven using
   surplus borrowed from exporting phases. Correlation-safe, but since TWC3
   applies `min(avail_a, avail_b, avail_c)`-equivalent logic, a single
   weak/importing phase capped at breakeven still bottlenecked the whole
   session to ~0A even with a large net export.
3. Third attempt: **water-filling** — raise the weakest phase(s) all the way
   toward the strongest. Turned out to be an actual double-counting bug
   (spending the same surplus pool twice), not just an over-optimization.
4. Fourth: `avail_mode_x = max(strict_x, pool/3)` — a correlation-safe,
   no-double-counting ceiling. Solved the availability-math problem
   correctly, but was superseded once the threshold-ignoring firmware
   behavior was confirmed and made the whole availability-math approach
   moot (see "Publication law" above).
5. A gain-damped self-balancing loop (`self_balance_gain`) and an EMA/
   low-pass smoothing attempt were both tried and reverted for this same
   underlying reason — see "Publication law" above for the design that
   replaced them (household-only signal + slew-rate limiter, eliminating
   the feedback loop and the correlation-vs-lag trade-off at the source
   instead of damping it).

**Escalation (`escalation_timeout_ms`)** was originally a continuous ramp
(`+0.1A` every 5s while `reported` sat at the limit) — useful as a
diagnostic tool to map TWC3's exact reaction threshold live, but replaced
with the current 2-stage step (see "Publication law" above) to match the
independently-validated reference design once the threshold was confirmed.

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
   - `twc_breaker_limit_a` — the value entered in the Tesla installer app's
     Home Load Management / CT clamps section for TWC3 — **must match
     exactly**, otherwise the limit will be offset. This is typically the
     physical branch breaker derated to 80% for continuous load (e.g. a
     3x25A breaker → 20A here), not necessarily the breaker's own rating —
     use whatever value is actually configured on the TWC3 itself, not a
     recomputed one
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
   - `twc_vitals_ip` — TWC3's own LAN IP address (find it in your router's
     DHCP client list — TWC3 exposes an unauthenticated local
     `/api/1/vitals` JSON endpoint used for the R1 hard floor, the
     household-only self-balancing signal, and diagnostic sensors)
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
- `sensor.*_twc_vitals_current_l1/l2/l3`, `*_twc_vitals_vehicle_current` —
  the car's own per-phase/total current, straight from TWC3's local vitals
  API (diagnostic; also feeds the R1 floor and the household-only signal).
- `binary_sensor.*_twc_vitals_contactor_closed` — TWC3's own reported
  contactor state, from vitals.
- `switch.*_aggregate_balance_metering` — **default OFF**, FVE-mode-only, see
  "aggregate/net balance metering" above.
- `switch.*_twc_control_enabled` — **default ON**, see "Disabling external
  control" above.
- `number.*_fve_offset` — manual kW offset added to the FVE-mode
  self-balancing target, runtime-adjustable in HA (`-5.0`..`+5.0`).
- `switch.*_zone_steering_mode`, `binary_sensor.*_zone_steering_engaged`,
  and the `number.*_zone_steering_*` tuning entities — see "Zone
  steering" above.

## Known limitations / behavior

- **Home Assistant is a hard dependency for power control**, not just
  start/stop — if the HA API connection or the 6 mirrored real-current/power
  entities go stale, the firmware fails safe (`reported = twc_breaker_limit_a`,
  0A available) rather than continuing to charge on old data. This trades
  away the earlier direct-HTTP-poll design's partial independence from HA
  in exchange for avoiding Shelly Gen2's brute-force login protection
  entirely (no local auth to manage at all). TWC3's own vitals API is
  polled directly (no HA dependency) but is used only as a diagnostic/
  supporting signal, not as a substitute data source for fail-safe purposes.
- TWC3 FW 26.26.1 does **not** proportionally track `reported` below its own
  configured breaker limit — confirmed live, repeatedly — and its reaction
  above the limit is not a sharp threshold but a time-integrated response
  that eventually reduces at any excess, reacting faster the further past
  the limit `reported` sits (see "TWC3 reaction curve" above for the full
  measured data). This is a firmware behavior, not something this project
  can compute around; the whole publication-law design (see "How it works"
  above) — and zone steering's discrete bands — exist to work with it
  rather than against it.
- FVE mode is a bang-bang-flavored controller (not PID) — near `signed≈0`
  (exactly balanced) it may pulse slightly around zero grid exchange, and
  the household-only self-balancing target is slew-rate-limited on the
  way down (`desired_avail_slew_down_a_per_s`, 1A/s), so it deliberately
  does not react instantly to a sudden load/export change in that
  direction — but recovers quickly (`desired_avail_slew_up_a_per_s`,
  10A/s) once conditions improve.
- The register map (identification block, Neurio meter MAC/model/serial
  number) is a fixed placeholder taken from the reverse-engineered project
  linked below — it's not real data from any physical device.

## Sources

- https://gist.github.com/LucaTNT/4adf01a7252386559070023612efa117 (register mapping)
- https://github.com/Klangen82/tesla-wall-connector-control (MIT License)
- https://community.home-assistant.io/t/tesla-wall-connector-gen-3-via-esphome-rs485-dynamic-current-control-no-wifi/985613
- https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/EM (EM.GetStatus RPC)
- https://github.com/zany92/tesla-loadpilot — independently-developed,
  live-validated solution to the same TWC3 threshold-ignoring behavior;
  the current publication law, anti-glitch firewall, R1 floor, and 2-stage
  escalation are adapted from its actual source
  (`esphome/packages/twc-core.yaml`), cross-checked directly rather than
  from its docs (see "Publication law" above for why)

## License

[MIT](LICENSE) — chosen for compatibility with
[Klangen82/tesla-wall-connector-control](https://github.com/Klangen82/tesla-wall-connector-control)
(also MIT), the primary code source referenced above.
