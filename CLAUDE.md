# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

T-Man's Snooze Schedule is a **pure client-side nap scheduling app** for parents managing baby sleep schedules. The entire application lives in a single `index.html` file (no build system, no dependencies, no backend). It works backwards from a fixed bedtime, automatically distributes wake windows, supports 2-nap vs 3-nap schedules, and intelligently adjusts schedules to satisfy time-based constraints while respecting biological wake window limits. Primary use case: parents on phones during chaos.

**Live URL:** https://bingpotstudio.com

## Tech Stack

- Pure vanilla JavaScript (ES6+)
- No frameworks, no build tools
- localStorage for persistence (schema v6)
- Hosted on GitHub Pages
- Single HTML file with embedded CSS/JS

## Current Version

- **APP_VERSION: "1.12.0"** (line 1153)
- **Settings schema:** `napPlannerSettings_v6` (mode-specific nap configs)
- **Recent features:** Unified Settings modal (v1.12), auto-update detection (v1.11), multiple schedule scenarios (v1.10)
- See `README.md` for complete feature list

## Commands

### Development
```bash
# Serve locally (any HTTP server)
python3 -m http.server 8000
# or
npx serve

# View in browser
open http://localhost:8000
```

### Testing
Manual testing only. Test scenarios:
```bash
# Test with defaults (3 naps: 30/60/30, wake: 07:00, bed: 20:30)
# Test mode switching (2 ↔ 3 naps)
# Test constraints: "Must be awake by", "Must be asleep during"
# Test edge cases: midnight-crossing bedtimes, extreme wake windows
# Test Power Sleep Mode with various targets
```

See `test_constraint_analysis.md` for detailed test cases.

## Git Workflow & Worktrees

**GitHub Operations:**
- ✅ **ALWAYS use GitHub CLI (`gh`) for all GitHub operations** (PRs, issues, API queries)
- Examples: `gh pr create`, `gh pr view`, `gh issue list`, `gh api`

**Branch Protection:**
- ❌ Cannot push directly to `main` (protected, requires PRs)
- ❌ Cannot force push to `main` or `dev`
- ✅ Can push to `dev` branch (integration branch)

**Current Worktrees:**
- **Main:** `/Users/davidhowlett/Documents/GitHub/Mumbolian.github.io` → `dev` branch
- **Test:** `/Users/davidhowlett/Documents/GitHub/Mumbolian.github.io-test-feature` → `test-feature` branch

### Worktree Workflow

For multi-file features or long-running work. **Not for quick fixes or single-file changes.**

**Always ask user before creating worktree** - EXCEPT when user directly requests it ("create a worktree for X").

**Create worktree:** `create-worktree feature-name`
- Creates new branch + worktree at `../Mumbolian.github.io-feature-name/`
- Git hooks automatically symlinked (branch protection active)
- User runs `work-tree feature-name` to open new session
- **You cannot switch to worktree mid-session** - only create it

**Cleanup:** Only suggest when user asks.
- `cleanup-worktrees` - lists and prompts for deletion
- `cleanup-worktrees --list-only` - inspect without cleaning
- `cleanup-worktrees --force` - clean merged even with uncommitted changes

### Deployment Workflow (MANDATORY)

Hosted on GitHub Pages. Changes go live at https://bingpotstudio.com after PR merge.

**From any worktree (e.g., `test-feature`):**

1. **Increment version** in `APP_VERSION` (line 1153): `"1.12.0"` → `"1.13.0"`
2. **Commit changes:**
   ```bash
   git add index.html
   git commit -m "Description

   🤖 Generated with [Claude Code](https://claude.com/claude-code)

   Co-Authored-By: Claude <noreply@anthropic.com>"
   ```

3. **Push to dev branch (NOT main):**
   ```bash
   git push origin test-feature:dev
   ```

4. **Create PR from dev to main:**
   ```bash
   gh pr create --base main --head dev --title "Deploy: [description]" --body "Description of changes"
   ```

5. **User approves and merges PR** → Changes go live

**When user requests deployment:**
- Increment version FIRST
- Commit with descriptive message + Claude footer
- Push current branch to `dev`: `git push origin <current-branch>:dev`
- Create PR: `dev` → `main`
- Inform user PR is ready for review

## Architecture

### Single-File Structure

