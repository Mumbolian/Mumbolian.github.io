# T Man's Snooze Schedule

A tiny browser-only nap scheduling tool built for tired parents.

You enter:

- Wake-up time
- Fixed bedtime (non-negotiable)
- Preferred number of naps (2 or 3)
- Nap lengths
- Wake window settings (collapsible)
  - Last wake window
  - Min and max wake window limits
- **NEW:** Schedule constraints (optional)
  - Must be awake by a certain time
  - Must be asleep during a time window

The app then builds a full day schedule that **always lands on the chosen bedtime**, automatically drops from 3 to 2 naps if needed, and **automatically adjusts to satisfy your constraints**.

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
- **Automatic downgrade**: If 3 naps won't fit your constraints, the app automatically switches to 2 naps with appropriate defaults (90 min, 30 min).
- **Wake window display**: Each nap and bedtime shows the active wake window duration before it (e.g., "↑ 2h30 awake").
- **Smart defaults**: Different default nap lengths for 2-nap (90, 30) vs 3-nap (30, 60, 30) schedules.
- **Copy to clipboard**: One-click copy of the full day schedule.
- **Responsive design**: Works on mobile and desktop.
- **Collapsible settings**: Wake window parameters collapse by default for a cleaner mobile interface.

### Schedule Constraints (NEW)
Set time-based requirements and let the app automatically adjust the schedule to meet them:

- **"Must be awake by"** constraint: Ensures baby is awake by a specific time (e.g., 12:30 PM for leaving the house)
- **"Must be asleep during"** constraint: Ensures baby is napping during a time window (e.g., 10:00-11:00 AM for a work call)

The app uses a unified linear programming approach to automatically satisfy constraints:

1. **Unified adjustment** - Simultaneously adjusts bedtime (±15-45 min) and nap lengths (±30 min) to meet constraints while respecting wake window limits
2. **Switch nap count** - Tries alternative 2-nap vs 3-nap schedule if unified adjustment can't find a solution

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

Optional constraint inputs:

- `mustBeAwakeBy` – time baby must be awake by (optional)
- `sleepWindow` – time window baby must be asleep during (optional)
  - `start` – start of required sleep window
  - `end` – end of required sleep window

All of these are configurable in the UI, and persisted in `localStorage` under `napPlannerSettings_v5`.

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

**Constraints:**
- Bedtime is **non-negotiable** - schedule must land exactly on target time
- Last wake window is **fixed** and must be within `[minWake, maxWake]`
- All earlier wake windows are dynamic, growing from `minWake` toward `maxWake`
- If 3 naps can't fit, automatically switches to 2-nap schedule

### Constraint Satisfaction Algorithm (Linear Programming Approach)

When schedule constraints are set, the app uses a unified approach to find a feasible solution:

**Important:** Wake window parameters (minWake, maxWake, lastWake) are biological constraints and are never modified. All adjustments work within these hard limits.

#### Strategy 1: Unified Linear Adjustment

Simultaneously adjusts multiple parameters to satisfy constraints:

**Variables adjusted:**
- Bedtime: ±15-45 minutes (tried in order: 0, -15, -30, -45, +15, +30)
- Nap lengths: ±30 minutes per nap (tried intelligently)
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
- Uses default lengths for that schedule type
  - 2-nap: 90, 30 minutes
  - 3-nap: 30, 60, 30 minutes
- Only used if unified adjustment finds no solution
- Can drastically change schedule layout

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
- **Modal for constraints**: Clean overlay (full-screen on mobile) for setting constraints
- **Real-time updates**: Schedule recalculates instantly as you adjust parameters
- **Visual indicators**:
  - Constraint button shows active count: "🎯 Constraints (2)"
  - Success/warning messages color-coded (green/red)
  - Wake window durations displayed: "↑ 2h30 awake"

---

## Technical Details

- **No backend**: Pure client-side JavaScript, runs entirely in browser
- **LocalStorage**: Settings persisted locally (v5 schema includes constraints)
- **No dependencies**: Vanilla HTML/CSS/JS, no frameworks
- **Mobile optimized**: Responsive design, native time pickers, touch-friendly
