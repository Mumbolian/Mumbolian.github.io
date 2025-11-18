# Schedule Engine – Detailed Logic Spec

This document describes the current production scheduling logic implemented in `index.html` and is intended as the canonical reference for re‑implementing the engine (e.g. in Swift for iOS). The goal is that a new implementation can reproduce the same behaviour and edge cases, with the UI built on top of a pure scheduling module.

It complements `docs/architecture.md`, `UNIFIED_LP_IMPLEMENTATION.md`, and `BUGFIX_WAKE_WINDOW_LIMITS.md` by spelling out the concrete rules and decision steps.

---

## 1. Scope and Core Responsibilities

The schedule engine is responsible for:

- Turning user preferences and biological limits into a day schedule of:
  - `Wake-up`
  - `Nap 1`, `Nap 2`, `Nap 3` (2–3 naps only)
  - `Bedtime`
- Respecting *biological wake window limits*:
  - No wake window below `minWake`.
  - No wake window above `maxWake`.
  - The final wake window before bed is fixed at `lastWake`.
  - The first wake window must not exceed `maxFirstWake`.
- Respecting optional *time constraints*:
  - “Must be awake during” window.
  - “Must be asleep during” window.
- Handling:
  - Two vs three nap modes, including auto upgrade/downgrade.
  - “Power Sleep” mode (boost total daytime sleep to a target).
  - Actual nap tracking and its impact on remaining naps.
  - Generating up to three alternative schedules (“styles”) per day.

All of this runs for a *single child / single day*; multi‑child support is a higher‑level concern.

---

## 2. Data Model and Inputs

### 2.1 Time Representation

- Internal time type: **minutes since midnight** (integer).
- Helpers:
  - `toMinutes("HH:MM") → minutes | null`
  - `fromMinutes(minutes) → "HH:MM"`
    - Values are normalised into `[0, 24*60)` via modulo arithmetic.
- If bedtime is earlier than wake time, it is treated as *next day*:
  - If `bedMins <= wakeMins`, add `24 * 60` to `bedMins`.

### 2.2 User Settings (conceptual)

For a given child/day, the engine works from:

- **Day bounds**
  - `wakeTime` (`wakeMins`): wake‑up time.
  - `bedTime` (`bedMins`): *target* bedtime (may be adjusted by the engine by ±15 minutes).

- **Nap structure**
  - Requested nap mode: `preferThree` → `requestedNapCount` ∈ {2, 3}.
  - Per‑mode nap lengths (minutes):
    - 2‑nap settings: `[nap1_2, nap2_2, nap3_2]` (third is a placeholder).
    - 3‑nap settings: `[nap1_3, nap2_3, nap3_3]`.
  - In the current UI, inputs `nap1`, `nap2`, `nap3` reflect either the 2‑nap or 3‑nap configuration depending on mode.

- **Wake window parameters (biological limits)**
  - `minWake`: minimum wake window between sleeps.
  - `maxWake`: maximum wake window between sleeps.
  - `lastWake`: fixed final wake window from last nap end → bedtime.
  - `maxFirstWake`: upper bound for the first wake window of the day.

These are *never* auto‑tuned by any algorithm; they are considered biological limits.

- **Power Sleep**
  - `powerSleepEnabled` (boolean).
  - `powerSleepTarget` (minutes) – target total daytime nap time, default 180 minutes (3h).

- **Constraints**
  - “Must be awake during” window:
    - `awakeWindowStart`, `awakeWindowEnd` ("HH:MM" each).
  - “Must be asleep during” window:
    - `sleepWindowStart`, `sleepWindowEnd` ("HH:MM" each).
  - Empty fields mean “no constraint” for that dimension.

- **Actual naps (optional)**
  - Array of 0–3 entries:
    - `{ napNum: 1|2|3, start: minutes, end: minutes }`.
  - Only naps that actually occurred are present; missing entries are treated as “not yet taken”.
  - There is a per‑day midnight reset (in the web app) that clears stale actuals; for the engine spec it is enough to assume “actuals are for this day”.