| Section | Lines | Contents |
|---------|-------|----------|
| HTML/CSS | 1-968 | Structure, embedded styles, dark mode design |
| JavaScript | 969-2900+ | All application logic (IIFE-wrapped) |
| Storage | - | localStorage only (`napPlannerSettings_v6`) |

### Key Architectural Patterns

1. **Reactive UI via central coordinator**
   - `recomputeSchedule()` (line ~2300): Called on every input change
   - Guards against recursion with `isRecomputing` flag
   - Validates inputs → builds schedule → updates DOM
   - All UI changes flow through this function

2. **Dual-mode configuration**
   - Separate settings for 2-nap vs 3-nap schedules
   - User configures both modes independently via modal
   - App auto-switches modes when constraints require it
   - Mode-specific settings preserved (`twoNapLengths`, `threeNapLengths`)

3. **localStorage persistence**
   - Schema v6 (mode-specific nap configs)
   - Backwards compatible with v5 settings
   - Auto-save on every change via `recomputeSchedule()` → `saveSettings()`
   - Migration: `initFromStorageOrDefaults()` (line ~1607)

## Core Algorithms

### Schedule Building: `tryBuildSchedule()` (line ~1833)

Works backwards from bedtime to build daily schedule:

1. Calculate total available awake time: `totalDay - totalNapTime - lastWake`
2. Distribute remaining time across wake windows (one before each nap)
3. Wake windows grow from `minWake` → `maxWake`, preferring increasing pattern
4. Last wake window (`lastWake`) is fixed
5. Returns: `{ok, items, wakeWindowDurations, ...}`

**Critical:** Wake window parameters (`minWake`, `maxWake`, `lastWake`) are **biological constraints** and are NEVER adjusted by any algorithm.

### Constraint Satisfaction: `adjustScheduleForConstraints()` (line ~2040)

Unified linear programming approach. Two strategies:

1. **Unified adjustment** - Simultaneously adjusts:
   - Bedtime: 0, -15, -30, -45, +15, +30 min (tested in order)
   - Nap lengths: ±5 to ±30 min (5-min increments)
   - Pairwise nap transfers (e.g., Nap 1 -10, Nap 2 +10)
   - Tests ~150 nap combinations per bedtime adjustment
   - Returns first feasible solution

2. **Switch nap count** - If unified adjustment fails:
   - Tries alternative mode (2 ↔ 3 naps)
   - Uses stored settings for that mode (or defaults)

**Validation:** `validateConstraints()` (line ~2176)
- "Must be awake by T": No nap overlaps with time T
- "Must be asleep during S-E": At least one nap fully covers window

See `UNIFIED_LP_IMPLEMENTATION.md` and `LINEAR_APPROACH.md` for detailed algorithm documentation.

### Auto-Adjustment: `tryAutoAdjustNapsToFit()` (line ~2000)

Runs automatically even without constraints:
- Tries minor nap adjustments (±5 to ±15 min) to fit schedule
- Never reduces naps below 30-minute minimum
- Runs before constraint satisfaction

### Power Sleep Mode: `applyPowerSleep()` (line ~2271)

Proportionally boosts nap times to reach target total sleep:
- Default target: 180 minutes (3 hours)
- Example: [30, 60, 30] (120 min) → [45, 90, 45] (180 min target)

### Actual Naps Tracking

- Users record actual nap times via modal
- Stored in separate localStorage key per date
- When actuals exist: `tryBuildScheduleWithActuals()` (line ~1687) modifies algorithm
- Midnight reset: `checkMidnightReset()` (line ~1283) clears stale data

## Code Style

- Arrow functions for new code
- Prefer `const` over `let`
- No semicolons (current style)
- Time handling via `toMinutes()` / `fromMinutes()` helpers (lines ~1249, ~1256)
- All UI updates through `recomputeSchedule()`
- Direct localStorage access only via `saveSettings()` / `loadSettings()`

## Version Management

**CRITICAL:** Increment version BEFORE every commit.

- **Current:** `const APP_VERSION = "1.12.0"` (line 1153)
- **Semantic versioning:** MAJOR.MINOR.PATCH
  - **MAJOR:** Breaking changes or major redesigns
  - **MINOR:** New features (e.g., unified Settings modal)
  - **PATCH:** Bug fixes, minor tweaks, documentation
- **Display format:** `v1.12` (patch version shown but not emphasized)

