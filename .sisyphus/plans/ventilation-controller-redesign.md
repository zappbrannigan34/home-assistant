## Goal

Redesign the Home Assistant ventilation controller so that each room gets its own equilibrium-seeking controller, with continuous room demand driven by environmental signals, automatic coefficient estimation instead of manual gain tuning, and a physically separate actuator layer.

## Corrected requirement set after user correction

1. The main control horizon is **10 minutes**, not 30 minutes.
2. Regulation must be **separate per room** (`zap`, `eva`), not one shared global behavior with mirrored formulas.
3. The controller must target **steady-state convergence** under quasi-constant load (for example one person continuously present in a room).
4. Resulting graphs should become **flatter / steadier**, not continue oscillating as a reactive heuristic loop.
5. Room-specific limits are required: setpoint, min/max opening, deadband, actuator constraints, and related controller bounds.
6. **Coefficients must not be manually tuned**. They must be estimated/computed automatically from observed room and actuator dynamics.
7. Actuator gating/safety/manual behavior remains a downstream physical layer, not part of controller-state estimation.

## Final architectural decision

The current ventilation redesign direction must be **replaced wholesale**, not revised incrementally.

Reason:

- The current heuristic family uses forecast/slope-driven reactive logic.
- That family can be smoothed or decoupled from actuator state, but it still does not naturally converge to a stable equilibrium opening under quasi-constant load.
- The corrected requirement set now explicitly demands steady-state convergence, flatter graphs, 10-minute horizon, room-local regulation, and automatic coefficient estimation.
- Those requirements are structurally aligned with a **per-room PI-style regulator plus automatic room-model identification**, not with continued heuristic forecast control.

Therefore:

1. Do **not** continue tuning the current `continuous_forecast_error` controller family.
2. Do **not** try to rescue the existing heuristic by changing constants only.
3. Replace the controller layer with a different control family.

## Confirmed root cause in current code

- `packages/ventilation/ventilation_control.yaml:280-282` seeds zap recommendation from `cover.ventilation_zap_group.current_position` and then from the previous recommended value.
- `packages/ventilation/ventilation_control.yaml:486-488` seeds eva recommendation from `cover.drivent_eva_driventokonnyi_privod.current_position` and then from the previous recommended value.
- Both controllers use fixed delta bands (`±2` inside deadband, capped `±4` outside) and then clamp to min/max.
- This creates a saturating incremental integrator tied to actuator state instead of a pure continuous recommendation.

## Non-negotiable invariants

1. Recommendation must not read actuator position as controller state.
2. Recommendation must not depend on cooldown or last adjustment time.
3. Manual actuator movement must not change recommendation when environmental inputs are unchanged.
4. Dead zone is centered on `forecast_error_30m`, not raw CO2.
5. Recommendation remains continuous and minute-level even during the 30-minute actuation lockout.
6. Actuation layer copies the latest recommendation only at allowed actuation time.
7. Existing public recommendation entity IDs stay stable unless proven impossible.

## Target architecture

### Layer 1 — signal chain

Preserve and use the existing per-room signal chain as the controller inputs:

- filtered CO2
- slope from filtered CO2
- 30-minute forecast CO2
- 30-minute forecast error

These already exist for both `zap` and `eva` and should remain the primary controller inputs.

### Layer 2 — per-room equilibrium-seeking controller

Replace the current heuristic forecast controller with a **per-room PI-style controller** whose primary controlled variable is filtered CO2 error, not forecast error.

Per room:

1. Compute `error_room = filtered_co2_room - setpoint_room`.
2. Maintain a room-local integral state that learns the steady opening required for persistent occupancy/load.
3. Compute room demand from `bias_room + Kp_room * error_room + I_room`, with room-local limits and anti-windup.
4. Use the 10-minute forecast only as a secondary bounded anticipatory/feedforward term or diagnostic, not as the main error term.
5. Keep room demand independent from current actuator position and gate timing.
6. Apply one final clamp to room-local min/max demand.

### Automatic coefficient estimation

The controller must not depend on hand-tuned room gains as the final design.

Required direction:

1. Estimate room response from observed history:
   - CO2 rise under occupancy / closed-window conditions
   - CO2 decay under open-window conditions
   - effective actuator influence versus opening percentage
2. Derive room-local controller parameters automatically from those measured dynamics.
3. Treat manually-entered gains only as temporary fallback/debug values if absolutely needed during bootstrap, not as the intended steady-state design.
4. Preserve visibility of estimated coefficients and model quality as diagnostics.

### Practical identification direction

The repository and external research together suggest the following practical path:

1. Use Home Assistant history/API data to estimate room response from real operating intervals.
2. Reuse the repo's existing calibration mindset from `packages/humidity`:
   - keep persistent learned parameters
   - update them incrementally from observed cycles
   - expose calibration quality and current estimates for debugging