---

## 3. Output Model

The core engine returns a structure conceptually like:

- `success`: boolean.
- On failure:
  - `reason`: human‑readable string describing the main blocking issue (e.g. “Awake windows would have to go outside the min/max limits”).
  - Optionally: extra context per nap mode (why 2‑nap and 3‑nap both failed).
- On success:
  - `napCountUsed`: 2 or 3 – the nap count actually used (may differ from requested).
  - `items`: ordered list of schedule items:
    - `label`: `"Wake-up"`, `"Nap 1"`, `"Nap 2"`, `"Nap 3"`, `"Bedtime"`.
    - `timeRange`: `"HH:MM – HH:MM"` or a single time `"HH:MM"` for wake/bed.
    - `extra`: e.g. `"60 min"` for naps.
    - `wakeWindow`: minutes of wake time preceding this item (for naps and bedtime).
    - `badge` / `badgeClass`: semantic tags (start of day, nap, actual, end of day).
  - `wakeWindowDurations`: list of per‑segment wake windows, including final wake to bed.
  - `napLengthsUsed`: nap lengths actually used for each nap, after any adjustments.
  - `bedtimeMins`: resulting bedtime in minutes (may differ from original `bedMins` by up to ±15 minutes when constraints/auto‑adjustment kick in).
  - Flags for explanation:
    - Whether auto‑adjustment was used.
    - Whether constraints required changes.
    - Whether nap count was auto‑upgraded/downgraded.

On top of this, the web app builds **multiple scenarios** (Section 7) with different wake‑window distribution strategies; the iOS implementation can mirror this by returning an array of scenarios.

---

## 4. Base Schedule Construction (No Constraints, No Actuals)

The core builder is `tryBuildSchedule(napCount, params, strategy)` where:

- `napCount` ∈ {2, 3}.
- `params` includes:
  - `wakeMins`, `bedMins`, `lastWake`, `minWake`, `maxWake`, `maxFirstWake`, `napLengths` (array of at least `napCount` integers).
- `strategy` ∈ {`AGGRESSIVE`, `GENTLE`, `BALANCED`} – see 4.2.

### 4.1 Steps

1. **Compute day length**
   - `totalDay = bedMins - wakeMins`.
   - If `totalDay <= 0`: fail (`"Bedtime must be after wake time (same or next day)."`) – this should already be prevented by the pre‑step that rolls bedtime to next day, but the guard remains.

2. **Compute nap and awake budgets**
   - `totalNapTime = sum(napLengths[0..napCount-1])`.
   - `totalAwake = totalDay - totalNapTime`.
   - `remainingBeforeLast = totalAwake - lastWake`.
   - If `remainingBeforeLast < 0`: fail (`"Not enough time for naps + last wake window before bed."`).

3. **Check biological wake‑window constraints**
   - Number of wake windows *before* the final wake is `nWindowsBeforeLast = napCount` (one before each nap).
   - Compute:
     - `minTotal = minWake * nWindowsBeforeLast`.
     - `maxTotal = maxWake * nWindowsBeforeLast`.
   - If `remainingBeforeLast < minTotal` or `remainingBeforeLast > maxTotal`: fail (`"Awake windows would have to go outside the min/max limits."`).

4. **Distribute wake windows before naps**
   - Call `distributeWakeWindows(nWindowsBeforeLast, remainingBeforeLast, minWake, maxWake, strategy)` to get an array of `wakeWindowsBefore[]`.
   - Append `lastWake` to obtain `wakeWindowsAll = [ ...wakeWindowsBefore, lastWake ]`.
   - Enforce `maxFirstWake`: if `maxFirstWake` is non‑zero and `wakeWindowsBefore[0] > maxFirstWake`: fail with a message including the actual and allowed value.

