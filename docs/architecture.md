# T‑Man’s Snooze Schedule – Architecture Overview

This doc describes the current architecture of the production web app at `https://t-man.net` (backed by `index.html`), and serves as the canonical high-level spec for the scheduling engine and UI.

---

## 1. High-Level Structure

- **Single-page app**
  - All HTML, CSS, and JavaScript live in `index.html` (and `test/index.html` for active development).
  - No backend or build system; everything runs in the browser.

- **Main responsibilities**
  - Collect user inputs (wake time, bedtime, nap prefs, wake windows, constraints, Power Sleep).
  - Compute one or more feasible nap schedules for the day.
  - Show those schedules with clear wake windows and adjustment explanations.
  - Persist user preferences locally via `localStorage`.

---

## 2. Core Concepts & Data Model

- **Time representation**
  - Internally, times are stored as **minutes from midnight** (integers).
  - Helpers convert between HH:MM strings and minutes (`toMinutes`, `fromMinutes`).

- **Settings**
  - Wake/bed times.
  - Nap mode preference: 2 vs 3 naps.
  - Mode-specific nap lengths:
    - `twoNapLengths` (nap1, nap2, nap3 placeholder).
    - `threeNapLengths` (nap1, nap2, nap3).
  - Wake window parameters:
    - `minWake`, `maxWake` (biological bounds).
    - `lastWake` (fixed final wake window).
    - `maxFirstWake` (upper bound on first wake).
  - Power Sleep:
    - `powerSleepEnabled`, `powerSleepTarget`.
  - Theme: light vs neon.

- **Constraints**
  - “Must be awake during” window (start/end).
  - “Must be asleep during” window (start/end).
  - Both normalized relative to the main wake time when applied.

- **Schedule items**
  - A schedule is a list of items:
    - `Wake-up`
    - `Nap 1`, `Nap 2`, `Nap 3` with start/end and nap length.
    - `Bedtime`.
  - Each item carries:
    - Time range.
    - Wake window before it.
    - Badges (start of day / nap / end of day).
    - Adjustment flags (e.g. adjusted nap).

---

## 3. Engine & Algorithms

The engine logic lives in the JavaScript portion of `index.html` / `test/index.html`. Canonical behavior is described by `README.md` and `UNIFIED_LP_IMPLEMENTATION.md`.

### 3.1 Base Schedule (`tryBuildSchedule`)

For a given nap count (2 or 3) and wake strategy:

1. Convert wake/bed to minutes and compute total day length.
2. Apply nap lengths to get total planned nap time.
3. Subtract nap time and `lastWake` from the day to find remaining time for earlier wake windows.
4. Distribute the remaining time into wake windows before each nap, according to the chosen strategy:
   - **Build Later**: larger increases toward the end of the day.
   - **Gentle**: smaller, more even increases.
   - **Balanced**: nearly equal wake windows.
5. Enforce constraints:
   - All wake windows within `[minWake, maxWake]`.
   - `lastWake` fixed and within `[minWake, maxWake]`.
   - First wake window ≤ `maxFirstWake`.
6. If feasible, return a schedule (wake + naps + bedtime) and associated wake-window info; otherwise return a failure reason.

### 3.2 Base Auto-Adjustment (`tryAutoAdjustNapsToFit`)

When a base schedule cannot be built (even with the user’s nap preferences) and no explicit constraints are set:

1. Generate nap-length combinations around the user’s values using `generateNapCombinations`:
   - Pairwise transfers.
   - Reductions/redistributions.
   - Single-nap ± adjustments (±5–15 minutes).
2. Try each set of nap lengths with `tryBuildSchedule`.
3. If still infeasible, try shifting bedtime by ±15 minutes and repeat.
4. Accept the first schedule that fits all biological constraints and report changes (e.g. “Auto-adjusted naps to fit schedule: Nap 1 +10 min”).

### 3.3 Unified Constraint Solver (`adjustScheduleForConstraints`)

When constraints are active (“awake during” / “asleep during”):

1. Build an initial schedule for the requested nap count.
2. If it already satisfies constraints, do nothing.
3. Otherwise, run the **unified adjustment**:
   - Loop over bedtime offsets: `0`, `-15`, `+15` minutes.
   - For each offset:
     - Build nap-length combinations using `generateNapCombinations` (±5–30 minutes per nap).
     - For each combination:
       - Call `tryBuildSchedule` with new bedtime and nap lengths.
       - Validate constraints using `validateConstraints`.
       - If both biological limits and constraints pass, return this schedule.
4. If no solution is found with the current nap count, try the **alternate nap count** (2 ↔ 3) using stored settings or defaults, and repeat the validation.
5. Expose a status message summarizing changes (e.g. “Adjusted schedule: bedtime earlier by 15 min to 20:15, adjusted naps: Nap 1 -10 min, Nap 3 +15 min”).

### 3.4 Biological Hard Limits

As established in `BUGFIX_WAKE_WINDOW_LIMITS.md`:

- **Never adjusted:**
  - `minWake`, `maxWake`, `lastWake`.
- **Only adjusted:**
  - Nap lengths (within min-nap and max-adjust bounds).
  - Bedtime (±15 minutes in unified solver; ±15 in base auto-adjust).

Any solution that violates biological bounds is discarded, even if it would satisfy time constraints.

---

## 4. UI & State Flow

- **Inputs → State**
  - Time inputs and number fields update DOM elements.
  - A central `recomputeSchedule()` function reads all inputs, normalizes them, and drives:
    - Base schedule building.
    - Automatic adjustments.
    - Constraint solver (if constraints set).

- **State → UI**
  - `recomputeSchedule()`:
    - Chooses nap mode (2 vs 3), performing auto-switching if needed.
    - Computes up to 3 scenarios (Build Later, Gentle, Balanced).
    - Renders scenario tabs and the active schedule list.
    - Updates warnings/success messages and summary text.
    - Populates a copy-to-clipboard text representation.

- **Persistence**
  - Settings persisted as `napPlannerSettings_v6` in `localStorage`: mode-specific nap configs, wake windows, Power Sleep target, theme, etc.
  - Actual naps persisted under a separate key and used to adjust remaining naps and compute insights.

- **Theme & Versioning**
  - Theme toggle switches between light and neon via a `data-theme` attribute on `<html>` and CSS overrides.
  - `APP_VERSION` constant in the JS; a focus-based fetch of `index.html` compares embedded version strings and hard-reloads when a new version is detected (settings preserved via `localStorage`).

---

## 5. Extensibility Notes

- **For iOS / other clients**
  - The Swift scheduling engine should mirror:
    - Time representations and wake-window semantics.
    - `tryBuildSchedule` behavior (including failure modes).
    - Unified constraint-solving logic and biological limits.
  - `UNIFIED_LP_IMPLEMENTATION.md` plus this architecture doc define the engine “contract”; Swift can reimplement it as pure functions with the same invariants.

- **For future web changes**
  - Keep `index.html` the canonical implementation.
  - Update `README.md`, `UNIFIED_LP_IMPLEMENTATION.md`, and this doc when any behavior around constraints, biological limits, or schedule building changes.

