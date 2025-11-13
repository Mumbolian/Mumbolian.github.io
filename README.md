# Bingpot Studio – Nap Planner

A tiny browser-only nap scheduling tool built for tired parents.

You enter:

- Wake-up time  
- Fixed bedtime (non-negotiable)  
- Preferred number of naps (2 or 3)  
- Nap lengths  
- First wake window  
- Last wake window  
- Min and max wake window limits  

The app then builds a full day schedule that **always lands on the chosen bedtime**, and automatically drops from 3 to 2 naps if the constraints can’t be satisfied.

Live at: **https://bingpotstudio.com**

---

## Concept

The design is based on how parents actually plan nap days:

- Bedtime is usually **fixed** (bath, bottle, sibling schedules, etc.).
- The **last wake window** before bed is also fairly fixed (e.g. 3–3.5 hours).
- Wake windows tend to **increase throughout the day**, not stay flat.
- It’s often easier to “work backwards” from bedtime rather than forward from wake-up.

This tool formalises that mental model in a way that’s simple enough to use on a phone mid-chaos.

---

## How the schedule is calculated

### Inputs

Core inputs:

- `wakeTime` – time baby wakes up  
- `bedTime` – target bedtime (fixed)  
- `preferThree` – checkbox: prefer 3 naps (fall back to 2 if needed)  
- `nap1`, `nap2`, `nap3` – nap lengths in minutes  
- `firstWake` – first wake window (currently just stored, not directly used in the math)  
- `lastWake` – wake window before bed (treated as fixed)  
- `minWake` – minimum allowed wake window  
- `maxWake` – maximum allowed wake window  

All of these are configurable in the UI, and persisted in `localStorage` under `napPlannerSettings_v4`.

### Constraints

- Bedtime is **non-negotiable**. The schedule must land exactly on that clock time (allowing for the “next day” if bedtime is earlier than wake on the clock).
- Last wake window is **fixed**, but clamped to `[minWake, maxWake]`.
- All earlier wake windows must also satisfy:  
  `minWake <= window <= maxWake`.
- If 3 naps can’t be fit between wake and bed while respecting the above, the planner automatically switches to a **2-nap day**.

### Algorithm (high level)

For a given nap count (2 or 3):

1. Convert wake time and bedtime into minutes, allowing bedtime to be “next day” if needed.
2. Compute total available day:
   ```text
   totalDay = bedMins - wakeMins