5. **Build schedule items**
   - Start with:
     - `items[0] = { label: "Wake-up", timeRange: fromMinutes(wakeMins), badge: "Start of day" }`.
   - Initialise `cursor = wakeMins`.
   - For each nap index `i` in `[0 .. napCount-1]`:
     - `ww = wakeWindowsBefore[i]`.
     - `cursor += ww`; `napStart = cursor`; `napEnd = napStart + napLengths[i]`.
     - Append a `Nap i+1` item with:
       - `timeRange: fromMinutes(napStart) – fromMinutes(napEnd)`.
       - `extra: "<length> min"`.
       - `wakeWindow: ww`.
     - Push `ww` into `wakeWindowDurations`.
     - Set `cursor = napEnd`.
   - Final wake to bedtime:
     - `cursor += lastWake`.
     - Append `Bedtime` item at `fromMinutes(cursor)` with `wakeWindow = lastWake`.
     - Add `lastWake` to `wakeWindowDurations`.

6. **Return schedule**
   - If all checks passed:
     - `ok: true`.
     - `items`, `wakeWindowDurations`.
     - `napCountUsed = napCount`.
     - `napLengthsUsed = napLengths.slice(0, napCount)`.
     - `bedtimeMins = cursor`.
     - `strategy` (the wake‑window distribution strategy used).

### 4.2 Wake‑Window Distribution Strategies (“Three Schedules”)

The three “schedule styles” correspond to different policies for distributing `remainingBeforeLast` across `nWindowsBeforeLast` wake windows:

- **AGGRESSIVE / “Build Later”**
  - Intended shape: shorter wake windows earlier in the day, longer later.
  - Implementation:
    - Start all windows at `minWake`.
    - Let `leftover = remainingBeforeLast - minWake * nWindows`.
    - Walk from the last window backwards, pushing time in ~25‑minute chunks (`targetIncrement = 25`) within `[minWake, maxWake]`.
    - Any remaining minutes are distributed from the back again, bounded by `maxWake`.
  - Effect: a more pronounced build‑up of awake time towards the evening.

- **GENTLE / “Gentle Build”**
  - Similar to AGGRESSIVE but with smaller increments:
    - Uses `targetIncrement = 12`.
    - Two passes from the back, each time bounded by `maxWake`.
  - Effect: gently increasing wake windows without dramatic jumps.

- **BALANCED / “Balanced”**
  - Aims for nearly equal wake windows:
    - Compute `perWindow = floor(leftover / nWindows)`, `remainder = leftover % nWindows`.
    - Add `perWindow` to each, capped at `maxWake`.
    - Distribute the remainder from the back in 1‑minute steps, still respecting `maxWake`.
  - Effect: wake windows are as equal as possible given the constraints.

Each strategy produces a separate scenario in the UI (tabs 1–3). For iOS, these can be exposed as separate “schedule style” options for the parent.

---

## 5. Auto‑Adjustment Without Constraints

When no “awake/asleep window” constraints are set, the engine still tries to salvage infeasible configurations via `tryAutoAdjustNapsToFit`.

### 5.1 Nap‑Length Combination Generator

`generateNapCombinations(originalNapLengths, napCount, minNap, maxAdjust, lockedIndices)` builds a list of candidate nap‑length arrays to test:

1. **Original lengths first**
   - The unmodified `originalNapLengths` is always the first candidate.
2. **Pairwise transfers (“shift time between naps”)**
   - For each pair `(i, j)` where neither nap is “locked” (see actual nap handling later), and for each transfer `t` in `{5, 10, ..., maxAdjust}`:
     - If `nap[i] - t >= minNap`: move `t` minutes from nap i to j.
     - If `nap[j] - t >= minNap`: move `t` minutes from nap j to i.
   - This keeps total daytime sleep constant and is preferred.
3. **Redistribution adjustments**
   - Reduce a single nap by `reduction ∈ {5, 10, ..., maxAdjust}` (without going below `minNap`), then redistribute that reduction across the other (unlocked) naps as evenly as possible.
