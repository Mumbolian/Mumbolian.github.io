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

1. **Adjust wake windows** - Compresses or expands wake windows to shift naps earlier or later (least invasive)
2. **Adjust bedtime** - Moves bedtime earlier (shortens day) or later (extends day) by 15 minutes
3. **Reallocate nap time** - Redistributes nap durations while keeping total sleep constant (max ±15 min per nap, minimum 30 min each)
4. **Switch nap count** - Tries alternative 2-nap vs 3-nap schedule if needed

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

#### Strategy 1: Adjust Wake Windows
- **Compress**: Reduces `maxWake` by 5-30 minutes in 5-minute increments to shift naps earlier
- **Expand**: Increases `maxWake` by 5-30 minutes in 5-minute increments (up to 5 hour maximum) to shift naps later
- Preserves nap lengths and bedtime
- **Least invasive** - tries this first

Example: `maxWake` 210 → 200 minutes shifts schedule ~10 minutes earlier
Example: `maxWake` 210 → 220 minutes shifts schedule ~10 minutes later

#### Strategy 2: Adjust Bedtime
- **Earlier**: Moves bedtime 15 minutes earlier, shortening the day and forcing schedule compression
- **Later**: Moves bedtime 15 minutes later, extending the day and allowing more flexibility
- All naps and wake windows adjust to fit the new day length
- Can trigger 3→2 nap reduction (when compressing) or 2→3 nap addition (when extending)

Example: Bedtime 20:30 → 20:15 compresses everything by 15 minutes
Example: Bedtime 20:30 → 20:45 extends everything by 15 minutes

#### Strategy 3: Reallocate Nap Time
- Reduces conflicting nap and redistributes time to other naps
- **Total sleep time stays constant**
- Constraints per nap:
  - Minimum 30 minutes
  - Maximum ±15 minutes change from original
- Reduction in 5-minute increments (5, 10, 15 min)

Example: Reduce Nap 2 by 10 min, add 5 min each to Nap 1 and Nap 3

#### Strategy 4: Switch Nap Count
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
