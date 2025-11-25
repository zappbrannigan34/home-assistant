# Commit Message Template

Use this template for structured, informative git commits.

---

## Format

```
<type>: <short summary> (max 72 characters)

<body - explain WHAT changed and WHY>

<optional footer - references, breaking changes, co-authors>
```

---

## Commit Types

- **fix:** Bug fix (template sensors, automation logic, etc.)
- **feat:** New feature (new automation, integration, dashboard)
- **refactor:** Code restructuring (no behavior change)
- **docs:** Documentation only
- **style:** Formatting, whitespace (no logic change)
- **test:** Add/modify tests
- **chore:** Maintenance (dependency updates, config tweaks)
- **revert:** Revert previous commit

---

## Example 1: Bug Fix

```
fix: template sensors stuck as unavailable

Problem:
- Template sensors using states() function never updated
- states() doesn't create entity tracking in HA
- Sensors stuck as unavailable despite source entities working

Solution:
- Converted to trigger-based templates (kvazis pattern)
- Added explicit triggers: state changes + time_pattern fallback
- Added availability templates using has_value()
- Kept states() function for safety during startup

Changes:
- packages/humidity/humidity_control.yaml:
  * Version 2.0.1 → 2.0.4
  * Added trigger block with platform: state for all sources
  * Added availability templates to prevent cascading unavailable
  * Removed automation description fields (kvazis pattern)

Verified:
- sensor.adaptive_target_humidity: 40.0% ✅
- sensor.humidity_error: 16.1% ✅
- sensor.recommended_intensity: 7 ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Example 2: New Feature

```
feat: add bedroom climate control automation

Adds adaptive climate control for bedroom based on:
- Temperature sensor readings
- Time of day (different targets for day/night)
- Outdoor temperature (condensation prevention)
- Manual override detection

Files:
- packages/climate/bedroom_climate.yaml (NEW)
- docs/NAMING_CONVENTIONS.md (updated examples)

Tested:
- Automation triggers on temperature changes ✅
- Manual override detection working ✅
- Night mode activates at 22:00 ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Example 3: Documentation

```
docs: add troubleshooting guide for template sensors

Created comprehensive guide for debugging unavailable template sensors:
- Symptoms checklist
- Diagnosis steps (API commands, entity registry checks)
- Root cause explanation (states() vs trigger-based)
- Step-by-step fix instructions
- Verification checklist

Files:
- docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md (NEW)
- docs/LESSONS_LEARNED.md (NEW)
- docs/DEBUGGING_HA.md (NEW)
- packages/humidity/CHANGELOG.md (NEW)
- .github/COMMIT_TEMPLATE.md (NEW - this file)

Rationale:
Future-proof against repeating entity tracking issues.
All debugging commands documented for quick reference.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Pre-Commit Checklist

Before committing, verify:

### Code Changes
- [ ] Configuration check passes (`ha core check`)
- [ ] No YAML syntax errors
- [ ] Entity_id references validated (all exist in HA)
- [ ] Tested with full restart (if structural changes)
- [ ] Cross-references checked (grep + API validation)

### Documentation
- [ ] CHANGELOG updated (if package modified)
- [ ] Version number incremented (if applicable)
- [ ] Migration notes added (if breaking changes)
- [ ] Comments updated (if logic changed)

### Git
- [ ] Commit message follows template format
- [ ] Changes split into logical commits (not huge monolithic commits)
- [ ] Sensitive data excluded (.env, tokens, secrets)
- [ ] Co-author attribution included (if AI-assisted)

---

## Breaking Changes Format

If commit introduces breaking changes, add to footer:

```
BREAKING CHANGE: Template sensor format changed from state-based to trigger-based

Migration required:
1. Update humidity_control.yaml to v2.0.4
2. Full restart (NOT reload_all)
3. Verify sensors show real values

See: docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md
```

---

## References Format

Link related issues, commits, docs:

```
Fixes: #123
Related: dee50d1
See: docs/TROUBLESHOOTING_TEMPLATE_SENSORS.md
Pattern: https://github.com/kvazis/newHA
```

---

## Claude Code Attribution

For AI-assisted commits, include footer:

```
🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Tips

1. **Write for future you**
   - Explain WHY, not just WHAT
   - Include diagnosis steps taken
   - Document verification performed

2. **Be specific**
   - ❌ "Fixed sensors"
   - ✅ "fix: template sensors stuck as unavailable (states() tracking issue)"

3. **Use bullet points**
   - Easier to scan
   - Clear structure
   - Quick reference

4. **Include verification**
   - Show tested values
   - Confirm functionality
   - Prevent future breakage

5. **Link documentation**
   - Reference troubleshooting guides
   - Point to related commits
   - Include external sources (kvazis, HA docs, GitHub issues)

---

**Last Updated:** 2025-11-25
**Related:** [CONTRIBUTING.md](../CONTRIBUTING.md) (if exists)