4. **Single‑nap ± adjustments**
   - For each nap and each adjustment `adj ∈ {-maxAdjust, ..., -5, 5, ..., maxAdjust}`:
     - As long as `nap[i] + adj >= minNap`, emit a variant with only that nap changed.
   - These change total nap time and are tried last.

In all cases, naps that are “locked” (because an actual nap has already occurred there) are not modified.

### 5.2 Auto‑Adjustment Flow (No Constraints)

`tryAutoAdjustNapsToFit(napCount, params, strategy)`:

1. If *any* actual naps exist for this nap count, **do nothing** (actual‑based scheduling will handle things; see Section 6).
2. Generate candidate nap‑length combinations around `params.napLengths` using:
   - `minNap = 30` minutes.
   - `maxAdjust = 15` minutes.
3. For each candidate `testNapLengths`:
   - Run `tryBuildSchedule(napCount, { ...params, napLengths: testNapLengths }, strategy)`.
   - If `ok`, accept the first success and return:
     - Adjusted schedule.
     - Adjusted nap lengths.
     - `bedtimeAdjustment = 0`.
     - Optional message summarising changes, e.g. `"Auto-adjusted naps to fit schedule: Nap 1 +10 min, Nap 2 -10 min"`.
4. If all nap‑only attempts fail:
   - Try bedtime adjustments `bedAdj ∈ {-15, +15}`:
     - For each `bedAdj`:
       - Test original nap lengths with `bedMins + bedAdj`.
       - If that fails, test each `testNapLengths` at the adjusted bedtime.
       - Accept the first successful combination and build a message describing both bedtime and nap changes.
5. If nothing fits, return `null` (no auto adjustment possible).

This mechanism is used in two places:

- Directly by `recomputeSchedule` when the requested nap count fails.
- Indirectly by scenario generation to try and salvage each wake‑window strategy.

---

## 6. Constraint Handling (“Awake During” / “Asleep During”)

When constraints are present, the engine uses `adjustScheduleForConstraints` on top of a base schedule.

### 6.1 Constraint Semantics

All times below are expressed in minutes and *normalised* to be near the schedule’s wake time:

- “Must be awake during” window `[awakeStart, awakeEnd]`:
  - No nap may overlap this window.
  - Overlap means: `napStart < awakeEnd && napEnd > awakeStart`.
  - If a nap overlaps, the schedule violates the constraint.
- “Must be asleep during” window `[sleepStart, sleepEnd]`:
  - At least one nap must *fully cover* this window:
    - `napStart <= sleepStart && napEnd >= sleepEnd`.
  - If no nap fully covers the window, the schedule violates the constraint.

Normalisation rules:

- For each constraint time:
  - While `< wakeMins - 12h`, add 24h.
  - While `> wakeMins + 12h`, subtract 24h.
- If end ≤ start after normalisation, add 24h to end (wrap‑around).
  - This allows constraints that cross midnight logically.

### 6.2 Validation Helper

`validateConstraints(scheduleResult, wakeMins, bedMins)`:

1. Reads the current constraint inputs.
2. Normalises them relative to `wakeMins`.
3. Extracts all naps (`scheduleResult.items` with labels `"Nap 1"`, `"Nap 2"`, `"Nap 3"`).
4. Applies the rules above.
5. Returns a list of violation objects, each with:
   - `type`: `'awake-window'` or `'sleep-window'`.
   - Human‑readable `message`.
   - `suggestion`: short guidance for the user.

An empty list means the schedule is constraint‑compliant.

### 6.3 Unified Adjustment Flow

`adjustScheduleForConstraints(params, strategy)`:

1. If no constraints at all are set, return `null` (do nothing).
2. Attempt to build a base schedule via `tryBuildSchedule(napCount, params, strategy)`.
   - If that fails, return `null` (constraints cannot be applied to a non‑existent schedule).