## Important Constraints

❌ **Never Modify:**
- Wake window parameters (`minWake`, `maxWake`, `lastWake`) - biological constraints
- Settings schema version without migration logic
- Direct localStorage manipulation outside `saveSettings()`

❌ **Git Workflow:**
- Never push directly to `main` (will be rejected)
- Never force push to `main` or `dev`
- Never skip version increment before commit

✅ **Always Do:**
- Increment version BEFORE committing code changes
- Test: defaults, mode switching, constraints, edge cases
- Use `recomputeSchedule()` for all UI updates
- Preserve user settings (adjustments affect display only, not stored configs)

✅ **Design Principles:**
- No backend (everything client-side)
- No frameworks (vanilla JavaScript)
- No build step (edit `index.html` and refresh)
- Mobile-first (primary use case: parents on phones)

## Key Files & Line References

| Function/Constant | Line | Purpose |
|-------------------|------|---------|
| `APP_VERSION` | 1153 | Version constant (increment before commit) |
| `STORAGE_KEY` | 1154 | localStorage key for settings |
| `initFromStorageOrDefaults()` | ~1607 | Settings migration (v5 → v6) |
| `tryBuildSchedule()` | ~1833 | Core schedule building algorithm |
| `tryAutoAdjustNapsToFit()` | ~2000 | Auto-adjustment (±15 min naps) |
| `adjustScheduleForConstraints()` | ~2040 | Constraint satisfaction (LP approach) |
| `generateNapCombinations()` | ~1953 | Generate nap adjustment combinations |
| `validateConstraints()` | ~2176 | Validate constraint satisfaction |
| `applyPowerSleep()` | ~2271 | Power Sleep Mode logic |
| `recomputeSchedule()` | ~2300 | Central UI coordinator |
| `toMinutes()` / `fromMinutes()` | ~1249, ~1256 | Time conversion helpers |
| `checkMidnightReset()` | ~1283 | Clear stale actual naps data |

## Modal Management

Three modals with open/close functions:

| Modal | Open Function | Close Function | Line | Purpose |
|-------|---------------|----------------|------|---------|
| Nap Config | `openNapConfigModal()` | `closeNapConfigModal()` | ~1144 | Edit 2-nap and 3-nap settings |
| Constraints | `openModal()` | `closeModal()` | - | Set time-based constraints |
| Track Naps | `openActualNapsModal()` | `closeActualNapsModal()` | - | Record actual nap times |

## Visual Feedback System

- **Amber haze:** Schedule cards when naps adjusted (border + "⚠️ Adjusted +X min")
- **Button highlight:** Configure button gets amber glow when adjustments active
- **Constraint count:** Button shows "🎯 Constraints (2)"
- **Success/warning messages:** Color-coded in `warningBox` div (green/red)
- **Wake window display:** Each item shows "↑ Xh Ymin awake"
- **Preference mismatch:** Toggle indicates when actual mode ≠ preference

## Common Modification Scenarios

### Adding a new input field
1. Add HTML input in appropriate card section
2. Add variable to `loadSettings()` / `saveSettings()` (lines ~1272, ~1295)
3. Read value in `recomputeSchedule()` (line ~2300)
4. Use in schedule calculation

### Modifying constraint satisfaction
1. Edit `adjustScheduleForConstraints()` (line ~2040)
2. Update `generateNapCombinations()` for different search space (line ~1953)
3. Modify `validateConstraints()` for new constraint types (line ~2176)
4. See `UNIFIED_LP_IMPLEMENTATION.md` for algorithm details

### Changing schedule calculation
1. Modify `tryBuildSchedule()` for base algorithm (line ~1833)
2. Update wake window distribution logic (lines ~1879-1888)
3. Ensure biological constraints (`minWake`, `maxWake`, `lastWake`) remain respected

## Documentation Files

- `README.md`: User-facing feature documentation (reference this for features)
- `LINEAR_APPROACH.md`: Mathematical formulation of constraint problem
- `UNIFIED_LP_IMPLEMENTATION.md`: Details of unified adjustment algorithm
- `IMPLEMENTATION_NOTES.md`: Historical notes on constraint satisfaction
- `BUGFIX_WAKE_WINDOW_LIMITS.md`: Wake window constraint bug fix notes
- `test_constraint_analysis.md`: Constraint testing scenarios
