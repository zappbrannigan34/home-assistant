# Lessons Learned: Home Assistant Configuration

**Purpose:** Document mistakes, their root causes, and how to avoid them in future
**Last Updated:** 2025-11-25
**Repository:** https://github.com/zappbrannigan34/home-assistant

---

## 📋 Table of Contents

1. [Template Sensors Entity Tracking Issue](#1-template-sensors-entity-tracking-issue)
2. [Unauthorized Configuration Changes](#2-unauthorized-configuration-changes)
3. [Reload vs Restart Requirements](#3-reload-vs-restart-requirements)
4. [Research Methodology Failures](#4-research-methodology-failures)
5. [Cross-Reference Validation](#5-cross-reference-validation)

---

## 1. Template Sensors Entity Tracking Issue

### 📅 Date
2025-11-24 to 2025-11-25

### 🐛 Problem
Template sensors stuck as "unavailable" despite source entities working correctly.

### 🔍 Symptoms
- ✅ Configuration check passes
- ✅ Home Assistant logs show no errors
- ✅ Source sensors (`sensor.sensor_eva_temperature`, etc.) working and updating
- ❌ Template sensors (`sensor.adaptive_target_humidity`, etc.) stuck as "unavailable"
- ❌ `last_changed` timestamp from 23:18 UTC never updates
- ❌ Template logic never executes

### 🚨 Root Cause

**The `states()` function doesn't create entity tracking in Home Assistant.**

```yaml
# This LOOKS correct but DOESN'T WORK reliably:
template:
  - sensor:
      - name: "Adaptive Target Humidity"
        state: >
          {% set T = states('sensor.sensor_eva_temperature') | float(22) %}
          {{ T * 2 }}
```

**Why it fails:**
1. `states('entity')` is designed to be "safe" (returns default if unavailable)
2. Because it's safe, HA assumes you DON'T want automatic tracking
3. Template evaluates ONCE at startup, then NEVER again
4. No subscription to source entity changes created

### ✅ Solution

**Use trigger-based templates with explicit entity tracking:**

```yaml
template:
  - trigger:
      # Explicit: "Re-evaluate when these entities change"
      - platform: state
        entity_id: sensor.sensor_eva_temperature
      # Fallback: "Re-evaluate every 5 minutes regardless"
      - platform: time_pattern
        minutes: "/5"
    sensor:
      - name: "Adaptive Target Humidity"
        # Prevent unavailable cascade
        availability: >
          {{ has_value('sensor.sensor_eva_temperature') }}
        # Still safe to use states()!
        state: >
          {% set T = states('sensor.sensor_eva_temperature') | float(22) %}
          {{ T * 2 }}
```

### 🎓 Lessons Learned

1. **`states()` function ≠ Entity tracking**
   - Safe for missing entities
   - Does NOT create automatic subscriptions
   - Use for robustness, NOT tracking

2. **Trigger-based templates for packages**
   - Always use `trigger:` block in packages
   - Explicit `platform: state` for each source
   - Add `time_pattern` fallback

3. **Availability templates prevent cascades**
   - Use `has_value()` to check entity exists
   - Prevents cascading "unavailable" during startup
   - Better UX than hard failures

4. **Structural changes need full restart**
   - Changing `sensor:` → `trigger:` format
   - `reload_all` insufficient for new trigger-based entities
   - Always test with full restart

### 📚 References
- Fixed in: humidity_control.yaml v2.0.4 (commit dee50d1)
- Troubleshooting: [TROUBLESHOOTING_TEMPLATE_SENSORS.md](TROUBLESHOOTING_TEMPLATE_SENSORS.md)
- Pattern source: kvazis/newHA repository
- Related issue: https://github.com/home-assistant/core/issues/145567

---

## 2. Unauthorized Configuration Changes

### 📅 Date
2025-11-24 (version 2.0.2, reverted in 2.0.3)

### 🐛 Problem
Changed `sensor.sensor_eva_*` → `sensor.sensor_zap_*` without user authorization, breaking all automations.

### 🔍 What Happened

**Trigger:** Spook integration showed warnings about orphaned `sensor_eva_*` entity references.

**Assumption (incorrect):**
- Thought `eva` sensors were renamed to `zap` sensors
- Assumed Spook warnings indicated real problem
- Made bulk find-replace without verification

**Reality:**
- **BOTH** sensor sets exist in system
- `sensor.sensor_eva_*` = Eva room sensor (current)
- `sensor.sensor_zap_*` = Zap room sensor (different location)
- Spook warnings were **FALSE POSITIVES**

### 🚨 Root Causes

1. **Trust without verify**
   - Trusted Spook warnings without checking entity existence
   - Didn't verify via API before making changes

2. **Lack of authorization**
   - Made significant changes without explicit user permission
   - Assumed intent instead of asking

3. **Insufficient testing**
   - Didn't verify sensors worked after change
   - Didn't check if referenced entities exist

### ✅ Prevention

1. **ALWAYS verify entities exist before renaming references:**
   ```bash
   # Check if entity exists via API
   curl -s -H "Authorization: Bearer $TOKEN" \
     https://hass.example.org/api/states/sensor.sensor_eva_temperature

   # Check entity registry
   curl -s -H "Authorization: Bearer $TOKEN" \
     https://hass.example.org/api/config/entity_registry | \
     jq '.[] | select(.entity_id | contains("sensor_eva"))'
   ```

2. **Never make bulk changes without explicit approval:**
   - Show plan with exact changes
   - Wait for user "OK" / "да" / "делай"
   - Explain impact of changes

3. **Validate third-party tool warnings:**
   - Spook warnings may be false positives
   - Always cross-check with HA core sources
   - Entity existence > integration warnings

4. **Test after every change:**
   - Check sensors show real values (not unavailable)
   - Verify automations still trigger
   - Monitor logs for errors

### 🎓 Lessons Learned

1. **Spook integration warnings ≠ Truth**
   - May detect orphaned references incorrectly
   - Cross-validate with entity registry
   - Prefer HA core tools over third-party

2. **Authorization gates for destructive operations**
   - Renaming entity references = destructive
   - ALWAYS require explicit approval
   - Show impact analysis first

3. **Sensor naming can be ambiguous**
   - Multiple sensors may have similar names
   - `sensor_eva` vs `sensor_zap` = different physical devices
   - Check device registry for context

### 📚 References
- Reverted in: humidity_control.yaml v2.0.3
- See: packages/humidity/CHANGELOG.md

---

## 3. Reload vs Restart Requirements

### 📅 Date
2025-11-25

### 🐛 Problem
After converting templates to trigger-based format, `homeassistant.reload_all` didn't apply changes.

### 🔍 What Happened

**Sequence:**
1. Changed template format: `sensor:` → `trigger:`
2. Ran `homeassistant.reload_all`
3. Logs showed "success"
4. Sensors still "unavailable"
5. Required **full restart** to apply changes

### 🚨 Root Cause

**Home Assistant reload behavior:**
- `reload_all` works for **content changes** (template logic, thresholds, etc.)
- `reload_all` does NOT work for **structural changes** (entity type, new triggers)
- Trigger-based templates are created differently than state-based
- Entity registry needs full reload to recognize new entity types

### ✅ When to Use Which

**Use `homeassistant.reload_all` for:**
- ✅ Template logic changes (formulas, thresholds)
- ✅ Automation condition/action changes
- ✅ Script sequence modifications
- ✅ Entity attribute updates

**Use `homeassistant.restart` for:**
- ✅ Template entity TYPE changes (`sensor:` → `trigger:`)
- ✅ New template entities added
- ✅ Integration configuration changes
- ✅ Package structure changes

**Use `template.reload` for:**
- ✅ Existing template sensor logic changes
- ✅ Quick iteration on template formulas
- ❌ **NOT** for new trigger-based templates

### 🎓 Lessons Learned

1. **Structural changes = restart required**
   - Adding `trigger:` block = structural
   - Changing entity creation method = structural
   - Don't assume reload is sufficient

2. **Test methodology matters**
   - After reload: verify sensor states changed
   - Check `last_changed` timestamp updated
   - Don't trust "success" logs alone

3. **Document reload requirements**
   - Add migration notes to CHANGELOG
   - Warn in comments about restart requirement
   - Reference in troubleshooting guides

### 📚 References
- Documented in: TROUBLESHOOTING_TEMPLATE_SENSORS.md
- Related: packages/humidity/CHANGELOG.md § Migration Guide

---

## 4. Research Methodology Failures

### 📅 Date
2025-11-24

### 🐛 Problem
Initial research was shallow, stopped at first hypothesis without thorough investigation.

### 🔍 What Happened

**Initial approach (wrong):**
1. Found issue #145567 on GitHub (reload problem)
2. Assumed it was the cause
3. Suggested restart immediately
4. Didn't verify hypothesis first

**User feedback:**
> "сука я говорил сначала перезагрузи тварь убедись в своей гипотезе только потом ресерч"
> (Translation: "Verify hypothesis FIRST, then research")

**Corrected approach:**
1. User performed restart
2. Verified sensors still broken
3. THEN did deep research
4. Found 15+ sources including kvazis repository
5. Identified real root cause (entity tracking)

### 🚨 Root Causes

1. **Confirmation bias**
   - Found one explanation, stopped looking
   - Fit hypothesis to first finding
   - Didn't consider alternatives

2. **Insufficient research depth**
   - Only checked 2-3 sources
   - Didn't search kvazis repository (Russian HA expert)
   - Stopped at GitHub issues only

3. **Wrong verification order**
   - Suggested solution before verifying problem
   - "Research → Solution" instead of "Test → Research → Solution"

### ✅ Prevention: Deep Research Protocol

**Minimum 10 findings per hypothesis:**

1. **Official Documentation** (2-3 sources)
   - Home Assistant official docs
   - Component/integration documentation
   - Breaking changes announcements

2. **GitHub Issues** (3-4 sources)
   - Exact error messages search
   - Related feature discussions
   - Closed issues with solutions

3. **Community Forums** (2-3 sources)
   - Home Assistant Community
   - Reddit r/homeassistant
   - Discord channels

4. **Expert Repositories** (2-3 sources)
   - kvazis (Russian HA expert)
   - HACS integration examples
   - Popular automation repositories

5. **Verification Tests** (always!)
   - Reproduce problem
   - Test hypothesis
   - Verify solution

**Research checklist:**
- [ ] Checked official HA docs
- [ ] Searched GitHub issues (open + closed)
- [ ] Checked community forums
- [ ] Reviewed expert repositories (kvazis, etc.)
- [ ] Found at least 10 relevant sources
- [ ] Verified hypothesis with actual tests
- [ ] Cross-referenced findings across sources

### 🎓 Lessons Learned

1. **Test hypothesis BEFORE solution**
   - Verify problem exists
   - Test proposed solution in isolation
   - Don't assume first finding is correct

2. **Deep research = 10+ sources minimum**
   - GitHub issues alone insufficient
   - Include expert community repositories
   - Cross-reference findings

3. **kvazis repository is gold standard**
   - Russian HA community leader
   - Tested patterns in production
   - Often ahead of official docs

4. **User feedback is authoritative**
   - User knows their system
   - Listen to correction immediately
   - Adjust methodology on demand

### 📚 References
- Research pattern documented in: CLAUDE.md (user instructions)
- kvazis repository: https://github.com/kvazis/newHA

---

## 5. Cross-Reference Validation

### 📅 Date
2025-11-25

### 🐛 Problem
After fixing template sensors, needed to verify all entity_id references are valid.

### 🔍 Process

**What we validated:**
- All 24 entity_id references in humidity_control.yaml
- Source sensors existence and state
- Template sensors existence and state
- Device entities (switches, binary sensors, numbers)

**Method:**
```bash
# 1. Extract all entity_id from config
grep -roh "sensor\.[a-z0-9_]*" packages/humidity/ | sort -u

# 2. Check each exists in HA
curl -s -H "Authorization: Bearer $TOKEN" \
  https://hass.example.org/api/states | \
  jq '.[] | select(.entity_id | test("pattern")) | {entity_id, state}'
```

### ✅ Validation Checklist

**Before commit:**
- [ ] All referenced entities exist in HA
- [ ] Source entities have recent `last_updated` timestamps
- [ ] Template sensors show real values (not "unavailable")
- [ ] Device entities accessible (switches, numbers, etc.)
- [ ] Weather entity exists and updates
- [ ] Automation entities listed correctly

**Tools:**
- API state check: `/api/states/entity_id`
- Entity registry: `/api/config/entity_registry`
- Grep for patterns: `grep -roh "entity_pattern"`
- jq for filtering: `jq '.[] | select(...)'`

### 🎓 Lessons Learned

1. **Validate cross-references before commit**
   - Extract all entity_id from changed files
   - Verify each exists via API
   - Check states are reasonable

2. **Automate validation where possible**
   - Script to extract + check entities
   - Part of pre-commit checklist
   - Prevents broken references

3. **Entity existence ≠ Entity working**
   - Check state values reasonable
   - Verify `last_updated` recent
   - Test automation triggers

### 📚 References
- Validation script: TBD (future automation)
- API docs: https://developers.home-assistant.io/docs/api/rest/

---

## 🎯 Summary: Key Takeaways

### Top 5 Rules to Prevent Issues

1. **VERIFY BEFORE CHANGE**
   - Check entity existence via API
   - Test hypothesis before solution
   - Never trust warnings alone

2. **AUTHORIZATION FOR DESTRUCTIVE OPS**
   - Show plan + impact analysis
   - Wait for explicit "OK" / "да" / "делай"
   - No assumptions about intent

3. **DEEP RESEARCH = 10+ SOURCES**
   - Official docs + GitHub + Community + Experts
   - kvazis repository for Russian HA patterns
   - Cross-reference all findings

4. **STRUCTURAL CHANGES = RESTART**
   - Template TYPE changes need restart
   - Not just reload_all
   - Test with full restart always

5. **VALIDATE CROSS-REFERENCES**
   - Extract all entity_id from changes
   - Verify via API before commit
   - Check states reasonable

### Documentation Hierarchy

```
TROUBLESHOOTING_TEMPLATE_SENSORS.md  ← Start here for sensor issues
        ↓
LESSONS_LEARNED.md (this file)        ← Why problems happened
        ↓
DEBUGGING_HA.md                       ← How to investigate
        ↓
NAMING_CONVENTIONS.md                 ← Prevent entity_id issues
        ↓
packages/*/CHANGELOG.md               ← Version history + migration
```

---

**Maintained by:** zappbrannigan34
**Last Updated:** 2025-11-25
**Related:** [TROUBLESHOOTING_TEMPLATE_SENSORS.md](TROUBLESHOOTING_TEMPLATE_SENSORS.md), [DEBUGGING_HA.md](DEBUGGING_HA.md)