3. If `validateConstraints` returns no violations, return `null` – constraints are already satisfied.
4. Otherwise, run a **unified search** over bedtime offsets and nap‑length combinations:
   - Use `MIN_NAP = 30`, `MAX_NAP_ADJUST = 30`.
   - Bedtime adjustments to try, in order:
     - `bedAdj ∈ {0, -15, +15}`.
   - For each `bedAdj`:
     - Build `adjustedBedMins = bedMins + bedAdj`.
     - Compute effective nap lengths:
       - Start from `napLengths`.
       - If actual naps exist:
         - Lock those naps (their indices go into `lockedIndices`).
         - Replace their lengths with the actual durations (if ≥ `MIN_NAP`).
     - Generate nap combinations with `generateNapCombinations(effectiveNapLengths, napCount, MIN_NAP, MAX_NAP_ADJUST, lockedIndices)`.
     - For each `testNapLengths`:
       - Build `testSchedule = tryBuildSchedule(napCount, { ...params, bedMins: adjustedBedMins, napLengths: testNapLengths }, strategy)`.
       - If `!testSchedule.ok`, skip.
       - Run `validateConstraints(testSchedule, wakeMins, adjustedBedMins)`.
       - If there are no violations:
         - Return this schedule along with:
           - `strategy: 'unified-adjustment'`.
           - `changes: { bedMins: adjustedBedMins, napLengths: testNapLengths }`.
           - A combined human‑readable message describing bedtime and nap changes.
5. If unified adjustment fails:
   - Try switching nap count (2 ↔ 3) using stored per‑mode nap settings or defaults and repeat the check (but without re‑running a second nested unified search; the code just tests the alternate nap count once and validates constraints).
   - If that works, return:
     - `strategy: 'nap-count'`.
     - `changes: { napCount: alternateNapCount, napLengths: alternateNapLengths }`.
6. If no combination satisfies constraints, return `null`.

In all cases, **wake window bounds** (`minWake`, `maxWake`, `lastWake`) and `maxFirstWake` remain untouched; invalid schedules are simply rejected.

---

## 7. Actual‑Nap‑Aware Scheduling

When the user records actual nap times, the engine should:

- Respect those naps as fixed.
- Adjust remaining naps to keep total daytime sleep close to the original plan.
- Recompute wake windows and the rest of the day around the actuals.

This is handled by `getActualNapTimes` and `tryBuildScheduleWithActuals`.

### 7.1 Actual Naps Collection

`getActualNapTimes(napCount)`:

- Reads up to three actual naps from inputs (start/end times).
- Converts them to minutes and returns an array of objects:
  - `{ napNum: 1|2|3, start, end }`.
- For 2‑nap schedules, Nap 3 actuals are ignored.

### 7.2 Building a Schedule With Actuals

If `actualNaps.length > 0`, `tryBuildSchedule` short‑circuits to `tryBuildScheduleWithActuals(napCount, params, actualNaps)`:

1. Initialise `adjustedNapLengths = napLengths`.
2. Compute net difference between actual and planned naps:
   - For each actual nap at index `idx`:
     - `planned = napLengths[idx]`.
     - `duration = end - start`.
     - Increment `netDelta` by `(duration - planned)`.
     - Set `adjustedNapLengths[idx] = duration`.
   - `netDelta > 0`: more sleep than planned so far (overshoot).
   - `netDelta < 0`: less sleep than planned so far (deficit).
3. Adjust remaining nap lengths:
   - Minimum nap length: `MIN_NAP = 30`.
   - If `netDelta > 0` (overshoot):
     - Walk remaining, *untracked* naps from the end backwards.
     - Reduce each by as much as possible (down to `MIN_NAP`) until the overshoot is “paid back” or no more can be reduced.
   - If `netDelta < 0` (deficit):
     - Find the highest actual nap index `maxActualIdx`.
     - Starting from `maxActualIdx + 1` to the last nap:
       - Increase untracked naps by as much as needed to absorb the deficit (subject to later feasibility checks).
