# Ventilation control redesign TODO

## Goal

Build a new ventilation controller where:
- `recommended_opening` is computed continuously and independently from actuator state
- the controller reasons on a 30-minute future horizon
- dead zone is ±50 ppm with hysteresis
- inside dead zone, changes happen only on sharp/meaningful trend drift
- outside dead zone, response scales dynamically with trend/error severity
- the physical actuator layer only applies the latest recommendation every 30 minutes

## Phase 1 — Freeze baseline and observability

- [ ] Re-read current multi-room implementation sections in `packages/ventilation/ventilation_control.yaml` for zap/eva controller flow.
- [ ] Identify which existing entities can remain as physical-layer pieces (`cover.ventilation_zap_group`, `cover.drivent_eva_driventokonnyi_privod`, `input_datetime.*last_adjustment`, safety automations).
- [ ] Identify which current entities belong to the old controller model and must be replaced or repurposed (`sensor.co2_trend`, `sensor.ventilation_recommended_position`, `sensor.ventilation_co2_error`, EVA equivalents).
- [ ] Preserve backward-compatible generic zap entity IDs unless removal is explicitly justified.

## Phase 2 — Continuous signal chain design

- [ ] Add a primary continuous CO2 smoothing layer using EMA / low-pass for zap.
- [ ] Add the same primary continuous CO2 smoothing layer for eva.
- [ ] Decide whether a tiny outlier/median pre-filter is needed before EMA; only keep it if there are real discrete spikes rather than normal airflow variation.
- [ ] Add a trend/slope signal per room derived from the filtered CO2, not raw CO2.
- [ ] Add a trend intensity / noisiness signal per room if needed to distinguish sharp drift from ordinary local airflow fluctuation.
- [ ] Add a 30-minute forecast signal per room (`forecast_co2_30m`) based on filtered level plus trend.
- [ ] Add a 30-minute forecast error signal per room (`forecast_error_30m = forecast - target`).

## Phase 3 — New controller logic

- [ ] Implement dead zone ±50 ppm around the forecast error, not raw instantaneous error.
- [ ] Implement hysteresis for the dead zone so boundary crossing does not chatter.
- [ ] Inside dead zone: default to hold.
- [ ] Inside dead zone: allow change only when trend/noisiness crosses a sharpness threshold.
- [ ] Outside dead zone: compute dynamic response from forecast error magnitude and trend steepness; do not use fixed steps.
- [ ] Keep the continuous recommendation independent from current cover position and from the 30-minute cooldown state.
- [ ] Ensure the recommendation answers only: “what opening should we want now for the next 30-minute horizon?”

## Phase 4 — Separate actuation layer

- [ ] Keep the 30-minute gated actuation logic as a separate automation layer.
- [ ] At each allowed actuation point, copy the latest recommendation to the grouped zap cover.
- [ ] At each allowed actuation point, copy the latest recommendation to the eva cover.
- [ ] Preserve room-specific `last_adjustment` tracking.
- [ ] Preserve overload/cold-weather/CO2-unavailable safety blocks.
- [ ] Confirm that manual actuator moves still do not rewrite continuous recommendation.

## Phase 5 — Diagnostics and dashboard

- [ ] Replace old diagnostics based on the legacy controller with the new signals:
  - filtered CO2
  - slope / trend intensity
  - forecast CO2 30m
  - forecast error 30m
  - continuous `recommended_opening`
  - actual window position
  - last adjustment time
- [ ] Update zap dashboard view to show grouped-actuator diagnostics plus both physical child positions.
- [ ] Update eva dashboard view to show the same controller-layer diagnostics.

## Phase 6 — Validation strategy

- [ ] Run local YAML diagnostics on modified package/dashboard files.
- [ ] Deploy repo-first changes to `R:\packages\ventilation\` and `R:\dashboards\`.
- [ ] Run HA `check_config`.
- [ ] Restart HA when entity graph changes require restart.
- [ ] After restart, verify all zap/eva controller entities exist and are not `unknown`/`unavailable` after stabilization.
- [ ] Verify `recommended_opening` updates continuously without actuator movement.
- [ ] Verify the 30-minute actuation layer copies recommendation correctly for zap group.
- [ ] Verify the 30-minute actuation layer copies recommendation correctly for eva.
- [ ] Check `home-assistant.log.fault`, Repairs, and Spook after deployment.

## Phase 7 — Cleanup and finish

- [ ] Remove or repurpose obsolete legacy controller signals if they are no longer needed.
- [ ] Clean leftover server-side junk / stale references created by earlier iterations after the new model is proven.
- [ ] Update `packages/ventilation/README.md` and project `AGENTS.md` to match the final controller architecture.
- [ ] Commit changes in atomic commits.
- [ ] Push to GitHub.
- [ ] Await user confirmation that the redesign is accepted.
