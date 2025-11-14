# T-Man's Snooze Schedule

A tiny browser-only nap scheduling tool built for tired parents.

You enter:

- Wake-up time
- Fixed bedtime (non-negotiable)
- Preferred number of naps (2 or 3)
- Nap lengths (configured via modal for both 2-nap and 3-nap modes)
- Wake window settings (collapsible)
  - Last wake window
  - Min and max wake window limits
- Schedule constraints (optional)
  - Must be awake by a certain time
  - Must be asleep during a time window

The app then builds a full day schedule that **always lands on the chosen bedtime**, automatically switches between 2 and 3 naps as needed, and **automatically adjusts nap lengths to satisfy your constraints** while preserving your preferred settings.

Live at: **https://bingpotstudio.com**

---

## Concept

The design is based on how parents actually plan nap days:

- Bedtime is usually **fixed** (bath, bottle, sibling schedules, etc.).
- The **last wake window** before bed is also fairly fixed (e.g. 3–3.5 hours).
- Wake windows tend to **increase throughout the day**, not stay flat.
- It's often easier to "work backwards" from bedtime rather than forward from wake-up.
- **Real life requires flexibility**: Sometimes you need baby awake by a certain time (appointments, leaving the house) or asleep during a window (calls, sibling pickup).

This tool formalises that mental model in a way that's simple enough to use on a phone mid-chaos.

## Features

### Core Features
- **Version tracking**: App version displayed in footer for debugging and testing purposes
- **Local-only**: All settings are saved in your browser (localStorage). No server, no login, no tracking.
- **Multiple schedule scenarios** (v1.10+): The app generates up to 3 different schedule options with varying wake window distribution strategies:
  1. **Build Later**: Wake windows increase by 20-30 minutes, prioritizing shorter first wake window (ideal default - builds more awake time toward end of day)
  2. **Gentle**: Wake windows increase by 10-15 minutes (less dramatic increases)
  3. **Balanced**: Wake windows are roughly equal length (minimal variation)
  - Switch between scenarios using compact tabs above the schedule (mobile-optimized)
  - Each scenario independently respects constraints and Power Sleep Mode
  - Only feasible scenarios are shown
  - Each scenario may use different bedtime adjustments (±15 min) as needed
- **Nap configuration modal**: Configure both 2-nap and 3-nap settings independently via a dedicated modal, allowing you to maintain separate preferences for each schedule type without losing your configurations when the app auto-switches modes.
- **Mode-specific settings**: Separate storage for 2-nap and 3-nap configurations (v6 schema). When the app switches modes to fit constraints, it uses your saved settings for that mode.
- **Settings preservation**: Your configured nap lengths are never modified—adjustments only affect the displayed schedule. Your preferences remain your desired targets.
- **Automatic mode switching**: If your preferred mode won't fit constraints, the app automatically switches to the alternative (3↔2 naps) using your saved settings for that mode.
- **Bedtime flexibility** (v1.10.1+): As a last resort before declaring a schedule impossible, the app can adjust bedtime by ±15 minutes. This happens automatically after trying nap adjustments, keeping bedtime as close to your target as possible.
- **Visual adjustment indicators**:
  - Amber haze on nap schedule cards when adjusted
  - "⚠️ Adjusted +10 min" shown on affected naps
  - Amber highlight on Configure button when adjustments are active
  - Amber bedtime note when bedtime differs from target (e.g., "Bedtime adjusted to 20:45 (+15min from target 20:30)")
- **Wake window display**: Each nap and bedtime shows the active wake window duration before it (e.g., "↑ 2h30 awake").
- **Copy to clipboard**: One-click copy of the full day schedule.
- **Responsive design**: Works on mobile and desktop.
- **Collapsible settings**: Wake window parameters collapse by default for a cleaner mobile interface.
- **Power Sleep Mode**: Boosts nap times proportionally to reach a target total sleep duration (default: 3 hours). If current naps are set to 30, 60, 30 (total 120 mins), enabling Power Sleep with a 180-minute target distributes the missing 60 minutes proportionally, resulting in 45, 90, 45 minutes. Target is configurable in the nap settings modal.