4. Build schedule items:
   - Start at `wakeMins` with a `Wake-up` item.
   - For each nap index `i` from 0 to `napCount-1`:
     - If there is an `actualNap` for `napNum = i+1`:
       - Use exact `start`/`end` times.
       - Wake window before this nap is `napStart - cursor`.
       - Tag nap as `"Actual"`.
     - If there is no actual nap:
       - For non‑last naps:
         - Compute `remainingTime = bedMins - cursor`.
         - Sum remaining adjusted nap lengths (clamped to `MIN_NAP`) to get `adjustedRemainingNapTime`.
         - Reserve `lastWake` for final wake window.
         - Compute `availableForWakeWindows = remainingTime - adjustedRemainingNapTime - lastWake`.
         - Number of remaining wake windows = number of remaining naps.
         - Choose a wake window:
           - `wakeWindow = clamp(availableForWakeWindows / wakeWindowsNeeded, minWake, maxWake)`.
         - Set nap start = `cursor + wakeWindow`, nap end = `napStart + adjustedNapLen`.
       - For the last nap:
         - Work backwards from bedtime:
           - `napEnd = bedMins - lastWake`.
           - Maximum allowable nap length:
             - `maxAllowedNapLen = napEnd - (cursor + minWake)`.
           - If `maxAllowedNapLen < MIN_NAP`: fail (not enough time left).
           - `adjustedNapLen = clamp(originalLen, MIN_NAP, maxAllowedNapLen)`.
           - `napStart = napEnd - adjustedNapLen`.
           - Ensure wake before this nap ≥ `minWake`; otherwise fail.
5. After all naps:
   - Final wake window = `bedMins - cursor` (may differ from `lastWake` because actual naps shift the day).
   - Append `Bedtime` item.
6. Enforce `maxFirstWake` as for the base builder.
7. Return a schedule object including:
   - `hasActuals: true`.
   - `napLengthsUsed` derived from the actual/planned durations that ended up in the schedule.

Note: when actuals are present, *other* auto‑adjustment functions (like `tryAutoAdjustNapsToFit`) treat those naps as locked, and only adjust later naps as needed.

---

## 8. Putting It Together: High‑Level Flow

The top‑level orchestration in the web app is `recomputeSchedule()`. In platform‑neutral terms, the high‑level flow is:

1. **Read and validate inputs**
   - Parse wake and bed times; roll bed to next day if needed.
   - Parse `minWake`, `maxWake`, `lastWake`, `maxFirstWake`; enforce:
     - `minWake > 0`, `maxWake ≥ minWake`, `lastWake ∈ [minWake, maxWake]`, `maxFirstWake ≥ minWake`.
   - Read nap lengths for the currently selected mode; require `nap1 > 0` and `nap2 > 0`.

2. **Determine requested nap mode**
   - `preferThree` → `requestedNapCount = 3` else `2`.

3. **Apply Power Sleep (if enabled)**
   - If `powerSleepEnabled`:
     - Load `powerSleepTarget` (default 180 minutes).
     - Call `applyPowerSleep(napLengths, requestedNapCount, powerSleepTarget)`:
       - For active naps (2 or 3), compute current total.
       - If total ≥ target, leave as‑is.
       - Otherwise distribute the missing minutes proportionally across naps.
       - For 2‑nap mode, the third nap value is preserved as placeholder.

