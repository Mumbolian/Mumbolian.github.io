# T Man's Snooze Schedule

A tiny browser-only nap scheduling tool built for tired parents.

You enter:

- Wake-up time
- Fixed bedtime (non-negotiable)
- Preferred number of naps (2 or 3)
- Nap lengths
- Last wake window
- Min and max wake window limits

The app then builds a full day schedule that **always lands on the chosen bedtime**, and automatically drops from 3 to 2 naps if the constraints can't be satisfied.

Live at: **https://bingpotstudio.com**

---

## Concept

The design is based on how parents actually plan nap days:

- Bedtime is usually **fixed** (bath, bottle, sibling schedules, etc.).
- The **last wake window** before bed is also fairly fixed (e.g. 3–3.5 hours).
- Wake windows tend to **increase throughout the day**, not stay flat.
- It’s often easier to “work backwards” from bedtime rather than forward from wake-up.

This tool formalises that mental model in a way that's simple enough to use on a phone mid-chaos.

## Features

- **Local-only**: All settings are saved in your browser (localStorage). No server, no login, no tracking.
- **Automatic downgrade**: If 3 naps won't fit your constraints, the app automatically switches to 2 naps with appropriate defaults (90 min, 30 min).
- **Wake window display**: Each nap and bedtime shows the active wake window duration before it (e.g., "↑ 2h30 awake").
- **Smart defaults**: Different default nap lengths for 2-nap (90, 30) vs 3-nap (30, 60, 30) schedules.
- **Copy to clipboard**: One-click copy of the full day schedule.
- **Responsive design**: Works on mobile and desktop.

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

All of these are configurable in the UI, and persisted in `localStorage` under `napPlannerSettings_v4`.

### Constraints

- Bedtime is **non-negotiable**. The schedule must land exactly on that clock time (allowing for the "next day" if bedtime is earlier than wake on the clock).
- Last wake window is **fixed** and must be within `[minWake, maxWake]`.
- All earlier wake windows are calculated dynamically to fit the day, starting at `minWake` and growing toward `maxWake`, ensuring:
  `minWake <= window <= maxWake`.
- If 3 naps can't be fit between wake and bed while respecting the above, the planner automatically switches to a **2-nap day** with appropriate defaults.

### Algorithm (high level)

For a given nap count (2 or 3):

1. Convert wake time and bedtime into minutes, allowing bedtime to be “next day” if needed.
2. Compute total available day:
   ```text
   totalDay = bedMins - wakeMins
