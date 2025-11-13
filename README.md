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

The app uses a sophisticated multi-strategy approach to automatically satisfy constraints:

1. **Adjust bedtime** - Moves bedtime earlier by 15-45 minutes or later by 15-30 minutes (respects user's bedtime preferences)
2. **Adjust naps** - Shortens conflicting naps by up to 30 minutes, extends naps to cover sleep windows, or redistributes nap time
3. **Switch nap count** - Tries alternative 2-nap vs 3-nap schedule if needed

**Note:** Wake window parameters (minWake, maxWake, lastWake) are treated as hard biological constraints and are never adjusted.

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

### Constraint Satisfaction Algorithm (NEW)

When schedule constraints are set, the app automatically tries to adjust the schedule using these strategies in order:

**Important:** Wake window parameters (minWake, maxWake, lastWake) are biological constraints and are never modified. All adjustments work within these hard limits.

#### Strategy 1: Adjust Bedtime (Bidirectional)
- **Earlier**: Moves bedtime 15-45 minutes earlier in 15-minute increments (shortens day)
- **Later**: Moves bedtime 15-30 minutes later in 15-minute increments (extends day)
- All naps and wake windows adjust to fit the new day length
- Can trigger 3→2 nap reduction (when compressing) or 2→3 nap addition (when extending)
- **Limited to ±45/-30 min to respect bedtime routines**

Example: Bedtime 20:30 → 19:45 compresses by 45 minutes
Example: Bedtime 20:30 → 21:00 extends by 30 minutes

#### Strategy 2: Adjust Naps
- **First attempt**: Reduces conflicting nap by 5-30 minutes WITHOUT redistribution
  - Avoids pushing nap start time later
  - Extra awake time distributed across wake windows naturally
  - Most effective for satisfying "awake by" constraints
- **Second attempt**: Reduces conflicting nap and redistributes to naps AFTER it only
  - Prevents pushing the conflicting nap's start time later
  - Only redistributes to later naps in the schedule
  - Maximum 15 minutes redistribution per receiving nap
- Constraints:
  - Minimum 30 minutes per nap
  - Maximum 30 minutes reduction for conflicting nap

- **For "must be asleep during" constraints**: Extends naps by up to 30 minutes or reduces earlier naps to shift schedule
  - Finds nap closest to required sleep window
  - Extends nap to fully cover the window
  - Or reduces earlier naps to shift target nap into position

Example: Nap 2 (12:00-13:00) conflicts with "awake by 12:48"
→ Reduce Nap 2 to 48 minutes → ends at 12:48 ✓

#### Strategy 3: Switch Nap Count
- Tries alternative nap count (2 ↔ 3)
- Uses default lengths for that schedule type
  - 2-nap: 90, 30 minutes
  - 3-nap: 30, 60, 30 minutes
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