4. **Attempt schedule for requested nap count**
   - Call `tryBuildSchedule(requestedNapCount, params, defaultStrategy)`:
     - `defaultStrategy` in the main path is `AGGRESSIVE`.
   - If it succeeds:
     - `actualNapCountInUse = requestedNapCount`.
   - If it fails:
     - Store failure reason (`threeNapFailReason` or `twoNapFailReason`).
     - Call `tryAutoAdjustNapsToFit(requestedNapCount, params)`.
       - If success:
         - Use returned schedule; keep inputs unchanged; attach adjustment message.
         - `actualNapCountInUse = requestedNapCount`.
       - If still no schedule:
         - **Auto‑downgrade or upgrade nap count**:
           - If requested was 3:
             - Load stored 2‑nap settings (or defaults `[90, 30, 30]`).
             - Optionally re‑apply Power Sleep (for 2 naps).
             - Try `tryBuildSchedule(2, newParams)`.
             - If success:
               - Mark `autoDowngraded = true`.
               - Set `requestedNapCount = 2`.
               - Update nap length inputs to match the 2‑nap settings.
               - Mark visually that preference and actual schedule differ.
           - If requested was 2:
             - Symmetric process but upgrading to 3 naps, using 3‑nap settings or defaults `[30, 60, 30]`.
   - If after all of the above no schedule exists for either nap count:
     - Return failure with:
       - The primary reason.
       - Supplementary notes for both the 2‑nap and 3‑nap attempts.

5. **Apply constraints (if any)**
   - Call `adjustScheduleForConstraints` with:
     - The nap count actually used.
     - The nap lengths in use.
   - If it returns an adjusted schedule:
     - Replace `scheduleResult` with that.
     - Track the message describing what changed.
     - Note: user inputs are *not* mutated – changes are only reflected in the schedule display.

6. **Generate alternative scenarios**
   - For the effective nap count and nap lengths (before constraint adjustments), call:
     - `generateScenarios(napCountUsed, params, baseNapLengths)`:
       - For each strategy (`AGGRESSIVE`, `GENTLE`, `BALANCED`):
         - Try `tryBuildSchedule`.
         - If that fails, try `tryAutoAdjustNapsToFit`.
         - If still infeasible, skip this strategy.
         - If feasible, optionally pass through `adjustScheduleForConstraints` to enforce constraints.
         - Add scenario with:
           - `schedule`.
           - `strategyKey`, human‑readable label and short label (“Build Later”, “Gentle”, “Balanced”).
           - Any auto‑adjust or constraint messages.
   - If no scenario is generated, fall back to the original `scheduleResult` as a single “Default Schedule”.
   - In the UI, scenario 1 is the default and others are available via tabs; in a Swift engine you can simply return all scenarios and let the UI choose how to expose them.

7. **Expose explanation data**
   - For the active scenario, summarise:
     - Nap count, total daytime sleep.
     - Differences between used nap lengths and the user’s base settings, per nap.
     - Any bedtime adjustment relative to the user’s target.
     - Whether the schedule had to diverge from the user’s preferred nap count.
     - Whether constraints required additional changes.

---

## 9. Notes for a Swift Re‑Implementation

For iOS, the recommended shape for a Swift module is:

- A pure `ScheduleEngine` type with a single public entry point, e.g.:

```swift
struct ScheduleEngine {
    func computeSchedules(request: ScheduleRequest) -> ScheduleResult
}
```

Where:

- `ScheduleRequest` encodes:
  - Day bounds, nap mode preference, per‑mode nap lengths.
  - Wake window parameters.
  - Power Sleep settings.
  - Constraints.
  - Actual naps for this day.
- `ScheduleResult` encodes:
  - `success` flag and error description (if any).
  - Active nap count.
  - One or more scenarios, each with:
    - Item list (wake, naps, bedtime) and wake windows.
    - Used nap lengths and total nap time.
    - Metadata on which strategy was used and what was adjusted.

The details above define the behaviour to match; naming and exact data structures can be adapted to fit Swift best practices as long as:

- Biological limits (`minWake`, `maxWake`, `lastWake`, `maxFirstWake`) are never auto‑tuned.
- Constraint semantics (“awake during” / “asleep during”) are preserved.
- Auto‑adjustment, Power Sleep, and actual‑nap handling behave as described.
- The three wake‑window distribution strategies remain available as distinct “schedule styles”.