### Schedule Constraints
Set time-based requirements and let the app automatically adjust the schedule to meet them:

- **"Must be awake by"** constraint: Ensures baby is awake by a specific time (e.g., 12:30 PM for leaving the house)
- **"Must be asleep during"** constraint: Ensures baby is napping during a time window (e.g., 10:00-11:00 AM for a work call)

The app uses a unified linear programming approach to automatically satisfy constraints:

1. **Unified adjustment** - Simultaneously adjusts bedtime (±15 min) and nap lengths (±30 min) to meet constraints while respecting wake window limits
2. **Switch nap count** - Tries alternative 2-nap vs 3-nap schedule if unified adjustment can't find a solution

**Automatic schedule fitting** (v1.10.1+): Even without explicit constraints, if a schedule cannot fit within wake window limits, the app automatically tries:
1. First: Adjust nap lengths by ±5-15 minutes
2. Last resort: Adjust bedtime by ±15 minutes
3. Finally: Switch between 2-nap and 3-nap modes

**Key principles:**
- Wake window parameters (minWake, maxWake, lastWake) are **hard biological constraints** and never adjusted
- Wake windows are distributed automatically with an **increasing pattern** throughout the day (preferred but not required)
- Adjustments are tried in order of minimal change (original bedtime + minor nap tweaks first)

Visual feedback shows which strategy was used and whether constraints are satisfied.

---

## How the schedule is calculated

### Inputs

Core inputs:

- `wakeTime` – time baby wakes up
- `bedTime` – target bedtime (fixed)
- `preferThree` – checkbox: prefer 3 naps (fall back to 2 if needed)
- `nap1`, `nap2`, `nap3` – nap lengths in minutes
  - 3-nap defaults: 30, 60, 30 minutes
  - 2-nap defaults: 90, 30 minutes
- `lastWake` – wake window before bed (treated as fixed, default: 210 minutes / 3h30)
- `minWake` – minimum allowed wake window (default: 135 minutes / 2h15)
- `maxWake` – maximum allowed wake window (default: 210 minutes / 3h30)
- `powerSleepEnabled` – checkbox: enable Power Sleep Mode to boost nap times to target
- `powerSleepTarget` – target total nap time in minutes (default: 180 minutes / 3 hours)

Optional constraint inputs:

- `mustBeAwakeBy` – time baby must be awake by (optional)
- `sleepWindow` – time window baby must be asleep during (optional)
  - `start` – start of required sleep window
  - `end` – end of required sleep window

All of these are configurable in the UI, and persisted in `localStorage` under `napPlannerSettings_v6`.

### Base Schedule Algorithm

For a given nap count (2 or 3):

1. Convert wake time and bedtime into minutes, allowing bedtime to be "next day" if needed.
2. Compute total available day:
   ```
   totalDay = bedMins - wakeMins
   ```
3. Subtract total nap time and last wake window from available time:
   ```
   remainingBeforeLast = totalDay - totalNapTime - lastWake
   ```
4. Distribute remaining time across wake windows before naps, starting at `minWake` and growing toward `maxWake`.
5. Build schedule items with wake windows displayed before each nap and bedtime.

**Auto-adjustment features:**
- If the base schedule can't fit with exact nap lengths, the app automatically tries adjusting nap lengths by ±5-15 minutes
- Naps are never reduced below 30 minutes minimum
- Adjustments prefer minimal changes (±5 min first, then ±10 min, etc.)
- Can also transfer time between naps (e.g., reduce one nap by 10 min, increase another by 10 min)
- If auto-adjustment succeeds, displays message like "✓ Auto-adjusted naps to fit schedule: Nap 1 +5 min"

**Constraints:**
- Bedtime is **non-negotiable** - schedule must land exactly on target time
- Last wake window is **fixed** and must be within `[minWake, maxWake]`
- All earlier wake windows are dynamic, growing from `minWake` toward `maxWake`
- If auto-adjustment fails for 3 naps, automatically switches to 2-nap schedule

### Constraint Satisfaction Algorithm (Linear Programming Approach)

When schedule constraints are set (e.g., "must be awake by" or "must be asleep during"), the app uses a more aggressive adjustment approach to satisfy them:

**Important:** Wake window parameters (minWake, maxWake, lastWake) are biological constraints and are never modified. All adjustments work within these hard limits.

**Note:** This is separate from the base auto-adjustment (±15 min) that happens automatically even without constraints. Constraint satisfaction can adjust naps by up to ±30 minutes and also adjust bedtime.

#### Strategy 1: Unified Linear Adjustment

Simultaneously adjusts multiple parameters to satisfy constraints:

**Variables adjusted:**
- Bedtime: ±15-45 minutes (tried in order: 0, -15, -30, -45, +15, +30)
- Nap lengths: ±30 minutes per nap (more aggressive than base auto-adjustment)
  - Single nap adjustments: ±5, ±10, ±15, ±20, ±25, ±30 minutes
  - Pairwise transfers: Move time between two naps (e.g., reduce Nap 1 by 10, extend Nap 2 by 10)

**Wake windows:**
- Automatically distributed by `tryBuildSchedule` algorithm
- Prefers increasing pattern (earlier windows shorter, later windows longer)
- Always respects [minWake, maxWake] bounds
- Last wake window is fixed

**How it works:**
1. For each bedtime adjustment (minimal first)
2. Try various nap length combinations (single adjustments, then pairwise transfers)
3. Let `tryBuildSchedule` distribute wake windows optimally
4. Validate constraints are satisfied
5. Return first feasible solution

**Example:** "Must be asleep 15:00-15:30"
- Tries: Original bedtime + extend Nap 3 by 15 min
- If that fails: Bedtime -15 min + reduce Nap 1 by 10 min
- If that fails: Bedtime -30 min + various nap combinations
- Result: "Adjusted schedule: bedtime earlier by 30 min to 20:00, adjusted naps: Nap 1 -10 min"

#### Strategy 2: Switch Nap Count
- Tries alternative nap count (2 ↔ 3)
- Uses your saved settings for that schedule type (falls back to defaults if none configured)
  - 2-nap defaults: 90, 30 minutes
  - 3-nap defaults: 30, 60, 30 minutes
- Only used if unified adjustment finds no solution
- Can drastically change schedule layout
- Your configured settings for the alternate mode are preserved and used

### Validation
After adjustments, the app validates constraints:
- **"Must be awake by"**: Checks if any nap overlaps with the constraint time
- **"Must be asleep during"**: Checks if at least one nap fully covers the required window

Displays:
- ✓ Green success message if all constraints satisfied
- ⚠️ Red warning with specific violation details and suggestions if not satisfied
- Clear message showing which strategy was used (e.g., "Compressed schedule by moving bedtime earlier to 20:15")

---

## UI/UX

- **Mobile-first design**: Works seamlessly on phones, the primary use case
- **Collapsible sections**: Wake window settings hidden by default to reduce clutter
- **Modals for configuration**:
  - **Nap Configuration Modal**: "⚙️ Configure Nap Lengths" - Edit both 2-nap and 3-nap settings side-by-side
  - **Constraints Modal**: "🎯 Schedule Constraints" - Set time-based requirements
  - **Track Naps Modal**: "📊 Track Actual Naps" - Record actual nap times for analysis
- **Real-time updates**: Schedule recalculates instantly as you adjust parameters
- **Visual indicators**:
  - Constraint button shows active count: "🎯 Constraints (2)"
  - Configure button gets amber haze when naps are adjusted: "⚙️ Configure Nap Lengths"
  - Adjusted naps show amber border and "⚠️ Adjusted +10 min" on schedule cards
  - Success/warning messages color-coded (green/red)
  - Wake window durations displayed: "↑ 2h30 awake"
  - Preference mismatch indicator when actual mode differs from preference

---

## Technical Details

- **No backend**: Pure client-side JavaScript, runs entirely in browser
- **LocalStorage**: Settings persisted locally (v6 schema with mode-specific nap configurations)
  - Stores separate `twoNapLengths` and `threeNapLengths` objects
  - Preserves both configurations even when switching modes
  - Backwards compatible with v5 settings
- **No dependencies**: Vanilla HTML/CSS/JS, no frameworks
- **Mobile optimized**: Responsive design, native time pickers, touch-friendly