3. Learn at least these room-local quantities automatically:
   - baseline CO2 rise rate when effectively closed / minimally ventilated
   - effective CO2 removal rate versus opening level
   - approximate room time constant / settling behavior
4. Derive PI gains and/or equivalent controller constants from the learned room model.
5. Bias the controller toward conservative / no-overshoot tuning rather than aggressive disturbance rejection.

### Why classic autotuning is not enough

- Classical Ziegler–Nichols style autotuning is too aggressive for the user goal here.
- The target outcome is not "fast correction with visible oscillation", but flatter graphs and stable demand under constant occupancy.
- Therefore the estimator/tuner must prefer low-overshoot / damped settings, not quarter-wave-decay behavior.

### Controller behavior inside dead zone

- Default behavior: hold a room-local steady demand near the learned equilibrium.
- Hysteresis and deadband are allowed, but must support convergence rather than create hunting.
- Do not use fixed discrete jumps.

### Controller behavior outside dead zone

- Response should correct filtered CO2 error smoothly and settle back toward a stable opening.
- No hard output bands.
- No dependence on current cover position.
- Forecast/trend may modify demand only as bounded assist terms.

### Layer 3 — actuation gate

Keep existing 30-minute actuator automations structurally separate.

- `ventilation_control_main` remains the zap gate.
- `ventilation_eva_control_main` remains the eva gate.
- They read the latest recommendation at actuation time.
- They compare recommendation against current actuator position only for the decision to move hardware.
- They keep `input_datetime.*last_adjustment` as physical-layer timing only.

### Safety layer

Preserve safety-close automations and overload / outdoor temperature / CO2-unavailable checks.

## Concrete code changes

### 1. `packages/ventilation/ventilation_control.yaml`

#### Zap controller block

Replace current zap recommendation formula family with a room-local equilibrium controller.

Required changes:

- Remove heuristic forecast-error-as-primary-control logic.
- Remove room-shared parameter assumptions where zap and eva need independent behavior.
- Introduce room-local demand state consistent with PI-style control and anti-windup.
- Add automatic estimation path for zap controller parameters from observed dynamics.
- Replace the existing `sensor.ventilation_recommended_position` implementation rather than iteratively patching it.

Add or update diagnostic attributes to expose:

- controller mode
- dead zone state
- trend gating state
- raw controller output before clamp
- continuous error and slope factors

#### Eva controller block

Apply the same controller architecture to `eva`, but with independent room-local state, limits, and automatically estimated parameters.

#### Migration note

The implementation should be treated as a controller-family migration, not a patch release of the current heuristic. Existing public entity IDs may stay stable, but the internal algorithm and diagnostics should be considered a new generation of the controller.

#### Actuation automations

Keep the 30-minute automations mostly intact.

Only ensure:

- recommendation remains independent from actuator state
- actuator comparison stays only in actuation conditions
- no recommendation logic depends on `input_datetime.*last_adjustment`

### 2. `dashboards/ventilation_dashboard.yaml`

Update dashboard cards so both rooms visibly show:

- raw CO2
- filtered CO2
- slope
- forecast CO2 30m
- forecast error 30m
- controller mode / dead zone / trend gate where useful
- continuous recommendation
- actual actuator position
- last adjustment timestamp

The dashboard must make actuator/recommendation decoupling obvious.

### 3. `packages/ventilation/README.md`

Update package docs to match the final implemented formula and diagnostics.

### 4. `AGENTS.md` and TODO docs

Update repo-level docs only if they materially misdescribe the final ventilation architecture.

## Verification plan

### Static verification

1. Read back all changed sections in:
   - `packages/ventilation/ventilation_control.yaml`
   - `dashboards/ventilation_dashboard.yaml`
   - `packages/ventilation/README.md`
   - `AGENTS.md` if changed
2. Run `lsp_diagnostics` on the repository root and require zero new YAML errors in changed files.
3. Run targeted content checks on `packages/ventilation/ventilation_control.yaml` and require:
   - no recommendation-state formula path seeded from `cover.*current_position`
   - no recommendation-state formula path seeded from `input_datetime.*last_adjustment`
   - actuator position references remain allowed only in:
     - actuation conditions/actions
     - diagnostic attributes
4. Pass criteria:
   - changed YAML files are syntactically clean
   - recommendation sensors are actuator-independent by code inspection

### Local config verification

1. Run `lsp_diagnostics` for local YAML validation before deployment.
2. If available in the repo or local environment, run Home Assistant config parsing on the changed package/dashboard files through the local validation workflow; otherwise treat repo-side LSP validation as the local gate and perform authoritative validation on the server via `check_config`.
3. Pass criteria:
   - zero local YAML syntax errors
   - no malformed template blocks
   - no duplicated or broken entity definitions introduced by the refactor

### Deployment verification

1. Deploy repo-first to:
   - `R:\packages\ventilation\ventilation_control.yaml` or active SMB-equivalent path
   - `R:\dashboards\ventilation_dashboard.yaml` or active SMB-equivalent path
2. Run server-side Home Assistant `check_config`.
3. Restart Home Assistant because trigger/template structure and entity surface may change.
4. Verify all expected entities recover through API/state checks.
5. Pass criteria:
   - `check_config` exits cleanly
   - Home Assistant restarts successfully
   - all expected zap/eva entities exist and settle to non-`unknown` / non-`unavailable`

### Live behavior proof

Collect fresh server-side evidence using Home Assistant API state reads, timestamps, dashboard graphs, and log inspection.

#### Proof 1 — recommendation changes during lockout while actuator stays still

Tools/evidence:
- Home Assistant API state reads for recommendation sensor and cover entity
- dashboard history graph if available

Steps:
1. Capture current states of:
   - zap: `sensor.ventilation_recommended_position`, `cover.ventilation_zap_group`, `input_datetime.ventilation_last_adjustment`
   - eva: `sensor.ventilation_eva_recommended_position`, `cover.drivent_eva_driventokonnyi_privod`, `input_datetime.ventilation_eva_last_adjustment`
2. Wait for at least one controller recompute interval well inside the 30-minute gate window.
3. Capture the same states again.

Expected result:
- recommendation state changed for at least one room
- actual cover position did not change
- `last_adjustment` did not change

#### Proof 2 — manual actuator movement does not feed back into recommendation

Tools/evidence:
- Home Assistant API state reads before/after manual cover movement

Steps:
1. Record recommendation, filtered CO2, slope, forecast error, and current actuator position.
2. Manually move the physical actuator via UI/service call without changing environment.
3. Re-read recommendation and controller inputs after state settles.

Expected result:
- actuator position changes
- recommendation remains unchanged or only changes if controller inputs changed independently
- no direct actuator-position-induced jump in recommendation

#### Proof 3 — next 30-minute gate copies latest recommendation

Tools/evidence:
- Home Assistant API state reads
- automation trace/history if available

Steps:
1. Record recommendation and actuator position shortly before a 30-minute boundary.
2. Let the scheduled actuation automation fire.
3. Re-read recommendation, actuator position, and `last_adjustment`.

Expected result:
- actuator position moves to match latest recommendation within normal execution delay
- `last_adjustment` updates
- recommendation itself remains a controller output rather than being rewritten from actuator position

#### Proof 4 — output converges instead of oscillating reactively

Tools/evidence:
- dashboard/history graph or repeated API snapshots over time

Steps:
1. Collect a representative sample of recommendation states over fresh runtime.
2. Compare whether intermediate values occur meaningfully rather than rare endpoint-only behavior.

Expected result:
- room demand settles toward a relatively stable opening under quasi-constant occupancy/load
- graphs show reduced swing amplitude versus the old heuristic loop
- controller output is not just moving constantly in reaction to slope reversals

#### Proof 5 — coefficients are automatically estimated, not manually hand-tuned

Tools/evidence:
- API states and diagnostics for the room-local estimator outputs
- package/dashboard diagnostics

Steps:
1. Inspect the exposed estimated room parameters and their timestamps/source diagnostics.
2. Verify they are derived from observed runtime data rather than static hardcoded gains.

Expected result:
- controller coefficients/derived model terms are produced automatically per room
- any manual fallback values are clearly marked as bootstrap-only, not the final tuning path

#### Log and health checks

After deployment, inspect:
- `home-assistant.log.fault`
- Repairs
- Spook

Expected result:
- no new errors attributable to the redesigned ventilation package

### Dashboard QA

After dashboard update, verify both zap and eva sections visibly show:

- filtered CO2
- slope
- forecast CO2 30m
- forecast error 30m
- continuous recommendation
- actual actuator position
- last adjustment time
- controller mode / dead zone / trend gate if exposed

Pass criteria:
- both room views load without broken entity references
- controller-layer and actuator-layer signals are simultaneously visible

### Documentation QA

Re-read and compare these files against the implemented architecture:

- `packages/ventilation/README.md`
- `AGENTS.md` if changed
- `TODO-ventilation-control-redesign-2026-04-20.md` if changed

Pass criteria:
- docs describe forecast-error-based dead zone, continuous recommendation, separate 30-minute actuation gate, and actuator independence accurately
- docs do not claim behavior that the YAML no longer implements

## Implementation order

1. Lock this plan.
2. Refactor zap/eva controller formulas in `ventilation_control.yaml`.
3. Verify YAML and diagnostics locally.
4. Update dashboard visibility.
5. Update README / AGENTS / TODO docs.
6. Deploy and verify on server.
7. Commit only after live verification passes.
